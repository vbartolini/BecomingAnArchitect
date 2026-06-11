# Lekcja 03 — Dane w Azure: SQL Database vs Cosmos DB

## Cel lekcji

Po tej lekcji rozumiesz dwa główne silniki danych Azure — Azure SQL Database (z jego modelami zakupu) i Cosmos DB (z modelem dokumentowym, partition key, RU i poziomami spójności) — i umiesz dla danego problemu wybrać: relacyjna, dokumentowa czy obie, z uzasadnieniem kosztowym.

## Dlaczego to ważne

"SQL czy Cosmos?" to w ekosystemie Microsoft odpowiednik klasycznego "SQL vs NoSQL" — i pytanie, na którym łatwo polec dwustronnie: dogmatyk relacyjny ("Cosmos to hipsterstwo") i entuzjasta NoSQL ("SQL nie skaluje") brzmią równie słabo. Architekt mówi o trade-offach: modelu spójności, wzorcach dostępu i koszcie. Do tego Cosmos DB to usługa, na której najłatwiej w całym Azure **przepalić budżet przez niezrozumienie modelu rozliczeń** — sekcja o błędach kosztowych jest tu nie mniej ważna niż teoria. Twoje 20 lat z SQL Serverem to atut: Azure SQL znasz w 80% z definicji, więc gros lekcji poświęcamy Cosmosowi.

## Teoria

### Azure SQL Database — twój SQL Server jako usługa

**Co to jest:** zarządzany silnik SQL Server (PaaS): Microsoft odpowiada za patche, backupy, HA; ty za schemat, indeksy i zapytania. Z perspektywy kodu .NET — to samo: ten sam T-SQL, EF Core, `Microsoft.Data.SqlClient`. Drobne różnice (brak cross-database queries w pojedynczej bazie, brak SQL Agent — zastępują go np. Functions z timerem) wyłapiesz w praktyce.

**Modele zakupu — pierwsza decyzja kosztowa:**

- **DTU (Database Transaction Unit)** — model historyczny: kupujesz abstrakcyjny "pakiet mocy" (mieszanka CPU/IO/pamięci) w tierach Basic/Standard/Premium. Prosty, ale nieprzejrzysty — nie wiesz, ile masz rdzeni; gdy brakuje IO, dokupujesz też CPU, czy chcesz czy nie. Spotkasz go w istniejących systemach; do nowych projektów Microsoft kieruje ku vCore.
- **vCore** — kupujesz konkretną liczbę wirtualnych rdzeni + storage osobno. Przejrzysty, porównywalny z on-premise, pozwala użyć **Azure Hybrid Benefit** (zniżka za posiadane licencje SQL Server). W ramach vCore wybierasz tier usługi: General Purpose (standard) lub Business Critical (lokalne SSD, repliki, niska latencja — znacznie drożej).
- **Serverless** (wariant vCore, General Purpose) — moc skaluje się automatycznie w zadanym przedziale vCore, rozliczanie za sekundę faktycznego użycia, a przy braku aktywności baza może się **auto-pauzować** (płacisz wtedy tylko za storage). Idealny do dev/test i obciążeń sporadycznych. Trade-off: wznowienie po pauzie trwa (cold start bazy — pierwsze połączenie po pauzie dostaje błąd lub czeka), więc dla produkcyjnego API z ciągłym ruchem wybierz provisioned.
- **Elastic pools** — wzmianka: wiele baz współdzieli jedną pulę mocy; sensowne przy modelu SaaS "baza per klient", gdzie szczyty klientów się nie nakładają. Zapamiętaj, że istnieje; wróci, jeśli kiedyś zaprojektujesz multi-tenant SaaS.

**Reguła kciuka:** nowy projekt → vCore; dev/test i ruch sporadyczny → serverless z auto-pauzą; dziesiątki małych baz → elastic pool.

### Cosmos DB — od zera

**Co to jest:** globalnie dystrybuowana, multi-model baza NoSQL Microsoftu. W praktyce używa się jej najczęściej przez **API NoSQL (dokumentowe)**: dane to dokumenty JSON w kontenerach (container ≈ tabela bez sztywnego schematu). Obietnica: latencja jednocyfrowych milisekund, elastyczna skala pozioma, replikacja na wiele regionów "jednym przełącznikiem". Cena tej obietnicy: musisz myśleć o partycjonowaniu i spójności **od pierwszego dnia** — rzeczy, które w SQL odkładasz na później, tu są decyzją projektową nr 1.

**Model dokumentowy.** Zamiast normalizować dane w tabele i składać JOIN-ami, modelujesz dokument pod **wzorzec odczytu**: zamówienie z pozycjami to jeden dokument JSON, czytany jedną operacją. Denormalizacja jest normą; JOIN-ów między dokumentami nie ma (są tylko wewnątrz dokumentu). Konsekwencja praktyczna: **najpierw spisujesz zapytania, potem projektujesz dokumenty** — odwrotnie niż w świecie relacyjnym.

