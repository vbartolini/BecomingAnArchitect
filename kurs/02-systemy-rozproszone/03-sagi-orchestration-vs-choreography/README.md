# Lekcja 03 — Sagi: orchestration vs choreography

## Cel lekcji

Po tej lekcji będziesz umiał wyjaśnić, dlaczego klasyczne transakcje rozproszone (2PC) nie nadają się do mikroserwisów, zaprojektować **sagę** — sekwencję lokalnych transakcji z akcjami kompensującymi — w obu wariantach (orchestration i choreography), oraz świadomie wybrać wariant do problemu. Zrozumiesz też **semantykę kompensacji**: co da się "cofnąć", a czego nie.

## Dlaczego to ważne

"Jak zrobisz transakcję obejmującą trzy mikroserwisy?" to pytanie obowiązkowe na rozmowach o role architektoniczne — i pułapka: odpowiedź "transakcją rozproszoną" dyskwalifikuje, odpowiedź "sagą, a konkretnie…" otwiera najlepszą część rozmowy. W praktyce każdy proces typu rezerwacja + płatność + powiadomienie (Aerotunel — dokładnie ten kształt) to saga, nawet jeśli nikt jej tak w projekcie nie nazwał. Po tej lekcji będziesz umiał nazwać i uporządkować to, co prawdopodobnie już kiedyś zbudowałeś ręcznie.

## Teoria

### Problem: transakcja przez granice usług

W monolicie z jedną bazą sprawa jest trywialna: rezerwacja + płatność + zapis powiadomienia w jednej transakcji SQL — albo wszystko, albo nic (ACID). W systemie rozproszonym booking, payment i notification to osobne usługi z osobnymi bazami. Potrzebujesz "albo wszystko, albo nic" **bez wspólnej bazy**.

### Naiwne rozwiązanie: 2PC — i dlaczego nie działa w mikroserwisach

**2PC** (*two-phase commit* — zatwierdzanie dwufazowe) to historyczna odpowiedź na ten problem. Koordynator pyta wszystkich uczestników: "czy możecie zatwierdzić?" (faza 1 — *prepare*; każdy blokuje zasoby i obiecuje, że commit się uda), a gdy wszyscy odpowiedzą "tak" — rozsyła "zatwierdzajcie" (faza 2 — *commit*). W świecie .NET znasz to jako MSDTC i `TransactionScope` obejmujący dwie bazy.

Dlaczego to umarło w architekturach rozproszonych:

- **Blokowanie**: między prepare a commit każdy uczestnik trzyma blokady. Jeśli koordynator padnie w tym oknie, uczestnicy są **w zawieszeniu** (*in-doubt*) — nie mogą ani zatwierdzić, ani wycofać, trzymając blokady w nieskończoność. 2PC jest protokołem *blokującym* z pojedynczym punktem awarii.
- **Dostępność = iloczyn**: transakcja przechodzi tylko, gdy *wszyscy* uczestnicy i koordynator żyją jednocześnie. Im więcej usług, tym gorzej — dokładnie odwrotnie niż cel mikroserwisów (niezależna zawodność).
- **Wsparcie**: brokery wiadomości, API HTTP, bazy NoSQL i usługi chmurowe po prostu **nie mówią** w 2PC. Nie obejmiesz `TransactionScope` wywołania REST do bramki płatności.

Wniosek: skoro nie możemy mieć jednej atomowej transakcji, podzielmy proces na kroki, z których **każdy jest lokalną transakcją**, i zaplanujmy, co robić, gdy któryś krok padnie.

### Saga od zera

**Saga** to wzorzec realizacji procesu biznesowego jako **sekwencji lokalnych transakcji**, gdzie każda transakcja zatwierdza się niezależnie, a na wypadek niepowodzenia późniejszego kroku każdy wcześniejszy krok ma zdefiniowaną **akcję kompensującą** (*compensating action*) — osobną transakcję, która biznesowo odwraca jego skutek.

Analogia: planujesz wakacje — rezerwujesz lot, potem hotel, potem wypożyczalnię aut. To trzy niezależne "transakcje" u trzech firm. Jeśli wypożyczalnia odmówi, nie istnieje magiczny "rollback wakacji" — dzwonisz i **anulujesz** hotel i lot (kompensacja), być może płacąc opłatę za anulowanie (kompensacja ma koszty i nie przywraca świata idealnie).

Kluczowa różnica względem rollbacku ACID: kompensacja to **nowa transakcja biznesowa**, nie cofnięcie w czasie. Między krokiem a jego kompensacją inni **widzą stan pośredni** (hotel przez chwilę był zarezerwowany). Saga rezygnuje z izolacji (litera "I" z ACID) w zamian za brak blokad i niezależność usług. To świadomy trade-off, nie niedoróbka.

### Przykład: saga rezerwacji z płatnością

