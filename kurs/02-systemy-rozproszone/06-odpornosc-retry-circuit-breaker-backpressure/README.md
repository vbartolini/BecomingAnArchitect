# Lekcja 06 — Odporność: timeouts, retry, circuit breaker, backpressure

## Cel lekcji

Po tej lekcji będziesz umiał zbudować kompletny pipeline odporności (*resilience*) dla wywołań między usługami: timeouts z budżetem błędów, retry z exponential backoff i jitterem (oraz wiedzą, kiedy retry **szkodzi**), circuit breaker z rozumieniem jego stanów, backpressure i bulkhead — wszystko z implementacją w .NET (Polly / Microsoft.Extensions.Resilience).

## Dlaczego to ważne

Pojedyncza usługa pada rzadko. Systemy rozproszone padają **kaskadowo**: jedna wolna zależność wyczerpuje wątki w usłudze wyżej, ta zaczyna odmawiać, jej klienci robią retry, ruch rośnie ×3 dokładnie wtedy, gdy system jest najsłabszy — i mała awaria staje się ogólnosystemową. Większość głośnych awarii dużych platform to właśnie kaskady napędzane retry, nie pierwotne błędy. Na rozmowach "co się stanie, gdy ta zależność zwolni?" to pytanie, które odróżnia projektowanie happy path od projektowania systemów. A Ty te kaskady widziałeś na żywo — czas nazwać mechanizmy obrony.

## Teoria

### Timeout — pierwsza linia obrony

**Timeout** to maksymalny czas, jaki godzisz się czekać na odpowiedź. Brzmi banalnie, ale to najważniejszy mechanizm tej lekcji, bo **wolna zależność jest groźniejsza niż martwa**: martwa odrzuca połączenie natychmiast (błąd jest tani), wolna trzyma Twoje wątki, sloty połączeń i pamięć — aż wyczerpie pulę i Twoja usługa też stanie. Bez timeoutu czas oczekiwania jest teoretycznie nieskończony; domyślne timeouty bibliotek (np. 100 s w `HttpClient`) są w praktyce nieskończonością.

Zasady:

- **Każde wywołanie sieciowe ma jawny timeout** — dobrany do realnej latencji zależności (np. p99 × 2–3), nie wzięty z sufitu ani z defaulta.
- **Budżet czasu i propagacja**: jeśli klient czeka na Ciebie maksymalnie 2 s, a Ty wołasz trzy zależności sekwencyjnie, ich timeouty muszą zmieścić się w tych 2 s — inaczej liczysz odpowiedź dla kogoś, kto już się rozłączył. Praktyka w .NET: przekazuj `CancellationToken` z żądania przychodzącego **w dół przez całą ścieżkę** (do `HttpClient`, EF Core, brokera) — to jest mechanizm propagacji budżetu. Wzorzec dojrzały: każde piętro dostaje pozostały budżet (deadline), nie własny niezależny timeout.
- Timeout to też **decyzja biznesowa**: po przekroczeniu — błąd? wartość z cache? odpowiedź zdegradowana ("cena niedostępna, spróbuj później")? To się nazywa *graceful degradation* i warto mieć na nią plan, zanim będzie potrzebna.

### Retry: exponential backoff + jitter — i kiedy retry SZKODZI

Skoro (lekcja 01) błędy przejściowe mijają, naturalny odruch to ponowić. Naiwny retry — natychmiast, w pętli — jest jednak **bronią obosieczną**:

**Retry storm**: zależność zwalnia pod obciążeniem → wszyscy klienci dostają timeouty → wszyscy ponawiają → ruch rośnie ×(1+liczba prób) dokładnie w momencie, gdy zależność ledwo żyje → zależność umiera na dobre, a po restarcie natychmiast dostaje falę zaległych retry i umiera znowu (*thundering herd* — efekt stada). Retry zamienia degradację w awarię. Zapamiętaj zdanie na rozmowę: **retry przenosi obciążenie w przyszłość i je multiplikuje — pomaga przy błędach incydentalnych, szkodzi przy przeciążeniu**.

Cywilizowany retry:

- **Exponential backoff** (wykładnicze odczekiwanie): odstępy rosną wykładniczo — 1 s, 2 s, 4 s, 8 s… Daje zależności czas na dojście do siebie.
- **Jitter** (losowe rozrzucenie): do odstępu dodaj losowość. Bez jittera wszyscy klienci, którym timeout wypadł w tej samej sekundzie, ponowią **równocześnie** — zsynchronizowane fale uderzeniowe co 1, 2, 4 s. Jitter rozsmarowuje falę w czasie. (Klasyczna analiza AWS "Exponential Backoff and Jitter" pokazuje, że *full jitter* — losuj z przedziału [0, backoff] — wygrywa.)
- **Skończony, mały limit prób** (2–3) i tylko dla błędów **przejściowych** (klasyfikacja z lekcji 05: 503/timeout — tak; 400/walidacja — nigdy).
- **Retry tylko na jednym piętrze**: jeśli każda z 4 warstw robi 3 próby, na dole ląduje 3⁴ = 81 wywołań. Ponawiaj blisko źródła błędu albo na krawędzi — nie wszędzie.
- Retry wymaga **idempotencji** wywoływanej operacji (lekcja 01) — inaczej ponawiając "pobierz płatność" produkujesz duplikaty.

