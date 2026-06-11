# Lekcja 04 — ADR: Architecture Decision Records

## Cel lekcji

Po tej lekcji będziesz umiał pisać ADR-y w formacie Nygarda: wiedzieć, **które** decyzje dokumentować, jak prowadzić cykl życia ADR-a (proposed → accepted → superseded), gdzie go trzymać — i będziesz miał za sobą pierwsze trzy napisane ADR-y.

## Dlaczego to ważne

Drugie prawo architektury (lekcja 02): *why is more important than how*. Kod doskonale dokumentuje "jak". **Nic** poza ADR-em nie dokumentuje "dlaczego" — a to "dlaczego" wyparowuje z organizacji w tempie rotacji ludzi.

Znasz ten scenariusz z każdego dojrzałego systemu: ktoś znajduje w kodzie dziwactwo ("czemu tu jest własna kolejka w tabeli SQL zamiast brokera?!"), uznaje je za głupotę poprzedników, "naprawia" — i odkrywa na produkcji powód, dla którego było jak było. ADR to szczepionka na ten cykl. Koszt: 30–60 minut pisania. Stąd teza twojego przyszłego posta: **najtańsze narzędzie architekta, którego prawie nikt nie używa**.

Na rozmowach rekrutacyjnych ADR działa podwójnie: po pierwsze pytanie "jak dokumentujecie decyzje?" pada wprost na rolach senior+; po drugie sam trening pisania ADR-ów to trening odpowiadania na "why not X?" — bo dobry ADR zawiera odrzucone opcje.

## Teoria

### Czym jest ADR

**ADR** (*Architecture Decision Record*) to **krótki dokument tekstowy opisujący jedną znaczącą decyzję architektoniczną**: jej kontekst, treść i konsekwencje. Format zaproponował Michael Nygard (autor "Release It!") w blogonotce z 2011 — i celowo jest minimalistyczny: jedna–dwie strony, zwykły Markdown, w repozytorium kodu.

ADR **nie jest**: specyfikacją, dokumentacją systemu, opisem "jak działa moduł". Jest zapisem **momentu wyboru**: co wiedzieliśmy, co rozważaliśmy, co wybraliśmy, co nas to kosztuje. Kluczowa cecha: ADR jest **niemutowalny** (ang. *immutable*) — zaakceptowanego ADR-a nie edytuje się, gdy zmieniamy zdanie. Pisze się **nowy**, który zastępuje stary (o statusach za chwilę). Dzięki temu zbiór ADR-ów czyta się jak dziennik pokładowy projektu: widać nie tylko stan, ale i **trajektorię** myślenia. To dokładnie ten format, który trenowałeś na sobie w module 0 (`notes/decisions-log.md`) — teraz formalizujemy go do standardu branżowego.

### Format Nygarda — sekcja po sekcji

```markdown
# ADR-NNN: Tytuł będący decyzją

## Status
Proposed | Accepted | Superseded by ADR-MMM | Deprecated

## Context  (Kontekst)
## Decision (Decyzja)
## Consequences (Konsekwencje)
```

**Tytuł** — pełne zdanie z decyzją, nie temat. Źle: "Kolejki". Dobrze: "ADR-007: Powiadomienia o odwołanych zajęciach idą przez kolejkę z workerem, nie synchronicznie z API". Tytuł ma działać jak nagłówek prasowy: po samym spisie treści katalogu ADR-ów da się zrekonstruować architekturę.

**Status** — gdzie decyzja jest w cyklu życia (sekcja niżej).

**Context** — siły działające na decyzję: fakty techniczne, wymagania (atrybuty jakościowe! — lekcja 01), ograniczenia biznesowe, polityczne i zespołowe — **opisane neutralnie**, bez sprzedawania rozwiązania. Test jakości: czy ktoś, kto przeczyta sam Context, zrozumie, czemu w ogóle trzeba było decydować? Tu też wypisujesz **rozważane opcje** z krótkim powodem odrzucenia (formalnie Nygard ich nie wymaga, w praktyce to najcenniejszy fragment — odpowiedź na przyszłe "why not X?"; to skrócona macierz z lekcji 02).

**Decision** — sama decyzja, czas dokonany, strona czynna: "Będziemy…", "Wybieramy…". Krótko. Z uzasadnieniem wiążącym decyzję z kontekstem.

**Consequences** — co staje się prawdą po decyzji: **dobre, złe i neutralne**. To druga sekcja, po której poznaje się dojrzałość autora: ADR bez negatywnych konsekwencji to materiał marketingowy, nie ADR. Tu lądują rezygnacje z lekcji 02 ("płacimy Y") i warunki rewizji ("wrócimy do tematu, jeśli Z").

### Statusy i cykl życia

