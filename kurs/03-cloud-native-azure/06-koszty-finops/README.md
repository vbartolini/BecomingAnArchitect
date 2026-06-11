# Lekcja 06 — Koszty jako atrybut architektury (FinOps w pigułce)

## Cel lekcji

Po tej lekcji traktujesz koszt jak atrybut jakościowy architektury — na równi z dostępnością czy latencją — znasz narzędzia Azure do szacowania i kontroli kosztów, rozpoznajesz główne pułapki kosztowe i umiesz policzyć oraz porównać koszt kilku wariantów tej samej architektury. Ćwiczenie kluczowe tej lekcji produkuje artefakt modułu: porównanie kosztów trzech wariantów.

## Dlaczego to ważne

W on-premise koszt infrastruktury był decyzją zakupową raz na kilka lat — architekt projektował "pod żelazo, które jest". W chmurze **każda decyzja architektoniczna jest decyzją kosztową**, naliczaną co godzinę: wybór tieru, polityka retencji logów, sposób skalowania. Dlatego "koszty" to dziś jeden z najmocniejszych tematów na rozmowach architektonicznych — kandydatów, którzy projektują ładne diagramy, jest wielu; kandydatów, którzy przy diagramie potrafią powiedzieć "wariant A kosztuje ~X/miesiąc, wariant B ~3X, a różnica kupuje nam Y" — mało. Historie typu "zmniejszyłem rachunek o 40% zmianą tieru i retencji logów" to gotowe, wiarygodne posty i odpowiedzi rekrutacyjne.

## Teoria

### FinOps w pigułce — model myślenia

**FinOps** to praktyka łącząca finanse i inżynierię wokół jednej idei: **koszt chmury jest sterowalny przez decyzje inżynierskie, więc inżynierowie muszą go widzieć i za niego współodpowiadać**. Kanoniczny cykl ma trzy fazy:

1. **Inform** — widoczność: kto, co i ile wydaje. Bez przypisania kosztów do systemów/zespołów (→ tagi) nie ma rozmowy.
2. **Optimize** — działanie: dopasowanie zasobów do realnego użycia (rightsizing), eliminacja marnotrawstwa, zobowiązania za zniżki (reservations/savings plans — rezerwujesz użycie na 1–3 lata w zamian za istotny rabat; sensowne dopiero przy stabilnym, przewidywalnym obciążeniu).
3. **Operate** — ciągłość: koszty przeglądane rytmicznie (jak metryki niezawodności), budżety z alertami, anomalie wykrywane wcześnie.

Dla architekta najważniejsza jest faza **zero**, którą cykl pomija: koszt projektuje się **przed wdrożeniem**. Szacunek kosztów wariantów powinien być sekcją każdego poważnego ADR — "wybieramy B, mimo że A jest tańsze o połowę, bo…" to wzorcowe zdanie architektoniczne.

I jedna uczciwość intelektualna: optymalizujesz **koszt całkowity**, nie rachunek za Azure. AKS bywa "tańszy na fakturze" od Container Apps przy dużej skali — dopóki nie doliczysz ułamka etatu na utrzymanie klastra. Czas zespołu to też koszt; na rozmowie to rozróżnienie (rachunek vs TCO — total cost of ownership) robi różnicę.

### Narzędzia — minimum operacyjne

