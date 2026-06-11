# Etap 02 — Architektura i backlog: projekt przed kodem

## Cel lekcji

Po tym etapie masz w repo komplet decyzji projektowych zanim napiszesz pierwszą linię kodu produkcyjnego: diagramy C4 (poziom 1–2), 3–5 ADR-ów dla kluczowych wyborów oraz backlog rozbity na epiki i tygodnie. Wiesz dokładnie, co budujesz w którym tygodniu — i potrafisz każdy wybór obronić.

## Dlaczego to ważne

"Projekt przed kodem" to nie biurokracja — to dokładnie ta praca, za którą płaci się architektom. Na system design interview dostajesz 45 minut na zrobienie tego, co tu zrobisz w tygodniu: aktorzy, kontenery, decyzje, trade-offy. Capstone jest więc jednocześnie treningiem formatu rozmowy.

Jest też powód czysto praktyczny: w etapie 03 czekają Cię 2–3 tygodnie implementacji. Każda decyzja podjęta "w locie" podczas kodowania to przerwany flow i ryzyko refaktoringu w połowie sagi. Decyzje podjęte teraz, na diagramie i w ADR, kosztują godzinę; te same decyzje podjęte w tygodniu 3 kosztują dzień. To jest dźwignia, o której mówi cały moduł 1.

## Teoria

### Diagram C4 — co ma pokazywać dla tego systemu

Format C4 znasz z modułu 1 (lekcja 03). Tu konkret: co **musi** znaleźć się na diagramach capstone, żeby recenzent zrozumiał system bez czytania kodu.

**Poziom 1 — System Context.** Aktorzy i systemy:
- **Klient** (osoba rezerwująca lot) — używa systemu przez API/minimalny UI.
- **Operator tunelu** (osoba) — opcjonalnie, jeśli FR tego wymaga (np. przegląd rezerwacji dnia); jeśli nie — nie dodawaj na siłę.
- **System rezerwacji** (Twój system) — jeden boks, jedno zdanie opisu.
- **TunnelOps** (system zewnętrzny, symulowany) — system operacyjny tunelu; strzałka "zgłasza potwierdzone rezerwacje".
- **Bramka płatności** (system zewnętrzny, symulowany) — strzałka "autoryzuje płatności".

Symulowane systemy rysuj jako zewnętrzne (szare w konwencji C4), z adnotacją "(simulated)" — uczciwość diagramu buduje zaufanie do całego repo.

**Poziom 2 — Containers.** Kontener w C4 = osobno uruchamialna jednostka (proces), nie kontener Dockera. Oczekiwany zestaw:
- **Booking API** — ASP.NET Core; przyjmuje żądania klienta (dostępność, rezerwacja, płatność, anulowanie); zapisuje do bazy i do outboxa.
- **Worker** — proces tła (.NET); publikuje z outboxa, konsumuje zdarzenia, prowadzi sagę, woła TunnelOps, generuje powiadomienia.
- **Broker** — Azure Service Bus (lokalnie: emulator Service Bus lub RabbitMQ — decyzja do ADR, patrz niżej); kolejki/topiki + DLQ.
- **Baza danych** — Azure SQL (lokalnie: kontener SQL Server); dane domenowe + outbox + tabela dedup + stan sagi.
- **TunnelOps simulator** — osobny minimalny serwis HTTP (może być w tym samym repo), konfigurowalne awarie: opóźnienia, błędy 5xx, odpowiedzi-śmieci.

Na strzałkach: protokół i charakter komunikacji ("HTTP/sync", "Service Bus/async, at-least-once"). To poziom, który trafi na górę README repo.

Poziom 3 (Components) rysuj tylko, jeśli w etapie 03 okaże się potrzebny do wyjaśnienia sagi — nie z automatu.

### Decyzja 1: modular monolith vs osobne serwisy

Najważniejszy ADR projektu. Opcje:

- **(A) Mikroserwisy**: booking, payments, notifications jako osobne procesy/deploymenty. Plus: "wygląda imponująco". Minus: trzy pipeline'y, trzy deploymenty, sieć między każdą parą — złożoność operacyjna, która nie demonstruje żadnej *dodatkowej* kompetencji architektonicznej, a zjada tygodnie. Klasyczny błąd projektów portfolio.
- **(B) Modular monolith**: jeden codebase z twardo wydzielonymi modułami (booking, payments, notifications) — osobne przestrzenie nazw/projekty, komunikacja między modułami **wyłącznie przez kontrakty** (zdarzenia, publiczne interfejsy), zakaz sięgania do cudzych tabel. Wdrażany jako dwa procesy: API + Worker.