Proces: **booking → payment → notification** (Twój chleb powszedni z Aerotunel).

```
Krok 1: BookingService   — utwórz rezerwację (status: Pending)     | kompensacja: anuluj rezerwację
Krok 2: PaymentService   — pobierz płatność                        | kompensacja: zwrot środków (refund)
Krok 3: BookingService   — potwierdź rezerwację (Pending→Confirmed)| kompensacja: (zwykle ostatni krok jej nie potrzebuje)
Krok 4: NotificationService — wyślij e-mail z potwierdzeniem       | kompensacja: NIE ISTNIEJE (o tym niżej)
```

Scenariusz szczęśliwy: 1 → 2 → 3 → 4, każdy krok to lokalna transakcja + zdarzenie (przez outbox z lekcji 02 — sagi i outbox to nierozłączna para: każdy krok sagi publikuje swoje zdarzenie transakcyjnie).

Scenariusz awarii: płatność odrzucona w kroku 2 → saga wykonuje kompensacje **w odwrotnej kolejności** dla kroków już zatwierdzonych → anuluj rezerwację z kroku 1 → saga kończy się stanem `BookingCancelled`. Spójnie, bez blokad, bez koordynatora 2PC.

Dwa wymagania techniczne: kroki i kompensacje muszą być **idempotentne** (lekcja 01 — zdarzenia mogą dojść podwójnie), a kompensacja musi być **zawsze wykonalna** (refund nie może zostać "odrzucony" biznesowo — może się co najwyżej opóźnić i być ponawiany).

### Choreography vs orchestration

Są dwa sposoby koordynowania kroków sagi.

**Choreography (choreografia)** — nie ma centralnego koordynatora; usługi reagują na zdarzenia kolejnych usług, jak tancerze, którzy znają układ i patrzą na siebie nawzajem:

```
BookingService:      PlaceBooking → publikuje BookingCreated
PaymentService:      słucha BookingCreated → pobiera płatność → publikuje PaymentCompleted / PaymentFailed
BookingService:      słucha PaymentCompleted → potwierdza | słucha PaymentFailed → anuluje (kompensacja)
NotificationService: słucha BookingConfirmed → wysyła e-mail
```

**Orchestration (orkiestracja)** — istnieje **orkiestrator** (osobny komponent, często w usłudze rozpoczynającej proces), który jak dyrygent mówi każdemu, co i kiedy grać: trzyma **stan sagi** (maszynę stanów utrwalaną w bazie) i wysyła **komendy** do usług, a one odpowiadają zdarzeniami:

```csharp
// Szkic orkiestratora jako maszyny stanów (koncepcyjnie; w praktyce np. MassTransit StateMachine)
public sealed class BookingSaga
{
    public Guid BookingId { get; set; }
    public BookingSagaState State { get; set; } // AwaitingPayment, Confirmed, Compensating, Cancelled

    public IReadOnlyList<ICommand> Handle(IEvent evt) => (State, evt) switch
    {
        (AwaitingPayment, PaymentCompleted) => Transition(Confirmed,
            new ConfirmBooking(BookingId), new SendConfirmationEmail(BookingId)),

        (AwaitingPayment, PaymentFailed)    => Transition(Cancelled,
            new CancelBooking(BookingId)),               // kompensacja kroku 1

        (AwaitingPayment, SagaTimedOut)     => Transition(Compensating,
            new CancelBooking(BookingId)),               // brak odpowiedzi też jest decyzją
        _ => [];
    };
}
```

Porównanie:

| Kryterium | Choreography | Orchestration |
|---|---|---|
| Sprzężenie | luźne — usługi nie znają się nawzajem | usługi znają tylko orkiestratora; orkiestrator zna wszystkich |
| Widoczność procesu | **rozproszona** — "gdzie jest ta rezerwacja?" wymaga śledztwa po logach | stan procesu w jednym miejscu, łatwy do podejrzenia |
| Złożoność przy 2–3 krokach | minimalna — duża zaleta | orkiestrator to dodatkowy komponent — narzut |
| Złożoność przy 5+ krokach / rozgałęzieniach | eksploduje: cykliczne zależności zdarzeń, nikt nie ogarnia całości | rośnie liniowo — logika w jednej maszynie stanów |
| Timeouty, deadline'y procesu | trudne — kto pilnuje, że "nic nie przyszło"? | naturalne — orkiestrator ma timery |
| Ryzyko | "proces-widmo", którego nikt nie widzi w całości | orkiestrator jako mini-monolit / wąskie gardło, jeśli wciągnie logikę biznesową usług |

Reguła kciuku: **2–3 kroki liniowo → choreography; dłuższe, rozgałęzione, z timeoutami i wymogiem audytu → orchestration.** Na rozmowie nie deklaruj wyznania — pokaż, że wybór zależy od kształtu procesu.

### Semantyka kompensacji: czego nie da się cofnąć

