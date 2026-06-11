# Lekcja 02 — Modular monolith i vertical slice architecture w praktyce

## Cel lekcji

Po tej lekcji będziesz umiał zorganizować kod aplikacji .NET tak, jak robi się to we współczesnych, dobrze prowadzonych projektach: monolit z twardo egzekwowanymi granicami modułów, a wewnątrz modułów — vertical slices zamiast poziomych warstw. Zrefaktoryzujesz do tej struktury fragment repo messaging-patterns z modułu 2.

## Dlaczego to ważne

W module 1 (lekcja 06, style architektoniczne) ustaliłeś, że modular monolith to dziś domyślna rekomendacja dla większości systemów — mikroserwisy dopiero, gdy boli organizacyjnie. Ale na poziomie stylu to była decyzja na diagramie. Ta lekcja schodzi piętro niżej: **jak ten styl wygląda w solucji .NET** — bo "monolit modularny" bez egzekwowanych granic to po prostu monolit, w którym wszyscy obiecują grzeczność. Na rozmowach rekrutacyjnych pytanie "jak pilnujesz granic modułów w monolicie?" odróżnia ludzi, którzy o modular monolith czytali, od ludzi, którzy go utrzymywali. Vertical slices z kolei to prawdopodobnie największa zmiana w *organizacji kodu* aplikacji .NET ostatniej dekady — i częsty temat sporów w zespołach, w których architekt musi mieć zdanie poparte trade-offami, nie modą.

## Teoria

### Przypomnienie: czym jest modular monolith

**Modular monolith** to jeden proces (jeden deployment), w środku podzielony na moduły o jawnych granicach — każdy moduł ma własny model domeny, własne dane (osobny schemat lub przynajmniej osobne tabele "na własność") i komunikuje się z innymi wyłącznie przez zadeklarowane kontrakty. Dostajesz większość korzyści mikroserwisów z zakresu porządku (granice, autonomia zespołów, możliwość późniejszego wycięcia modułu do osobnej usługi) bez ich kosztów operacyjnych (sieć, częściowe awarie, rozproszone transakcje — całe cierpienie z modułu 2).

Naiwna wersja wygląda tak: foldery `Bookings/`, `Payments/`, `Notifications/` w jednym projekcie. Dlaczego się psuje: w jednym projekcie **wszystko jest `public` dla wszystkich**. Pierwszy deadline i ktoś z `Payments` sięga bezpośrednio po encję `Booking` i jej DbSet, "bo tak było szybciej". Po roku granice istnieją tylko w nazwach folderów, a "wytniemy Payments do osobnej usługi" okazuje się niewykonalne, bo zależności biegną we wszystkie strony. To dokładnie ten mechanizm, przez który większość "modularnych" monolitów modularna jest tylko w prezentacji dla zarządu.

### Egzekwowanie granic w jednym procesie

Granica, której nie pilnuje kompilator albo test, nie istnieje. Narzędzia w .NET, od najtańszego:

**1. Osobne projekty + `internal`.** Każdy moduł to osobny projekt (assembly). Wszystko w środku jest `internal`; publiczny jest tylko mały projekt kontraktów:

```
src/
  Modules/
    Bookings/
      Bookings.Contracts/    ← public: eventy, komendy, interfejs modułu
      Bookings/              ← internal: cała implementacja
    Payments/
      Payments.Contracts/
      Payments/
  Host/                      ← składa moduły w jeden proces (AppHost/WebApplication)
```

Projekt `Payments` referencuje **tylko** `Bookings.Contracts` — próba użycia wewnętrznej encji `Booking` to błąd kompilacji. Kompilator staje się strażnikiem architektury, za darmo i bez dyskusji na code review.

**2. Komunikacja przez kontrakty, nie przez współdzielony model.** Moduł wystawia dwa rodzaje drzwi:

- **synchroniczne zapytanie/komenda** — wąski publiczny interfejs w `Contracts`, implementowany wewnątrz modułu i rejestrowany w DI:

```csharp
// Bookings.Contracts — to widzą inne moduły
public interface IBookingsModule
{
    Task<BookingSummary?> GetBookingAsync(Guid id, CancellationToken ct);
}
public record BookingSummary(Guid Id, Guid CustomerId, decimal Price);
```

