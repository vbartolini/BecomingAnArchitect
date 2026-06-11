# Lekcja 01 — Atrybuty jakościowe ("-ilities") i ich konflikty

## Cel lekcji

Po tej lekcji będziesz umiał nazwać i zmierzyć sześć kluczowych atrybutów jakościowych systemu (scalability, availability, maintainability, security, performance, observability), wyjaśnić, czym są SLA/SLO/SLI — a przede wszystkim pokazać, **jak te atrybuty ze sobą konfliktują** i dlaczego "chcemy wszystkiego" nie jest wymaganiem, tylko brakiem decyzji.

## Dlaczego to ważne

Na rozmowach na role architektoniczne pytanie "zaprojektuj system X" prawie nigdy nie jest pytaniem o funkcje. Jest pytaniem o to, czy zaczniesz od: *"Zanim narysuję cokolwiek — jakie są wymagania niefunkcjonalne? Ilu użytkowników? Jaka tolerancja na niedostępność?"*. Kandydat, który od razu rysuje kafelki, odpada na starcie.

W realnej pracy atrybuty jakościowe to język, którym architekt rozmawia z biznesem. Biznes nie rozumie "Kafka vs RabbitMQ", ale rozumie "jeśli chcemy, żeby system działał także podczas awarii data center, to kosztuje X i spowalnia development o Y". Znasz to z produkcji: w systemie dla siłowni bramka wejściowa **musi** otworzyć się w sekundę nawet gdy moduł płatności leży — to jest decyzja o atrybutach jakościowych, nawet jeśli nikt jej tak wtedy nie nazywał.

## Teoria

### Czym jest atrybut jakościowy

**Atrybut jakościowy** (ang. *quality attribute*, potocznie *"-ility"*, po polsku też "wymaganie niefunkcjonalne") to cecha systemu opisująca **jak** system działa, a nie **co** robi. "Członek klubu może zarezerwować zajęcia" — to funkcja. "Rezerwacja działa, gdy 5000 osób rzuci się na grafik o 6:00 rano" — to atrybut jakościowy (scalability + availability).

Kluczowa obserwacja Richardsa i Forda z *Fundamentals of Software Architecture*: **architektura to w praktyce struktura systemu + atrybuty jakościowe, które ta struktura wspiera + decyzje, które do niej doprowadziły**. Funkcje można dodać do prawie każdej architektury. Atrybutów jakościowych nie da się "dokleić później" — są konsekwencją struktury. Nie dorobisz wysokiej dostępności do systemu, który ma jedną bazę danych jako pojedynczy punkt awarii, tak jak nie dorobisz fundamentów do stojącego domu.

Druga kluczowa obserwacja: **nie wybiera się wszystkich atrybutów**. Każdy wspierany atrybut kosztuje (pieniądze, złożoność, czas developmentu) i zwykle pogarsza inny atrybut. Architekt, który mówi "system będzie skalowalny, bezpieczny, szybki, tani i prosty w utrzymaniu", mówi w istocie: "nie podjąłem żadnej decyzji".

### Sześć atrybutów, które musisz znać od podszewki

#### 1. Scalability (skalowalność)

Zdolność systemu do obsługi **rosnącego obciążenia** bez degradacji działania. Uwaga na częste pomylenie: skalowalność to nie "system jest szybki", tylko "system pozostaje wystarczająco szybki, gdy obciążenie rośnie 10×".

Dwa podstawowe kierunki:
- **Vertical scaling (scale up)** — mocniejsza maszyna. Proste, ale ma sufit i pojedynczy punkt awarii.
- **Horizontal scaling (scale out)** — więcej maszyn. Bez sufitu, ale wymusza pytania: gdzie jest stan sesji? jak rozdzielać ruch? co z bazą danych, która nagle jest wspólnym wąskim gardłem?

Jak mierzyć: przepustowość (requests/s, rezerwacje/min) przy zadanym percentylu czasu odpowiedzi. Np. "system obsługuje 500 rezerwacji/min z p95 < 800 ms" (p95 = 95% żądań mieści się w tym czasie; percentyle są uczciwsze niż średnia, bo średnią zaniżają tysiące tanich żądań, a użytkownik pamięta te najwolniejsze).

