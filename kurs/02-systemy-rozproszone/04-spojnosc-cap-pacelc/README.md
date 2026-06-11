# Lekcja 04 — Spójność: strong vs eventual, CAP i PACELC

## Cel lekcji

Po tej lekcji będziesz rozumiał, czym różni się strong consistency od eventual consistency (na przykładach, nie sloganach), co **naprawdę** mówi twierdzenie CAP — i jakie są jego typowe błędne odczytania — oraz dlaczego **PACELC** jest użyteczniejszym narzędziem do codziennych decyzji. Poznasz też modele spójności sesji (read-your-own-writes i spółkę), które są praktycznym językiem rozmowy o UX w systemach rozproszonych.

## Dlaczego to ważne

CAP to chyba najczęściej **błędnie** cytowane twierdzenie w branży — "wybierz dwa z trzech" usłyszysz na co drugiej rozmowie i od co drugiego kandydata. Architekt, który umie powiedzieć "CAP mówi mniej, niż się powszechnie sądzi — w praktyce ciekawszy jest trade-off latency vs consistency, czyli PACELC", natychmiast pozycjonuje się piętro wyżej. A praktycznie: każda decyzja o replikacji bazy, cache'u przed API czy poziomie spójności w Cosmos DB (moduł 3) to dokładnie ten temat. Każdy "zapisałem, odświeżyłem i nie widzę swoich danych" bug, jaki widziałeś w produkcji, to była niedomówiona spójność.

## Teoria

### Skąd w ogóle bierze się problem: replikacja

Dopóki dane żyją w jednej bazie na jednej maszynie, spójność jest "za darmo". Problem zaczyna się, gdy te same dane istnieją w **wielu kopiach** (replikach) — a istnieją prawie zawsze: replika do odczytu, standby do failovera, cache, drugi region. Kopie trzeba synchronizować, synchronizacja zabiera czas, a w tym czasie ktoś może czytać. Cała teoria spójności to odpowiedź na jedno pytanie: **co wolno zobaczyć czytelnikowi, zanim wszystkie kopie się zgodzą?**

### Strong vs eventual consistency — na przykładach

**Strong consistency** (spójność silna; ściśle: *linearizability*) — system zachowuje się tak, **jakby istniała tylko jedna kopia danych**. Gdy zapis się potwierdzi, każdy kolejny odczyt — z dowolnej repliki, dowolnego klienta — widzi nową wartość. Cena: zapis musi poczekać na synchronizację replik (wyższa latencja), a gdy repliki nie mogą się porozumieć — system musi odmówić obsługi, zamiast skłamać.

**Eventual consistency** (spójność ostateczna) — gwarancja minimalna: jeśli przestaniesz zapisywać, to **po jakimś czasie** (niezdefiniowanym!) wszystkie repliki zbiegną do tej samej wartości. Do tego momentu odczyty mogą zwracać starą wartość, a dwa kolejne odczyty mogą się "cofać w czasie" (trafisz w świeżą, potem w spóźnioną replikę).

Przykład z życia (klasyka): zmieniasz zdjęcie profilowe w serwisie społecznościowym. Ty widzisz nowe od razu, znajomy w innym kraju jeszcze przez 20 sekund stare — i nikomu nic się nie dzieje. To eventual consistency w naturalnym środowisku. Kontrprzykład: stan konta przy autoryzacji przelewu — tu odczyt spóźnionej wartości to realna strata pieniędzy; chcesz strong consistency (albo przynajmniej spójności na poziomie pojedynczego konta).

Przykład bliższy Twojej produkcji: po `OrderPlaced` (lekcja 02) usługa raportowa buduje swój widok zamówień ze zdarzeń. Między commitem w usłudze zamówień a skonsumowaniem zdarzenia raport "nie wie" o zamówieniu — **architektura zdarzeniowa jest z definicji eventually consistent**. To nie bug, to właściwość, którą trzeba zaprojektować (UI, komunikaty, SLA na opóźnienie).

### Twierdzenie CAP — co naprawdę mówi

Definicje liter (precyzyjne, bo tu rodzą się przekłamania):

- **C (Consistency)** — w sensie CAP: linearizability, czyli strong consistency jak wyżej. (Nie mylić z "C" z ACID — to zupełnie co innego.)
- **A (Availability)** — *każde* żądanie do *działającego* węzła dostaje (niebłędną) odpowiedź. Nie "system zwykle działa", tylko twarda gwarancja odpowiedzi.
- **P (Partition tolerance)** — system działa dalej mimo **partycji sieciowej** (*network partition*) — sytuacji, gdy węzły żyją, ale sieć między nimi pękła i grupy węzłów nie mogą się komunikować.

