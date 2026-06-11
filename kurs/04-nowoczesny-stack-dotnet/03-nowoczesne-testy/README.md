# Lekcja 03 — Nowoczesne testy: Testcontainers, testy architektury, contract testing

## Cel lekcji

Po tej lekcji będziesz testował aplikacje integracyjne tak, jak robi się to dziś: testy integracyjne na prawdziwej infrastrukturze w kontenerach (Testcontainers + WebApplicationFactory) zamiast lasu mocków, testy architektury pilnujące granic modułów z lekcji 02 oraz świadomość, czym jest contract testing i kiedy po niego sięgnąć.

## Dlaczego to ważne

Twoje systemy — Perfect Gym, Aerotunel, messaging-patterns — to aplikacje **integracyjne**: większość ich ryzyka mieszka na szwach (baza, broker, zewnętrzne API), nie w czystej logice. Tymczasem klasyczna szkoła testowania, którą pamiętasz, każe te szwy zamockować — czyli wyciąć z testów dokładnie to, co najczęściej się psuje. Na rozmowach na poziom architekta pytanie "jak testujecie?" jest pytaniem o architekturę: strategia testów zdradza, czy kandydat rozumie, gdzie w jego systemie jest ryzyko. Odpowiedź "unit testy z mockami, 80% coverage" w 2026 brzmi jak odpowiedź z 2012. Dodatkowo testy architektury to jedyny tani sposób, by decyzje z lekcji 02 (granice modułów) przeżyły dłużej niż entuzjazm po refaktoryzacji.

## Teoria

### Piramida testów a "trofeum" — gdzie naprawdę jest ryzyko

**Piramida testów** (Mike Cohn): dużo szybkich unit testów na dole, mniej testów integracyjnych w środku, garstka testów end-to-end na szczycie. Uzasadnienie z epoki: testy integracyjne były **wolne, drogie i flaky** (niestabilne — raz przechodzą, raz nie), bo wymagały współdzielonego środowiska testowego, "tej jednej bazy na serwerze", którą ktoś właśnie zepsuł.

Dlaczego ten kształt bywa dziś zły: dla aplikacji integracyjnej unit test z mockami repozytorium i brokera testuje głównie... konfigurację mocków. Klasyka, którą na pewno widziałeś: 400 zielonych unit testów, a system wywala się na pierwszym prawdziwym zapytaniu, bo mock zwracał posortowaną listę, a SQL Server — nie (albo: bo InMemory provider EF nie egzekwuje constraintów, transakcji ani collation). Mock testuje Twoje *wyobrażenie* o zależności, nie zależność.

Stąd alternatywny kształt — **testing trophy** (trofeum, Kent C. Dodds): najgrubsza warstwa to **testy integracyjne** — uruchamiają prawdziwy kod aplikacji przeciwko prawdziwym zależnościom, unit testy zostają dla czystej logiki (algorytmy, reguły domenowe, maszyny stanów — np. logika `CanCancel` z lekcji 02), a E2E nadal jest mało. Co zmieniło ekonomię i umożliwiło ten kształt? Docker: prawdziwy SQL Server albo RabbitMQ wstaje w kontenerze w kilka sekund, **per test run, jednorazowy, izolowany**. Testy integracyjne przestały być wolne i flaky — więc kara, która uzasadniała piramidę, w dużej mierze zniknęła.

Pragmatyczna rama (nie dogmat): kształt strategii testów ma odpowiadać **rozkładowi ryzyka w systemie**. Biblioteka algorytmiczna → piramida. Aplikacja CRUD-owo-integracyjna → trofeum.

### Testcontainers — prawdziwa infrastruktura w testach

**Testcontainers** to biblioteka (dla .NET: pakiety `Testcontainers.*`), która z poziomu kodu testu uruchamia kontenery Dockera, czeka aż będą gotowe, podaje connection string i sprząta po wszystkim. Test deklaruje swoją infrastrukturę tak, jak deklaruje swoje dane:

```csharp
using Testcontainers.MsSql;
using Testcontainers.RabbitMq;

public sealed class MessagingFixture : IAsyncLifetime
{
    public MsSqlContainer Sql { get; } = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public RabbitMqContainer Rabbit { get; } = new RabbitMqBuilder().Build();

    public async Task InitializeAsync() =>
        await Task.WhenAll(Sql.StartAsync(), Rabbit.StartAsync());

    public async Task DisposeAsync() =>
        await Task.WhenAll(Sql.DisposeAsync().AsTask(), Rabbit.DisposeAsync().AsTask());
}
```

I test, który w module 2 byłby niemożliwy bez mocków — a tu sprawdza **prawdziwe** zachowanie outboxa na prawdziwym SQL i prawdziwym brokerze:

```csharp
public class OutboxTests(MessagingFixture fx) : IClassFixture<MessagingFixture>
{
    [Fact]
    public async Task Confirmed_booking_event_survives_in_outbox_when_broker_is_down()
    {
        await using var db = CreateDbContext(fx.Sql.GetConnectionString());
        var handler = new ConfirmBooking.Handler(db, eventPublisher: null!);

        await handler.HandleAsync(new ConfirmBooking.Command(TestData.BookingId), default);

        var pending = await db.OutboxMessages.Where(m => m.ProcessedAt == null).ToListAsync();
        Assert.Single(pending);
        Assert.Equal(nameof(BookingConfirmed), pending[0].Type);
    }
}
```

Kluczowe szczegóły warsztatowe:

- **Fixture per kolekcja testów, nie per test.** Start kontenera SQL to kilka(naście) sekund — dzielisz kontener między testy (`IClassFixture` / `ICollectionFixture` w xUnit), a izolację zapewniasz tańszą walutą: osobna baza/schemat per test albo respawn danych.
- **Przypinaj wersje obrazów** (`:2022-latest` to minimum; lepiej konkretny tag) — test ma być deterministyczny.
- **CI:** Testcontainers działa wszędzie, gdzie jest Docker — GitHub Actions ma go out-of-the-box.
- Czego Testcontainers **nie zastąpi**: usług bez obrazu kontenera (np. zewnętrzne SaaS API — tam zostaje mock/stub na poziomie HTTP, np. WireMock.Net) oraz pełnych usług chmurowych (dla części istnieją emulatory w kontenerach — np. Azurite dla Azure Storage, emulator Cosmos DB — sprawdź dostępność dla swojego stacku).

### WebApplicationFactory — test integracyjny całego API

Testcontainers daje infrastrukturę; drugi klocek to uruchomienie **własnej aplikacji** w teście. `WebApplicationFactory<TEntryPoint>` (pakiet `Microsoft.AspNetCore.Mvc.Testing`) hostuje całe ASP.NET Core API **in-memory** — pełny pipeline (routing, middleware, DI, walidacja, serializacja), bez sieci i bez deploymentu — i daje `HttpClient` do strzelania w nie:

```csharp
public sealed class ApiFactory(MessagingFixture fx) : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder) =>
        builder.ConfigureAppConfiguration(cfg => cfg.AddInMemoryCollection(new Dictionary<string, string?>
        {
            ["ConnectionStrings:Bookings"] = fx.Sql.GetConnectionString(),
            ["ConnectionStrings:Messaging"] = fx.Rabbit.GetConnectionString(),
        }));
}

[Fact]
public async Task Cancelling_unknown_booking_returns_404()
{
    var client = _factory.CreateClient();

    var response = await client.PostAsJsonAsync(
        $"/bookings/{Guid.NewGuid()}/cancel", new { reason = "test" });

    Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
}
```

Połączenie **WebApplicationFactory + Testcontainers** to dzisiejszy standard testu integracyjnego w .NET: prawdziwe API + prawdziwa baza + prawdziwy broker, a jedyne, co podmieniasz, to konfiguracja (connection stringi wskazują kontenery). Zauważ, jak dobrze to współgra z vertical slices z lekcji 02: **naturalną jednostką testowania jest slice testowany przez HTTP** — od requestu po wiersz w bazie. Nie potrzebujesz interfejsu repozytorium "dla testowalności", bo testujesz przez prawdziwe drzwi.