Najdojrzalsza część tematu — i najlepszy materiał na rozmowę. Skutki kroków dzielą się na trzy klasy:

1. **Kompensowalne** (*compensatable*) — da się biznesowo odwrócić: rezerwacja → anulowanie, obciążenie → refund (uwaga: refund to nie "cofnięcie" — klient widzi na wyciągu dwie operacje, a prowizje mogły przepaść).
2. **Pivot** — punkt bez powrotu: krok, po którym saga **musi** iść do przodu (np. zlecenie przelewu wyszło do systemu zewnętrznego). Projektując sagę, ustaw kroki tak, by ryzykowne/odmawialne były **przed** pivotem, a pewne — po nim.
3. **Niekompensowalne** (*retriable/non-compensatable*) — wysłany e-mail, SMS, wydany towar fizyczny. **Nie cofniesz e-maila.** Strategie: (a) przesuń taki krok na sam koniec sagi, żeby nigdy nie wymagał kompensacji; (b) jeśli musi być wcześniej — kompensuj *semantycznie*: wyślij drugą wiadomość "przepraszamy, rezerwacja anulowana" (świat nie wraca do stanu sprzed — godzimy się z tym jawnie); (c) opóźnij skutek (e-mail z buforem 2 minut, jak "cofnij wysyłkę" w Gmailu — to dosłownie ten wzorzec).

### Trade-offy i "kiedy tego NIE używać"

- Saga = rezygnacja z izolacji: stany pośrednie są widoczne (klient może dostać "przetwarzamy płatność" — zaprojektuj te stany jawnie w UI/statusach, zamiast udawać, że ich nie ma).
- Saga to maszyna stanów do utrzymania: więcej kodu, więcej testów (każda ścieżka kompensacji!), trudniejszy debugging — dlatego lekcja 07 (tracing) jest jej naturalnym dopełnieniem.
- **Kiedy NIE używać:** gdy proces mieści się w jednej usłudze i jednej bazie — zwykła transakcja ACID bije sagę pod każdym względem. Nie rozbijaj transakcji na usługi tylko po to, żeby mieć sagę. Najlepsza saga to ta, której uniknąłeś dobrym wyznaczeniem granic usług (bounded contexts z modułu 1).

## Praktyka

- [ ] Rozpisz sagę "rezerwacja slotu + płatność + powiadomienie" (realia Aerotunel) w obu wariantach: choreography (diagram zdarzeń między usługami) i orchestration (maszyna stanów orkiestratora). Mermaid/ASCII wystarczy.
- [ ] Dla każdego kroku wypisz akcję kompensującą i sklasyfikuj ją: kompensowalna / pivot / niekompensowalna. Zaznacz pivot na diagramie.
- [ ] Rozpisz minimum 3 scenariusze awarii (płatność odrzucona; payment service nie odpowiada — timeout; awaria orkiestratora w połowie sagi) i prześledź na diagramie, co się dzieje w każdym wariancie koordynacji.
- [ ] Zaimplementuj szkielet orkiestratora w C# (maszyna stanów jak wyżej, stan w pamięci lub w tabeli, zdarzenia symulowane in-process) — pełną sagę na brokerze zrobisz w mini-projekcie (lekcja 08).

## Artefakt

Folder `03-sagi-orchestration-vs-choreography/` w repo nauki: dwa diagramy (choreography + orchestration) z zaznaczonym pivotem i kompensacjami, tabela porównawcza wariantów z Twoją rekomendacją dla procesu rezerwacji **z uzasadnieniem** (to ćwiczenie formatu ADR z modułu 1), szkic kodu orkiestratora. Diagramy wejdą do repo mini-projektu i do Capstone.

## Definition of Done

- [ ] Umiesz w 3 minuty wyjaśnić, dlaczego 2PC nie działa w mikroserwisach (blokowanie + in-doubt + brak wsparcia), nie myląc tego z "bo jest stare".
- [ ] Umiesz narysować z pamięci sagę booking→payment→notification w obu wariantach i wskazać, przy którym kształcie procesu wybrałbyś który.
- [ ] Na pytanie "a jak cofniesz wysłany e-mail?" odpowiadasz trzema strategiami, nie ciszą.
- [ ] Wiesz, dlaczego saga wymaga idempotencji kroków i dlaczego naturalnie łączy się z outboxem.

## Materiały

1. **microservices.io — [Saga](https://microservices.io/patterns/data/saga.html)** + rozdział 4 książki "Microservices Patterns" (Richardson), jeśli chcesz głębiej — przykład Order/Customer jest rozpisany krok po kroku.
2. **Oryginalny paper "Sagas"** (Garcia-Molina, Salem, 1987) — 12 stron, zaskakująco czytelny; warto znać źródło, na rozmowach robi wrażenie, że wzorzec ma 40 lat i pochodzi z baz danych, nie z mikroserwisów.
