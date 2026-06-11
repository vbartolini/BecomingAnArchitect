# Lekcja 04 — Wydajność okiem architekta: profilowanie, EF Core, source generators

## Cel lekcji

Po tej lekcji będziesz wiedział, **jak podejść do problemu wydajnościowego w .NET metodycznie**: którym narzędziem zacząć diagnozę (dotnet-counters, dotnet-trace, profiler VS), jak poprawnie zmierzyć mikro-wydajność (BenchmarkDotNet), gdzie EF Core typowo gubi wydajność i co z tym robić, oraz co architektonicznie zmieniają source generators. To przegląd z lotu architekta, nie głęboki kurs performance engineeringu.

## Dlaczego to ważne

Architekt nie musi być performance engineerem, ale musi umieć dwie rzeczy. Po pierwsze — **poprowadzić diagnozę**: gdy produkcja zwalnia (pamiętasz takie poranki z Perfect Gym), ktoś musi zdecydować, czy patrzymy w bazę, w GC, czy w sieć — i zrobić to pomiarem, nie zgadywaniem. Po drugie — **rozstrzygać spory o optymalizacje**: zespół chce przepisać serializację "bo Span jest szybszy", a architekt musi zapytać "ile to kosztuje teraz i czy to w ogóle jest na ścieżce krytycznej?". Na rozmowach rekrutacyjnych wątek perf pojawia się zwykle jako "opowiedz o problemie wydajnościowym, który rozwiązałeś" — i odpowiedź zdradza, czy kandydat mierzy, czy wróży.

## Teoria

### Zasada zerowa: najpierw zmierz, potem optymalizuj

Naiwne podejście: "to pewnie ta pętla / ten LINQ / ta refleksja" → tydzień optymalizowania → zero efektu, bo 90% czasu i tak szło na N+1 do bazy. Dlaczego intuicja zawodzi: rozkład kosztów w realnych systemach jest skrajnie nierówny (kilka hot pathów odpowiada za prawie wszystko) i **prawie nigdy nie leży tam, gdzie wskazuje pamięć o kodzie**. Właściwy proces:

1. **Zdefiniuj cel liczbowo** ("p95 endpointu rezerwacji < 300 ms przy 200 RPS") — bez celu nie wiesz, kiedy skończyć, a optymalizować można w nieskończoność.
2. **Zmierz na poziomie systemu** (metryki/tracing z modułu 2 — observability to pierwsza linia diagnozy perf!).
3. **Zejdź profilerem do hot pathu.**
4. **Zmień jedną rzecz, zmierz ponownie.** Bez "przy okazji poprawiłem trzy inne".

To dokładnie ta sama dyscyplina co retro z modułu 0 i debugowanie: jedna zmienna naraz.

### Narzędzia diagnostyczne — kiedy które

Trzy narzędzia pokrywają 90% potrzeb; różnią się momentem i miejscem użycia:

| Narzędzie | Co daje | Kiedy sięgasz |
|---|---|---|
| **dotnet-counters** | metryki runtime'u na żywo: CPU, alokacje/s, czas w GC, gen 0/1/2, ThreadPool queue, exceptions/s | **pierwszy rzut oka** na żywym procesie (także na produkcji — narzut minimalny): "czy to GC, CPU, czy głodzony ThreadPool?" |
| **dotnet-trace** | nagranie śladu (CPU sampling, eventy runtime) z działającego procesu do pliku; analiza w PerfView / Visual Studio / speedscope | gdy counters pokazały *że* jest źle, a trzeba ustalić *gdzie* — **działa na serwerze/w kontenerze**, bez instalowania czegokolwiek poza CLI toolem |
| **Visual Studio Profiler** (CPU Usage, .NET Object Allocation, Database tool) | wygodna analiza interaktywna z nawigacją do kodu | diagnoza **lokalna/dev**: reprodukujesz problem u siebie i chcesz najszybszej ścieżki od wykresu do linii kodu |

