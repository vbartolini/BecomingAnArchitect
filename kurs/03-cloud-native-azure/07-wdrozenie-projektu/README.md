# Lekcja 07 — Projekt: wdrożenie aplikacji messagingowej na Azure

## Cel lekcji

Spiąć cały moduł w jedno działające wdrożenie: aplikacja messagingowa z modułu 2 (producer API → broker → consumer → baza) działa na Azure w architekturze **Container Apps + Service Bus + Azure SQL**, wdrożona **w 100% z Bicep**, z **managed identities zamiast connection stringów** i **pełną observability** (Application Insights + distributed tracing przez OpenTelemetry). Po tej lekcji masz w portfolio system, o którym możesz powiedzieć: "od pustej subskrypcji do działającego, obserwowalnego systemu jednym poleceniem".

## Dlaczego to ważne

To jest moment, w którym wiedza z lekcji 01–06 zamienia się w dowód. Na rozmowach architektonicznych najsilniejsze odpowiedzi zaczynają się od "w moim projekcie…" — ten projekt daje ci takie otwarcie dla większości tematów modułu: wyboru compute, semantyki Service Busa, tożsamości bez sekretów, IaC i kosztów. Diagram tego wdrożenia idzie do Featured na LinkedIn, repo staje się wizytówką, a sam projekt — fundamentem capstone'a (moduł 7). Lekcje teoretyczne można podrobić oczytaniem; działającego, odtwarzalnego wdrożenia — nie.

## Teoria

Nowej teorii jest tu celowo niewiele — dwa tematy domykające.

### Observability w Azure: Application Insights + OpenTelemetry

Z modułu 2 znasz pojęcia: distributed tracing (śledzenie żądania przez wiele usług), korelacja (wspólny identyfikator spinający logi/spany jednego przepływu), metryki. W Azure składają się one tak:

- **Azure Monitor** — parasolowa platforma telemetrii; **Log Analytics workspace** — baza, w której telemetria ląduje (i za której **ingest płacisz** — lekcja 06); **Application Insights** — aplikacyjna nakładka APM: mapa zależności, wyszukiwarka trace'ów end-to-end, analiza błędów i latencji, język zapytań **KQL** (Kusto Query Language — SQL-opodobny język do przeszukiwania telemetrii; nauczysz się go w praktyce, pisząc 3–4 zapytania w tym projekcie).
- **OpenTelemetry (OTel)** — otwarty, branżowy standard instrumentacji (trace'y, metryki, logi) — następca rozwiązań vendor-specific. Piszesz instrumentację raz, backend (App Insights, Jaeger, Datadog…) jest wymienny. Microsoft uczynił OTel docelowym sposobem zasilania Application Insights z .NET — w nowym kodzie używaj **Azure Monitor OpenTelemetry Distro** (`Azure.Monitor.OpenTelemetry.AspNetCore`), nie klasycznego SDK App Insights. Zweryfikuj w dokumentacji bieżący stan wsparcia — ten obszar zmieniał się w ostatnich latach dynamicznie.
- **Propagacja kontekstu przez Service Bus** — rzecz, którą w module 2 robiłeś świadomie (correlation id w nagłówkach), tu w dużej mierze dostajesz: instrumentacje .NET propagują kontekst trace'u standardem **W3C Trace Context** (nagłówek `traceparent`), a SDK `Azure.Messaging.ServiceBus` przenosi go w metadanych wiadomości. Efekt: w Application Insights widzisz **jeden trace**: HTTP request do producera → publikacja → (czas w kolejce) → odbiór przez consumera → zapytania SQL. Twoim zadaniem jest to **zweryfikować**, nie zbudować — i wiedzieć, gdzie kontekst się urywa (np. przy ręcznym przetwarzaniu batchy lub własnych pętlach odbioru trzeba spiąć go jawnie).
- **Sampling** — od pierwszego dnia ustaw sampling i poziomy logowania per środowisko (Information na prod, Debug tylko lokalnie); rachunek za ingest to najczęstsza niespodzianka observability (lekcja 06, pułapka nr 2).

### Architektura docelowa

```
                        ┌───────────────────────────────────────────────┐
                        │  Container Apps Environment                   │
 HTTP →  ingress  ───►  │  producer-api (min 1 replika)                 │
                        │     │ MI: send                                │
                        │     ▼                                        │
                        │  [outbox w SQL] ──relay──► Service Bus queue │
                        │                              │ MI: receive   │
                        │                              ▼               │
                        │  consumer-worker (KEDA: 0–N replik           │
                        │     │ MI            po długości kolejki)     │
                        └─────┼─────────────────────────────────────────┘
                              ▼
                         Azure SQL (serverless, Entra-only auth)

 wszystko → OpenTelemetry → Application Insights / Log Analytics
 całość opisana w infra/ (Bicep), tożsamość: user-assigned MI, zero sekretów
```

Decyzje są już podjęte w lekcjach 01–05 (i zapisane w twoich ADR-ach): Container Apps zamiast AKS/Functions (lekcja 01), Service Bus z DLQ i peek-lock (lekcja 02), SQL serverless — bo outbox potrzebuje transakcji relacyjnej (lekcja 03), user-assigned MI + RBAC data-plane (lekcja 04), całość z Bicep z parametryzacją środowisk (lekcja 05), tagi + budżet + świadomy ingest logów (lekcja 06).

## Praktyka

Projekt rozpisany na dwa tygodnie (tygodnie 3–4 modułu). Punkty kontrolne (✅ CHECKPOINT) traktuj twardo: nie przechodź dalej, dopóki checkpoint nie jest zielony — to one wyznaczają, że fundament jest zdrowy.

### Tydzień 1 — infrastruktura i aplikacja w chmurze

- [ ] **Przygotowanie repo:** aplikacja z modułu 2 buduje się do obrazów kontenerowych (producer, consumer — osobne Dockerfile); lokalnie działa docker compose z emulatorami/lokalnym brokerem. Jeśli w module 2 używałeś RabbitMQ — przepnij abstrakcję na `Azure.Messaging.ServiceBus` (to dobry test, czy twoje warstwy z modułu 2 naprawdę izolują brokera).
- [ ] **Bicep — rozbudowa `infra/` z lekcji 05:** dodaj moduły `containerapps.bicep` (środowisko + 2 aplikacje + Azure Container Registry lub obrazy z GHCR) i `monitoring.bicep` (Log Analytics + Application Insights). Connection do App Insights przekaż aplikacjom przez zmienną środowiskową (connection string App Insights nie jest sekretem w sensie credentiali, ale i tak prowadź go przez parametry Bicep, nie hardkod).
- [ ] **Tożsamość:** user-assigned MI podpięta do obu aplikacji; role w Bicep: *Service Bus Data Sender* (producer, scope: kolejka/topic), *Data Receiver* (consumer), użytkownik kontekstowy w SQL dla MI (`CREATE USER ... FROM EXTERNAL PROVIDER` — ten krok bywa półręczny, zanotuj go w README repo).
- [ ] **Deploy:** `az deployment group what-if` → przeczytaj → deploy. Obrazy wypchnięte do rejestru, aplikacje wystartowały.
- [ ] ✅ **CHECKPOINT 1:** wysyłasz HTTP request do producer-api w chmurze → wiadomość przechodzi przez Service Bus → consumer zapisuje wynik w SQL. W konfiguracji aplikacji **nie ma ani jednego sekretu** (przejrzyj zmienne środowiskowe w portalu, żeby to potwierdzić).
- [ ] **KEDA:** reguła skalowania consumera po długości kolejki, `minReplicas: 0`. Wstrzymaj consumera, napompuj kolejkę, włącz — obejrzyj skalowanie 0→N→0 w metrykach.
- [ ] ✅ **CHECKPOINT 2:** `az group delete` + ponowny deploy od zera = działający system bez ręcznych kroków (poza udokumentowanymi, jak user SQL). To jest test "infrastruktura naprawdę jest kodem".

### Tydzień 2 — observability, odporność, artefakty

- [ ] **OpenTelemetry:** Azure Monitor OTel Distro w obu aplikacjach; trace'y, metryki i logi płyną do Application Insights; sampling i poziomy logów ustawione per środowisko.
- [ ] ✅ **CHECKPOINT 3 (najważniejszy):** w Application Insights widzisz **jeden distributed trace przechodzący przez producer → Service Bus → consumer → SQL**, z czasem oczekiwania w kolejce. Zrób zrzut ekranu — wejdzie do posta/Featured.
- [ ] **Scenariusze odpornościowe z modułu 2, teraz w chmurze:** (a) rzuć wyjątkiem w consumerze → wiadomość po `MaxDeliveryCount` ląduje w DLQ → znajdź ją i jej trace w App Insights; (b) wyślij duplikat → idempotencja consumera działa; (c) napisz zapytanie KQL: liczba wiadomości przetworzonych/failed per godzina.
- [ ] **Mapa zależności:** obejrzyj Application Map w App Insights — czy pokazuje topologię, którą narysowałeś w lekcji 02?
- [ ] **Koszty domknięcie:** po kilku dniach działania porównaj rzeczywisty koszt w Cost Management z szacunkiem wariantu B z lekcji 06; dopisz wnioski do dokumentu porównania.
- [ ] **Artefakt 1 — diagram architektury wdrożenia:** narysuj (C4 container-level z modułu 1 + adnotacje: tożsamości, role, przepływ trace'u, model skalowania). Wrzuć do repo i do **Featured na LinkedIn**.
- [ ] **Artefakt 2 — post:** opublikuj porównanie wariantów z lekcji 06, teraz wzmocnione rzeczywistymi kosztami i zrzutem trace'u ("policzyłem na sucho, potem wdrożyłem i zmierzyłem — oto różnice").
- [ ] **README repo:** sekcja "Deploy from zero" (wymagania, 3–4 polecenia, znane kroki ręczne) + diagram. Repo, którego nie da się uruchomić z README, nie jest portfolio.

### Jeśli zostanie czas (nieobowiązkowe rozszerzenia)

- [ ] CI/CD: GitHub Actions z what-if w PR i deployem po merge (federated credentials zamiast sekretu service principala — wzorzec z lekcji 04 w wersji dla pipeline'u).
- [ ] Alert w Azure Monitor: rosnący DLQ → e-mail. Mały krok, świetnie wygląda w demo.

## Artefakt

1. **Działające wdrożenie** + repo: kod aplikacji, `infra/` (Bicep), README "deploy from zero" — kandydat na przypięte repo na GitHubie.
2. **Diagram architektury wdrożenia** → Featured na LinkedIn (artefakt modułu z planu).
3. **Post z porównaniem kosztów** wzmocniony rzeczywistymi danymi (artefakt modułu z planu).

## Definition of Done

- [ ] Wszystkie trzy CHECKPOINTY zielone — w szczególności trace end-to-end producer → Service Bus → consumer → SQL widoczny w Application Insights.
- [ ] Pełny redeploy od zera działa i jest opisany w README; w konfiguracji zero connection stringów z credentialami.
- [ ] DLQ-scenariusz przetestowany w chmurze; umiesz pokazać poison message i jego trace na żywo (to świetne 5 minut demo na rozmowie).
- [ ] Diagram w Featured, post opublikowany, rzeczywiste koszty dopisane do dokumentu z lekcji 06.
- [ ] Decyzja AZ-305 (README modułu) podjęta i zapisana w `notes/decisions-log.md` — to formalnie zamyka moduł 3.
- [ ] Po zakończeniu pracy: środowisko albo skalowane do zera/usunięte, albo objęte budżetem — rachunek za "pomnik projektu" to porażka lekcji 06.

## Materiały

- [Microsoft Learn — Azure Monitor OpenTelemetry for .NET](https://learn.microsoft.com/azure/azure-monitor/app/opentelemetry-enable?tabs=aspnetcore) — instrumentacja, którą wdrażasz w tygodniu 2; sprawdź bieżący stan Distro.
- [Microsoft Learn — Tutorial: Deploy to Azure Container Apps](https://learn.microsoft.com/azure/container-apps/tutorial-code-to-cloud) — referencja przepływu obraz → rejestr → Container App; resztę robisz z własnego Bicep.
