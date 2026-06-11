# Lekcja 02 — Outbox, inbox i transactional messaging

## Cel lekcji

Po tej lekcji będziesz umiał rozpoznać problem **dual write** (i wskazać go w cudzym kodzie na pierwszy rzut oka), zaimplementować **outbox pattern** w .NET + EF Core krok po kroku, wyjaśnić rolę **inbox pattern** oraz powiedzieć, czym jest CDC i kiedy zastępuje poller.

## Dlaczego to ważne

Dual write to prawdopodobnie najczęstszy *cichy* bug w systemach zdarzeniowych — kod wygląda poprawnie, code review go przepuszcza, testy przechodzą, a w produkcji raz na tysiąc przypadków znika zdarzenie albo publikuje się zdarzenie o czymś, co nigdy nie zaszło. Na rozmowach rekrutacyjnych "jak zagwarantujesz, że zdarzenie zostanie opublikowane, skoro broker i baza to dwa różne systemy?" to pytanie-klasyk, a "outbox pattern" to oczekiwana odpowiedź. Jeśli w Perfect Gym widziałeś rozjazd między stanem bazy a tym, co poszło na kolejkę — to było dokładnie to.

## Teoria

### Problem: dual write

**Dual write** to sytuacja, w której jedna operacja biznesowa wymaga zapisu do **dwóch niezależnych systemów** — np. bazy danych i brokera wiadomości — a te dwa zapisy **nie są objęte wspólną transakcją**, bo nie mogą być: baza i broker to osobne procesy, osobne protokoły, brak wspólnego mechanizmu transakcyjnego.

Naiwny kod, który pisze 90% zespołów:

```csharp
public async Task PlaceOrderAsync(PlaceOrder cmd)
{
    db.Orders.Add(new Order(cmd));
    await db.SaveChangesAsync();                          // zapis nr 1: baza

    await bus.PublishAsync(new OrderPlaced(cmd.OrderId)); // zapis nr 2: broker
}
```

Rozpiszmy scenariusze awarii — to jest sedno tej lekcji:

| Scenariusz | Co się dzieje | Skutek |
|---|---|---|
| Crash między `SaveChangesAsync` a `PublishAsync` | zamówienie w bazie, zdarzenie nie wyszło | **zdarzenie zgubione** — magazyn nigdy się nie dowie, klient czeka na paczkę w nieskończoność |
| `PublishAsync` rzuca (broker niedostępny) | jw., tylko z wyjątkiem; jeśli wyjątek poleci do klienta — klient widzi błąd, choć zamówienie *istnieje* | niespójność + zdezorientowany klient |
| Odwrócenie kolejności: najpierw publish, potem save — i save się nie udaje | zdarzenie wyszło, zamówienia nie ma | **zdarzenie-widmo** — system reaguje na coś, co nie zaszło (np. e-mail "dziękujemy za zamówienie" bez zamówienia) |
| "To owińmy w transakcję bazy" — publish wewnątrz `tx` | publish nie cofnie się przy rollbacku (broker nie zna transakcji bazy); a jeśli commit padnie po publishu — znowu widmo | transakcja bazy **nie obejmuje** brokera, to tylko iluzja bezpieczeństwa |

Wniosek: **nie da się atomowo zapisać do dwóch systemów, które nie dzielą transakcji**. (Kiedyś próbowano tego transakcjami rozproszonymi 2PC/MSDTC — dlaczego to umarło, zobaczysz w lekcji 03.) Trzeba sprowadzić problem do **jednego** zapisu atomowego.

### Outbox pattern krok po kroku

Pomysł jest piękny w swojej prostocie. Analogia: urzędowa **skrzynka nadawcza**. Urzędnik, załatwiając sprawę, nie biegnie z listem na pocztę (mógłby zginąć po drodze, a sprawa już zatwierdzona). Zamiast tego wkłada list do skrzynki nadawczej **w tym samym akcie co podpisanie dokumentu** — a osobny goniec regularnie opróżnia skrzynkę i nosi listy na pocztę. Jak goniec zgubi list — w skrzynce jest kopia, zaniesie ponownie.

Technicznie: zamiast publikować do brokera, **zapisz zdarzenie do tabeli w tej samej bazie i tej samej transakcji** co dane biznesowe. To już jest jeden zapis atomowy — baza umie to zagwarantować. Osobny proces (relay/poller) czyta tabelę i publikuje do brokera.

