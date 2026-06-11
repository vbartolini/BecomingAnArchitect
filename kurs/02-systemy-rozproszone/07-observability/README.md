# Lekcja 07 — Observability: logi, metryki, distributed tracing

## Cel lekcji

Po tej lekcji będziesz rozumiał różnicę między monitoringiem a observability, znał trzy filary (logi strukturalne, metryki RED/USE, distributed tracing) i umiał zainstrumentować usługę .NET przez OpenTelemetry tak, żeby dało się prześledzić jedno żądanie przez cztery usługi i kolejkę — bez zgadywania.

## Dlaczego to ważne

Wszystkie wzorce z lekcji 01–06 mają wspólną cechę: **rozsmarowują jedną operację biznesową po wielu usługach i czasie**. Outbox publikuje z opóźnieniem, saga skacze między usługami, retry powtarza, DLQ odkłada na później. Pytanie "gdzie utknęła rezerwacja klienta X?" w takim systemie bez observability to seans spirytystyczny: grep po czterech zestawach logów z czterema formatami i bez wspólnego identyfikatora. Na rozmowach na poziom architekta "jak będziesz to debugować na produkcji?" pada zaraz po "jak to zaprojektujesz?" — i wielu kandydatów dopiero wtedy zauważa, że ich piękny diagram jest nieobserwowalny. A Ty wiesz z produkcji, że system, którego nie widać, to system, którego się boisz dotykać.

## Teoria

### Observability vs monitoring

**Monitoring** odpowiada na pytania, które zadałeś **z góry**: definiujesz wskaźniki (CPU, liczba 500-ek, długość kolejki), progi i alerty. Świetny na **znane niewiadome** — awarie, które umiesz przewidzieć.

**Observability** (obserwowalność — termin z teorii sterowania: system jest obserwowalny, gdy jego stan wewnętrzny da się wywnioskować z wyjść) to właściwość systemu: czy emituje dość bogate dane, żeby odpowiadać na pytania, których **nie przewidziałeś** — *unknown unknowns*. "Czemu żądania klientów z aplikacji mobilnej w wersji 3.2 są wolne tylko we wtorki?" — żaden z góry zdefiniowany dashboard tego nie pokaże; bogate, skorelowane dane telemetryczne pozwolą to wykopać.

Skrót myślowy: monitoring mówi **że** coś jest nie tak; observability pozwala ustalić **dlaczego**. Jedno nie zastępuje drugiego — monitoring budzi, observability prowadzi śledztwo.

### Filar 1: logi strukturalne i korelacja

**Log strukturalny** to zdarzenie z polami (JSON), nie linijka tekstu. Różnica praktyczna: po `"OrderId": "abc-123"` możesz **filtrować i agregować**; po `"Processing order abc-123 for customer..."` możesz tylko grepować i płakać.

W .NET masz to w standardzie — pod warunkiem używania **message template**, nie interpolacji:

```csharp
// DOBRZE — template: OrderId i Total to osobne, indeksowane pola
logger.LogInformation("Order {OrderId} placed, total {Total}", order.Id, order.Total);

// ŹLE — interpolacja: jedna płaska stringa, pola utracone na zawsze
logger.LogInformation($"Order {order.Id} placed, total {order.Total}");
```

**Korelacja**: każdy wpis logu związany z obsługą żądania/wiadomości musi nieść wspólny identyfikator przepływu (correlation id — dziś w praktyce **trace id**, patrz filar 3). W .NET służą do tego scope'y (`logger.BeginScope`) i automatyczne dołączanie trace id przez integrację logowania z tracingiem — wtedy "pokaż wszystkie logi tej jednej rezerwacji, ze wszystkich usług" to jeden filtr, nie archeologia.

### Filar 2: metryki — RED i USE

