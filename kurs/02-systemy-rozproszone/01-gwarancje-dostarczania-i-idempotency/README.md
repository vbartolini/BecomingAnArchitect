# Lekcja 01 — Gwarancje dostarczania i idempotency

## Cel lekcji

Po tej lekcji będziesz rozumiał, dlaczego żaden broker ani sieć nie da Ci gwarancji "dokładnie raz", co naprawdę oznaczają at-most-once / at-least-once / exactly-once, oraz będziesz umiał zaprojektować i zaimplementować **idempotent consumer** w C# — czyli odbiorcę, któremu duplikaty nie robią krzywdy.

## Dlaczego to ważne

To jest temat numer jeden każdej rozmowy o systemach rozproszonych. Pytanie "co się stanie, gdy ta wiadomość przyjdzie dwa razy?" pada na praktycznie każdym system design interview — i odróżnia ludzi, którzy "używali RabbitMQ", od ludzi, którzy rozumieją, co się dzieje pod spodem. Jeśli kiedykolwiek w Perfect Gym widziałeś podwójne naliczenie, dwa e-maile do klienta albo dwa razy przetworzoną płatność — to była dokładnie ta lekcja, tylko odebrana w produkcji o nieludzkiej porze. Tu nadajemy temu nazwy.

## Teoria

### Problem: sieć jest zawodna, a potwierdzenia też są wiadomościami

Zacznijmy od najprostszego możliwego scenariusza. Usługa A wysyła wiadomość do usługi B (bezpośrednio albo przez broker — bez znaczenia). B ma ją przetworzyć i potwierdzić (wysłać **ack** — *acknowledgement*, potwierdzenie odbioru).

Co może pójść nie tak:

1. Wiadomość ginie po drodze — B nigdy jej nie dostaje.
2. B dostaje wiadomość, przetwarza ją… i **pada przed wysłaniem acka**.
3. B przetwarza, wysyła ack — ale **ack ginie po drodze**.
4. B przetwarza wolno, A uznaje, że timeout, i wysyła ponownie — a pierwsza wiadomość jednak dotarła.