Twierdzenie CAP mówi: **w czasie partycji sieciowej nie można mieć jednocześnie C i A.** Intuicja jest prosta: pękła sieć, klient pisze do lewej połowy, drugi klient czyta z prawej. Prawa połowa ma dwa wyjścia: odpowiedzieć starymi danymi (jest A, poświęca C) albo odmówić, bo nie wie, czy dane są aktualne (jest C, poświęca A). Trzeciej opcji nie ma — i to całe twierdzenie.

**Typowe błędne odczytania (must-know na rozmowę):**

1. *"Wybierz dwa z trzech: C, A, P."* — Nie. **P nie jest do wyboru.** Partycje sieciowe w systemie rozproszonym po prostu się zdarzają (awaria switcha, przeciążony link, GC pause udający partycję) — możesz najwyżej udawać, że nie. Realny wybór brzmi: **gdy nastąpi partycja — C czy A?** "System CA" to system, który przy pierwszej partycji robi coś niezdefiniowanego.
2. *"CAP klasyfikuje bazy: Mongo jest CP, Cassandra AP…"* — Naklejki mylą, bo zachowanie zależy od konfiguracji (np. w Cassandrze poziomy spójności per zapytanie zmieniają odpowiedź) i bo wiele systemów nie daje ani twardego C, ani twardego A z definicji.
3. *"CAP opisuje normalną pracę systemu."* — Nie. CAP mówi **wyłącznie o zachowaniu podczas partycji**, czyli w trybie awaryjnym. O normalnej pracy — czyli 99,9% czasu — nie mówi **nic**. I to jest największa praktyczna słabość CAP, którą naprawia PACELC.

### PACELC — użyteczniejsze rozszerzenie

**PACELC** (Daniel Abadi, czyt. "pas-elk"): **P**artition → **A**vailability czy **C**onsistency; **E**lse (gdy partycji nie ma) → **L**atency czy **C**onsistency.

Druga połowa to brakujący kawałek CAP: nawet w zdrowym systemie replikacja kosztuje czas. Chcesz strong consistency? Zapis musi być potwierdzony przez repliki (np. kworum), a odczyt musi trafić w aktualną kopię — płacisz **latencją** każdego żądania, codziennie. Godzisz się na eventual consistency? Zapis potwierdzasz lokalnie, replikujesz w tle — szybko, ale czytelnicy mogą widzieć starość. **Latency vs consistency to trade-off, który płacisz zawsze, nie tylko podczas awarii** — dlatego PACELC jest lepszym językiem codziennych decyzji niż CAP.

Notacja: system "PA/EL" (np. Dynamo/Cassandra w typowej konfiguracji) wybiera dostępność przy partycji i niską latencję na co dzień; "PC/EC" (np. systemy oparte o konsensus, jak etcd) — spójność w obu sytuacjach. Cosmos DB pozwala to wybrać pokrętłem (o czym niżej).

### Modele spójności sesji — praktyczny środek skali

Między strong a eventual jest całe spektrum gwarancji "sesyjnych" — słabszych niż strong (tanich), ale ratujących UX. Definiują, co widzi **konkretny użytkownik/sesja**, nie wszyscy naraz:

- **Read-your-own-writes** (czytaj własne zapisy) — po swoim zapisie użytkownik zawsze widzi jego skutek. Klasyczny bug bez tej gwarancji: edytujesz profil, klikasz zapisz, strona się odświeża z repliki do odczytu… i widzisz stare dane. Implementacje: czytaj z primary przez X sekund po własnym zapisie; przypnij sesję do repliki; porównuj wersję/timestamp.
- **Monotonic reads** (odczyty monotoniczne) — czas się nie cofa: skoro raz zobaczyłeś wartość z chwili T, nie zobaczysz już starszej. Bez tego: odświeżasz listę i komentarz znika, potem wraca (kolejne żądania trafiają w różnie spóźnione repliki). Implementacja: sticky routing sesji do jednej repliki.
- **Consistent prefix / causal consistency** — skutek nie wyprzedza przyczyny: nie zobaczysz odpowiedzi na pytanie przed samym pytaniem.

Te nazwy to złoto na rozmowach, bo pokazują myślenie o spójności **per wymaganie biznesowe**, a nie zero-jedynkowo.

### Jak to się przekłada na praktykę