Zwróć uwagę: `BookingSummary` to **DTO kontraktu**, nie encja. Encja z jej nawigacjami i DbContextem nigdy nie opuszcza modułu.

- **asynchroniczne eventy in-process** — moduł publikuje fakt ("BookingConfirmed"), nie wie i nie chce wiedzieć, kto słucha:

```csharp
// Bookings.Contracts
public record BookingConfirmed(Guid BookingId, Guid CustomerId, decimal Amount);

// w Payments — handler subskrybujący event z innego modułu
internal sealed class CreateInvoiceOnBookingConfirmed : IEventHandler<BookingConfirmed>
{
    public Task HandleAsync(BookingConfirmed e, CancellationToken ct) =>
        /* utwórz fakturę w module Payments */ ...;
}
```

To te same wzorce co w module 2 (publish/subscribe, eventual consistency między modułami), tylko transport jest tani: in-process zamiast brokera. I to jest właśnie ścieżka ewolucji: gdy moduł kiedyś wyjedzie do osobnej usługi, **kontrakt eventu zostaje, zmienia się transport** na Service Bus — reszta systemu nie zauważa.

**3. Test architektury** — siatka bezpieczeństwa na to, czego kompilator nie złapie (np. ktoś dodał referencję projektową "na chwilę"). Pełna implementacja w lekcji 03.

### Vertical slice architecture — od zera

Problem z warstwami poziomymi. Klasyczna struktura, którą znasz z każdego projektu ostatnich 20 lat: `Controllers/`, `Services/`, `Repositories/`, `Models/` (albo szlachetniej: Api / Application / Domain / Infrastructure). Kod jest pokrojony **technologicznie**. Tymczasem zmiany biznesowe są krojone **funkcjonalnie**: "dodaj możliwość anulowania rezerwacji" oznacza dotknięcie kontrolera, serwisu, repozytorium, DTO i mapowania — **pięciu plików w pięciu folderach**, żeby zrobić jedną rzecz. Co gorsza, warstwy zachęcają do współdzielenia: jeden `BookingService` na 14 metod, jeden `IBookingRepository` używany przez 9 funkcji — zmiana pod jedną funkcję ryzykuje regresję ośmiu pozostałych. Sprzężenie (coupling) między *funkcjami* rośnie, bo wszystkie przechodzą przez te same współdzielone klasy.

**Vertical slice** odwraca cięcie: jednostką organizacji kodu jest **feature** (przypadek użycia), a nie warstwa. Wszystko, czego feature potrzebuje — endpoint, walidacja, logika, dostęp do danych — mieszka razem, w jednym folderze, często w jednym pliku:

```
Modules/Bookings/
  Features/
    CancelBooking/
      CancelBooking.cs        ← endpoint + handler + walidacja
    GetBookingDetails/
      GetBookingDetails.cs
    ConfirmBooking/
      ConfirmBooking.cs
      BookingConfirmedEvent.cs
  Domain/
    Booking.cs                ← encja/agregat wspólny dla slice'ów modułu
  Infrastructure/
    BookingsDbContext.cs
```

Przykładowy slice w całości (styl: statyczna klasa feature z zagnieżdżonymi typami):

```csharp
public static class CancelBooking
{
    public record Command(Guid BookingId, string Reason);

    public static void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapPost("/bookings/{id:guid}/cancel",
            async (Guid id, CancelRequest req, Handler handler, CancellationToken ct) =>
                await handler.HandleAsync(new Command(id, req.Reason), ct) switch
                {
                    CancelResult.Ok => Results.NoContent(),
                    CancelResult.NotFound => Results.NotFound(),
                    CancelResult.TooLate => Results.Conflict("Past cancellation deadline"),
                    _ => Results.Problem()
                });

    internal sealed class Handler(BookingsDbContext db, IEventPublisher events)
    {
        public async Task<CancelResult> HandleAsync(Command cmd, CancellationToken ct)
        {
            var booking = await db.Bookings.FindAsync([cmd.BookingId], ct);
            if (booking is null) return CancelResult.NotFound;
            if (!booking.CanCancel(DateTimeOffset.UtcNow)) return CancelResult.TooLate;

            booking.Cancel(cmd.Reason);
            await db.SaveChangesAsync(ct);
            await events.PublishAsync(new BookingCancelled(booking.Id), ct);
            return CancelResult.Ok;
        }
    }
}
```

