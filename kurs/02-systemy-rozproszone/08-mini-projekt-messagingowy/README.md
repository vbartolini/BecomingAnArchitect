# Lekcja 08 — Mini-projekt: repo "messaging-patterns"

## Cel lekcji

Spiąć wszystkie wzorce modułu w jeden działający projekt: przepływ **zamówienie → płatność → powiadomienie** na lokalnym brokerze, z outboxem, idempotent consumerem, retry + DLQ, sagą z kompensacją i tracingiem — oraz z symulacjami awarii, które udowadniają, że to działa. To nie jest ćwiczenie "na boku": **ta wizytówka wejdzie do Featured na LinkedIn, a potem stanie się bazą Capstone** (moduł 7) i aplikacji wdrażanej na Azure (moduł 3).

## Dlaczego to ważne

Wiedza z lekcji 01–07 jest na razie rozproszona po małych ćwiczeniach. Rozmowa rekrutacyjna i prawdziwy projekt wymagają czegoś więcej: umiejętności **złożenia** wzorców w spójny system i obrony każdej decyzji. Repo publiczne robi dodatkową robotę: jest dowodem zamiast deklaracji ("tu jest mój outbox z testem na crash relaya" bije każde "znam outbox pattern" na CV), daje materiał na posty i pokazuje Twój znak firmowy — sekcję "czego świadomie NIE ma". Dla kogoś z 20+ latami doświadczenia, ale bez publicznego portfolio, to najtańszy sposób uwidocznienia tego, co i tak umiesz.

## Wymagania projektu

### Scenariusz biznesowy

Minimalny sklep/rezerwacje (celowo nudny — bohaterem są wzorce, nie domena):

```
[OrderService]  --OrderPlaced-->  [PaymentService]  --PaymentCompleted/Failed-->  [OrderService]
                                                                                       |
                                                  [NotificationService]  <--OrderConfirmed/OrderCancelled--
```

1. **OrderService** — REST API: `POST /orders` tworzy zamówienie (`Pending`), publikuje `OrderPlaced` **przez outbox**; nasłuchuje wyników płatności i potwierdza (`Confirmed`) lub anuluje (`Cancelled`) zamówienie. Tu mieszka logika sagi.
2. **PaymentService** — konsumuje `OrderPlaced`, symuluje pobranie płatności (konfigurowalnie: sukces / odmowa / wyjątek / opóźnienie — to Twoje pokrętła do symulacji awarii), publikuje `PaymentCompleted` lub `PaymentFailed`. Idempotent consumer z tabelą dedup.
3. **NotificationService** — konsumuje `OrderConfirmed` / `OrderCancelled`, "wysyła e-mail" (log). Celowo ostatni krok sagi — krok niekompensowalny (lekcja 03).

### Broker

Do wyboru (decyzję zapisz w README jako mini-ADR):

- **RabbitMQ w Dockerze** — rekomendacja na start: najprostszy lokalnie, management UI z podglądem kolejek, naturalne DLQ przez dead-letter exchange.
- **Azure Service Bus emulator** (oficjalny kontener Dockera) — bliżej modułu 3 (to samo SDK `Azure.Messaging.ServiceBus` pojedzie potem na prawdziwy Service Bus), wbudowane DLQ; emulator ma ograniczenia, sprawdź aktualny stan przed decyzją.

Całość stawiana jednym `docker compose up` (broker + SQL/Postgres + Jaeger lub Aspire Dashboard); usługi przez `dotnet run` albo też w compose.

### Wymagane wzorce (mapa: wymaganie → lekcja)

| # | Wymaganie | Lekcja |
|---|---|---|
| 1 | Outbox w OrderService i PaymentService (własna implementacja: tabela + relay; bez MassTransit — celem jest zrozumienie bebechów) | 02 |
| 2 | Idempotent consumer w każdej usłudze konsumującej (tabela `ProcessedMessages`, unique constraint jako gwarancja) | 01 |
| 3 | Saga zamówienia z kompensacją: `PaymentFailed` → anulowanie zamówienia; wybierz orchestration albo choreography i **uzasadnij w ADR** | 03 |
| 4 | Retry z backoff+jitter dla błędów przejściowych; klasyfikacja błędów; DLQ dla trwałych/wyczerpanych + runbook z lekcji 05 w repo | 05, 06 |
| 5 | Klucz partycji / sesji: zdarzenia jednego zamówienia przetwarzane w kolejności (RabbitMQ: consistent-hash lub jedna kolejka per typ + wersjonowanie stanu; Service Bus: sessions po `OrderId`) — wybór i ograniczenia opisz | 05 |
| 6 | OpenTelemetry: trace end-to-end przez 3 usługi i broker (propagacja `traceparent` w nagłówkach wiadomości); metryki RED + głębokość DLQ | 07 |