Heurystyka: **produkcja → dotnet-counters, potem dotnet-trace; lokalnie → profiler VS.** Do tego `dotnet-dump`/`dotnet-gcdump`, gdy problemem jest pamięć (wycieki, LOH). Wszystkie `dotnet-*` instalujesz jako global tools — warto mieć je przećwiczone *zanim* płonie produkcja, stąd ćwiczenie poniżej.

Typowe sygnatury, które warto umieć czytać z counters: wysoki "% Time in GC" + duże alokacje → problem alokacyjny; rosnąca ThreadPool queue przy niskim CPU → blokowanie wątków (sync-over-async — `.Result`/`.Wait()` na kodzie async); CPU 100% na jednym z wielu rdzeni → wąskie gardło jednowątkowe.

### BenchmarkDotNet — mikrobenchmarki robione poprawnie

Naiwny benchmark, który każdy kiedyś napisał: `Stopwatch` wokół pętli. Dlaczego kłamie: pierwszy przebieg mierzy JIT, debug build mierzy brak optymalizacji, dead code elimination potrafi usunąć mierzony kod, a GC i inne procesy dodają szum. **BenchmarkDotNet** rozwiązuje to wszystko: osobny proces per benchmark, rozgrzewka (warmup), wiele iteracji ze statystyką, pilnowanie release builda, diagnozer alokacji:

```csharp
[MemoryDiagnoser]
public class SerializationBenchmarks
{
    private readonly Booking _booking = TestData.SampleBooking();

    [Benchmark(Baseline = true)]
    public string Newtonsoft() => JsonConvert.SerializeObject(_booking);

    [Benchmark]
    public string SystemTextJson() => JsonSerializer.Serialize(_booking);

    [Benchmark]
    public string SystemTextJson_SourceGen() =>
        JsonSerializer.Serialize(_booking, BookingContext.Default.Booking);
}
// uruchomienie: BenchmarkRunner.Run<SerializationBenchmarks>(); (Release!)
```

Wynik to tabela: średni czas, odchylenie, **Gen0/Allocated** (dzięki `[MemoryDiagnoser]` — zawsze go dodawaj, alokacje bywają ważniejsze niż czas). Reguły poprawności: mierz w Release, porównuj do `Baseline`, mierz realistyczne dane (serializacja 3-polowego obiektu nic nie mówi o 5-kilobajtowym DTO), i pamiętaj, że **mikrobenchmark odpowiada tylko na pytanie mikro** — "metoda A szybsza od B" nie znaczy "system będzie szybszy" (patrz zasada zerowa).

### EF Core pod kątem wydajności

EF Core jest dziś szybki — ale domyślne zachowania są zoptymalizowane pod wygodę, nie pod hot path. Pięć rzeczy, które architekt musi znać:

**1. N+1** — odwieczny klasyk: pętla po rezerwacjach, w środku dostęp do `booking.Customer.Name` → 1 zapytanie + N zapytań (przy lazy loadingu) albo `null` (bez niego). Wykrywanie: logi EF (`LogTo`) albo licznik zapytań w teście integracyjnym z lekcji 03. Leczenie: `Include` albo — lepiej — projekcja (punkt 5).

**2. AsNoTracking** — domyślnie EF śledzi każdą zmaterializowaną encję (change tracking), żeby wykryć zmiany przy `SaveChanges`. Dla zapytań tylko-do-odczytu to czysty koszt CPU i pamięci. Reguła: **każde query, które nie kończy się SaveChanges, dostaje `AsNoTracking()`** (w vertical slices z lekcji 02 widać to naturalnie: slice'y odczytowe — no-tracking, slice'y komend — tracking).

**3. Split queries** — `Include` kilku kolekcji naraz generuje jednego JOIN-owego potwora z eksplozją kartezjańską (wiersze = iloczyn liczności kolekcji). `AsSplitQuery()` rozbija to na osobne zapytania per kolekcja. Trade-off: mniej danych na druciе, ale brak spójności migawkowej między zapytaniami i więcej round-tripów.

**4. Compiled queries** — `EF.CompileAsyncQuery(...)` pomija powtarzaną translację LINQ→SQL dla zapytań wołanych tysiące razy na sekundę. To optymalizacja hot pathu, nie domyślna praktyka.

