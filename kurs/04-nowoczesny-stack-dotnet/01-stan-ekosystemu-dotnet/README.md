# Lekcja 01 — Stan ekosystemu .NET: mapa dla wracającego z dłuższej podróży

## Cel lekcji

Po tej lekcji będziesz miał aktualną mapę ekosystemu .NET: jak działa rytm wydań i wsparcia, które zmiany językowe i platformowe mają znaczenie architektoniczne (a które są kosmetyką), czym jest .NET Aspire i jaki problem rozwiązuje — oraz zweryfikowany, zapisany w repo stan "co jest aktualne na dziś".

## Dlaczego to ważne

Na rozmowie na poziom architekta pytanie "co nowego w .NET Cię ostatnio zainteresowało?" pada niemal zawsze — i nie służy sprawdzeniu encyklopedycznej wiedzy, tylko temu, **czy kandydat śledzi platformę, na której projektuje**. Odpowiedź "records i top-level statements" (rzeczy sprzed wielu lat) brzmi gorzej niż szczere "nadrabiam — i wiem dokładnie co". Druga stawka jest projektowa: decyzje typu "minimal APIs czy kontrolery", "czy wchodzimy w Aspire" podejmuje się raz na projekt i żyje z nimi latami. Architekt, który nie zna opcji, nie podejmuje decyzji — dziedziczy domyślne.

## Teoria

### Unifikacja: jedna platforma zamiast trzech

Stan, który pamiętasz: .NET Framework (Windows-only, WebForms/MVC, `web.config`), do tego Mono na inne platformy, potem .NET Core jako "ten nowy, jeszcze niepełny". Ten rozjazd skończył się wraz z **.NET 5 (2020)** — od tego momentu istnieje **jedna platforma** o nazwie po prostu ".NET", cross-platform, open source, rozwijana w rocznym rytmie:

- **co listopad wychodzi nowa wersja główna** (.NET 5, 6, 7, 8…),
- **wersje parzyste to LTS** (Long Term Support — 3 lata wsparcia), nieparzyste to STS (Standard Term Support — 18 miesięcy),
- .NET Framework 4.8 żyje wiecznie jako komponent Windows, ale **nie dostaje nowych funkcji** — to platforma legacy do utrzymania, nie do nowych projektów.