### Wymagane symulacje awarii (dowód, że system działa)

Każdy scenariusz = skrypt/test + sekcja w README z opisem: co wstrzyknięto → co system zrobił → zrzut/log na dowód.

1. **Duplikaty** — opublikuj `OrderPlaced` dwukrotnie z tym samym `MessageId` (oraz: ubij relay między publish a oznaczeniem `ProcessedAt`); dowód: jedna płatność, drugi odbiór zalogowany jako zignorowany duplikat.
2. **Out-of-order** — dostarcz `OrderCancelled` przed `OrderPlaced` (lub `PaymentCompleted` dla nieznanego zamówienia); dowód: system nie wywraca się i dochodzi do poprawnego stanu końcowego (wersjonowanie/tombstone z lekcji 05).
3. **Poison message** — wstrzyknij zdeformowany payload; dowód: brak retry (błąd trwały!), wiadomość w DLQ z powodem, alert/metryka DLQ > 0, redrive po "naprawie" zgodnie z runbookiem.
4. **Awaria zależności** — PaymentService rzuca 100% błędów przejściowych przez 30 s; dowód: retry z backoff w logach, brak retry storm, saga kończy się poprawnie po ustąpieniu awarii; bonus: timeout → kompensacja sagi.
5. **Crash w pół sagi** — ubij OrderService po pobraniu płatności, przed potwierdzeniem; dowód: po restarcie saga dochodzi do końca (stan sagi utrwalony, zdarzenia at-least-once).

### README repo (wizytówka — całe repo po angielsku)

Architektura z diagramem (C4 container — moduł 1 w praktyce), tabela wzorców z linkami do kodu, "How to run" (docker compose + krok po kroku), sekcja **Failure simulations** ze zrzutami, 2–4 mini-ADR-y (broker, orchestration vs choreography, poller vs CDC, klucz partycji), oraz sekcja **"What's intentionally NOT here"** — np. brak autoryzacji, brak prawdziwej bramki płatności, jedna baza per usługa zamiast osobnych serwerów — z uzasadnieniami. Ta ostatnia sekcja to Twój znak firmowy z posta o DayChunks.

Język: to artefakt publiczny, więc zgodnie z polityką językową kursu **po angielsku jest cały folder `messaging-patterns/`** — nie tylko README, ale też ADR-y, runbook DLQ, komentarze w kodzie i komunikaty commitów go dotyczących. Notatki robocze po polsku zostają poza wizytówką (folder lekcji, `notes/`, `exercises/`).

## Plan tygodniowy

### Tydzień 4 modułu — szkielet i fundament niezawodności

- [ ] Zacznij pracę w folderze-wizytówce `messaging-patterns/` (już istnieje w repo nauki, ze stubem README). Commity przyrostowe od pierwszego dnia — historia tego folderu też jest treścią portfolio (`git log -- messaging-patterns/`), więc komunikaty po angielsku (np. `code(mp): add transactional outbox relay`). Upewnij się, że repo nauki jest (albo przed publikacją będzie) publiczne.
- [ ] `docker-compose.yml`: broker + baza + Jaeger/Aspire Dashboard; trzy projekty usług + projekt `Contracts` (definicje zdarzeń) w jednym solution.
- [ ] Happy path bez wzorców: `POST /orders` → `OrderPlaced` → płatność → `PaymentCompleted` → `Confirmed` → "e-mail". Najpierw ma działać, potem być niezawodne.
- [ ] Dołóż outbox + relay w OrderService i PaymentService (przenieś i uogólnij kod z lekcji 02).
- [ ] Dołóż idempotent consumery (kod z lekcji 01).
- [ ] Symulacja 1 (duplikaty) działa i jest opisana w README.

