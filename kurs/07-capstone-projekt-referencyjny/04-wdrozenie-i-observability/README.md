# Etap 04 — Wdrożenie i observability: system w chmurze, który widać

## Cel lekcji

Po tym etapie capstone działa na Azure, wdrożony w całości z Bicep (Container Apps + Service Bus + Azure SQL + Application Insights, z managed identities zamiast connection stringów), z distributed tracingiem przez cały przepływ rezerwacji, minimalnym dashboardem i alertami — oraz policzoną i zoptymalizowaną sekcją kosztów w README repo.

## Dlaczego to ważne

"Działa u mnie w Dockerze" to poziom seniora. "Działa w chmurze, wdrożone z IaC, widzę każdy przepływ w tracingu i wiem, ile to kosztuje miesięcznie" — to poziom, na który celujesz. Dla recenzenta repo różnica jest natychmiast widoczna: katalog `infra/`, screenshot trace'a w README i sekcja kosztów to trzy rzeczy, których w 95% projektów portfolio po prostu nie ma.

Robiłeś już to w module 3 (lekcja 07) dla repo messagingowego — ten etap to powtórka **bez taryfy ulgowej**: bez prowadzenia za rękę, na większym systemie, z sagą w środku. Powtórzenie samodzielne jest dokładnie tym, co zamienia "zrobiłem raz z instrukcją" w kompetencję.

## Teoria

### Architektura wdrożenia

Mapowanie kontenerów C4 (etap 02) na zasoby Azure:

| Kontener C4 | Zasób Azure | Uwagi |
|---|---|---|
| Booking API | Container Apps — app z ingress | HTTP z internetu; skalowanie po HTTP (min 0–1 replik) |
| Worker | Container Apps — app bez ingress | skalowanie po długości kolejki (KEDA scaler dla Service Bus); min 0 replik |
| TunnelOps simulator | Container Apps — app z ingress wewnętrznym | celowo wdrażany — demo awaryjności ma działać też w chmurze |
| Broker | Service Bus (Standard) | topiki/kolejki + DLQ; Standard wystarcza (topiki są od Standard wzwyż) |
| Baza | Azure SQL Database | najtańszy sensowny tier dla demo — patrz sekcja kosztów |
| Telemetria | Application Insights (workspace-based) + Log Analytics | wspólny workspace dla wszystkiego |

Do tego: Container Registry (obrazy), Container Apps Environment (wspólne dla trzech appek), Key Vault tylko jeśli zostaje jakikolwiek sekret (cel: nie zostaje).

### Bicep: struktura i kolejność

Szablon z modułu 3 (lekcja 05) jest punktem wyjścia; tu rośnie do struktury modułowej: `infra/main.bicep` + moduły per obszar (`servicebus.bicep`, `sql.bicep`, `apps.bicep`, `monitoring.bicep`). Zasady, które recenzent doceni:

- **Zero sekretów w parametrach.** Połączenia przez **managed identity** (tożsamość zarządzana — Azure sam wydaje i rotuje poświadczenia aplikacji): API i Worker dostają tożsamość, role `Azure Service Bus Data Sender/Receiver` na brokerze i dostęp do SQL przez uwierzytelnianie Entra ID (użytkownik bazodanowy dla tożsamości). Jedyny trudny punkt: utworzenie użytkownika w bazie dla managed identity wymaga kroku poza Bicep (skrypt wdrożeniowy) — zaplanuj go w pipeline, nie rób ręcznie w portalu.
- **Idempotentne wdrożenie**: `az deployment group create` uruchamiane drugi raz niczego nie psuje. Przetestuj to wprost — to ta sama idempotencja, którą ćwiczyłeś na wiadomościach, tylko na poziomie infrastruktury.
- **Parametryzacja środowiska** (sufiks nazw, tier bazy) — nawet jeśli środowisko jest jedno, pokazujesz, że umiesz myśleć o wielu.