- **Replika do odczytu w SQL** (Azure SQL read replica, klasyczny read-scale-out): w momencie, gdy kierujesz odczyty na replikę, **wybrałeś eventual consistency** dla tych odczytów — nawet jeśli nikt w zespole nie wypowiedział tego zdania. Pytanie kontrolne do każdego takiego designu: "które ekrany wymagają read-your-own-writes i jak to zapewnimy?".
- **Cache** (Redis przed API): cache to replika z TTL — każde cache'owanie to decyzja o spójności. "Jak długo użytkownik może widzieć starą cenę?" to pytanie biznesowe, nie techniczne; TTL jest tylko jego implementacją.
- **Cosmos DB** — rzadki przypadek bazy z jawnie nazwanym pokrętłem: pięć poziomów spójności (strong / bounded staleness / **session** / consistent prefix / eventual), gdzie session = read-your-own-writes w obrębie sesji i jest domyślna — dokładnie modele z tej lekcji, sprzedawane jako feature. Szczegóły w module 3; tu wystarczy wiedzieć, że ta lekcja to teoria pod tamto pokrętło.
- **Messaging z lekcji 01–03**: cały model outbox + zdarzenia + sagi zakłada eventual consistency między usługami. Strong consistency między mikroserwisami praktycznie nie istnieje — granica usługi to granica spójności (kolejny argument za dobrym wyznaczaniem granic).

### Trade-offy i "kiedy czego NIE używać"

- Nie żądaj strong consistency "na wszelki wypadek" — płacisz latencją i dostępnością za gwarancję, której większość odczytów nie potrzebuje. Pytaj o **koszt biznesowy spóźnionego odczytu** dla konkretnego ekranu/operacji.
- Nie wciskaj eventual consistency tam, gdzie spóźniony odczyt = pieniądze lub bezpieczeństwo (autoryzacja, saldo, limity) — albo tam, gdzie koszt obsługi anomalii w UX przewyższa oszczędność.
- Najczęściej właściwa odpowiedź jest **mieszana per operacja**: strong na ścieżce zapisu i krytycznych odczytach, session na ekranach "moje dane", eventual na listach, raportach i wyszukiwaniu.

## Praktyka

- [ ] Zbuduj **tabelę decyzyjną spójności** dla znanego Ci systemu (rezerwacje z Aerotunel albo członkostwa z Perfect Gym): 8–12 operacji (odczyty i zapisy) × kolumny: wymagany model (strong / read-your-own-writes / monotonic / eventual), koszt biznesowy spóźnionego odczytu, jak zapewnić. To jest główne ćwiczenie lekcji.
- [ ] Opisz słowami (3–4 zdania każde) dwa scenariusze partycji: (a) system wybiera C — co widzi użytkownik? (b) system wybiera A — co widzi i co trzeba potem posprzątać?
- [ ] Sklasyfikuj w notacji PACELC trzy systemy, z którymi pracowałeś (np. SQL Server z repliką, Redis cache, broker + konsumenci) — z jednym zdaniem uzasadnienia.
- [ ] Pytanie kontrolne (odpowiedz pisemnie): dlaczego "wybierz dwa z trzech" jest błędnym odczytaniem CAP?

## Artefakt

Folder `04-spojnosc-cap-pacelc/` w repo nauki: tabela decyzyjna spójności + szkic posta na LinkedIn **"CAP mówi mniej, niż myślisz"** (struktura: popularny slogan → co twierdzenie mówi naprawdę → PACELC → przykład tabeli decyzyjnej z prawdziwego systemu). Tabela decyzyjna to też gotowy rekwizyt na system design interview.

## Definition of Done

- [ ] Umiesz bez notatek powiedzieć, co CAP mówi naprawdę (jedno zdanie o zachowaniu podczas partycji) i wymienić dwa błędne odczytania.
- [ ] Umiesz rozwinąć PACELC i wyjaśnić, czemu "E" jest praktycznie ważniejsze od "P".
- [ ] Na przykładzie "zapisałem profil i nie widzę zmian" umiesz nazwać brakującą gwarancję i podać dwie implementacje naprawy.
- [ ] Tabela decyzyjna istnieje i ma niejednorodne odpowiedzi (jeśli wszędzie wyszło "strong" albo wszędzie "eventual" — wróć i policz koszty jeszcze raz).

## Materiały

1. **"Designing Data-Intensive Applications"**, M. Kleppmann — rozdz. 5 ("Replication", tam modele sesji) + rozdz. 9 ("Consistency and Consensus", tam uczciwa krytyka CAP).
2. **Daniel Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design"** (IEEE Computer, 2012) — oryginalny artykuł o PACELC, krótki i przystępny.