**Rekomendacja: (B), z komunikacją przez Service Bus tam, gdzie uczy wzorców.** Zdarzenia domenowe (BookingConfirmed, PaymentFailed...) idą przez realny broker — bo to na nich demonstrujesz outbox, idempotency i DLQ. Wywołania, które brokera nie potrzebują (odczyt dostępności), zostają zwykłym kodem. W ADR zapisz wprost: granice modułów są tak postawione, że wydzielenie payments do osobnego serwisu to zmiana deploymentu, nie architektury — i to jest puenta, którą powiesz na rozmowie. (Moduł 1, lekcja 06 — style architektoniczne — daje pełną argumentację.)

### Decyzja 2: wybór bazy danych

Opcje: Azure SQL vs Cosmos DB (macierz z modułu 3, lekcja 03). Rekomendacja: **Azure SQL**. Argumenty do ADR: outbox i tabela deduplikacji wymagają transakcji obejmującej dane domenowe i outbox (dual write — moduł 2, lekcja 02); model danych jest relacyjny (sloty, instruktorzy, rezerwacje — naturalne FK); blokada slotu to unique constraint, najtańsza możliwa kontrola współbieżności; koszt i znajomość operacyjna. Cosmos odrzucony: globalna dystrybucja i elastyczny schemat to atrybuty, z których ADR-001 świadomie zrezygnował. Odrzucenie z uzasadnieniem zapisz — to połowa wartości ADR-u.

### Decyzja 3: saga — orchestration vs choreography

Przepływ rezerwacji (hold → płatność → potwierdzenie → zgłoszenie do TunnelOps, z kompensacją przy porażce) to saga — definicje i porównanie znasz z modułu 2, lekcja 03. Do ADR:

- **Choreography**: każdy moduł reaguje na zdarzenia poprzedniego, nikt nie trzyma całości. Mniej elementów, ale przepływ jest "rozsmarowany" — trudniej pokazać go recenzentowi i trudniej debugować timeout płatności.
- **Orchestration**: jawny orkiestrator (w Workerze) trzyma stan sagi w bazie i decyduje o krokach oraz kompensacjach.

**Rekomendacja: orchestration** dla przepływu rezerwacji. Powody dydaktyczno-praktyczne: stan sagi w tabeli = scenariusz "restart workera w środku sagi" da się pokazać i przetestować; timeout płatności ma naturalne miejsce obsługi; recenzent widzi cały przepływ w jednej klasie. W ADR uczciwie zapisz koszt: orkiestrator to pojedynczy punkt wiedzy o procesie i dodatkowy stan do migrowania.

### Decyzja 4 (mniejsza, ale zapisz): broker lokalnie

Azure Service Bus nie działa offline; do developmentu lokalnego: emulator Service Bus w kontenerze (sprawdź aktualny stan — Microsoft wydał oficjalny emulator) albo abstrakcja transportu (np. przez bibliotekę messagingową) z RabbitMQ lokalnie. Trade-off: emulator = wierność produkcji; abstrakcja = czystszy kod, ale ryzyko różnic zachowań (np. semantyka DLQ). Wybierz, zapisz w ADR, nie wracaj do tematu.

### Backlog: epiki i tygodnie

**Epik** (epic) — duży, spójny kawałek pracy rozbijany na zadania. Proponowany podział, skorelowany z harmonogramem modułu:

| Epik | Zakres | Tydzień |
|---|---|---|
| **E1 — Szkielet + CI** | struktura solution (moduły, testy, docker-compose), pipeline CI (build + testy + badge), TunnelOps simulator "happy path" | 2 (start) |
| **E2 — Booking + outbox** | FR-1, FR-2: dostępność, rezerwacja z holdem i unique constraint, zdarzenia przez outbox, idempotent consumer po stronie workera | 2 |
| **E3 — Płatność + saga** | FR-3: symulowana bramka (sukces/odmowa/timeout), orkiestrator sagi ze stanem w bazie, kompensacja (zwolnienie slotu), wygasanie holdów | 3 |
| **E4 — Powiadomienia + integracja + DLQ** | FR-4, FR-5, FR-6: powiadomienia ze zdarzeń, zgłoszenia do TunnelOps z retry, poison message → DLQ + runbook, anulowanie | 4 |
| **E5 — Azure + observability** | Bicep (Container Apps, Service Bus, SQL, App Insights), managed identities, tracing end-to-end, dashboard, koszty | 5 |
| **E6 — Dokumentacja + publikacja** | README rekruterskie, trade-offs, porządki, Featured, post | 6 |