Drobiazg techniczny: przy top-level statements klasa `Program` jest niejawna i internal — żeby `WebApplicationFactory<Program>` ją widziała, dodaj w API `public partial class Program {}` albo `[assembly: InternalsVisibleTo(...)]`.

### Testy architektury — reguły z lekcji 02 jako kod

Problem: w lekcji 02 ustaliłeś granice modułów. Kompilator pilnuje braku referencji projektowych — ale nie upilnuje konwencji ("handlery mają być internal i sealed") ani nie zadziała, gdy ktoś *doda* referencję projektową. **Testy architektury** (styl wywodzący się z javowego ArchUnit; w .NET: **NetArchTest** — prosty, fluent — albo **ArchUnitNET** — port pełnego ArchUnit) wyrażają reguły architektoniczne jako zwykłe testy jednostkowe nad metadanymi assembly:

```csharp
using NetArchTest.Rules;

public class ModuleBoundaryTests
{
    [Fact]
    public void Payments_module_does_not_depend_on_Bookings_internals()
    {
        var result = Types.InAssembly(typeof(Payments.AssemblyMarker).Assembly)
            .ShouldNot()
            .HaveDependencyOn("MessagingPatterns.Modules.Bookings")   // implementacja
            .GetResult();                                              // Contracts są OK

        Assert.True(result.IsSuccessful,
            "Payments może zależeć tylko od Bookings.Contracts. Naruszenia: " +
            string.Join(", ", result.FailingTypeNames ?? []));
    }

    [Fact]
    public void Handlers_are_internal_and_sealed()
    {
        var result = Types.InCurrentDomain()
            .That().HaveNameEndingWith("Handler")
            .Should().NotBePublic().And().BeSealed()
            .GetResult();

        Assert.True(result.IsSuccessful);
    }
}
```

To zmienia status reguł architektonicznych: z dokumentu, który się dezaktualizuje, w **test, który blokuje merge**. Architektura przestaje być opinią — staje się wymaganiem wykonywalnym. Dla architekta to też narzędzie skali: nie musisz osobiście pilnować granic na każdym code review w trzech zespołach.

### Contract testing — w zarysie

Problem: konsument i producent API to osobne deploymenty (Twój moduł 3: API na Container Apps, konsumenci gdzie indziej). Producent zmienia pole `price` na `pricing.amount`, jego testy są zielone, testy konsumenta (na mockach producenta!) też — a produkcja leży. Mocki konsumenta **dryfują** względem rzeczywistego API. Test E2E by to wykrył, ale E2E całych ekosystemów są drogie i flaky.

**Contract testing** (najbardziej znane narzędzie: **Pact**) odwraca układ: konsument *nagrywa* swoje oczekiwania ("na GET /bookings/{id} oczekuję pól id, price") jako **kontrakt** — plik trafia do brokera kontraktów, a w pipeline **producenta** test odtwarza wszystkie kontrakty konsumentów przeciwko prawdziwemu API. Producent fizycznie nie może zmergować zmiany łamiącej konsumenta, o którym nawet nie wie. To "consumer-driven contracts": konsumenci definiują, co z API naprawdę jest używane.

Uczciwa rama: contract testing ma sens, gdy **wiele zespołów rozwija komunikujące się usługi niezależnie**. W monolicie modularnym kontrakty pilnuje kompilator + testy architektury; w systemie jednego zespołu wystarczą testy integracyjne. Na tym etapie kursu wystarczy, że umiesz problem i rozwiązanie **nazwać** — wdrożenie Pact wraca jako opcja przy Capstone.

### Co testować na jakim poziomie — pragmatyczne reguły

1. **Czysta logika** (reguły domenowe, maszyny stanów, obliczenia) → unit testy, bez mocków, bo nie ma czego mockować.
2. **Slice / przepływ przez moduł** (API → handler → baza → event) → test integracyjny: WebApplicationFactory + Testcontainers. To Twoja domyślna, najgrubsza warstwa.
3. **Reguły architektury** (granice modułów, konwencje) → NetArchTest, kilka testów, raz napisane.
4. **Szwy między niezależnie deployowanymi usługami** → contract testing (gdy zespołów/usług jest dość, by bolało).
5. **Krytyczne ścieżki biznesowe end-to-end** → 2–5 testów E2E, nie 200.