Blizna z produkcji: poranny szczyt w klubach fitness — wszyscy rezerwują zajęcia po otwarciu grafiku. To klasyczny **spike load**: średnie obciążenie niskie, szczyt 50× wyższy. Skalowanie "pod średnią" tu nie działa.

#### 2. Availability (dostępność)

Procent czasu, w którym system **odpowiada poprawnie**. Wyraża się w "dziewiątkach":

| Dostępność | Niedostępność rocznie | Komentarz |
|---|---|---|
| 99% ("dwie dziewiątki") | ~3,65 dnia | OK dla narzędzi wewnętrznych |
| 99,9% | ~8,8 godziny | typowy SaaS |
| 99,99% | ~53 minuty | wymaga redundancji wszystkiego i automatycznego failover |
| 99,999% | ~5 minut | telekomy, płatności; bardzo drogie |

Każda kolejna dziewiątka kosztuje **mniej więcej rząd wielkości więcej** — bo wymaga eliminacji kolejnej klasy pojedynczych punktów awarii (ang. *single point of failure*, SPOF): druga instancja aplikacji, replika bazy, drugi region, redundantny load balancer…

Ważne rozróżnienie: dostępność **całego systemu** vs **krytycznej ścieżki**. Bramka wejściowa do klubu musi mieć więcej dziewiątek niż raport miesięczny dla managera. Dobra architektura pozwala różnym częściom systemu mieć różną dostępność — to tańsze niż "wszystko na 99,99%".

#### 3. Performance (wydajność)

Jak szybko system odpowiada przy **normalnym** obciążeniu. Dwie miary, których nie wolno mylić:
- **Latency (opóźnienie)** — czas pojedynczej operacji ("rezerwacja trwa 300 ms"),
- **Throughput (przepustowość)** — ile operacji na jednostkę czasu ("1000 rezerwacji/min").

Można mieć świetny throughput i fatalną latency (przetwarzanie wsadowe: milion rekordów na godzinę, ale każdy czeka w kolejce 40 minut) i odwrotnie. Architektura często wymienia jedno na drugie — np. kolejka (jak te, które znasz z integracji) poprawia throughput i odporność na szczyty, ale dodaje latency i zmienia operację z synchronicznej na "przyjąłem, zrobię".

#### 4. Security (bezpieczeństwo)

Ochrona danych i operacji przed niepowołanym dostępem i modyfikacją. Dla architekta to nie "dodamy HTTPS", tylko decyzje strukturalne: gdzie jest uwierzytelnianie (ang. *authentication* — "kim jesteś") i autoryzacja (ang. *authorization* — "co ci wolno"), jak izolowane są dane różnych klientów (w systemie multi-tenant dla sieci siłowni: czy klub A może przez błąd w zapytaniu zobaczyć członków klubu B?), co jest szyfrowane w spoczynku i w tranzycie, jak wygląda audyt.

Jak mierzyć: trudniej niż inne atrybuty — zwykle przez zgodność (OWASP ASVS, RODO/GDPR), wyniki pentestów, czas wykrycia i reakcji na incydent.

#### 5. Maintainability (utrzymywalność)

Jak łatwo system **zmieniać**: dodawać funkcje, naprawiać błędy, podmieniać komponenty. To atrybut najbardziej niedoceniany na rozmowach i najdroższy w praktyce — bo 80%+ kosztu systemu to lata utrzymania, nie pierwsze wdrożenie.

Składowe: modularność (czy zmiana w module płatności może zepsuć rezerwacje?), testowalność (ile trwa pewność, że nic nie zepsułem?), czytelność, łatwość wdrożenia (ang. *deployability* — czas od commita do produkcji).

Jak mierzyć pośrednio: lead time zmiany (od decyzji do produkcji), częstotliwość wdrożeń, odsetek wdrożeń powodujących awarię (to metryki DORA — branżowy standard mierzenia zdolności wytwórczej zespołów).

#### 6. Observability (obserwowalność)

Zdolność do **zrozumienia, co dzieje się wewnątrz systemu, na podstawie tego, co system emituje na zewnątrz** — bez dokładania kodu i bez debuggera na produkcji. Trzy filary:
- **Logi** — co się wydarzyło (zdarzenia dyskretne),
- **Metryki** — ile/jak szybko (liczby w czasie: CPU, p95, długość kolejki),
- **Trace'y (distributed tracing)** — droga pojedynczego żądania przez wiele komponentów ("rezerwacja przeszła przez API → serwis grafiku → kolejkę → serwis płatności i utknęła tu, na 4 sekundy").