- **Proposed** — propozycja na stole, do dyskusji/recenzji. ADR może żyć w tym statusie przez review pull requesta.
- **Accepted** — decyzja obowiązuje. Od teraz dokument jest niemutowalny (poza zmianą statusu).
- **Superseded by ADR-NNN** — decyzję zastąpiła nowsza. Stary ADR **zostaje w repo** z odnośnikiem do następcy; nowy linkuje wstecz ("Supersedes ADR-007"). Historia myślenia jest wartością — nie kasuj jej.
- **Deprecated** — decyzja przestała obowiązywać, ale nic jej nie zastąpiło (np. wycofano cały feature).

Spotyka się też "Rejected" — propozycja odrzucona po dyskusji. Wbrew intuicji warto ją zachować: "rozważaliśmy i odrzuciliśmy" to wiedza, która zaoszczędzi komuś powtórki tej samej dyskusji za dwa lata.

### Kiedy pisać ADR — a kiedy nie

Pisz, gdy decyzja jest **architektonicznie znacząca** (ang. *architecturally significant*): wpływa na strukturę, atrybuty jakościowe, zewnętrzne zależności, interfejsy między zespołami/modułami — albo jest **drogo odwracalna** (one-way door z lekcji 02; to najlepsza pojedyncza heurystyka). Przykłady: wybór bazy/brokera, model multi-tenancy, granice modułów, styl komunikacji (sync/async), strategia wersjonowania publicznego API, "budujemy vs kupujemy".

**Nie pisz**, gdy: decyzja jest tanio odwracalna i lokalna (nazewnictwo, wybór drobnej biblioteki — od tego masz linijkę w dzienniku decyzji); gdy "decyzja" jest standardem zespołu opisanym gdzie indziej (coding guidelines); gdy próbujesz ADR-em opisać **jak działa system** (od tego jest C4 i dokumentacja). Sygnał ostrzegawczy przerostu: jeśli piszesz więcej niż ~1–2 ADR-y tygodniowo przez dłuższy czas, prawdopodobnie dokumentujesz decyzje niearchitektoniczne i format umrze ze zmęczenia.

Wariant specjalny: **ADR pisany wstecz** (retroaktywny) — dokumentujesz decyzję podjętą dawno temu, bo system na niej stoi, a "dlaczego" istnieje tylko w głowach. Wartościowe przy wdrażaniu ADR-ów w istniejącym projekcie: 3–5 retroaktywnych ADR-ów dla decyzji nośnych daje nowym ludziom mapę szybciej niż tygodnie onboardingu. Dokładnie to przećwiczysz.

### Gdzie trzymać

**W repozytorium kodu**, katalog `docs/adr/` (lub `docs/decisions/`), pliki `NNNN-tytul-kebab-case.md`, numeracja narastająca, nigdy nie reużywana. Dlaczego repo, a nie Confluence/SharePoint: (1) ADR przechodzi **code review jak kod** — pull request z ADR-em w statusie Proposed to naturalny mechanizm dyskusji i akceptacji; (2) wersjonowanie i blame za darmo; (3) developer znajdzie go tam, gdzie pracuje — dokumentacja oddalona od kodu umiera pierwsza. Jeśli decyzja obejmuje wiele repozytoriów — osobne repo `architecture/` albo repo systemu "wiodącego", z linkami. Istnieją narzędzia CLI (adr-tools) — miłe, niekonieczne; szablon w Markdownie wystarcza.

### Pełny przykładowy ADR

```markdown
# ADR-001: DayChunks działa w 100% lokalnie, bez własnego backendu

## Status
Accepted

## Context
DayChunks to aplikacja do planowania dnia "chunkami", rozwijana przez
jedną osobę po godzinach. Kluczowe siły:
- Budżet czasowy: kilka godzin tygodniowo. Każda godzina utrzymania
  infrastruktury to godzina zabrana funkcjom.
- Budżet finansowy: ~0 zł stałych kosztów do czasu przychodów.
- Dane użytkownika to prywatny plan dnia — wrażliwe; przechowywanie
  ich na serwerze tworzy obowiązki (RODO, backupy, bezpieczeństwo).
- Doświadczenie autora z systemów produkcyjnych: backend = dyżury,
  monitoring, awarie o 6 rano. Świadomie NIE chcę tej klasy problemów
  w projekcie hobbystycznym.
- Ryzyko biznesowe nr 1 to brak użytkowników, nie brak skalowalności.

Rozważane opcje:
1. Klasyczny backend (ASP.NET Core + SQL na Azure) — odrzucone:
   stały koszt + utrzymanie nieproporcjonalne do etapu produktu.
2. Backend-as-a-Service (np. Supabase/Firebase) — odrzucone NA TERAZ:
   tańsze od opcji 1, ale nadal konto, wendor, sync do utrzymania;
   główny argument "za" (synchronizacja między urządzeniami) nie jest
   potwierdzoną potrzebą użytkowników.
3. Brak backendu: dane lokalnie na urządzeniu — wybrane.

## Decision
DayChunks przechowuje wszystkie dane wyłącznie lokalnie na urządzeniu
użytkownika. Nie budujemy backendu, kont użytkowników ani synchronizacji.
Eksport/import danych do pliku jest jedyną formą przenoszenia danych
i backupu — świadomie przerzuconą na użytkownika.

## Consequences
Pozytywne:
- Zero kosztów stałych i zero utrzymania serwerów; całość czasu idzie
  w produkt.
- Aplikacja działa offline z definicji; brak całych klas awarii.
- Prywatność jako cecha produktu i argument marketingowy.
Negatywne:
- Brak synchronizacji między urządzeniami — najczęstsze możliwe
  życzenie użytkowników; jego realizacja później będzie droga
  (model danych musi od dziś być projektowany pod przyszły sync,
  np. identyfikatory niezależne od urządzenia).
- Brak telemetrii po stronie serwera — mniej wiemy o użyciu.
- Utrata urządzenia bez eksportu = utrata danych użytkownika.
Warunek rewizji: powtarzające się prośby użytkowników o sync
na 2+ urządzeniach lub model monetyzacji wymagający kont.
```