Kluczowe obserwacje:

- **Cały feature czyta się od góry do dołu w jednym pliku.** Nowa osoba w zespole rozumie "anulowanie rezerwacji" w 2 minuty, bez skakania po pięciu projektach.
- **Handler pisze do DbContextu bezpośrednio.** Tak, bez repozytorium. To nie herezja, tylko świadomy wybór: abstrakcja repozytorium broniła przed wymianą ORM (która nie następuje) kosztem rozmycia zapytań po klasach wspólnych. Slice może sięgać po tyle abstrakcji, ile *ten konkretny feature* potrzebuje — prosty odczyt może być surowym SQL przez Dapper, skomplikowana komenda może budować pełny agregat domenowy. **Spójność wymagana wewnątrz slice'a, nie między slice'ami.**
- **To co wspólne, pozostaje wspólne świadomie:** encja `Booking` z logiką domenową (`CanCancel`, `Cancel`) jest współdzielona przez slice'y modułu — bo reguły biznesowe muszą być jedne. Wspólny kod wyciąga się, gdy duplikacja zaczyna boleć, nie prewencyjnie.

### Mediator czy własne handlery?

W ekosystemie .NET vertical slices historycznie skleja się biblioteką **MediatR**: każdy slice to `IRequest` + `IRequestHandler`, endpoint robi `mediator.Send(command)`, a cross-cutting concerns (logowanie, walidacja, transakcje) wpina się jako pipeline behaviors. **Uwaga licencyjna:** MediatR (podobnie jak kilka innych popularnych bibliotek tego autorstwa, m.in. AutoMapper) ogłosił przejście na model komercyjny — **zanim go wybierzesz, zweryfikuj aktualny model licencji i cennik** i potraktuj to jako normalną decyzję architektoniczną (koszt vs wartość), nie domyślną zależność.

Alternatywa, pokazana w przykładzie wyżej: **własne handlery rejestrowane w DI** — klasa `Handler` per slice, wstrzykiwana do endpointu. Tracisz gotowe pipeline behaviors (zastępują je endpoint filters minimal APIs i dekoratory), zyskujesz zero zależności, pełną kontrolę i brak refleksyjnej magii. Dla nowego kodu w tym kursie używamy własnych handlerów — mediator dodasz, jeśli kiedyś policzy się jego wartość.

### Clean architecture vs vertical slices — uczciwie

**Clean architecture** (Uncle Bob; w .NET spopularyzowana m.in. przez szablony Jasona Taylora): warstwy koncentryczne — Domain w środku, Application wokół, Infrastructure i UI na zewnątrz, zależności tylko do środka, interfejsy w Application, implementacje w Infrastructure. Cel: domena niezależna od technologii.

| Kryterium | Clean architecture | Vertical slices |
|---|---|---|
| Optymalizuje pod | ochronę domeny przed technologią | szybkość i lokalność zmian per feature |
| Koszt dodania feature'a | dotykasz 3–4 warstw, ale każda zmiana "wie, gdzie mieszka" | jeden folder; zero ceremonii |
| Bogata, współdzielona logika domenowa | ✅ naturalne miejsce (Domain) | wymaga dyscypliny, by nie zdublować reguł w slice'ach |
| Aplikacje głównie CRUD/integracyjne | dużo abstrakcji bez zwrotu | ✅ dopasowana ilość abstrakcji per slice |
| Próg wejścia dla zespołu | znany, dobrze opisany | wymaga odwagi "każdy slice może być inny" |
| Testowalność | przez interfejsy i mocki | przez testy integracyjne slice'a (lekcja 03) |

I najważniejsze: **to nie jest wybór wykluczający ani religia.** W praktyce dojrzałe projekty łączą oba: moduł ma slice'y (organizacja per feature) **oraz** wydzielony model domenowy tam, gdzie logika jest naprawdę bogata. Pytanie kontrolne architekta brzmi nie "które jest lepsze", tylko: *ile współdzielonej logiki domenowej naprawdę ma ten moduł?* Dużo → warto inwestować w wyodrębnioną domenę. Mało (CRUD, integracje, przepływy jak w messaging-patterns) → warstwy to podatek bez zwrotu.