- **Azure Pricing Calculator** ([azure.com/pricing/calculator](https://azure.microsoft.com/pricing/calculator/)) — szacowanie **przed** wdrożeniem: składasz koszyk usług z parametrami (tier, region, wolumen) i dostajesz miesięczny szacunek. To narzędzie ćwiczenia kluczowego. Nawyk: szacunek zapisujesz (link/eksport) obok ADR-a, żeby po fakcie porównać z rzeczywistością.
- **Cost Management** (w portalu) — rzeczywistość **po** wdrożeniu: analiza kosztów per zasób/grupa/tag, trendy, prognoza końca miesiąca. Pierwsze miejsce, do którego wchodzisz, gdy "coś drożeje".
- **Budżety i alerty** — budżet to kwota progowa na scope (subskrypcja/RG) + alerty e-mail przy np. 50/80/100% (także prognozowanych). **Budżet nie zatrzymuje zasobów** — tylko informuje; automatyczne reakcje musisz dobudować sam. Na koncie nauki: budżet z alertami to higiena obowiązkowa — ustaw dziś, jeśli jeszcze nie masz.
- **Tagi** — pary klucz-wartość na zasobach (`system=orders`, `env=dev`, `owner=...`, `cost-center=...`), po których Cost Management tnie raporty. Bez tagów koszt subskrypcji to jedna nieczytelna liczba. Tagi nadawaj w Bicep (lekcja 05) — otagowanie ręczne nie przeżyje pierwszego redeployu.

### Główne pułapki kosztowe

1. **Egress** — transfer danych **wychodzących** z Azure (i między regionami) jest płatny; wchodzący zwykle nie. Pułapki: architektura multi-region z ciągłą replikacją, aplikacja w regionie A z bazą w regionie B (egress + latencja), serwowanie dużych plików bez CDN.
2. **Logi w Application Insights / Log Analytics** — **klasyk numer jeden**. Płaci się za **ingest GB** logów i telemetrii; gadatliwa aplikacja z poziomem logowania Debug i bez samplingu potrafi generować koszty observability **wyższe niż koszt compute**. Mechanizmy obrony: poziomy logowania per środowisko, sampling telemetrii, retencja (ile dni trzymasz dane), capy dzienne. W lekcji 07 skonfigurujesz to świadomie.
3. **Provisioned throughput / zarezerwowana moc** — wszystko, co rezerwujesz, nalicza się 24/7 niezależnie od użycia: RU/s w Cosmosie (lekcja 03), Messaging Units Service Bus Premium, App Service Plan, vCore w SQL bez auto-pauzy, node'y AKS. Pytanie kontrolne do każdego zasobu: *"ile płacę, gdy nikt nie używa systemu?"* — i czy mnie to urządza.
4. **Zapomniane środowiska** — wyklikany "na chwilę" zasób do testów, środowisko po zakończonym POC, dysk po skasowanej maszynie. Obrona systemowa, nie siła woli: osobne resource groups per eksperyment (kasujesz jedną grupą), tagi z datą ważności, alert budżetowy na subskrypcji dev, nawyk "usuń RG" z lekcji 01–04.
5. Mniejsze, ale warte znania: publiczne adresy IP i private endpoints (grosze, ale ×N), snapshoty i backupy bez polityki retencji, środowiska non-prod w tierach produkcyjnych ("dev na Premium, bo skopiowaliśmy szablon" — patrz parametryzacja z lekcji 05).

### Architektura kosztowa: serverless vs provisioned przy różnych profilach ruchu

Najważniejsza zależność tej lekcji. Modele rozliczeń dzielą się na **pay-per-use** (Functions Consumption, Container Apps z scale-to-zero, Cosmos serverless, SQL serverless) i **provisioned** (App Service Plan, Service Bus Premium, Cosmos provisioned, AKS). Która wygrywa — decyduje **profil ruchu**:

- **Ruch sporadyczny/nocny zerowy** (wewnętrzne narzędzia, integracje, dev/test): pay-per-use wygrywa bezapelacyjnie — płacisz za minuty pracy w miesiącu, provisioned płaci za 720 godzin.
- **Ruch zmienny z wyraźnymi szczytami** (e-commerce, systemy klienckie): pay-per-use lub autoscale; provisioned wymiarowany na szczyt marnuje pieniądze poza szczytem, wymiarowany na średnią — throttluje w szczycie.
- **Ruch wysoki i stały** (przetwarzanie ciągłe, duże API): provisioned wygrywa — stawka jednostkowa pay-per-use jest wyższa (płacisz premię za elastyczność) i przy pełnym wykorzystaniu rezerwacji przestaje się opłacać. Tu wchodzą też reservations.

Wniosek architektoniczny: **istnieje punkt przecięcia** (break-even) — wolumen, od którego provisioned staje się tańszy. Architekt nie zna go "z czuja", tylko **liczy** dla swojego profilu ruchu (zrobisz to w ćwiczeniu). I drugi wniosek: scale-to-zero to nie ficzer "nice to have", tylko mechanizm, dzięki któremu architektura **śpi razem z ruchem** — w systemach o nierównym obciążeniu to największa pojedyncza dźwignia kosztowa.

Uwaga praktyczna: **nie cytuj cen z pamięci** (ani z tej lekcji — celowo nie ma tu kwot). Ceny i tiery zmieniają się często; na rozmowie podawaj rzędy wielkości i metodę liczenia, a konkretne liczby zawsze z aktualnego kalkulatora.

### Jak liczyć wariant architektury — metoda

1. **Ustal profil ruchu liczbowo**: żądań/dzień, wiadomości/dzień, średni rozmiar, godziny aktywności, GB danych i logów/miesiąc. Bez liczb nie ma kalkulacji — przyjmij założenia i **zapisz je** (porównanie wariantów jest wiarygodne tylko przy wspólnych założeniach).
2. **Rozpisz wariant na zasoby** z tierami (np. Container Apps Consumption + Service Bus Standard + SQL serverless + Log Analytics z X GB/mies).
3. **Policz w Pricing Calculator** każdy zasób; zsumuj.
4. **Policz wrażliwość**: ten sam wariant przy ruchu ×10 i ×0,1. To pokazuje, gdzie jest break-even i który wariant "skaluje się kosztowo".
5. **Dolicz koszt ludzki** jakościowo (kto to utrzymuje i ile czasu) — przynajmniej jako komentarz.

## Praktyka

- [ ] Higiena konta: utwórz budżet na swojej subskrypcji nauki z alertami (np. 50/80/100%). Dodaj standardowe tagi (`env`, `system`) do szablonów Bicep z lekcji 05.
- [ ] Przejrzyj Cost Management swojej subskrypcji za ostatni miesiąc: co kosztowało najwięcej? Czy jest tam coś, co powinno było zostać usunięte po ćwiczeniach?
- [ ] **Ćwiczenie kluczowe** — policz i udokumentuj koszty trzech wariantów aplikacji messagingowej z modułu 2, na wspólnych założeniach ruchu (przyjmij np. 100k wiadomości/dzień, API 10 req/s w godzinach 8–20, baza 10 GB; zapisz założenia!):
  - **Wariant A — minimalny serverless:** Azure Functions (Consumption) + Storage Queues + SQL serverless.
  - **Wariant B — rekomendowany:** Container Apps (Consumption, scale-to-zero dla consumera) + Service Bus Standard + SQL serverless. (To architektura lekcji 07.)
  - **Wariant C — maksymalna kontrola:** AKS (mały node pool) + self-hosted broker (np. RabbitMQ na klastrze) + SQL provisioned.

  Dla każdego: lista zasobów z tierami → kalkulacja w Pricing Calculator → suma miesięczna → koszt przy ruchu ×10 i ×0,1 → 2–3 zdania o kosztach nieujętych w fakturze (ops, ryzyko, krzywa uczenia).
- [ ] Przeanalizuj wyniki: gdzie jest break-even między B i C? Który wariant najgorzej znosi ruch ×0,1 i dlaczego? (Spodziewany wniosek: C płaci pełną stawkę nawet przy zerowym ruchu.)
- [ ] Napisz na tej bazie **post na LinkedIn**: porównanie wariantów z liczbami, założeniami i wnioskiem "co bym wybrał i kiedy zmieniłbym zdanie". To jeden z dwóch artefaktów modułu.

## Artefakt

1. **Dokument porównania kosztów trzech wariantów** (tabela: zasoby, założenia, koszt/miesiąc, koszt przy ×10 i ×0,1, koszty miękkie) — zacommitowany w repo obok ADR-ów.
2. **Post na LinkedIn** oparty na tym dokumencie — artefakt modułu z planu. Liczby + założenia + wniosek; posty z konkretnymi kalkulacjami mają nieporównanie większą siłę niż ogólniki o "optymalizacji kosztów".

## Definition of Done

- [ ] Budżet z alertami działa na twojej subskrypcji; zasoby z lekcji 05+ mają tagi nadawane z Bicep.
- [ ] Dokument trzech wariantów istnieje, ma jawne założenia i analizę wrażliwości (×10, ×0,1) — i umiesz go obronić w 5-minutowej rozmowie.
- [ ] Wymieniasz z pamięci 4+ pułapki kosztowe z mechanizmem ("logi: płacisz za ingest GB, więc Debug na prodzie to rachunek, nie tylko szum").
- [ ] Umiesz wyjaśnić break-even serverless vs provisioned i wskazać, po której stronie jest twój system — z uzasadnieniem liczbowym.
- [ ] Post opublikowany.

## Materiały

- [Microsoft Learn — Microsoft Cost Management documentation](https://learn.microsoft.com/azure/cost-management-billing/) — narzędzia: analiza kosztów, budżety, alerty, tagi.
- [Azure Well-Architected Framework — Cost Optimization pillar](https://learn.microsoft.com/azure/well-architected/cost-optimization/) — koszt jako filar projektowania; czytaj wybiórczo: principles + checklist.