Anty-reguły: nie testuj frameworka (serializacja JSON "czy działa"), nie mockuj tego, co możesz mieć naprawdę, nie mierz jakości coverage liczonym na unit testach getterów.

### Kiedy tego NIE używać

- **Testcontainers nie wszędzie:** na maszynach bez Dockera (część korpo-laptopów) i przy bardzo ciasnym budżecie czasu CI warto mieć podział: szybkie unit testy na PR, integracyjne na merge. Nie dogmatyzuj — wolny feedback loop też zabija jakość.
- **Unit testy z mockami nadal mają miejsce** — tam, gdzie testujesz *protokół współpracy* ("po błędzie wołamy kompensację, dokładnie raz") albo zależność jest niekontenerowalna. Problemem jest mock jako *domyślna* strategia, nie mock jako narzędzie.
- **Testy architektury przesadzone** (50 reguł o nazewnictwie) stają się biurokracją. Pilnuj testami rzeczy drogich w naprawie (granice, zależności), nie stylu — od stylu jest analizator/formatter.

## Praktyka

- [ ] **Ćwiczenie główne: dopisz testy do repo messaging-patterns zrefaktoryzowanego w lekcji 02.** (a) Fixture z Testcontainers: SQL Server (lub Postgres — co masz w repo) + RabbitMQ; (b) minimum 3 testy integracyjne przez WebApplicationFactory, w tym jeden sprawdzający zachowanie z modułu 2 na prawdziwej infrastrukturze (np. duplikat wiadomości → idempotent consumer nie tworzy drugiego rekordu); (c) uruchom całość w CI (GitHub Actions).
- [ ] **Dodaj test architektury:** moduł A nie referencuje wnętrza modułu B (tylko Contracts) + jedna reguła konwencji. Zweryfikuj, że test naprawdę działa: wprowadź naruszenie, zobacz czerwień, cofnij.
- [ ] Zmierz czas pełnego przebiegu testów lokalnie i w CI — zanotuj; jeśli > ~2–3 min, zastosuj współdzielenie fixture i zanotuj zysk.
- [ ] Skreśl z dotychczasowych testów repo te, które po dodaniu integracyjnych nic już nie wnoszą (mocki repozytoriów itp.) — usuwanie testów to też praca inżynierska.
- [ ] Przeczytaj dokumentację Pact (sama strona "how it works") i napisz 5-zdaniowe podsumowanie własnymi słowami do repo nauki — kiedy w Twoich projektach z przeszłości by się przydał.

## Artefakt

1. **Repo messaging-patterns z kompletem testów:** integracyjne na Testcontainers + WebApplicationFactory, test architektury, zielony pipeline CI — to domyka ćwiczenie modułowe z planu ("vertical slices + pełne testy").
2. **Notatka "strategia testów"** w repo (sekcja README): co testujemy na jakim poziomie i dlaczego — 10–15 linii, według reguł z tej lekcji.

## Definition of Done

- [ ] `dotnet test` na czystej maszynie z Dockerem przechodzi bez żadnej ręcznej konfiguracji środowiska.
- [ ] Co najmniej jeden test łapie realne zachowanie infrastruktury, którego mock by nie złapał (np. constraint/unikalność w bazie przy duplikacie wiadomości).
- [ ] Test architektury istnieje i udowodniłeś, że wykrywa naruszenie.
- [ ] CI uruchamia testy integracyjne na każdy push/PR.
- [ ] Umiesz bez notatek wyjaśnić: piramida vs trofeum (i od czego zależy wybór), co daje Testcontainers ponad InMemory provider, czym jest contract testing i kiedy go NIE wdrażać.

## Materiały

1. [Dokumentacja Testcontainers for .NET](https://dotnet.testcontainers.org/) — krótka; przeczytaj całą sekcję o fixtures i wsparciu xUnit.
2. Andrew Lock, seria o testach integracyjnych z `WebApplicationFactory` na [andrewlock.net](https://andrewlock.net) — najlepsze omówienie detali (podmiana konfiguracji, `Program` przy top-level statements).