**Krok 1 — tabela outbox:**

```sql
CREATE TABLE OutboxMessages (
    Id          UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,  -- = MessageId = idempotency key
    Type        NVARCHAR(300)  NOT NULL,                -- np. 'OrderPlaced'
    Payload     NVARCHAR(MAX)  NOT NULL,                -- JSON zdarzenia
    OccurredAt  DATETIME2      NOT NULL,
    ProcessedAt DATETIME2      NULL                     -- NULL = jeszcze nie wysłane
);
CREATE INDEX IX_Outbox_Unprocessed ON OutboxMessages(OccurredAt) WHERE ProcessedAt IS NULL;
```

**Krok 2 — zapis biznesowy + outbox w jednej transakcji (EF Core):**

```csharp
public async Task PlaceOrderAsync(PlaceOrder cmd, CancellationToken ct)
{
    var order = new Order(cmd);
    db.Orders.Add(order);

    db.OutboxMessages.Add(new OutboxMessage
    {
        Id         = Guid.NewGuid(),
        Type       = nameof(OrderPlaced),
        Payload    = JsonSerializer.Serialize(new OrderPlaced(order.Id, order.Total)),
        OccurredAt = DateTime.UtcNow
    });

    await db.SaveChangesAsync(ct); // JEDEN atomowy zapis — i tylko on
}
```

Koniec metody biznesowej. Żadnego `bus.PublishAsync` w handlerze — to najważniejsza zmiana mentalna.

**Krok 3 — relay (poller) jako `BackgroundService`:**

```csharp
public sealed class OutboxRelay(IServiceScopeFactory scopes, IMessageBus bus,
                                ILogger<OutboxRelay> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await using var scope = scopes.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            var batch = await db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.OccurredAt)
                .Take(50)
                .ToListAsync(ct);

            foreach (var msg in batch)
            {
                // MessageId = Id z outboxa → konsument może deduplikować
                await bus.PublishAsync(msg.Type, msg.Payload, messageId: msg.Id, ct);
                msg.ProcessedAt = DateTime.UtcNow;
                await db.SaveChangesAsync(ct); // oznaczaj pojedynczo, nie całą paczkę
            }

            if (batch.Count == 0)
                await Task.Delay(TimeSpan.FromSeconds(1), ct);
        }
    }
}
```

Przeanalizuj awarie relaya — i zauważ, że **żadna nie gubi danych**:

- Relay padnie po `PublishAsync`, a przed `SaveChangesAsync` → po restarcie opublikuje **ponownie**. Czyli outbox daje **at-least-once** — duplikaty są możliwe i to jest OK, bo (lekcja 01!) konsument jest idempotentny, a `MessageId` wędruje z wiadomością. To dlatego mówimy o *idempotent publish*: publikacja może się powtórzyć, skutek u odbiorcy — nie.
- Broker leży → relay próbuje dalej w kolejnych obrotach pętli; zdarzenia czekają bezpiecznie w tabeli.
- Wiele instancji usługi = wiele relayów → ryzyko podwójnej publikacji rośnie (dalej tylko duplikaty, nie zguby); w praktyce: blokada lidera, `SELECT ... FOR UPDATE SKIP LOCKED`, albo gotowiec — w .NET outbox mają wbudowany **MassTransit** i **NServiceBus**, w mini-projekcie napiszesz własny, żeby rozumieć bebechy.

### Inbox pattern

**Inbox** to lustrzane odbicie outboxa po stronie konsumenta: przychodzącą wiadomość najpierw **zapisz do tabeli inbox w transakcji z deduplikacją**, ack-nij brokerowi, a przetwarzaj z tabeli (od razu lub asynchronicznie). W praktyce tabela `ProcessedMessages` z lekcji 01 to minimalny inbox — pełny inbox dodatkowo przechowuje payload, co pozwala szybko zdjąć wiadomość z brokera (krótki lock) i przetwarzać we własnym tempie z lokalnym retry. Outbox = "na pewno wyślę", inbox = "na pewno przetworzę dokładnie raz". Razem tworzą **transactional messaging** end-to-end.

### CDC jako alternatywa dla pollera