### Kiedy tego NIE używać

- **Vertical slices w zespole bez code review i dyscypliny** potrafią zdegenerować do "każdy plik to inny framework". Swoboda per slice wymaga kultury inżynierskiej; w juniorskim zespole ujednolicony szablon (nawet warstwowy) może być bezpieczniejszy.
- **Modular monolith nie jest celem samym w sobie:** mała aplikacja jednego zespołu (DayChunks!) nie potrzebuje granic modułowych z osobnymi projektami kontraktów — wystarczą foldery i zdrowy rozsądek. Granice wprowadzasz, gdy pojawia się więcej domen lub zespołów.
- **Nie refaktoryzuj działającego systemu warstwowego "na slice'y" hurtem.** Strategia dla legacy: nowe feature'y jako slice'y obok starych warstw, stare przepisuj tylko przy okazji zmian biznesowych.

## Praktyka

- [ ] Wróć do macierzy stylów z modułu 1 (lekcja 06) i dopisz do wiersza "modular monolith" kolumnę "jak egzekwuję granice w .NET" — 3 mechanizmy z tej lekcji.
- [ ] **Ćwiczenie główne: refaktoryzacja repo messaging-patterns (moduł 2, lekcja 08) do vertical slices.** Wybierz jedną usługę (np. obsługę zamówień) i: (a) wydziel moduły jako osobne projekty z projektami `*.Contracts`, (b) przepisz endpointy i handlery wiadomości na strukturę `Features/NazwaFeature/`, (c) komunikację między modułami przeprowadź wyłącznie przez kontrakty/eventy, (d) upewnij się, że implementacje są `internal`. Outbox i idempotent consumer z modułu 2 zostają — zmienia się organizacja kodu, nie wzorce niezawodności.
- [ ] W trakcie refaktoryzacji zanotuj 3 miejsca, w których stara struktura (warstwy/wspólne serwisy) utrudniała zmianę — to konkrety do posta modułowego.
- [ ] Zweryfikuj aktualny model licencjonowania MediatR (i ewentualnych alternatyw, np. Wolverine, własne handlery) — dopisz wniosek do notatki o stanie ekosystemu z lekcji 01.
- [ ] Napisz mini-ADR (format z modułu 1): "Vertical slices + własne handlery zamiast warstw + MediatR w messaging-patterns" — kontekst, decyzja, konsekwencje.

## Artefakt

1. **Repo messaging-patterns zrefaktoryzowane do modular monolith + vertical slices** — z czytelnym układem folderów widocznym w README repo (drzewko struktury + 3 zdania "dlaczego tak").
2. **Mini-ADR** o wyborze organizacji kodu — do repo nauki.

## Definition of Done

- [ ] W zrefaktoryzowanym kodzie moduł nie może sięgnąć do wnętrza innego modułu — sprawdzone próbą: dodaj testowo `using` do wewnętrznej encji innego modułu i potwierdź błąd kompilacji (potem usuń).
- [ ] Każdy feature da się przeczytać w całości w jednym folderze, bez skakania po warstwach.
- [ ] Umiesz wyjaśnić w 5 minut: czym różni się slice od warstwy, dlaczego współdzielenie serwisów między feature'ami zwiększa coupling, i kiedy mimo wszystko wybrałbyś clean architecture.
- [ ] ADR jest w repo, a kwestia licencji MediatR — zweryfikowana i zanotowana.
- [ ] Repo nadal działa: przepływ wiadomości z modułu 2 (duplikaty, DLQ) przechodzi jak przed refaktoryzacją.

## Materiały

1. Jimmy Bogard, ["Vertical Slice Architecture"](https://www.jimmybogard.com/vertical-slice-architecture/) — tekst źródłowy pojęcia, krótki i konkretny (plus nagrania jego talków pod tym samym tytułem).
2. Kamil Grzybek, ["Modular Monolith: A Primer"](https://www.kamilgrzybek.com/blog/posts/modular-monolith-primer) + jego przykładowe repo `modular-monolith-with-ddd` — najlepsze dostępne studium egzekwowania granic modułów w .NET.