**Partition key — najważniejsza decyzja.** Cosmos skaluje się przez fizyczne rozproszenie danych na partycje — dokładnie ten mechanizm, który znasz z lekcji o partycjonowaniu w module 2 i z partycji Event Hubs. Wybierasz **partition key** (np. `/customerId`): dokumenty z tą samą wartością trafiają do tej samej partycji logicznej.

- Zapytanie z wartością partition key → trafia w jedną partycję → szybkie i tanie.
- Zapytanie bez niej → **cross-partition fan-out** (odpytanie wszystkich partycji) → wolne i drogie.
- Zły klucz = **hot partition**: np. `/date` w systemie logującym — wszystkie dzisiejsze zapisy walą w jedną partycję, a reszta klastra się nudzi; limit przepustowości pojedynczej partycji staje się sufitem całego systemu.
- Klucza **nie zmienisz** po utworzeniu kontenera bez migracji danych. Dobry klucz: wysoka kardynalność + równomierny rozkład ruchu + obecny w większości zapytań. Klasyk: `/tenantId` lub `/customerId`.

**RU — jednostka kosztu.** Cosmos rozlicza się w **Request Units (RU)**: znormalizowanej walucie kosztu operacji (umownie: odczyt dokumentu 1 KB po id ≈ 1 RU; zapis ≈ 5+ RU; zapytania — zależnie od złożoności, fan-out mnoży koszt przez liczbę partycji). Płacisz za przepustowość RU/s w jednym z trybów:

- **Provisioned throughput** — rezerwujesz stałe RU/s; przekroczenie = throttling (błąd 429, SDK robi retry). Płacisz za rezerwację **niezależnie od użycia** — 24/7.
- **Autoscale** — podajesz maksimum, Cosmos skaluje w przedziale 10–100% maksimum; droższa stawka jednostkowa, ale płacisz za faktyczny szczyt godziny.
- **Serverless** — czysty pay-per-use za zużyte RU; świetny dla dev/test i małego/sporadycznego ruchu (ma limity pojemności i funkcji — zweryfikuj aktualne w dokumentacji).

Kluczowy nawyk: każda odpowiedź SDK zawiera nagłówek z **request charge** — ile RU kosztowała operacja. Architekt Cosmosa patrzy na RU zapytań tak, jak ty patrzyłeś na plany wykonania w SQL Server.

**5 poziomów spójności — najlepsza część Cosmosa dydaktycznie.** W module 2 spójność była binarna w teorii (strong vs eventual, CAP/PACELC: płacisz latencją albo świeżością danych). Cosmos daje **suwak z pięcioma pozycjami**, ustawiany per konto i osłabialny per żądanie:

| Poziom | Gwarancja (ludzkim językiem) | Cena |
|---|---|---|
| **Strong** | zawsze czytasz ostatni zatwierdzony zapis, globalnie | najwyższa latencja zapisu; w multi-region zapis czeka na repliki |
| **Bounded staleness** | odczyt spóźniony maksymalnie o K wersji lub T sekund — opóźnienie ma kontrakt | prawie-strong, nieco tańszy; dobry kompromis multi-region |
| **Session** (domyślny) | **read-your-own-writes w ramach sesji**: klient widzi własne zapisy; inni — eventual | sweet spot: rozwiązuje 90% problemów UX ("zapisałem i nie widzę") prawie za darmo |
| **Consistent prefix** | nigdy nie zobaczysz zapisów w złej kolejności (możesz widzieć stare, ale nie poprzestawiane) | tani; kolejność bez świeżości |
| **Eventual** | kiedyś się zsynchronizuje; możliwe odczyty w dowolnej kolejności | najtańszy, najniższa latencja |

Mapowanie na teorię z modułu 2: Strong = wybór C (consistency) w CAP; Eventual = wybór A/latencji; pozycje pośrodku to PACELC w praktyce — "else, czyli przy braku partycji sieci, ile latencji płacisz za ile spójności". **Session jako default** to ważna lekcja projektowa: większość systemów nie potrzebuje globalnej spójności — potrzebuje, żeby użytkownik widział własne zmiany. Bonus dla architekta: odczyty na poziomach słabszych niż Strong potrafią być tańsze w RU — spójność ma w Cosmosie dosłownie cennik.

**Jeszcze jedno: change feed.** Cosmos wystawia strumień zmian kontenera (posortowany per partycja) — naturalny zaczep pod wzorce zdarzeniowe, w tym wariant outboxa (zapis dokumentu → change feed → publikacja na Service Bus). Zapamiętaj, że istnieje.

### Kiedy relacyjna, kiedy dokumentowa, kiedy obie

**Azure SQL, gdy:** dane naturalnie relacyjne (wiele encji, więzy, raporty ad-hoc), potrzebujesz transakcji obejmujących wiele encji i JOIN-ów, zespół i narzędzia są SQL-owe, wzorce zapytań nieznane z góry (SQL wybacza — doindeksujesz później). **Domyślny wybór dla typowego systemu biznesowego .NET** — i dla projektu w lekcji 07 (m.in. dlatego, że outbox pattern z modułu 2 opiera się na transakcji SQL).