### Circuit breaker od zera

Retry odpowiada na pytanie "co zrobić z *tym jednym* nieudanym wywołaniem". **Circuit breaker** (bezpiecznik, jak w instalacji elektrycznej) odpowiada na pytanie systemowe: "czy w ogóle **warto teraz wołać** tę zależność?". Gdy zwarcie w instalacji, bezpiecznik odcina prąd — nie próbujesz dalej wciskać prądu w zwarty obwód.

Trzy stany:

- **Closed** (zamknięty — prąd płynie): ruch przechodzi normalnie; breaker liczy błędy. Gdy odsetek/liczba błędów w oknie przekroczy próg (np. >50% błędów przy min. 10 wywołaniach w 30 s) → **Open**.
- **Open** (otwarty — obwód przerwany): wywołania **nie wychodzą w ogóle** — breaker odrzuca je natychmiast lokalnym wyjątkiem (`BrokenCircuitException`). Zalety: klient dostaje błąd w mikrosekundy zamiast czekać na timeout (fail fast), a chora zależność dostaje to, czego najbardziej potrzebuje — **ciszę**. Po zadanym czasie (np. 30 s) → **Half-open**.
- **Half-open** (półotwarty — próba): breaker przepuszcza pojedyncze wywołanie-sondę. Sukces → Closed (wracamy do normalności); porażka → Open (kolejna runda ciszy).

Circuit breaker i retry współpracują: retry obsługuje pojedyncze czknięcia, breaker chroni przed wołaniem trupa (i ucina retry storm u źródła — otwarcie breakera zatrzymuje też ponawianie). Breaker per zależność (lub per endpoint), nie jeden globalny. Otwarcie breakera to zdarzenie do **alertu** — to system mówi "uznałem zależność za chorą".

### Backpressure i bulkhead

**Backpressure** (przeciwciśnienie) to mechanizm, którym odbiorca komunikuje nadawcy: "zwolnij, nie nadążam". Bez niego nadmiarowy ruch gromadzi się w niejawnych buforach (pula wątków, socket backlog, pamięć) aż do wybuchu. Kluczowy element: **bounded queues** (kolejki ograniczone) — każdy bufor w systemie musi mieć limit, bo nieograniczona kolejka to nie jest brak decyzji, to decyzja "przyjmę wszystko i umrę później" (rosnąca kolejka = rosnąca latencja = timeouty u klientów = retry = …). Gdy limit osiągnięty, masz trzy uczciwe opcje: **blokuj** producenta (naturalne w procesie — `Channel` z `BoundedChannelFullMode.Wait`), **odrzucaj** z jasnym sygnałem (HTTP 429 + `Retry-After` — to jest backpressure w REST), albo **zrzucaj** najmniej ważne (load shedding). Broker wiadomości to backpressure w skali systemu: kolejka buforuje, a konsument **ciągnie** (pull) we własnym tempie — m.in. po to istnieje messaging. Limit prefetch konsumenta to też bounded queue.

**Bulkhead** (gródź — jak przegrody wodoszczelne w kadłubie statku: przebicie zalewa jeden przedział, nie cały statek): izolacja zasobów per zależność/funkcja. Zamiast jednej wspólnej puli — osobne limity równoległości: np. wywołania do wolnego systemu raportowego mają własny limit 10 równoległych slotów, więc gdy raporty flaczeją, ścieżka płatności nie odczuwa braku wątków. W .NET: osobne `HttpClient`/handlery z limitem połączeń, semafory, strategia rate-limiter/bulkhead w pipeline odporności.

### Implementacja w .NET: Microsoft.Extensions.Resilience (Polly v8)

Polly to standardowa biblioteka odporności w .NET; od wersji 8 jej silnik (`ResiliencePipeline`) jest podstawą oficjalnych pakietów `Microsoft.Extensions.Resilience` i `Microsoft.Extensions.Http.Resilience`. Pipeline składasz ze strategii — kolejność ma znaczenie (od zewnątrz: timeout całkowity → retry → breaker → timeout pojedynczej próby):