**Metryka** to liczba agregowana w czasie (licznik, histogram) — tania w zbieraniu i przechowywaniu, idealna do dashboardów i alertów, ale pozbawiona kontekstu pojedynczego żądania (od tego są trace'y). Dwa kanoniczne zestawy, które warto znać z nazwy:

**RED** — dla **usług** (request-driven), patrzy oczami klienta usługi:
- **R**ate — ile żądań na sekundę obsługuje usługa,
- **E**rrors — ile z nich kończy się błędem,
- **D**uration — jak długo trwają (koniecznie percentyle: p50/p95/p99 — średnia kłamie, bo maskuje ogon, a w ogonie żyją Twoi najbardziej wkurzeni użytkownicy).

**USE** — dla **zasobów** (CPU, dysk, pula połączeń, kolejka), patrzy oczami infrastruktury:
- **U**tilization — jaki procent czasu/pojemności zasób jest zajęty,
- **S**aturation — ile pracy **czeka** w kolejce do zasobu (nadmiar ponad pojemność; saturacja > 0 przy utylizacji ~100% = zasób jest wąskim gardłem),
- **E**rrors — błędy zasobu (odrzucone połączenia, I/O errors).

Zapamiętaj parę: **RED dla usług, USE dla zasobów** — razem dają komplet: RED mówi, że klienci cierpią; USE mówi, który zasób ich dusi. Dla systemu kolejkowego: RED konsumenta (rate przetwarzania, błędy, czas obsługi) + USE kolejki (głębokość = saturacja!) + metryka z lekcji 05: rozmiar DLQ.

### Filar 3: distributed tracing od zera

**Trace** (ślad) to zapis drogi **jednego żądania** przez cały system — drzewo operacji. **Span** to jedna operacja w tym drzewie (obsługa HTTP, zapytanie SQL, publikacja do kolejki): ma nazwę, czas startu i trwania, atrybuty (np. `db.statement`), status i wskazanie rodzica. Trace = drzewo spanów o wspólnym **trace id**.

Wizualizacja trace'a (wodospad) odpowiada na pytania nieosiągalne dla logów i metryk: *gdzie* w łańcuchu 4 usług zniknęło 800 ms; czy wywołania szły sekwencyjnie, choć mogły równolegle; który konkretnie SQL był wolny **w tym** żądaniu; czy opóźnienie to przetwarzanie, czy czekanie w kolejce.

**Context propagation** (propagacja kontekstu) — serce tracingu: żeby spany z czterech usług skleiły się w jedno drzewo, identyfikator musi **podróżować z żądaniem**. Standard **W3C Trace Context** definiuje nagłówek HTTP `traceparent`:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              │  └────────── trace id ──────────┘ └── parent span ──┘ └ flagi (sampled)
```

Usługa odbierająca kontynuuje trace zamiast zaczynać nowy; przez kolejkę kontekst jedzie w **nagłówkach wiadomości** (to samo pole `traceparent` w application properties). Dzięki standardowi propagacja działa między usługami w różnych technologiach i bibliotekach.

W trace'ach przez messaging jest subtelność: span konsumpcji może być *dzieckiem* spanu publikacji (jeden długi trace przez całą sagę) albo osobnym trace'em z *linkiem* do nadawcy — przy długich, asynchronicznych przepływach linki bywają czytelniejsze. Na razie wystarczy wiedzieć, że oba warianty istnieją.

**Sampling**: w dużym ruchu nie przechowujesz 100% trace'ów (koszt!) — próbkuje się np. 10% plus wszystkie błędne/wolne (tail sampling). Decyzja podróżuje we flagach `traceparent`.

### OpenTelemetry w .NET

**OpenTelemetry (OTel)** to otwarty standard (API + SDK + protokół OTLP) zbierania telemetrii wszystkich trzech filarów — branżowy konsensus, który uwolnił telemetrię od vendor lock-inu: instrumentujesz raz, backend (Jaeger, Prometheus, Application Insights, Grafana…) wybierasz i zmieniasz konfiguracją. W .NET sytuacja jest wyjątkowo dobra, bo tracing jest **wbudowany w platformę** (`System.Diagnostics.Activity` = span; `ActivitySource` = źródło spanów), a OTel je tylko zbiera i eksportuje. `HttpClient`, ASP.NET Core i coraz więcej bibliotek tworzy spany i propaguje `traceparent` **automatycznie**.

```csharp
// dotnet add package OpenTelemetry.Extensions.Hosting
//   + OpenTelemetry.Instrumentation.AspNetCore, .Http, .EntityFrameworkCore
//   + OpenTelemetry.Exporter.OpenTelemetryProtocol
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("booking-service"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()        // span na każde żądanie przychodzące
        .AddHttpClientInstrumentation()        // span + traceparent na wychodzące HTTP
        .AddEntityFrameworkCoreInstrumentation()
        .AddSource("BookingService.Messaging") // nasze własne spany (niżej)
        .AddOtlpExporter())                    // → Jaeger/Aspire Dashboard/App Insights
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()        // RED za darmo: http.server.request.duration
        .AddRuntimeInstrumentation()           // GC, wątki — kawałek USE
        .AddOtlpExporter());
```

Własny span wokół operacji messagingowej (broker bez auto-instrumentacji = ręczna propagacja, raz, w jednym miejscu):

```csharp
private static readonly ActivitySource Source = new("BookingService.Messaging");