**5. Projekcje zamiast pełnych encji** — najważejsza pozycja tej listy. Pobieranie pełnej encji z nawigacjami, żeby pokazać 3 pola na liście, to płacenie za transfer, materializację i tracking rzeczy, których nie używasz:

```csharp
// zamiast: db.Bookings.Include(b => b.Customer).ToListAsync()
var rows = await db.Bookings
    .Where(b => b.Date == today)
    .Select(b => new BookingRow(b.Id, b.Customer.Name, b.Slot))  // SELECT tylko 3 kolumn
    .ToListAsync(ct);
```

Projekcja `Select` do DTO/record załatwia jednocześnie: N+1 (JOIN robi baza), tracking (projekcje nie są śledzone) i transfer. **Reguła architekta: encje do zapisu, projekcje do odczytu** — to zresztą CQRS w wersji minimalnej, bez żadnej infrastruktury.

### Span, Memory i alokacje — w pigułce

`Span<T>`/`ReadOnlySpan<T>` to typy pozwalające pracować na **wycinku istniejącej pamięci bez kopiowania i bez alokacji** — np. parsować fragment stringa bez `Substring` (która alokuje nowy string). `Memory<T>` to wersja, którą można przechowywać na heapie i używać w async. Wokół nich wyrosło API platformy (`ArrayPool<T>`, `stackalloc`, przeciążenia `int.Parse(ReadOnlySpan<char>)` itd.).

Kiedy to **ma** znaczenie: kod wywoływany miliony razy — parsery, serializatory, middleware na ścieżce każdego requestu, przetwarzanie strumieni — gdzie alokacje napędzają GC, a GC zżera latencję (to widzisz w dotnet-counters jako "% Time in GC"). Tam zamiana `string.Split` na przejście Spanem realnie obniża p99.

Kiedy to **przedwczesna optymalizacja**: niemal cała logika biznesowa. Handler wykonujący jedno zapytanie do bazy (miliony nanosekund) nie odczuje oszczędności 40 ns na parsowaniu. Span komplikuje kod (ograniczenia ref struct: nie w async, nie na heap) — to koszt, który musi się zwracać. Rola architekta: **wiedzieć, że ta półka istnieje i kiedy po nią posłać** — a posyłać dopiero z profilerem w ręku. Dobra wiadomość: większość zysków ze Span dostajesz za darmo, bo dostała je sama platforma (Kestrel, System.Text.Json, parsery).

### Source generators — koniec refleksji w runtime

**Source generator** to komponent kompilatora (Roslyn), który podczas budowania analizuje Twój kod i **dogenerowuje dodatkowy kod C#** do kompilacji. Po co: rzeczy, które przez 20 lat robiło się refleksją w runtime (serializacja, mapowanie, DI), mogą być wygenerowane raz, w compile time. Konsekwencje: zero kosztu refleksji w runtime, błędy wykrywane przy kompilacji zamiast w nocy na produkcji, oraz zgodność z **AOT** (ahead-of-time compilation — kompilacja do natywnego binarium bez JIT; refleksja emitująca kod tam nie działa).

Trzy wbudowane przykłady, które warto znać i używać:

```csharp
// 1. System.Text.Json — serializacja bez refleksji:
[JsonSerializable(typeof(Booking))]
internal partial class BookingContext : JsonSerializerContext { }
// użycie: JsonSerializer.Serialize(booking, BookingContext.Default.Booking)

// 2. Regex — wyrażenie kompilowane w build time, z analizą poprawności:
[GeneratedRegex(@"^[A-Z]{2}\d{6}$")]
private static partial Regex MembershipNumber();

// 3. Logging — zero alokacji, gdy poziom logowania wyłączony:
[LoggerMessage(Level = LogLevel.Warning,
    Message = "Booking {BookingId} cancelled after deadline by {UserId}")]
static partial void LogLateCancellation(ILogger logger, Guid bookingId, Guid userId);
```