```csharp
// dotnet add package Microsoft.Extensions.Http.Resilience
builder.Services.AddHttpClient<PaymentApiClient>(c =>
        c.BaseAddress = new Uri("https://payments.internal"))
    .AddResilienceHandler("payments", pipeline =>
    {
        pipeline.AddTimeout(TimeSpan.FromSeconds(5));            // budżet całego wywołania (z retry)

        pipeline.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType      = DelayBackoffType.Exponential,
            UseJitter        = true,                             // backoff + jitter w jednej linii
            Delay            = TimeSpan.FromMilliseconds(500)
            // domyślnie ponawia tylko błędy przejściowe (5xx, 408, timeouts) — nie 4xx
        });

        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio      = 0.5,                             // >50% błędów…
            MinimumThroughput = 10,                              // …przy min. 10 wywołaniach…
            SamplingDuration  = TimeSpan.FromSeconds(30),        // …w oknie 30 s → Open
            BreakDuration     = TimeSpan.FromSeconds(30)         // cisza przed Half-open
        });

        pipeline.AddTimeout(TimeSpan.FromSeconds(2));            // timeout pojedynczej próby
    });
```

Jest też gotowy zestaw rozsądnych domyślnych wartości jedną linią: `.AddStandardResilienceHandler()` (rate limiter + total timeout + retry + breaker + attempt timeout). Zaczynaj od niego, dostrajaj świadomie.

### Trade-offy i "kiedy tego NIE używać"

- Każdy mechanizm tej lekcji **wymienia spójność/kompletność na dostępność**: retry dodaje latencję i duplikaty, breaker odrzuca żądania, które *może* by przeszły, load shedding gubi pracę. To dobre wymiany — ale trzeba je nazywać, nie udawać, że są darmowe.
- **Nie ponawiaj**: błędów trwałych, operacji nieidempotentnych bez idempotency key, na wielu piętrach naraz, w nieskończoność.
- **Breaker bywa zbędny** dla zależności wewnątrz procesu albo wołanych raz na godzinę (MinimumThroughput i tak nie zadziała sensownie) — nie instaluj pełnego pipeline'u wszędzie rytualnie; dobieraj do krytyczności ścieżki.
- Parametry (progi, okna, limity) **muszą wynikać z pomiarów** (latencje p99, realny ruch) — pipeline z parametrami z sufitu potrafi szkodzić bardziej niż jego brak. To łączy się wprost z lekcją 07: bez metryk nie ma odporności, jest zgadywanie.

## Praktyka

- [ ] Zbuduj mini-laboratorium: API "Payment" z wstrzykiwanymi awariami (parametry: opóźnienie, procent 500-ek) + klient z pipeline'em jak wyżej. Wystarczą dwa projekty w jednym solution.
- [ ] Eksperyment 1 — retry storm: ustaw 100% błędów i klienta z agresywnym retry bez backoffu z 50 równoległych wywołań; obejrzyj liczbę żądań przychodzących do API. Potem włącz backoff+jitter i porównaj. Zapisz liczby.
- [ ] Eksperyment 2 — circuit breaker: zaobserwuj w logach przejścia Closed → Open → Half-open → Closed; zmierz różnicę czasu odpowiedzi klienta przy zależności martwej z breakerem (fail fast) i bez (czekanie na timeout).
- [ ] Eksperyment 3 — bounded queue: `Channel.CreateBounded<T>(20)` z wolnym konsumentem i szybkim producentem; porównaj zachowanie `Wait` vs `DropWrite`. Zanotuj, co to oznacza dla wyboru "blokuj vs odrzucaj" w API (429).
- [ ] Rozpisz budżet czasu dla ścieżki ze swojego mini-projektu (lekcja 08): klient czeka X → ile dostaje każde piętro i gdzie propagujesz `CancellationToken`.

## Artefakt

Folder `06-odpornosc-retry-circuit-breaker-backpressure/` w repo nauki: laboratorium z trzema eksperymentami + `notes.md` z liczbami (przed/po dla retry storm — te liczby to gotowy materiał na post lub slajd "retry storm w liczbach"). Pipeline z tej lekcji przeniesiesz żywcem do mini-projektu.

## Definition of Done

- [ ] Umiesz wyjaśnić, dlaczego wolna zależność jest groźniejsza niż martwa, i co z tym robi timeout + fail fast.
- [ ] Umiesz opowiedzieć mechanikę retry storm i rolę backoffu oraz jittera (osobno! każdy rozwiązuje inny problem).
- [ ] Rysujesz z pamięci trzy stany circuit breakera z warunkami przejść.
- [ ] Umiesz powiedzieć, czym jest backpressure, czemu bounded queue to decyzja, a nieograniczona kolejka to też decyzja (tylko gorsza), i jak backpressure wyraża się w REST (429) i w messagingu (pull + prefetch).
- [ ] Eksperymenty wykonane, liczby zapisane.

## Materiały

1. **"Release It!" (2nd ed.)**, Michael Nygard — rozdziały o stability patterns (circuit breaker, bulkhead, timeouts); to książka, z której te wzorce weszły do branży, pisana przez praktyka produkcji.
2. **Dokumentacja [Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/dotnet/core/resilience/http-resilience)** — aktualne API, standard handler i opcje strategii.
