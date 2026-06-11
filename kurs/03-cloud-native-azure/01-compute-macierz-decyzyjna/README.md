# Lekcja 01 — Compute w Azure: macierz decyzyjna

## Cel lekcji

Po tej lekcji znasz wszystkie sensowne opcje uruchamiania kodu .NET w Azure (App Service, Functions, Container Apps, AKS, ACI), rozumiesz ich modele rozliczeń i skalowania, i umiesz dla konkretnego systemu wybrać opcję compute wraz z uzasadnieniem — to znaczy: także powiedzieć, czego NIE wybrałeś i dlaczego.

## Dlaczego to ważne

"Gdzie to uruchomimy?" to pierwsza decyzja każdego projektu chmurowego i klasyk rozmów na role architektoniczne. Pytanie rekrutacyjne rzadko brzmi "co to jest AKS?" — częściej: "macie 5-osobowy zespół i API z nierównym ruchem, co wybierzesz i dlaczego nie Kubernetes?". Zła decyzja compute to najdroższy rodzaj długu: migracja między modelami compute (np. z Functions na AKS) dotyka CI/CD, sieci, observability i nawyków zespołu. Po latach pracy z systemami on-premise / IIS ta lekcja daje ci mapę: co w chmurze odpowiada temu, co znasz, i co istnieje "nowego", czego on-premise nie miało (scale-to-zero, płacenie za wykonanie).

## Teoria

### Wspólna oś: spektrum kontrola ↔ prostota

Wszystkie opcje compute w każdej chmurze układają się na jednej osi. Po lewej: pełna kontrola, pełna odpowiedzialność (maszyna wirtualna — ty patchujesz OS). Po prawej: zero kontroli nad infrastrukturą, zero jej utrzymania (serverless — wrzucasz funkcję, chmura robi resztę). **Im dalej w prawo, tym mniej ops, ale więcej ograniczeń i mniej przewidywalny model kosztów przy dużym, stałym ruchu.** Zapamiętaj tę oś — cała lekcja to poruszanie się po niej.

```
VM ── AKS ── Container Apps ── App Service ── Functions
więcej kontroli, więcej ops          mniej ops, więcej ograniczeń
```

(VM-ki pomijamy — w 2026 wybieranie surowych maszyn wirtualnych dla aplikacji .NET wymaga specjalnego uzasadnienia, np. legacy zależne od pełnego Windows Server.)

### Azure App Service — "IIS w chmurze, którym nie zarządzasz"