Zwróć uwagę: negatywne konsekwencje są konkretne i bolesne, odrzucone opcje mają powody, jest warunek rewizji. Po tym ADR-ze nikt za 3 lata nie powie "autor nie pomyślał o synchronizacji" — pomyślał i **zapłacił świadomie**.

### Kiedy ADR-y NIE zadziałają (uczciwie)

ADR-y umierają w organizacjach, gdzie: nikt ich nie czyta przy podejmowaniu kolejnych decyzji (pisanie do szuflady), są pisane po fakcie jako alibi, albo wymaganie ADR-a stało się biurokratyczną bramką dla każdej pierdoły. Lekarstwo jest kulturowe, nie techniczne: ADR czytany na onboardingu, linkowany w dyskusjach ("to już rozstrzygnięte w ADR-012, masz nowe argumenty?") i pisany tylko dla decyzji znaczących.

## Praktyka

- [ ] Przeczytaj oryginalną notkę Nygarda "Documenting Architecture Decisions" (cognitect.com/blog/2011/11/15/documenting-architecture-decisions) — 10 minut, źródło pierwotne.
- [ ] Załóż w repo nauki katalog `docs/adr/` z plikiem `template.md` (format Nygarda + sekcja opcji odrzuconych i warunku rewizji).
- [ ] **Ćwiczenie główne — 3 ADR-y wstecz** dla decyzji z własnej kariery:
  - [ ] ADR "DayChunks bez backendu" (możesz wyjść od przykładu z lekcji, ale przepisz go po swojemu — twoje powody, twoje liczby).
  - [ ] Jedna decyzja z Perfect Gym lub Aerotunelu, która **okazała się słuszna** (np. wybór sposobu integracji, kolejka w miejscu X) — udokumentuj kontekst, który ją uzasadniał.
  - [ ] Jedna decyzja, która **okazała się bolesna** — napisz ADR tak, jak powinien był wyglądać wtedy: z konsekwencjami negatywnymi, które wówczas zignorowano. To najbardziej pouczające z trzech ćwiczeń.
- [ ] Przepisz analizę trade-offów z lekcji 02 (artefakt `analiza-trade-off-01.md`) do formatu ADR — zobacz, jak macierz opcji kompresuje się do sekcji Context.
- [ ] Zanotuj szkic posta LinkedIn "ADR — najtańsze narzędzie architekta, którego prawie nikt nie używa": teza, twój przykład bolesnej decyzji bez ADR-a, format w 4 sekcjach, CTA. Publikacja wg planu modułu (tydzień 3–4).

## Artefakt

Katalog `docs/adr/` w repo nauki: `template.md` + minimum 3 pełne ADR-y (w tym "DayChunks bez backendu" i jeden o decyzji bolesnej) + ADR przepisany z lekcji 02. Plus szkic posta LinkedIn w `notes/`. Od tej lekcji **każda znacząca decyzja w DayChunks i w ćwiczeniach kursu dostaje ADR** — to nawyk, nie jednorazowe zadanie.

## Definition of Done

- [ ] Umiesz z pamięci wymienić 4 sekcje formatu Nygarda i wyjaśnić, czemu ADR jest niemutowalny i co robi status Superseded.
- [ ] Masz jasną heurystykę "pisać czy nie pisać" (znaczenie architektoniczne / odwracalność) i umiesz podać po 3 przykłady z każdej strony granicy.
- [ ] Każdy z twoich 3 ADR-ów ma: opcje odrzucone z powodami, minimum 2 **negatywne** konsekwencje i warunek rewizji. ADR bez negatywnych konsekwencji = do poprawki.
- [ ] Druga osoba (lub ty po 3 dniach przerwy) czyta ADR i potrafi odtworzyć, czemu zdecydowano tak, a nie inaczej — bez dopytywania.

## Materiały

1. **Michael Nygard, "Documenting Architecture Decisions"** (blog Cognitect, 2011) — źródło pierwotne formatu; krótkie i kompletne.
2. **adr.github.io** — przegląd wariantów formatu (MADR i inne) i przykładów; przejrzyj dla świadomości, ale na potrzeby kursu trzymaj się prostego Nygarda.