public async Task PublishAsync(OutboxMessage msg, CancellationToken ct)
{
    using var activity = Source.StartActivity("publish BookingCreated", ActivityKind.Producer);
    activity?.SetTag("messaging.message.id", msg.Id);

    // propagacja: wstrzyknij kontekst do nagłówków wiadomości
    var headers = new Dictionary<string, string>();
    if (activity is not null)
        headers["traceparent"] = activity.Id!; // format W3C

    await bus.PublishAsync(msg.Type, msg.Payload, headers, ct);
}
// Konsument: odczytaj traceparent z nagłówków i przekaż jako parentId:
// Source.StartActivity("consume BookingCreated", ActivityKind.Consumer, parentId: traceparent)
```

Do oglądania lokalnie: **Jaeger** w Dockerze albo **.NET Aspire Dashboard** (lekki, przyjmuje OTLP, pokazuje trace'y/metryki/logi — idealny do mini-projektu).

### Jak debugować przepływ przez 4 usługi — procedura

Scenariusz: klient zgłasza "zapłaciłem, brak potwierdzenia". System: API → booking → payment → notification, sklejone kolejkami. Kolejność śledztwa:

1. **Znajdź trace id** — po atrybutach biznesowych (`booking.id` jako tag spanu) albo z logów API (filtr po OrderId → wpis ma trace id).
2. **Otwórz wodospad trace'a** — od razu widzisz, dokąd przepływ **doszedł**: ostatni span pokazuje miejsce zatrzymania (np. payment opublikował `PaymentCompleted`, ale brak spanu konsumpcji w notification).
3. **Zawęź hipotezy metrykami** — skoro notification nie skonsumował: głębokość jego kolejki (saturacja rośnie = konsument nie nadąża lub stoi), rozmiar DLQ (poison? — lekcja 05), RED konsumenta (errors?).
4. **Dopiero teraz logi** — z filtrem po trace id: dokładny wyjątek, wartości pól. Logi są **ostatnim**, najdrobniejszym szczebłem, nie pierwszym — w tym tkwi zmiana nawyku, którą daje ta lekcja.

Bez tracingu ta sama diagnoza to godziny grepowania; z tracingiem — minuty. To jest różnica, którą opowiadasz na rozmowie jako "jak ustawiam zespołowi pracę z incydentami".

### Trade-offy i "kiedy tego NIE używać"

- **Telemetria kosztuje**: narzut CPU (zwykle mały), ale głównie **koszt przechowywania i transferu** — pełne logowanie + 100% trace'ów w dużym ruchu potrafi kosztować jak compute. Sampling i retencja to decyzje architektoniczne, nie szczegół.
- **Kardynalność metryk**: metryka z etykietą o milionie wartości (np. `user_id`) wysadza system metryk — wysokokardynalne wymiary należą do trace'ów i logów, nie do metryk.
- **Nie buduj wszystkiego od razu**: w małym systemie zacznij od auto-instrumentacji OTel + logów strukturalnych z trace id; własne spany i metryki dodawaj tam, gdzie faktycznie debugujesz. Dashboard nikt-nie-patrzy to dług, nie atut.
- Observability **nie zastępuje** testów ani projektowania pod awarie — pozwala szybko zrozumieć awarię, nie zapobiega jej.

## Praktyka

- [ ] Dodaj OpenTelemetry (tracing + metryki) do dwóch usług z dotychczasowych ćwiczeń (np. klient + Payment API z lekcji 06); odpal Jaeger albo Aspire Dashboard i obejrzyj pierwszy trace przechodzący przez obie.
- [ ] Przepuść trace przez kolejkę: zaimplementuj propagację `traceparent` w nagłówkach wiadomości (publish + consume jak w przykładzie) i potwierdź w UI, że spany producenta i konsumenta są w jednym drzewie.
- [ ] Skonfiguruj logi strukturalne tak, żeby każdy wpis zawierał trace id; wykonaj ćwiczenie "od zgłoszenia do przyczyny": wstrzyknij błąd w jednej usłudze i przejdź procedurę 4 kroków z teorii, notując czasy.
- [ ] Zdefiniuj minimalny zestaw metryk dla swojego mini-projektu: RED per usługa + głębokość kolejek + rozmiar DLQ. Zapisz listę (nazwa, typ, etykiety) — zaimplementujesz w lekcji 08.

## Artefakt

Folder `07-observability/` w repo nauki: dwie usługi z działającym tracingiem przez HTTP **i** przez kolejkę (zrzut ekranu wodospadu w `notes.md` — wejdzie do README mini-projektu), lista metryk dla mini-projektu, notatka z ćwiczenia debugowania. Zrzut wodospadu trace'a przez 4 usługi to też świetny materiał wizualny pod post na LinkedIn.

## Definition of Done

- [ ] Umiesz wyjaśnić różnicę monitoring vs observability przez "znane vs nieznane niewiadome".
- [ ] Rozwijasz z pamięci RED i USE i wiesz, który zestaw do czego (usługi vs zasoby) — oraz czemu percentyle, nie średnia.
- [ ] Umiesz narysować trace jako drzewo spanów i wyjaśnić, jak `traceparent` skleja usługi — także przez kolejkę.
- [ ] Masz działający trace end-to-end przez minimum 2 usługi i kolejkę, widoczny w UI.
- [ ] Umiesz opowiedzieć 4-krokową procedurę debugowania przepływu przez 4 usługi (trace → metryki → logi) i uzasadnić kolejność.

## Materiały

1. **Dokumentacja [.NET Observability with OpenTelemetry](https://learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel)** — kanoniczne wprowadzenie Microsoftu: Activity/ActivitySource, konfiguracja, przykłady.
2. **"Observability Engineering"** (Majors, Fong-Jones, Miranda; O'Reilly) — rozdziały 1–3 wystarczą: skąd się wzięło pojęcie, czym różni się od monitoringu, czemu high-cardinality ma znaczenie.