Rozbij epiki na zadania w GitHub Issues + Projects (publiczna tablica to dodatkowy sygnał profesjonalizmu) albo w `docs/backlog.md` — forma jest drugorzędna, ważne: zadania ≤ pół dnia pracy każde i jasne kryterium "done". W wariancie 4-tygodniowym: E1+E2 w tygodniu 2, E3+E4 (okrojone — patrz etap 03) w tygodniu 3, E5+E6 w tygodniu 4.

### Struktura repo — ustal teraz, oszczędzisz refaktoring

Propozycja zgodna z decyzjami powyżej (modular monolith, vertical slices — szczegóły organizacji kodu w etapie 03):

```
/src
  Booking.Api/            # ASP.NET Core — endpointy wszystkich modułów
  Booking.Worker/         # outbox publisher, saga, konsumenci
  Modules/
    Bookings/             # moduł: sloty, holdy, rezerwacje (slices + kontrakty zdarzeń)
    Payments/             # moduł: saga płatności, bramka (symulowana)
    Notifications/        # moduł: powiadomienia
  TunnelOps.Simulator/    # symulowany system zewnętrzny
/tests                    # jednostkowe + integracyjne (Testcontainers)
/infra                    # Bicep (etap 04)
/docs
  adr/  diagrams/  runbooks/  requirements.md  domain.md  backlog.md
```

Granice modułów egzekwuj od pierwszego commita (osobne projekty, internal domyślnie, komunikacja przez kontrakty) — granica, którą "doda się później", nie powstaje nigdy.

## Praktyka

- [ ] Narysuj C4 poziom 1 i 2 (narzędzie z modułu 1 — np. Structurizr DSL albo diagrams-as-code w Mermaid/PlantUML; źródło diagramu commituj do repo, nie tylko PNG).
- [ ] Napisz ADR-002: modular monolith z modułami booking/payments/notifications, wdrażany jako API + Worker (opcje, decyzja, konsekwencje — w tym ścieżka wydzielenia serwisu).
- [ ] Napisz ADR-003: Azure SQL jako baza (z uczciwym odrzuceniem Cosmos DB).
- [ ] Napisz ADR-004: saga orchestration dla przepływu rezerwacji (z kosztami tej decyzji).
- [ ] Napisz ADR-005: broker i strategia developmentu lokalnego (emulator vs abstrakcja transportu).
- [ ] Rozpisz backlog: epiki E1–E6 → zadania z kryteriami done; przypnij do tygodni.
- [ ] Przejrzyj całość pod kątem ADR-001: czy żadna decyzja nie przemyca wyciętego zakresu z powrotem?

## Artefakt

W repo capstone:
1. `docs/diagrams/` — C4 L1 + L2 (źródło + render),
2. `docs/adr/ADR-002…005` — cztery ADR-y (plus ADR-001 z etapu 01 = komplet pięciu),
3. backlog: GitHub Issues/Projects albo `docs/backlog.md` — epiki E1–E6 rozbite na zadania.

## Definition of Done

- [ ] Diagram C4 L2 pokazuje wszystkie 5 kontenerów i charakter komunikacji na strzałkach; osoba z zewnątrz odtworzy z niego przepływ rezerwacji.
- [ ] Każdy ADR ma minimum 2 rozważone opcje i sekcję konsekwencji — żaden nie jest "wybrałem X, bo X jest fajny".
- [ ] Backlog: każde zadanie ≤ pół dnia, każdy epik ma przypisany tydzień.
- [ ] Test rozmowy: ustaw timer na 10 minut i opowiedz architekturę na głos (poziomy C4 + 3 kluczowe decyzje z trade-offami). Jeśli nie mieścisz się w 10 minutach — diagramy są za szczegółowe albo decyzje nieprzemyślane.

## Materiały

1. **Moduł 1, lekcje 03 i 06** (`kurs/01-fundamenty-myslenia-architektonicznego/`) — C4 oraz argumentacja modular monolith vs mikroserwisy.
2. **Moduł 2, lekcja 03 — Sagi** (`kurs/02-systemy-rozproszone/03-sagi-orchestration-vs-choreography/`) — pełne porównanie orchestration/choreography, które tu tylko streszczono.