Kluczowa obserwacja: z perspektywy nadawcy **przypadki 1, 2 i 3 są nierozróżnialne**. A wie tylko tyle: "nie dostałem potwierdzenia". Nie wie, czy wiadomość zginęła, czy zginął ack, czy B właśnie kończy przetwarzanie. To nie jest kwestia lepszego brokera czy lepszej sieci — to fundamentalne ograniczenie (w literaturze: *Two Generals' Problem* — dwóch generałów nie może przez zawodnego posłańca uzgodnić ze 100% pewnością wspólnego ataku, bo każde potwierdzenie samo wymaga potwierdzenia, w nieskończoność).

Nadawca ma więc do wyboru dokładnie dwie strategie i obie są złe:

- **Nie ponawiać** → ryzykujesz, że wiadomość przepadła (przypadek 1 i 2).
- **Ponawiać** → ryzykujesz duplikat (przypadek 3 i 4).

Z tych dwóch strategii biorą się nazwy gwarancji dostarczania (*delivery guarantees*).

### Trzy gwarancje dostarczania

**At-most-once** ("co najwyżej raz") — wyślij i nie ponawiaj. Wiadomość dotrze 0 lub 1 raz. Nigdy nie będzie duplikatu, ale może w ogóle nie dotrzeć. Analogia: zwykły list — wrzucasz do skrzynki i tyle. Sensowne tam, gdzie pojedyncza zguba nie boli: metryki, logi, odczyt telemetrii, ping "user is typing…".

**At-least-once** ("co najmniej raz") — ponawiaj, aż dostaniesz potwierdzenie. Wiadomość dotrze 1 lub więcej razy. Nic nie zginie, ale **duplikaty są wpisane w kontrakt**. Analogia: list polecony za potwierdzeniem odbioru — jak potwierdzenie nie wraca, poczta próbuje doręczyć ponownie, nawet jeśli adresat już raz odebrał, a kwit się zgubił. To domyślny tryb praktycznie wszystkich brokerów (RabbitMQ, Azure Service Bus w trybie peek-lock, Kafka po stronie konsumenta).

**Exactly-once** ("dokładnie raz") — święty Graal: każda wiadomość dostarczona dokładnie jeden raz. I tu najważniejsze zdanie tej lekcji:

> **"Exactly-once delivery" w sensie transportowym nie istnieje.** Nie da się zagwarantować, że wiadomość *fizycznie dotrze* do odbiorcy dokładnie raz, bo ack też może zginąć (patrz wyżej).

To, co realnie da się osiągnąć — i co marketing niektórych brokerów nazywa "exactly-once" — to **exactly-once processing**, czyli **effectively-once**: wiadomość może *dotrzeć* wiele razy, ale jej **skutek** wystąpi dokładnie raz. Wzór, który warto znać na pamięć:

```
effectively-once = at-least-once delivery + idempotent processing
```

Czyli: godzimy się na duplikaty w transporcie i sprawiamy, że duplikaty są nieszkodliwe po stronie odbiorcy. Gdy ktoś na rozmowie mówi "Kafka ma exactly-once" — chodzi o exactly-once *semantics* wewnątrz ekosystemu Kafki (transakcje producenta + idempotent producer + offsety w tej samej transakcji), a nie o magiczny transport. Na granicy z Twoją bazą danych i Twoim kodem dalej obowiązuje wzór powyżej.

### Idempotency od zera

**Idempotencja** (*idempotency*) — operacja jest idempotentna, jeśli wykonanie jej wiele razy daje ten sam skutek co wykonanie jej raz.

Analogia z życia: przycisk windy. Naciśniesz raz czy pięć razy — winda przyjedzie raz. Włącznik światła w trybie "ustaw na WŁĄCZONE" — drugie naciśnięcie nic nie zmienia. Kontrprzykład: dzwonek do drzwi — każde naciśnięcie dzwoni ponownie.

W kodzie:

```csharp
// IDEMPOTENTNE — ustawiamy stan docelowy; powtórka nic nie psuje
order.Status = OrderStatus.Paid;

// NIE-idempotentne — każda powtórka zmienia stan
account.Balance += 100m;
```

Część operacji jest idempotentna z natury (`SET status = 'Paid'`, `DELETE` po id, upsert po kluczu). Ale większość ciekawych operacji biznesowych — "pobierz opłatę", "wyślij e-mail", "zarezerwuj slot" — nie jest. Dla nich idempotencję trzeba **zbudować**, i robi się to dwoma klockami:

**1. Idempotency key** — unikalny identyfikator *operacji* (nie wiadomości transportowej!), nadawany przez nadawcę i podróżujący z komunikatem. Zwykle `Guid` typu `MessageId` albo klucz biznesowy ("płatność za zamówienie #123 — próba pierwsza"). Ważne: przy retry nadawca wysyła **ten sam** klucz. Tak działa np. API Stripe: nagłówek `Idempotency-Key` sprawia, że dwa identyczne POST-y obciążają kartę raz.

**2. Deduplikacja po stronie konsumenta** — odbiorca zapisuje klucze już przetworzonych operacji i przed przetworzeniem sprawdza: "czy ja tego już nie robiłem?". Krytyczny szczegół: zapis klucza i skutek biznesowy muszą być **w jednej transakcji bazodanowej** — inaczej odtwarzasz problem ack-a piętro wyżej (przetworzyłeś, padłeś przed zapisem klucza → duplikat przejdzie).

A deduplikacja w brokerze (np. Azure Service Bus ma wbudowaną deduplikację po `MessageId`)? Pomaga, ale nie wystarcza: działa w oknie czasowym, dotyczy duplikatów *publikacji*, a nie ponownych *doręczeń* po utracie acka, i nie obejmuje Twojej transakcji bazodanowej. Traktuj ją jako optymalizację, nie jako gwarancję.

### Implementacja idempotent consumera w C#

Tabela przetworzonych wiadomości:

```sql
CREATE TABLE ProcessedMessages (
    MessageId   UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,  -- idempotency key
    Consumer    NVARCHAR(200)    NOT NULL,              -- nazwa handlera (jedna wiadomość, wielu konsumentów)
    ProcessedAt DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
-- jeśli kluczem jest para (MessageId, Consumer), zrób z niej PRIMARY KEY złożony
```

Konsument (EF Core, SQL Server — ale wzorzec jest uniwersalny):

```csharp
public sealed class PaymentRequestedHandler(AppDbContext db, IPaymentGateway gateway)
{
    public async Task HandleAsync(PaymentRequested message, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);

        // 1. Czy już przetworzone? (szybka ścieżka)
        var alreadyProcessed = await db.ProcessedMessages
            .AnyAsync(p => p.MessageId == message.MessageId
                        && p.Consumer == nameof(PaymentRequestedHandler), ct);
        if (alreadyProcessed)
            return; // ack i koniec — duplikat jest nieszkodliwy

        // 2. Skutek biznesowy
        var order = await db.Orders.SingleAsync(o => o.Id == message.OrderId, ct);
        order.Status = OrderStatus.Paid;

        // 3. Zapis idempotency key W TEJ SAMEJ transakcji
        db.ProcessedMessages.Add(new ProcessedMessage(
            message.MessageId, nameof(PaymentRequestedHandler)));

        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
        // ack do brokera dopiero PO commicie
    }
}
```

Dlaczego to działa nawet przy wyścigu (dwa wątki przetwarzają ten sam duplikat równolegle i oba przechodzą sprawdzenie w kroku 1)? Bo `MessageId` jest **kluczem głównym** — drugi `SaveChangesAsync` poleci wyjątkiem naruszenia unikalności, transakcja się wycofa, skutek wystąpi raz. Sprawdzenie w kroku 1 to optymalizacja; **gwarancją jest unique constraint**. To częste pytanie podchwytliwe na rozmowach: "a co, jak dwa duplikaty wejdą równolegle?".

Uwaga na pułapkę: jeśli skutkiem jest wywołanie **zewnętrznego systemu** (bramka płatności, SMTP), nie obejmiesz go transakcją bazy. Wtedy: przekaż idempotency key dalej (bramki płatności go przyjmują) albo zastosuj outbox — o tym w lekcji 02.

### Trade-offy i "kiedy tego NIE używać"

- **Tabela dedup rośnie** → potrzebujesz retencji (kasuj wpisy starsze niż maksymalny realny czas redelivery, np. 7–30 dni) — to dodatkowy proces do utrzymania.
- **Koszt na gorącej ścieżce** — jeden odczyt + jeden insert na każdą wiadomość. Zwykle pomijalne, ale przy setkach tysięcy msg/s liczy się każdy roundtrip.
- **Nie potrzebujesz dedupa**, gdy operacja jest idempotentna z natury (czysty upsert stanu docelowego) — wtedy tabela to zbędna złożoność. Zawsze najpierw pytaj: "czy mogę przeprojektować operację tak, by była naturalnie idempotentna?". To tańsze niż infrastruktura dedup.
- **At-most-once bywa OK** — nie dorabiaj at-least-once + dedup do telemetrii, której strata nie boli.

## Praktyka

- [ ] Rozpisz (na kartce / w notatce) 3 incydenty z Twojej kariery, które po tej lekcji umiesz nazwać: gdzie był brak idempotencji, gdzie utrata acka, gdzie duplikat. Nazwij gwarancję, która tam obowiązywała.
- [ ] Zaimplementuj idempotent consumer: konsolowa aplikacja .NET + SQLite/LocalDB, handler jak wyżej, tabela `ProcessedMessages` z kluczem głównym.
- [ ] Napisz test, który wywołuje handler **dwa razy tą samą wiadomością** i asercją sprawdza pojedynczy skutek; drugi test: dwa wywołania **równolegle** (`Task.WhenAll`) — sprawdź, że unique constraint ratuje sytuację.
- [ ] Zrób przegląd jednego ze swoich realnych systemów (z pamięci): które operacje są idempotentne z natury, a które wymagałyby klucza? Zanotuj.

## Artefakt

Folder `01-gwarancje-dostarczania-i-idempotency/` w repo nauki zawiera:
1. działający projekt z idempotent consumerem i dwoma testami (duplikat sekwencyjny + równoległy),
2. notatkę `notes.md` z trzema nazwanymi incydentami z produkcji — to surowiec na post "Exactly-once delivery nie istnieje" (publikacja możliwa teraz albo po lekcji 02, gdy dojdzie wątek outboxa).

## Definition of Done

- [ ] Umiesz w 2 minuty, bez notatek, wyjaśnić koledze, dlaczego exactly-once delivery jest niemożliwe (argument z ginącym ackiem) i co oznacza effectively-once.
- [ ] Test "duplikat równoległy" przechodzi, i umiesz wskazać, **który element** kodu daje gwarancję (unique constraint, nie `if`).
- [ ] Umiesz odpowiedzieć: kiedy at-most-once jest właściwym wyborem i kiedy tabela dedup to przerost formy.

## Materiały

1. **"Designing Data-Intensive Applications"**, M. Kleppmann — rozdz. 8 ("The Trouble with Distributed Systems") + sekcja o exactly-once w rozdz. 11.
2. **microservices.io — [Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html)** — zwięzły opis wzorca, dobra ściąga przedrozmowowa.