Weryfikuj nazwy i wersje zasobów z bieżącą dokumentacją (plan pisany 2026) — w Bicep wersje API zasobów potrafią się zmieniać; nie kopiuj ślepo ze starych przykładów.

Skalowanie Workera po kolejce — szkic reguły KEDA w definicji Container App (Bicep), bo to ona daje scale-to-zero z budzeniem na wiadomość:

```bicep
scale: {
  minReplicas: 0
  maxReplicas: 2
  rules: [{
    name: 'sb-queue'
    custom: {
      type: 'azure-servicebus'
      metadata: { queueName: 'booking-events', messageCount: '10' }
      identity: workerIdentity.id   // scaler też uwierzytelnia się managed identity
    }
  }]
}
```

Składnię i nazwy pól zweryfikuj z bieżącą dokumentacją — to szkic intencji, nie wzorzec do kopiowania.

### Distributed tracing przez cały przepływ

Cel: **jeden trace od POST /bookings do wywołania TunnelOps**, mimo że po drodze są: transakcja w bazie, outbox, broker, saga i HTTP na zewnątrz. Instrumentację OpenTelemetry znasz z modułu 2 (lekcja 07); specyfika tego systemu:

- **Outbox przerywa automatyczną propagację kontekstu** — zdarzenie nie jest publikowane w żądaniu HTTP, tylko później, przez publisher. Rozwiązanie: zapisuj `traceparent` (identyfikator kontekstu trace'a wg W3C Trace Context) w wierszu outboxa i odtwarzaj go przy publikacji jako link/parent. To subtelność, którą mało kto robi dobrze — opisz ją w README lub krótkim ADR, bo to świetny materiał na rozmowę.
- **Saga = wiele spanów w czasie** — kroki sagi mogą być oddalone o minuty (timeout płatności). Trace będzie "długi i dziurawy"; to normalne. Upewnij się, że każdy krok sagi ma span z atrybutami (`booking.id`, `saga.step`), żeby dało się filtrować w Application Insights.
- **Korelacja w logach**: każdy log ma `TraceId` — wtedy z alertu przechodzisz do logów, z logów do trace'a. Bez tego observability to trzy niepowiązane silosy.

Test akceptacyjny: w Application Insights (Application map / transaction search) widzisz pełny przepływ rezerwacji jako jedną transakcję end-to-end. Zrób **screenshot — trafi do README** jako dowód.

### Dashboard i alerty — minimum, które ma sens

Nie buduj ściany wykresów. Minimum architekta (metryki RED z modułu 2: Rate, Errors, Duration):

- **Dashboard** (workbook w Application Insights albo dashboard Azure): żądania API (rate/errors/duration), głębokość kolejek Service Bus, **liczba wiadomości w DLQ**, aktywne/zawieszone sagi, repliki Container Apps (widać scale-to-zero!).
- **Alerty — dwa wystarczą na demo**: (1) DLQ > 0 przez 5 minut — bo wiadomość w DLQ to zdarzenie wymagające człowieka (masz runbook z etapu 03); (2) error rate API powyżej progu. Każdy alert bez jasnej akcji odbiorcy to spam — zasada z produkcji, zapisz ją przy alertach w README.

### Koszty: sekcja w README repo

Koszt to atrybut architektury (moduł 3, lekcja 06). Sekcja "Running costs" w README capstone zawiera:

1. **Tabela miesięczna** zasobów z cenami (policz w kalkulatorze Azure dla swojego regionu; podaj datę wyceny — ceny się zmieniają). Orientacyjne rzędy wielkości dla demo: Container Apps przy scale-to-zero i sporadycznym ruchu — grosze do kilku USD (consumption plan rozlicza za użycie; jest też darmowy przydział miesięczny); Service Bus Standard — kilkanaście USD bazowej opłaty; Azure SQL — od ~5 USD (Basic/serverless z auto-pause) wzwyż; Application Insights — koszt za GB ingestowanych danych, przy demo zwykle w darmowym przydziale. **Zweryfikuj aktualne ceny — nie cytuj tych liczb bez sprawdzenia.**
2. **Jak zminimalizowano**: scale-to-zero dla API i Workera (min replicas = 0; trade-off: cold start pierwszego żądania — opisz go!), SQL serverless z auto-pause (trade-off: wznowienie trwa), sampling telemetrii, retencja logów 30 dni.
3. **Wyłącznik awaryjny**: `azd down` / skrypt usuwający resource group — środowisko demo stawiasz na czas pokazów i rozmów, nie trzymasz wiecznie. Budżet z alertem (ustawiony w module 3) obejmuje też ten projekt.

To podejście — "policzyłem, zoptymalizowałem, udokumentowałem trade-offy oszczędności" — jest rzadkie w portfolio i bardzo dobrze widziane przez hiring managerów, którzy płacą rachunki za chmurę.

## Praktyka — checklist wdrożeniowy

- [ ] `infra/` z modułowym Bicep: Container Apps Environment + 3 appki, Service Bus z kolejkami/topikami i DLQ, Azure SQL, Application Insights + Log Analytics, Container Registry.
- [ ] Managed identities: role Service Bus przypisane w Bicep; użytkownik SQL dla tożsamości tworzony skryptem w pipeline; **zero connection stringów i haseł** w konfiguracji appek (audyt: przejrzyj zmienne środowiskowe w portalu).
- [ ] Pipeline wdrożeniowy (GitHub Actions): build obrazów → push do registry → `az deployment` → krok SQL — uruchamialny od zera na czystym resource group.
- [ ] Test idempotencji infrastruktury: drugie uruchomienie wdrożenia przechodzi bez zmian i bez błędów.
- [ ] Smoke test w chmurze: pełna rezerwacja przez publiczny endpoint API; sprawdź trace end-to-end w Application Insights (z propagacją przez outbox) — zrób screenshot.
- [ ] Demo awaryjne w chmurze: wymuś poison message przez TunnelOps simulator → wiadomość w DLQ → alert zadziałał.
- [ ] Dashboard z metrykami RED + DLQ + replikami; 2 alerty z opisaną akcją.
- [ ] Sekcja "Running costs" w README: tabela, optymalizacje z trade-offami, sposób ubicia środowiska; budżet z alertem aktywny.

## Artefakt

W repo: katalog `infra/` (Bicep + skrypty), pipeline wdrożeniowy, sekcja "Running costs" w README, screenshot trace'a end-to-end i dashboardu (do `docs/images/`, użyte w README). W Azure: działające środowisko demo (stawiane na żądanie).

## Definition of Done

- [ ] Czysty resource group → jedno uruchomienie pipeline'u → działający system. Bez kroków ręcznych poza ewentualnym jednorazowym bootstrapem opisanym w README.
- [ ] Rezerwacja wykonana w chmurze ma jeden spójny trace: API → outbox → broker → saga → TunnelOps.
- [ ] Wiadomość w DLQ wywołuje alert; wiesz (runbook), co z nią zrobić.
- [ ] Koszt miesięczny środowiska demo policzony, zoptymalizowany i opisany w README razem z trade-offami optymalizacji (cold start, auto-pause).
- [ ] Umiesz odpowiedzieć: dlaczego Container Apps, a nie AKS/Functions/App Service — dla tego konkretnego systemu (macierz z modułu 3, lekcja 01, zastosowana, nie wyrecytowana).

## Materiały

1. **Moduł 3, lekcje 04, 05 i 07** (`kurs/03-cloud-native-azure/`) — managed identities, Bicep i wzorzec całego wdrożenia, które tu powtarzasz samodzielnie.
2. **[Azure Architecture Center — Container Apps](https://learn.microsoft.com/azure/container-apps/)** — dokumentacja pierwotna: KEDA scalery, scale-to-zero, ingress; weryfikuj tu bieżący stan usługi.