Obserwowalność staje się atrybutem krytycznym dokładnie w momencie, gdy system przestaje być jednym procesem. Znasz ten ból: wiadomość "zniknęła" gdzieś między kolejką a konsumentem i bez korelacji logów szukasz jej godzinami. To był deficyt obserwowalności, nie pech.

### SLA, SLO, SLI — jak mówić o jakości liczbami

Te trzy skróty to standardowy język mierzenia atrybutów (spopularyzowany przez Google SRE). Od zera:

- **SLI** (*Service Level Indicator*) — **wskaźnik**: konkretna mierzona wielkość. Np. "odsetek żądań HTTP zakończonych kodem 2xx/3xx w czasie < 500 ms, liczony w oknie 5 minut". SLI to termometr.
- **SLO** (*Service Level Objective*) — **cel wewnętrzny**: jaką wartość SLI chcemy utrzymywać. Np. "99,9% żądań spełnia powyższe kryterium w oknie 30 dni". SLO to temperatura, poniżej której zespół zaczyna działać.
- **SLA** (*Service Level Agreement*) — **umowa z klientem, z konsekwencjami** (kary, rabaty), zwykle łagodniejsza niż SLO, żeby mieć margines. Np. SLA 99,5% przy wewnętrznym SLO 99,9%.

Analogia: SLI = prędkościomierz, SLO = "jeżdżę maks 120, żeby mieć zapas", SLA = "ograniczenie 140, za przekroczenie mandat".

Z SLO wynika pojęcie **error budget** (budżet błędów): SLO 99,9% oznacza, że ~43 minuty niedostępności miesięcznie są *dozwolone*. Dopóki budżet nie jest wyczerpany — wdrażamy odważnie. Wyczerpany — stop, stabilizujemy. To genialne narzędzie, bo zamienia jałowy spór "stabilność vs tempo zmian" w arytmetykę.

### Najważniejsze: atrybuty KONFLIKTUJĄ

To jest sedno lekcji. Atrybuty jakościowe nie są listą życzeń — są **systemem naczyń połączonych**. Podnosisz jeden, inny spada. Kilka kanonicznych konfliktów:

**Availability vs Consistency (spójność danych).** W systemie rozproszonym, gdy sieć między węzłami zawiedzie, musisz wybrać: odpowiadać dalej (ryzykując nieaktualne dane) albo odmówić odpowiedzi (tracąc dostępność). To twierdzenie CAP — w module 2 rozbierzemy je formalnie; na razie intuicja: zajęcia mają 1 wolne miejsce, dwie osoby klikają "rezerwuję" na dwóch węzłach, które chwilowo się nie widzą. Albo jeden dostanie błąd (spójność kosztem dostępności), albo oboje "sukces" i konflikt rozwiążesz później (dostępność kosztem spójności — i ktoś dostanie maila z przeprosinami). Nie ma trzeciej opcji.

**Security vs Usability (i performance).** Każda warstwa bezpieczeństwa to tarcie: MFA wydłuża logowanie, krótkie sesje wylogowują recepcję w środku obsługi klienta, szyfrowanie pól utrudnia wyszukiwanie. System "maksymalnie bezpieczny" to system, z którego nikt nie chce korzystać — a wtedy użytkownicy zaczynają obchodzić zabezpieczenia (karteczka z hasłem na monitorze recepcji), co per saldo **obniża** bezpieczeństwo.

**Scalability vs Simplicity (a więc i maintainability).** Skalowanie horyzontalne wymusza: bezstanowość, cache rozproszony, kolejki, czasem sharding bazy. Każdy z tych elementów to złożoność, którą ktoś będzie utrzymywał przez lata. Skalowalność kupuje się walutą prostoty.

**Performance vs Maintainability.** Denormalizacja, cache, ręcznie strojone SQL-e, rezygnacja z warstw abstrakcji — wszystko to przyspiesza system i spowalnia jego zmienianie.