Konsekwencja architektoniczna: pytanie "którą wersję wybrać" ma dziś prostą heurystykę — **nowy projekt = bieżący LTS**, chyba że projekt jest krótkożyciowy albo potrzebuje funkcji ze świeżego STS. Plan tego kursu pisany jest w czerwcu 2026 — która wersja jest *aktualnie* bieżącym LTS, zweryfikujesz sam w pierwszym ćwiczeniu (to celowe: nawyk sprawdzania w [oficjalnej polityce wsparcia](https://dotnet.microsoft.com/platform/support/policy/dotnet-core) zamiast cytowania z pamięci to część kompetencji).

### Minimal APIs vs kontrolery

W klasycznym ASP.NET MVC, które pamiętasz, każdy endpoint to klasa kontrolera + routing w konfiguracji + ceremonia. **Minimal APIs** (od .NET 6) to alternatywny model: endpoint definiujesz jako lambdę wpiętą bezpośrednio w pipeline, bez kontrolerów i atrybutów:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IBookingService, BookingService>();

var app = builder.Build();

app.MapGet("/bookings/{id:guid}", async (Guid id, IBookingService svc) =>
    await svc.GetAsync(id) is { } booking
        ? Results.Ok(booking)
        : Results.NotFound());

app.MapPost("/bookings", async (CreateBooking cmd, IBookingService svc) =>
{
    var id = await svc.CreateAsync(cmd);
    return Results.Created($"/bookings/{id}", new { id });
});

app.Run();
```

Zwróć uwagę: parametry lambdy są **automatycznie bindowane** — `Guid id` z route, `CreateBooking cmd` z body (deserializacja JSON), `IBookingService svc` z kontenera DI. Zero atrybutów `[FromBody]`/`[FromServices]` w typowych przypadkach.

Kiedy które — uczciwie:

| | Minimal APIs | Kontrolery |
|---|---|---|
| Mała/średnia usługa, mikroserwis, vertical slices | ✅ naturalny wybór | działa, ale ceremonia |
| Duże API z rozbudowanymi filtrami, wersjonowaniem, konwencjami | da się, ale rośnie własny framework | ✅ dojrzały ekosystem |
| Zespół przyzwyczajony do MVC | krzywa zmiany nawyków | ✅ zero tarcia |
| Wydajność / startup | minimalnie lżejsze, wspierają AOT | cięższe |

To nie jest wybór religijny — od .NET 7–8 oba modele mają filtry, walidację i OpenAPI. Kierunek rozwoju platformy (i przykładów w dokumentacji) to jednak minimal APIs; w tym kursie używamy ich domyślnie, bo świetnie pasują do vertical slices (lekcja 02): **jeden endpoint = jeden feature = jeden plik**.

### Zmiany językowe istotne architektonicznie

C# dostaje nową wersję co rok i większość zmian to wygoda. Cztery rzeczy mają jednak wagę większą niż cukier składniowy:

**1. Top-level statements i implicit usings** — `Program.cs` bez klasy `Program`, `Main` i 20 usingów (widać to w przykładzie wyżej). Kosmetyka? Prawie — ale to dlatego współczesne przykłady i szablony wyglądają "obco" dla kogoś z ery `Global.asax`. Trzeba to po prostu znać, żeby czytać dzisiejszy kod.

**2. Nullable reference types (NRT)** — od C# 8, w nowych szablonach **włączone domyślnie** (`<Nullable>enable</Nullable>`). Typy referencyjne są nie-nullowalne, chyba że jawnie napiszesz `string?`. Kompilator analizuje przepływ i ostrzega przed dereferencją potencjalnego nulla. Architektonicznie to zmiana kontraktów: sygnatura `Customer? FindCustomer(string email)` mówi wprost "może nie znaleźć" — coś, co przez 20 lat było wiedzą plemienną albo komentarzem. Warunek: NRT działa tylko, gdy traktujesz ostrzeżenia serio (`<WarningsAsErrors>nullable</WarningsAsErrors>` w nowym kodzie). Wyłączone albo ignorowane — nie daje nic.

**3. Records** — typy o semantyce wartości (równość po zawartości, nie po referencji), domyślnie niemutowalne, z niemal zerową ceremonią:

```csharp
public record CreateBooking(Guid CustomerId, DateOnly Date, TimeSlot Slot);

// niemutowalna "zmiana" przez kopię:
var moved = booking with { Slot = TimeSlot.Evening };
```

Architektonicznie: records to naturalny budulec **kontraktów** — komend, eventów, DTO, value objects z DDD. Niemutowalność eliminuje całą klasę błędów ("ktoś po drodze zmodyfikował obiekt"), którą znasz z wielowarstwowych aplikacji przekazujących encje przez referencję.

**4. Pattern matching** — rozbudowywany od C# 7 do dziś: `switch` jako wyrażenie, wzorce właściwości, wzorce list. Zmienia styl pisania logiki gałęziowej z drabinek if-else na deklaratywne dopasowania:

```csharp
var fee = booking switch
{
    { Status: BookingStatus.Cancelled } => 0m,
    { Slot: TimeSlot.Peak, Participants: > 10 } => basePrice * 1.5m,
    { Customer.IsVip: true } => basePrice * 0.8m,
    _ => basePrice
};
```

Wzorzec `is { } x` z przykładu minimal API to też pattern matching: "nie-null i przypisz do `x`".

### Generic host i DI jako standard

W klasycznym ASP.NET kontener DI był opcjonalnym dodatkiem (Autofac, Ninject — każdy projekt inny). Dziś **wbudowany kontener DI + generic host** (`Microsoft.Extensions.Hosting`) to kręgosłup *każdej* aplikacji .NET — webowej, workera, konsolówki. Jeden model: rejestrujesz usługi w `builder.Services`, konfigurację czytasz przez `IConfiguration`/`IOptions<T>`, logujesz przez `ILogger<T>`, długotrwałe procesy to `IHostedService`/`BackgroundService`. Konsekwencja: biblioteki i frameworki integrują się przez te same abstrakcje — to de facto **system operacyjny aplikacji .NET**. Jeśli ostatnio rejestrowałeś zależności w `Global.asax` — to jest największa pojedyncza zmiana nawyku do przyswojenia.

### .NET Aspire — orkiestracja lokalnego developmentu

Problem, który znasz z każdego systemu rozproszonego (i z repo messaging-patterns z modułu 2): aplikacja to API + worker + Postgres + RabbitMQ + Redis. Lokalnie spinasz to ręcznie pisanym `docker-compose.yml`, connection stringami w pięciu miejscach i plemienną wiedzą "najpierw odpal broker, potem API". Każdy nowy członek zespołu traci na tym dzień.

**.NET Aspire** to odpowiedź Microsoftu: **model aplikacji rozproszonej zapisany w C#** zamiast w YAML-u. Projekt AppHost deklaruje komponenty i zależności:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var rabbit = builder.AddRabbitMQ("messaging");
var db = builder.AddPostgres("postgres").AddDatabase("bookings");

var api = builder.AddProject<Projects.Bookings_Api>("api")
    .WithReference(db)
    .WithReference(rabbit);

builder.AddProject<Projects.Bookings_Worker>("worker")
    .WithReference(rabbit)
    .WaitFor(api);

builder.Build().Run();
```

Co dostajesz w zamian:

- **`dotnet run` na AppHost stawia wszystko** — kontenery, projekty, w dobrej kolejności;
- **service discovery** — `WithReference(db)` wstrzykuje connection string do API automatycznie; kod prosi o połączenie po nazwie ("bookings"), nie zna hostów ani portów;
- **dashboard** — strona z logami, distributed tracingiem (OpenTelemetry) i metrykami wszystkich komponentów, bez konfiguracji — to lokalna wersja tego, co w module 2 (lekcja 07) budowałeś ręcznie;
- **service defaults** — gotowy pakiet telemetrii, health checków i resilience dla każdego projektu.

Uczciwa rama: Aspire celuje przede wszystkim w **developer experience lokalnego developmentu i orkiestrację**, nie zastępuje Kubernetesa/Container Apps na produkcji (choć potrafi generować artefakty wdrożeniowe). Projekt rozwijał się dynamicznie i jego zakres oraz model wydawniczy zmieniały się z wersji na wersję — **status na dziś (zakres, wersjonowanie, rekomendacje produkcyjne) zweryfikujesz w ćwiczeniu 1** w oficjalnej dokumentacji, zamiast polegać na stanie zamrożonym w tej lekcji.

### Kiedy tego wszystkiego NIE używać

- **Nie przepisuj działających aplikacji "na nowe", bo nowe.** Migracja z .NET Framework to decyzja biznesowa (koszty utrzymania, bezpieczeństwo, rekrutacja) — nie estetyczna. Działający monolit na 4.8, który nie wymaga zmian, może sobie żyć.
- **NRT w starym kodzie:** włączenie w dużym legacy generuje tysiące ostrzeżeń — włącza się je per plik/projekt, stopniowo, albo wcale. Wartość jest w nowym kodzie.
- **Aspire w solo-projekcie z jednym API i jedną bazą** to przerost — zwykły `docker compose` wystarczy. Aspire zarabia na siebie od ~3+ komponentów i więcej niż jednej osoby w zespole.

## Praktyka

- [ ] **Ćwiczenie 1 (obowiązkowo pierwsze): weryfikacja stanu ekosystemu.** W oficjalnych źródłach (polityka wsparcia .NET, release notes, dokumentacja Aspire) sprawdź i zapisz w repo nauki notatkę `stan-ekosystemu-2026-06.md`: (a) bieżąca wersja LTS .NET i daty końca jej wsparcia, (b) bieżąca wersja STS i co istotnego wniosła, (c) aktualna wersja C# i 2–3 nowości z ostatniego roku, (d) status .NET Aspire: aktualna wersja, model wydawniczy, czy/jak Microsoft pozycjonuje go produkcyjnie. Ta notatka to Twoje źródło prawdy na resztę modułu.
- [ ] Utwórz nowy projekt `dotnet new web` na bieżącym LTS i przeczytaj **każdą linię** wygenerowanego szablonu, aż nic w nim nie będzie magiczne (top-level statements, builder, brak `Startup.cs`).
- [ ] Przepisz jeden kontroler z dowolnego swojego starszego projektu (lub wymyślony endpoint CRUD) na minimal API z bindowaniem parametrów i `Results.*`.
- [ ] Weź jedną klasę DTO/encję ze starego kodu i przepisz na record z NRT; zanotuj, ile niejawnych założeń o nullach wyszło na jaw.
- [ ] Przejdź quickstart .NET Aspire z oficjalnej dokumentacji (API + Redis/Postgres) i obejrzyj dashboard — porównaj z tym, co w module 2 konfigurowałeś ręcznie pod OpenTelemetry.

## Artefakt

1. **Notatka `stan-ekosystemu-2026-06.md`** w repo nauki — zweryfikowany stan platformy z linkami do źródeł.
2. **Szkic posta modułowego** ("Co realnie zmieniło się w .NET…") — na razie sama struktura: lista 5–7 zmian, które z perspektywy kogoś od WebForms są naprawdę istotne, po jednym zdaniu każda. Mięso dopiszesz po lekcjach 02–04.

## Definition of Done

- [ ] Notatka o stanie ekosystemu istnieje, ma daty i linki do oficjalnych źródeł (nie blogów).
- [ ] Umiesz bez notatek powiedzieć: jaki jest rytm wydań .NET, która wersja jest bieżącym LTS i do kiedy wspierana, czym jest Aspire w jednym zdaniu i kiedy go NIE brać.
- [ ] Masz w repo działający przykład minimal API napisany własnoręcznie (nie wklejony).
- [ ] Rozumiesz każdą linię współczesnego szablonu projektu — test: potrafisz wyjaśnić koledze, gdzie "podziało się" `Global.asax`/`Startup.cs` i co je zastąpiło.
- [ ] Szkic posta ma kompletną listę punktów.

## Materiały

1. [Oficjalna polityka wsparcia .NET](https://dotnet.microsoft.com/platform/support/policy/dotnet-core) + sekcje "What's new in .NET / What's new in C#" na Microsoft Learn — źródło pierwotne do ćwiczenia 1.
2. [Dokumentacja .NET Aspire](https://learn.microsoft.com/dotnet/aspire/) — overview + quickstart; czytaj świeżą wersję, nie artykuły z okresu wczesnych preview.