**Cosmos DB, gdy:** wzorce dostępu znane i proste (odczyt po kluczu, dokument-agregat), skala lub globalna dystrybucja realnie przekracza wygodny zasięg pojedynczego SQL-a, schemat zmienny, wymagana stała niska latencja na odczytach punktowych. Przykłady: koszyk, profile/preferencje, katalog produktów, sesje IoT.

**Obie (polyglot persistence):** częsty wzorzec dojrzałych systemów — SQL jako system of record (transakcje, raporty), Cosmos jako zoptymalizowany widok odczytowy (np. read model w CQRS), synchronizowane zdarzeniami z lekcji 02. Trade-off: dwa silniki = dwa zestawy kompetencji, backupów, kosztów i jeden nowy problem — spójność między nimi. Nie zaczynaj od polyglot; dorastaj do niego.

**Anty-wzorzec do nazwania na rozmowie:** "Cosmos, bo NoSQL jest nowocześniejszy" przy danych relacyjnych i nieznanych zapytaniach — skończy się cross-partition queries, przepalonymi RU i bólem. Symetrycznie: SQL z kolumną `Data NVARCHAR(MAX)` pełną JSON-ów odpytywanych po polach to często sygnał, że ta część danych chce być dokumentem.

### Typowe błędy kosztowe w Cosmos DB

1. **Provisioned throughput na dev/test** — zarezerwowane RU/s naliczają się 24/7, także nocą i w weekendy. Dev/test → serverless lub darmowy emulator lokalny.
2. **Cross-partition queries jako codzienność** — zły partition key lub zapytania bez niego: każde zapytanie płaci za fan-out po wszystkich partycjach. Koszt rośnie z liczbą partycji, czyli **z sukcesem produktu**.
3. **Hot partition** — wymusza podniesienie throughputu całego kontenera, żeby nakarmić jedną partycję; płacisz za moc, której 90% partycji nie używa.
4. **Throughput per kontener zamiast shared/database-level** przy wielu małych kontenerach — minimalna rezerwacja × N kontenerów potrafi zaskoczyć.
5. **Wielkie dokumenty i pełne indeksowanie** — RU zapisu rośnie z rozmiarem dokumentu i liczbą indeksowanych pól; domyślnie Cosmos indeksuje **wszystko**. Tuning indexing policy (wykluczanie nieodpytywanych ścieżek) to najtańsza optymalizacja Cosmosa.
6. **Ignorowanie request charge** — zespoły, które nie logują RU zapytań, dowiadują się o problemie z faktury. Loguj RU od pierwszego dnia.

## Praktyka

- [ ] Dla aplikacji z modułu 2 odpowiedz pisemnie: które dane są relacyjne, a które (jeśli jakieś) skorzystałyby z modelu dokumentowego? Uzasadnij wybór SQL jako bazy projektu w lekcji 07.
- [ ] Utwórz Azure SQL Database w tierze **serverless** z auto-pauzą; podłącz się z .NET, zaobserwuj zachowanie pierwszego połączenia po pauzie. Obejrzyj w portalu wykres zużycia vCore.
- [ ] Uruchom Cosmos DB (serverless lub lokalny emulator). Zaprojektuj kontener `orders` z partition key `/customerId`; zapisz ~50 dokumentów dla kilku klientów. Wykonaj i porównaj request charge: (a) point read po id+partition key, (b) zapytanie z filtrem po `customerId`, (c) zapytanie cross-partition po polu spoza klucza. Zanotuj różnice RU.
- [ ] Eksperyment ze spójnością: opisz (na bazie dokumentacji) różnicę między Session a Eventual dla scenariusza "użytkownik edytuje profil i odświeża stronę". Dlaczego Session jest domyślny?
- [ ] **Usuń resource groups po ćwiczeniach.**

## Artefakt

1. **Notatka decyzyjna** "SQL vs Cosmos vs obie" z twoimi kryteriami + tabela 5 poziomów spójności z mapowaniem na CAP/PACELC — ściągawka na rozmowy.
2. **Wyniki eksperymentu RU** (tabelka: operacja → request charge) — mocny, konkretny materiał na post LinkedIn ("Policzyłem, ile kosztuje złe zapytanie w Cosmos DB").

## Definition of Done

- [ ] Umiesz wyjaśnić DTU vs vCore vs serverless i wskazać model dla: produkcyjnego API, środowiska dev, SaaS z 40 małymi bazami.
- [ ] Umiesz opowiedzieć 5 poziomów spójności Cosmosa z przykładem użycia każdego — i powiązać je z PACELC.
- [ ] Masz zmierzone na własnym koncie request charge dla point read vs cross-partition query.
- [ ] Wymieniasz z pamięci co najmniej 4 błędy kosztowe Cosmosa i ich mechanizm.

## Materiały

- [Microsoft Learn — Partitioning and horizontal scaling in Azure Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/partitioning-overview) — najważniejszy dokument tej lekcji; przeczytaj w całości.
- [Microsoft Learn — Consistency levels in Azure Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/consistency-levels) — pięć poziomów z diagramami; skonfrontuj z tabelą z lekcji.