Architektonicznie source generators to trend szerszy niż te trzy API: ekosystem przesuwa się od "magii w runtime" (refleksja, dynamic proxy, skanowanie assembly) do "jawności w compile time". Dla kogoś, kto pamięta epokę, gdy każdy framework opierał się na refleksji — to jedna z najgłębszych zmian filozofii platformy i mocny punkt do posta modułowego.

### Kiedy tego NIE robić — trade-offy optymalizacji

- **Optymalizacja bez pomiaru i bez celu** — patrz zasada zerowa; czas architekta wydany na 5% zysku poza ścieżką krytyczną to czysta strata.
- **Czytelność to też atrybut jakościowy** (moduł 1!). Compiled queries, Span, pooling — każda z tych technik czyni kod trudniejszym. Na hot path warto; rozsiane "prewencyjnie" po całym kodzie — obniżają maintainability za nic.
- **Mikrobenchmark nie zastępuje testu obciążeniowego.** Decyzje o skalowaniu i pojemności (moduł 3) podejmuje się na podstawie load testów systemu, nie tabelek BenchmarkDotNet.

## Praktyka

- [ ] Zainstaluj `dotnet-counters` i `dotnet-trace`; podłącz counters do działającego lokalnie repo messaging-patterns i przez 5 minut poobserwuj metryki pod małym obciążeniem (pętla curl/k6). Zanotuj 3 metryki i ich wartości "zdrowe" — to Twoja baza odniesienia.
- [ ] Wprowadź celowo sync-over-async (`.Result`) w jednym handlerze, obciąż API i zobacz sygnaturę w counters (ThreadPool queue). Cofnij zmianę. (Ćwiczysz rozpoznawanie wzorca, zanim spotka Cię na produkcji.)
- [ ] Znajdź lub sprokuruj N+1 w repo (odczyt listy z nawigacją), wykryj je logami EF, napraw projekcją; porównaj liczbę zapytań i czas przed/po — zapisz liczby.
- [ ] Napisz jeden benchmark BenchmarkDotNet z `[MemoryDiagnoser]` i `Baseline` — np. serializacja DTO z repo: refleksja vs source-generated context. Wklej tabelę wyników do notatki.
- [ ] Włącz w repo logging source generator (`[LoggerMessage]`) dla 2–3 najczęstszych logów na hot path.
- [ ] Spisz notatkę "moje reguły perf" (10 linii): kolejność diagnozy, narzędzie per sytuacja, reguły EF (no-tracking/projekcje), kiedy NIE optymalizować.

## Artefakt

1. **Notatka "reguły perf" + wyniki benchmarku** w repo nauki (liczby przed/po dla N+1 i tabela BenchmarkDotNet).
2. **Finalna wersja posta modułowego** "Co realnie zmieniło się w .NET przez ostatnie lata — okiem kogoś, kto pamięta WebForms" — masz już komplet materiału z lekcji 01–04 (unifikacja, minimal APIs, slices, Testcontainers, source generators vs refleksja). Publikacja domyka moduł.

## Definition of Done

- [ ] Dla scenariusza "endpoint nagle wolny na produkcji" umiesz bez notatek podać kolejność kroków i narzędzi.
- [ ] Naprawiłeś realne N+1 i masz liczby przed/po.
- [ ] Masz działający, poprawny metodycznie benchmark (Release, warmup, baseline, MemoryDiagnoser) i potrafisz wyjaśnić jego tabelę wyników.
- [ ] Umiesz wyjaśnić koledze: co robi source generator, dlaczego wypiera refleksję i co to ma wspólnego z AOT.
- [ ] Post modułowy opublikowany; moduł 4 odhaczony w `architect-track-plan.md`, `progress.md` zaktualizowany.

## Materiały

1. [Dokumentacja diagnostyki .NET](https://learn.microsoft.com/dotnet/core/diagnostics/) (dotnet-counters, dotnet-trace, scenariusze "diagnose performance issues") + [EF Core: Efficient Querying](https://learn.microsoft.com/ef/core/performance/) — dwa rozdziały źródłowe tej lekcji.
2. [Dokumentacja BenchmarkDotNet](https://benchmarkdotnet.org/) — sekcje "Good Practices" i "Diagnosers" wystarczą.