Poller odpytuje tabelę co sekundę — to działa, ale dodaje opóźnienie i obciąża bazę pustymi odczytami. Alternatywa: **CDC** (*Change Data Capture* — przechwytywanie zmian danych). Narzędzie takie jak **Debezium** czyta **log transakcyjny bazy** (ten sam, którego baza używa do recovery i replikacji) i zamienia każdy commit na zdarzenie — w tym inserty do tabeli outbox (Debezium ma gotowy *outbox event router*). Zalety: niższa latencja, zero pollingu, zachowana kolejność commitów. Koszty: nowy element infrastruktury do utrzymania (Debezium = Kafka Connect), uprawnienia do logu transakcyjnego, więcej ruchomych części. Reguła kciuku: zaczynaj od pollera; CDC wtedy, gdy latencja/obciążenie pollingu realnie boli.

### Trade-offy i "kiedy tego NIE używać"

- **Opóźnienie** — zdarzenie wychodzi po commicie + interwale pollera (sekundy). Jeśli potrzebujesz odpowiedzi "tu i teraz" w ramach żądania — to nie jest komunikacja zdarzeniowa, tylko synchroniczne wywołanie; outbox niczego tu nie naprawi.
- **Porządek** — pojedynczy relay z `ORDER BY OccurredAt` zachowuje kolejność publikacji; wiele relayów lub retry pojedynczych wiadomości może ją zaburzyć. Kolejnością per klucz zajmiemy się w lekcji 05.
- **Sprzątanie tabeli** — outbox rośnie; potrzebujesz retencji (DELETE wpisów z `ProcessedAt` starszym niż X dni) jako osobnego joba. Nieusuwany outbox to klasyczna bomba z opóźnionym zapłonem na produkcji.
- **Kiedy NIE używać:** gdy nie ma dual write — np. zdarzenie jest tylko sygnałem "odśwież cache" i jego zguba jest tania (wystarczy at-most-once), albo gdy jedynym skutkiem operacji jest publikacja (nie ma zapisu do bazy — nie ma czego sklejać). Outbox to koszt: tabela, relay, retencja — płać go tam, gdzie zguba zdarzenia naprawdę boli.

## Praktyka

- [ ] Weź swój kod z lekcji 01 i dobuduj: tabelę `OutboxMessages`, zapis biznesowy + outbox w jednej transakcji, `OutboxRelay` jako `BackgroundService` (broker możesz na razie zasymulować interfejsem `IMessageBus` logującym na konsolę).
- [ ] Zasymuluj awarię: rzuć wyjątek w relayu po "publikacji", przed oznaczeniem `ProcessedAt` — uruchom ponownie i potwierdź, że wiadomość wyszła drugi raz z **tym samym** `MessageId`, a idempotent consumer z lekcji 01 ją zignorował.
- [ ] Napisz job sprzątający (drugi `BackgroundService` albo zwykły `DELETE` w pętli) i ustal retencję.
- [ ] Narysuj prosty diagram sekwencji (może być ASCII/Mermaid w README): handler → baza (tx: order + outbox) → relay → broker → konsument (inbox/dedup). Ten diagram wejdzie do repo mini-projektu.

## Artefakt

Folder `02-outbox-inbox-transactional-messaging/` w repo nauki: działający outbox + relay + test awarii relaya, diagram sekwencji, oraz szkic posta **"Dual write: bug, którego pewnie masz w produkcji i o tym nie wiesz"** (problem → tabela scenariuszy awarii → outbox w 3 krokach). To mocny kandydat na pierwszy post serii "production engineering".

## Definition of Done

- [ ] Umiesz z pamięci rozpisać minimum 3 scenariusze awarii dual write i powiedzieć, który gubi zdarzenie, a który tworzy zdarzenie-widmo.
- [ ] Twój relay po zasymulowanym crashu publikuje duplikat, a system jako całość daje effectively-once — i umiesz wskazać, które dwa elementy razem to gwarantują (outbox po stronie nadawcy + dedup po stronie odbiorcy).
- [ ] Umiesz wyjaśnić, czym CDC różni się od pollera i kiedy warto dopłacić za Debezium.
- [ ] Umiesz odpowiedzieć: kiedy outbox to przerost formy?

## Materiały

1. **microservices.io — [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)** (+ podlinkowane tam *Polling Publisher* i *Transaction Log Tailing* — to właśnie poller vs CDC).
2. **"Designing Data-Intensive Applications"**, M. Kleppmann — rozdz. 11 ("Stream Processing"), sekcje o keeping systems in sync i CDC.