**Consistency vs Performance.** Transakcja obejmująca wiele zasobów (albo synchroniczna replikacja) daje gwarancje, ale trzyma blokady i czeka na najwolniejszy element. Stąd cała rodzina wzorców "eventual consistency" (spójność ostateczna — dane *dojdą* do zgodności, ale nie natychmiast), które poznasz w module 2.

Praktyczny wniosek: **pytanie architektoniczne nigdy nie brzmi "czy chcemy X?", tylko "ile Y jesteśmy gotowi oddać za X?"**. Dlatego pierwszym krokiem każdego projektu jest wybór 2–3 atrybutów **priorytetowych** — takich, dla których świadomie poświęcimy pozostałe. Richards i Ford nazywają to wprost: dążenie do "najlepszej architektury" jest błędem; istnieje tylko **least worst architecture** dla danego zestawu priorytetów.

### Kiedy NIE drążyć atrybutów jakościowych

Pytanie kontrolne każdej lekcji: kiedy tego nie używać? Jeśli budujesz prototyp na 2 tygodnie, narzędzie wewnętrzne dla 5 osób albo walidujesz pomysł biznesowy — formalna analiza atrybutów to przerost formy. Wystarczy jedno zdanie: "optymalizuję pod szybkość iteracji, świadomie ignoruję resztę". To też jest decyzja architektoniczna — i lepsza niż udawanie, że MVP potrzebuje 4 dziewiątek.

## Praktyka

- [ ] Spisz definicje 6 atrybutów własnymi słowami (po 1–2 zdania), bez zaglądania do lekcji. Porównaj z treścią — różnice pokazują, czego nie zrozumiałeś.
- [ ] **Ćwiczenie główne:** wybierz 3 najważniejsze atrybuty jakościowe dla dwóch systemów, które znasz najlepiej: (a) platforma do zarządzania siłowniami klasy Perfect Gym, (b) DayChunks. Dla każdego atrybutu napisz: dlaczego jest w top-3, jaki atrybut świadomie poświęcasz na jego rzecz i jak byś go zmierzył (zaproponuj konkretny SLI). Zapisz jako tabelę w repo nauki.
- [ ] Dla systemu z siłowniami rozpisz dostępność per ścieżka: bramka wejściowa, rezerwacja zajęć, płatności cykliczne, raporty. Przypisz każdej liczbę dziewiątek i uzasadnij, dlaczego nie wszystkie potrzebują tyle samo.
- [ ] Przypomnij sobie jedną realną awarię lub problem z produkcji (Perfect Gym, Aerotunel, kolejki) i nazwij go językiem tej lekcji: który atrybut zawiódł, jaki SLI by go wcześnie wykrył, jaki konflikt atrybutów był u źródła.

## Artefakt

Plik `atrybuty-perfect-gym-daychunks.md` w repo nauki: dwie tabele (po jednej na system) z kolumnami: **atrybut | dlaczego top-3 | co poświęcam | proponowany SLI**. Plus akapit o awarii z produkcji opisanej językiem atrybutów. Ten artefakt to surowiec na przyszły post LinkedIn ("3 atrybuty, które naprawdę liczyły się w systemie dla 1000+ siłowni").

## Definition of Done

- [ ] Umiesz z pamięci wyjaśnić każdy z 6 atrybutów i podać sposób jego pomiaru.
- [ ] Umiesz wyjaśnić różnicę SLI/SLO/SLA i pojęcie error budget na własnym przykładzie.
- [ ] Umiesz wymienić minimum 3 pary konfliktujących atrybutów z konkretnym scenariuszem, gdzie konflikt się materializuje.
- [ ] Artefakt jest w repo i dla każdego atrybutu w top-3 jest jasno nazwane, **co poświęcasz** — jeśli ta kolumna jest pusta, ćwiczenie nie jest zrobione.

## Materiały

1. **"Fundamentals of Software Architecture"** (Richards, Ford) — rozdz. 4–7 (architecture characteristics: definicja, identyfikacja, pomiar). Czytaj z ołówkiem: przy każdej charakterystyce notuj przykład z własnej kariery.
2. **Google SRE Book** — rozdział *"Service Level Objectives"* (dostępny bezpłatnie: sre.google/sre-book/service-level-objectives/) — kanoniczne źródło SLI/SLO/SLA i error budget.