### Tydzień 5 modułu — saga, awarie, observability, szlif

- [ ] Saga z kompensacją: ścieżka `PaymentFailed` → `OrderCancelled` → powiadomienie o anulowaniu; utrwalony stan sagi; ADR orchestration vs choreography.
- [ ] Retry + klasyfikacja błędów + DLQ; runbook DLQ w repo; symulacje 3 i 4.
- [ ] Kolejność per zamówienie (wymaganie 5) + symulacja 2; symulacja 5 (crash w pół sagi).
- [ ] OpenTelemetry end-to-end; zrzut wodospadu trace'a całej sagi do README.
- [ ] README na pełny połysk (diagram, ADR-y, "What's intentionally NOT here").
- [ ] Publikacja: link do folderu `messaging-patterns/` w sekcji **Featured** na LinkedIn (z własnym opisem; GitHub renderuje README folderu) + post podsumowujący moduł (np. "Zbudowałem od zera outbox, sagę i DLQ — żeby w końcu nazwać to, co produkcja robiła ze mną od lat" — z linkiem).

**Jeśli zabraknie czasu** — tnij w tej kolejności (od najmniej krytycznego): symulacja 5 → wymaganie 5 (kolejność; zostaw notatkę "known limitation" w README) → metryki (zostaw sam tracing). **Nie tnij**: outboxa, idempotencji, DLQ, sagi — to rdzeń Definition of Done modułu.

## Praktyka

Checkboxy planu tygodniowego powyżej **są** praktyką tej lekcji. Dodatkowo:

- [ ] Po ukończeniu przeprowadź "rozmowę rekrutacyjną z samym sobą" (albo z Claude jako interviewerem): 15 minut opowiadania o architekturze repo z pytaniami wgłębnymi ("a co jak relay padnie?", "czemu nie 2PC?", "czemu ten klucz partycji?"). Nagraj się — to trening pod moduł 6.
- [ ] Zaktualizuj `progress.md` i odhacz artefakty modułu w `architect-track-plan.md`.

## Artefakt

Folder-wizytówka **`messaging-patterns/`** (publicznie widoczny w repo nauki): działający system 3 usług na lokalnym brokerze, 6 wzorców, 5 udokumentowanych symulacji awarii, README z diagramem i ADR-ami — wszystko po angielsku. Podlinkowany w Featured na LinkedIn, z postem ogłaszającym. To jest **artefakt flagowy całego modułu** — i fundament, na którym moduł 3 postawi wdrożenie na Azure, a Capstone rozbuduje domenę.

## Definition of Done

- [ ] `git clone` + `docker compose up` + `dotnet run` (lub samo compose) wystarczą, żeby obcy człowiek uruchomił system w 10 minut, podążając za README.
- [ ] Wszystkie 5 symulacji awarii przechodzi i jest udokumentowane dowodami w README.
- [ ] Każda decyzja architektoniczna w repo ma uzasadnienie (ADR lub sekcja README) — test: na dowolne "dlaczego tak?" odpowiadasz konkretem, nie "bo tak było w tutorialu".
- [ ] Trace całej sagi (od `POST /orders` do "e-maila") widoczny jako jedno drzewo w UI tracingu.
- [ ] Wizytówka podlinkowana w Featured, post opublikowany.
- [ ] Umiesz opowiedzieć architekturę repo w 5 minut bez patrzenia w kod — bo to repo jest teraz Twoją najlepszą odpowiedzią na pytanie "zaprojektuj niezawodny przepływ komunikatów" (czyli na Definition of Done całego modułu).

## Materiały

1. **microservices.io — [katalog wzorców](https://microservices.io/patterns/)** — jako checklista nazewnictwa: upewnij się, że README używa kanonicznych nazw wzorców (transactional outbox, idempotent consumer, saga…), bo po tych frazach repo będzie znajdowane i oceniane.
2. **Dokumentacja wybranego brokera** (RabbitMQ: reliability guide + dead lettering; albo Azure Service Bus: sessions + dead-letter queues) — czytana punktowo, pod konkretne wymagania.