**Co to jest:** PaaS (Platform as a Service — platforma, gdzie dostarczasz kod/kontener, a dostawca zarządza systemem, runtime'em i serwerem WWW) do hostowania aplikacji webowych i API. Najbliższy odpowiednik tego, co znasz z IIS: deployujesz aplikację ASP.NET, dostajesz URL, TLS, deployment sloty (środowiska do blue-green deployment — wgrywasz nową wersję na slot "staging" i przełączasz ruch jednym ruchem), autoskalowanie.

**Model rozliczeń:** płacisz za **App Service Plan** — zarezerwowaną moc obliczeniową (rozmiar instancji × liczba instancji), **niezależnie od ruchu**. Plan działa i kosztuje 24/7, nawet gdy nikt nie używa aplikacji. Na jednym planie możesz hostować wiele aplikacji (współdzielą zasoby). Tiery od darmowego/Basic (dev) po Premium (produkcja, sloty, VNet integration).

**Kiedy używać:** klasyczne API/aplikacja webowa o w miarę przewidywalnym ruchu, zespół chce minimum zmian względem znanego modelu "wgrywam apkę na serwer". Najkrótsza droga z IIS do chmury.

**Kiedy nie:** ruch mocno zmienny lub sporadyczny (płacisz za bezczynność — App Service nie skaluje do zera w standardowych tierach), architektura wielousługowa z messaging (Container Apps zrobią to lepiej), potrzeba skalowania na podstawie długości kolejki (App Service skaluje głównie po CPU/pamięci/harmonogramie).

### Azure Functions — serverless, czyli płacisz za wykonanie

**Co to jest:** FaaS (Functions as a Service). Piszesz pojedyncze funkcje uruchamiane przez **triggery**: HTTP request, wiadomość na kolejce, timer, zdarzenie z Event Grid. Nie ma "aplikacji, która działa i czeka" — jest kod, który budzi się do zdarzenia. W .NET piszesz je w modelu *isolated worker* (funkcje działają w osobnym procesie z normalnym hostem .NET — pełna kontrola nad wersją frameworka i middleware).

**Model rozliczeń — tu jest sedno:**
- **Consumption (Flex Consumption)** — prawdziwy serverless: płacisz za liczbę wykonań i czas×pamięć wykonania. Zero ruchu = zero kosztu (lub blisko zera). Trade-off: **cold start** — gdy funkcja długo nie była wywoływana, platforma zwalnia jej instancję; pierwsze wywołanie po przerwie musi wystartować proces (dla .NET: setki ms do kilku sekund). Dla webhooka to nic, dla API z SLA na latencję — problem. Flex Consumption pozwala dokupić *always-ready instances*, łagodząc cold start za dopłatą.
- **Premium plan** — pre-warmed instancje (brak cold startu), VNet integration, dłuższy czas wykonania. Ale: płacisz za co najmniej jedną instancję **stale** — to już nie jest "zero ruchu = zero kosztu". Premium przy stałym ruchu potrafi kosztować więcej niż równoważny App Service — policz, zanim wybierzesz.
- (Functions można też hostować na App Service Planie lub w Container Apps — opcje niszowe, odnotuj, że istnieją.)

**Kiedy używać:** obciążenia zdarzeniowe i nieregularne — webhooki, harmonogramy, glue code między usługami, przetwarzanie wiadomości o małym/średnim wolumenie. Idealne na "drobnicę integracyjną", którą w on-premise robiły Windows Services i taski w schedulerze.

**Kiedy nie:** długotrwałe procesy (limity czasu wykonania — sprawdź aktualne w dokumentacji), aplikacje ze stanem w pamięci, duży stały ruch (provisioned compute wychodzi taniej), oraz gdy zespół zaczyna budować "rozproszony monolit z 80 funkcji" — to sygnał, że potrzebujecie usług, nie funkcji.

### Azure Container Apps — środek spektrum i domyślny wybór dla .NET w 2026

**Co to jest:** serverless platforma kontenerowa. Dostarczasz **kontener** (czyli dowolną aplikację .NET — Web API, worker, consumer kolejki), a platforma zarządza orkiestracją. Pod spodem działa Kubernetes, ale jest **całkowicie ukryty** — nie piszesz YAML-i Kubernetesa, nie zarządzasz klastrem. Dostajesz: revisions (wersjonowane wdrożenia, traffic splitting), Dapr (opcjonalny runtime do service discovery i pub/sub), ingress z TLS.

**KEDA i scale-to-zero — kluczowa cecha.** KEDA (Kubernetes Event-Driven Autoscaling) to wbudowany mechanizm skalowania na podstawie **zdarzeń**, nie tylko CPU: liczba wiadomości w kolejce Service Bus, lag w Event Hubs, liczba żądań HTTP, metryki własne. Przykład: consumer kolejki ustawiasz na "0–10 replik, +1 replika na każde 20 wiadomości w kolejce". Pusta kolejka → **zero replik → zero kosztu compute**. To jest dokładnie model, którego brakowało ci on-premise: worker, który "nie istnieje", gdy nie ma pracy.

**Model rozliczeń:** Consumption — za vCPU-sekundy i GiB-sekundy aktywnych replik (plus pula darmowa miesięcznie); Dedicated (workload profiles) — za zarezerwowane maszyny, gdy potrzebujesz gwarantowanej mocy lub GPU. Dla większości scenariuszy startujesz z Consumption.

**Kiedy używać:** mikroserwisy i workery messagingowe w .NET, API ze zmiennym ruchem, wszystko co już jest w kontenerze. To **domyślna rekomendacja tego kursu** dla systemów rozproszonych — i platforma projektu w lekcji 07.

**Kiedy nie:** potrzebujesz pełnego API Kubernetes (operatory, custom controllers, DaemonSets, service mesh wg własnego wyboru), workloadów stanowych z dyskami, albo wieloklastrowej kontroli — wtedy AKS.

### AKS (Azure Kubernetes Service) — kiedy NAPRAWDĘ potrzebujesz Kubernetesa

**Co to jest:** zarządzany Kubernetes — Azure utrzymuje control plane (mózg klastra), ty zarządzasz node poolami (maszynami roboczymi), konfiguracją, upgrade'ami, siecią, dodatkami. Pełna moc i pełne API Kubernetes.

**Model rozliczeń:** płacisz za VM-ki node'ów (24/7, niezależnie od ruchu) + ewentualnie za tier control plane. Do tego **koszt ukryty i największy: ludzie**. Klaster produkcyjny wymaga kogoś, kto rozumie networking, RBAC, upgrade'y wersji (Kubernetes wydaje wersje co kilka miesięcy i stare przestają być wspierane), monitoring klastra. To realnie ułamek etatu — stały.

**Kiedy używać — uczciwa lista:** (1) wiele zespołów współdzieli platformę i potrzebna jest izolacja namespace'ami + standaryzacja, (2) przenosisz istniejące workloady już opisane w Kubernetes/Helm, (3) wymagania, których Container Apps nie obsłuży: własny service mesh, operatory (np. samodzielnie hostowany Kafka), GPU z custom schedulingiem, ścisłe wymogi sieciowe, (4) strategia multi-cloud z Kubernetesem jako warstwą przenośności. **Zauważ, czego nie ma na liście:** "bo mamy mikroserwisy". Mikroserwisy same w sobie nie wymagają Kubernetesa.

**Anty-wzorzec do nazwania na rozmowie:** "resume-driven Kubernetes" — klaster dla 5-osobowego zespołu z trzema usługami. Koszt ops zjada każdą korzyść. Umiejętność powiedzenia "tu NIE wziąłbym AKS" jest silniejszym sygnałem seniority niż znajomość AKS.

### Azure Container Instances (ACI) — krótko

Pojedynczy kontener na żądanie, rozliczany za sekundy działania, bez orkiestracji (brak autoskalowania, load balancingu, revisions). Zastosowania: zadania wsadowe ad-hoc, agenci CI, szybkie eksperymenty. Nie jest platformą aplikacyjną — w architekturach pojawia się jako "wykonaj ten kontener raz i zniknij".

### Macierz decyzyjna

| Wymiar | App Service | Functions (Consumption) | Container Apps | AKS |
|---|---|---|---|---|
| Kontrola vs prostota | prostota; model "aplikacja na serwerze" | maksymalna prostota, najwięcej ograniczeń | środek: kontener = pełna swoboda kodu, zero ops klastra | maksymalna kontrola, pełny ops |
| Jednostka wdrożenia | aplikacja (kod lub kontener) | funkcja/trigger | kontener | wszystko (pody, YAML, Helm) |
| Model skalowania | instancje po CPU/pamięci/harmonogramie; bez zera | automatyczny, per zdarzenie, do zera | KEDA: zdarzenia (kolejka, HTTP, custom), **scale-to-zero** | dowolny (HPA/KEDA), ale konfigurujesz sam |
| Koszt przy małym/nieregularnym ruchu | słaby (plan działa 24/7) | **najlepszy** (płacisz za wykonania) | bardzo dobry (zero replik = ~zero kosztu) | najgorszy (node'y 24/7 + ludzie) |
| Koszt przy dużym stałym ruchu | dobry, przewidywalny | słaby (per-wykonanie się sumuje) → Premium/zmiana platformy | dobry (Dedicated profiles) | dobry przy dużej skali (reservations) |
| Cold start | nie | tak (Consumption) | przy skalowaniu od zera (sterowalne `minReplicas`) | nie |
| Ops overhead | niski | najniższy | niski | **wysoki — stały koszt ludzki** |
| Typowy przypadek | klasyczne web API, migracja z IIS | webhooki, harmonogramy, glue code | mikroserwisy + workery messagingowe | platforma wielu zespołów, special needs |

### Reguła kciuka dla typowego systemu .NET (2026)

1. **Zaczynaj od Container Apps** — API i workery jako kontenery, KEDA do skalowania po kolejce, scale-to-zero dla rzadko używanych usług.
2. **Functions** na zdarzeniową drobnicę wokół rdzenia: webhooki, timery, integracje.
3. **App Service**, gdy to jedna klasyczna aplikacja webowa i zespół chce zerowej krzywej uczenia.
4. **AKS tylko z powodem zapisanym w ADR** — jeśli nie umiesz nazwać konkretnego wymagania, którego Container Apps nie spełnia, nie bierz AKS.

Trade-off całej reguły: wybierając Container Apps oddajesz dostęp do pełnego API Kubernetes — jeśli za 2 lata go zażądasz, migracja na AKS jest możliwa (te same kontenery), ale nie darmowa (sieć, deploy, monitoring od nowa).

## Praktyka

- [ ] Wypisz z pamięci 5 opcji compute i dla każdej: model rozliczeń + jedno zdanie "kiedy tak / kiedy nie". Sprawdź z lekcją.
- [ ] Weź aplikację messagingową z modułu 2 (producer API + consumer + baza). Dla każdego komponentu przejdź macierz i wybierz compute. Zapisz decyzję jako **ADR** (kontekst → decyzja → odrzucone alternatywy z powodami → konsekwencje).
- [ ] Adwokat diabła: napisz 5–8 zdań uzasadnienia, w jakich warunkach twój system **jednak** powinien iść na AKS. Jeśli nie umiesz — wróć do sekcji o AKS.
- [ ] W Azure Portal utwórz środowisko Container Apps i wdróż dowolny przykładowy kontener (może być `mcr.microsoft.com/k8se/quickstart`). Obejrzyj: revisions, reguły skalowania, ustaw `minReplicas: 0` i zaobserwuj scale-to-zero. **Usuń resource group po ćwiczeniu.**
- [ ] Sprawdź w dokumentacji aktualne limity Functions Consumption (czas wykonania, pamięć) — bywają zmieniane, nie ucz się ich z tej lekcji na pamięć.

## Artefakt

1. **Macierz decyzyjna compute** w twoich notatkach (możesz przerobić tabelę z lekcji, ale dopisz własną kolumnę "moje przypadki użycia" z przykładami z twoich projektów).
2. **ADR**: wybór compute dla aplikacji z modułu 2 — wejdzie wprost do projektu w lekcji 07. Kandydat na post LinkedIn ("Dlaczego NIE wybrałem Kubernetesa").

## Definition of Done

- [ ] Umiesz odpowiedzieć w 60 sekund na każde z pytań: "Functions czy Container Apps dla consumera kolejki?", "kiedy App Service mimo wszystko?", "po czym poznasz, że zespół potrzebuje AKS?".
- [ ] ADR istnieje, zawiera odrzucone alternatywy z powodami i jest zacommitowany w repo.
- [ ] Uruchomiłeś (i usunąłeś) co najmniej jedną Container App i widziałeś scale-to-zero na własne oczy.

## Materiały

- [Azure Architecture Center — Choose an Azure compute service](https://learn.microsoft.com/azure/architecture/guide/technology-choices/compute-decision-tree) — oficjalne drzewo decyzyjne; porównaj z macierzą z tej lekcji.
- [Microsoft Learn — Azure Container Apps overview](https://learn.microsoft.com/azure/container-apps/overview) — przeczytaj w całości; to platforma projektu w lekcji 07.
