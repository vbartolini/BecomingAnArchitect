# Lekcja 02 — Repo nauki w git: jedno miejsce na plan, notatki i kod

## Cel lekcji

Po tej lekcji będziesz miał jedno repozytorium git, w którym mieszka cały Architect Track: plan, treść kursu, notatki i kod ćwiczeń. Ustalisz strukturę katalogów, konwencję commitów jako dziennika postępu oraz plik `progress.md` aktualizowany co tydzień.

## Dlaczego to ważne

Dla architekta repozytorium to nie tylko magazyn kodu — to **zapis procesu myślenia w czasie**. Na rozmowach rekrutacyjnych na role Senior/Architect coraz częściej pada prośba "pokaż coś, nad czym pracujesz". Publiczne repo nauki z półroczną historią commitów odpowiada na nią lepiej niż jakakolwiek deklaracja: pokazuje konsekwencję, sposób organizacji pracy i ewolucję myślenia — rzeczy, których nie da się podrobić w tydzień przed rozmową. To samo repo zasili Twój profil LinkedIn (sekcja Featured) i dostarczy materiału na posty.

Jest też powód operacyjny, który znasz z produkcji: **rozproszenie to koszt**. Notatki w jednej aplikacji, kod w drugiej, plan w trzeciej — to trzy miejsca do synchronizowania i trzy miejsca, w których coś ginie. Jedno repo to jedno źródło prawdy (single source of truth — jedno autorytatywne miejsce, z którego wszyscy czytają stan; każda kopia poza nim to potencjalna rozbieżność).

## Teoria

### Problem: gdzie trzymać naukę?

Naiwne podejście nr 1: **bez wersjonowania w ogóle** — notatki w Notion/OneNote, kod luzem w katalogach. Dlaczego się psuje: brak historii ("co ja właściwie zrobiłem w zeszłym miesiącu?"), brak kopii zapasowej z prawdziwego zdarzenia, zero wartości sygnalizacyjnej na zewnątrz.

Naiwne podejście nr 2: **osobne repo na każdą rzecz** — repo na notatki, repo na ćwiczenia z modułu 2, repo na ćwiczenia z modułu 3... Dlaczego się psuje: po trzech miesiącach masz sześć repozytoriów, z których każde wygląda na porzucone, bo commity rozkładają się rzadko. Kontekst się rozjeżdża: notatka o outbox pattern w jednym repo, kod outboxa w innym. Nikt (łącznie z Tobą za pół roku) nie złoży tego w całość.

Właściwy wzorzec: **monorepo nauki** — jedno repozytorium na cały plan. Monorepo (jedno repozytorium zawierające wiele logicznych części projektu) ma w świecie firm swoje wady i zalety, ale dla osobistej nauki zalety wygrywają zdecydowanie: jedna historia commitów = jeden ciągły dziennik postępu, pełen kontekst w jednym `git clone`, jedna rzecz do pokazania na rozmowie.

Decyzja tego projektu (podjęta świadomie, do dziennika decyzji w lekcji 03): **wizytówki też mieszkają w monorepo** — `messaging-patterns/` (moduł 2) i `windtunnel-booking/` (capstone) to foldery w tym samym repozytorium, a nie osobne repa. Granica między warsztatem a wizytówką jest **językowo-jakościowa, nie gitowa**: wszystko w tych dwóch folderach (README, ADR-y, kod, komentarze) oraz commity ich dotyczące — po angielsku i w jakości "do oceny przez obcego człowieka"; reszta repo — polski brudnopis nauki.

Trade-off do nazwania uczciwie, bo ten wybór ma cenę: (+) jedno repo do ogarniania, jedna historia, jeden push, zero żonglowania kontekstami gita; (−) wizytówka nie ma własnego URL-a repozytorium — w Featured linkujesz **folder** (GitHub renderuje README folderu przy jego przeglądaniu, więc to działa, ale wygląda mniej "produktowo" niż samodzielne repo); (−) historia commitów miesza polskie wpisy nauki z angielskimi wpisami wizytówek. Mitygacje: główny README repo dostaje na górze krótką sekcję po angielsku z wyraźnymi linkami do obu wizytówek (recenzent z Featured trafia tam jednym kliknięciem), a konwencja commitów (niżej) pozwala odfiltrować historię wizytówki: `git log -- messaging-patterns/`. Jeśli kiedyś uznasz, że wizytówka zasługuje na własne repo (np. przed intensywnym aplikowaniem) — wydzielenie folderu to godzina pracy, decyzja jest odwracalna.

### Struktura katalogów

Proponowana struktura — zgodna z tym, co już istnieje w tym projekcie:

```
Architect/                      # ← jedno repo git na wszystko
├── architect-track-plan.md     # źródło prawdy dla zakresu planu
├── CLAUDE.md                   # zasady pracy nad kursem (dla Claude Code)
├── progress.md                 # dziennik postępu, aktualizowany co tydzień
├── kurs/                       # treść kursu: moduły i lekcje (PL)
│   └── NN-nazwa-modulu/
│       └── NN-nazwa-lekcji/
│           └── README.md
├── notes/                      # notatki przekrojowe (PL)
│   ├── decisions-log.md        # dziennik decyzji (lekcja 03)
│   └── post-ideas.md           # kandydaci na posty LinkedIn (opcjonalnie)
├── exercises/                  # kod ćwiczeń — brudnopis nauki (PL)
│   └── NN-nazwa-cwiczenia/     # np. 02-outbox-pattern/ (solucja .NET per ćwiczenie)
├── messaging-patterns/         # folder-WIZYTÓWKA modułu 2 — w całości EN
└── windtunnel-booking/         # folder-WIZYTÓWKA capstone — w całości EN
```

Zasady, które robią różnicę:

- **Numeracja `NN-` wyznacza kolejność** — katalogi czytają się jak spis treści, bez zaglądania do plików.
- **Kod ćwiczeń osobno od treści lekcji** (`exercises/` vs `kurs/`), ale notatki z lekcji mogą żyć obok jej `README.md` — kontekst lekcji zostaje w jednym folderze. Drobne pliki ćwiczeniowe (snippet, diagram) też mogą zostać w folderze lekcji; do `exercises/` trafiają rzeczy z własną solucją/projektem.
- **`notes/` jest dla treści przekrojowych**, które nie należą do żadnej pojedynczej lekcji: dziennik decyzji, pomysły na posty, pytania "do sprawdzenia później".
- Nie projektuj struktury na zapas. Katalog tworzysz, gdy pojawia się pierwsza rzecz do włożenia — YAGNI (You Aren't Gonna Need It — zasada: nie buduj czegoś, zanim nie jest potrzebne) dotyczy też katalogów.

### Konwencja commitów jako dziennik postępu

W pracy commit message opisuje zmianę dla zespołu. W repo nauki commit message pełni drugą rolę: **wpis w dzienniku postępu**. Za pół roku `git log --oneline` ma się czytać jak historia Twojej nauki — co, kiedy i w jakim module.

Konwencja oparta o Conventional Commits (popularny standard formatowania commit message: `typ(zakres): opis`), uproszczona pod naukę:

```
typ(moduł): co zostało zrobione

Typy:
  notes  – notatki z nauki         notes(m2): outbox pattern — notatki z DDIA rozdz. 9
  code   – kod ćwiczeń             code(m2): działający outbox na SQL Server + hosted service
  docs   – treść kursu, plan       docs(m0): format przeglądu tygodnia
  post   – materiały LinkedIn      post: szkic posta o ADR
  weekly – wpis tygodniowy         weekly: tydzień 1 — rytm 4/4, pierwsze ADR-y
```

Dwie zasady ważniejsze niż sam format:

1. **Każdy blok deep work kończy się commitem.** To "zdefiniowane wyjście" z lekcji 01 — nawet jeśli commitujesz tylko pół strony notatek. Commit jest dowodem, że blok się odbył, i jednostką postępu.
2. **Commit message pisz dla siebie-za-pół-roku.** "wip" i "update" nic nie powiedzą; "notes(m2): dlaczego at-least-once wymaga idempotency" — powie wszystko.

Wyjątek językowy (polityka językowa kursu): commity dotyczące folderów-wizytówek pisz **po angielsku**, np. `code(mp): add transactional outbox relay` albo `code(capstone): saga compensation for payment timeout` — bo historię wizytówki recenzent może przeglądać przez `git log -- messaging-patterns/`.

### Publiczne repo jako sygnał na rynku pracy

Czy repo nauki ma być publiczne? Argumenty za:

- **Sygnalizacja kosztowna** (costly signal — pojęcie z teorii sygnalizacji: wiarygodny jest sygnał, którego nie da się tanio podrobić). Wpis "passionate about architecture" w CV kosztuje 3 sekundy. Publiczna historia 150 commitów przez 6 miesięcy kosztuje 6 miesięcy — i dlatego jest wiarygodna.
- Rekruterzy techniczni i hiring managerowie naprawdę zaglądają na GitHub przed rozmową. Zielona mapa aktywności + sensownie opisane repo to przewaga na starcie rozmowy: "widziałem, że uczysz się systemów rozproszonych — opowiedz o tym" to najlepsze możliwe otwarcie.
- Publiczność dyscyplinuje jakość: notatki pisane "na zewnątrz" są pełniejsze, a to one staną się surowcem na posty.
- Repo linkujesz w Featured na LinkedIn — domyka się pętla: nauka → artefakt → widoczność.

Argumenty przeciw i mitygacje: (a) "ktoś zobaczy moje błędy" — repo nauki z definicji pokazuje proces, nie perfekcję; dojrzali inżynierowie czytają to jako zaletę; (b) **nigdy nie commituj niczego z pracy** — przykłady z Perfect Gym / Aerotunel zawsze anonimizuj i odtwarzaj z pamięci jako uogólnione case'y, bez kodu, nazw i danych firmowych. To twarda zasada, nie sugestia.

Kwestia języka (polityka językowa kursu): część warsztatowa repo **zostaje po polsku** — uczysz się szybciej w swoim języku, a autentyczny polski dziennik nauki jest wiarygodnym sygnałem, nie obciachem; terminy fachowe i tak są w oryginale. Główny README repo dostaje na górze 2–3 zdania po angielsku mówiące, czym to repo jest, plus wyraźne linki do wizytówek ("learning-in-public repo for my Solution Architect track — notes in Polish; the showcase projects below are fully in English: → messaging-patterns, → windtunnel-booking") — recenzent spoza Polski ma w jedno kliknięcie trafić tam, gdzie wszystko zrozumie. Foldery-wizytówki: w całości po angielsku.

Decyzja "publiczne czy prywatne" to dobry, prawdziwy wpis do dziennika decyzji w lekcji 03. Jeśli masz opór — kompromis: zacznij prywatnie, upublicznij po module 1, gdy repo będzie miało już kształt. Jedno ograniczenie tego kompromisu: skoro wizytówki mieszkają w tym repo, **najpóźniej przy publikacji mini-projektu z modułu 2 repo musi być publiczne** — niewidoczna wizytówka nie istnieje.

### `progress.md` — tygodniowy stan planu

`progress.md` w korzeniu repo to **agregat przeglądów tygodnia** z lekcji 01: jedno miejsce, w którym widać przebieg całego planu. Aktualizujesz go raz w tygodniu, podczas przeglądu. Szablon wpisu:

```markdown
# Progress — Architect Track

> Aktualizowany co tydzień podczas przeglądu. Najnowszy tydzień na górze.

## Tydzień NN (RRRR-MM-DD – RRRR-MM-DD) — Moduł X

**Rytm:** bloki nauki 3/4 · post: tak/nie · komentowanie 3/4 · przegląd: tak

**Co zrobiłem:**
- (1–4 punkty: ukończone lekcje, ćwiczenia, artefakty)

**Artefakt tygodnia:** (link/ścieżka — commit, notatka, diagram, post)

**Retro — co działa:** ...
**Retro — co nie zadziałało i dlaczego:** ...
**Retro — co zmieniam w przyszłym tygodniu:** (max 1–2 korekty)

**Następny tydzień — pierwszy krok:** (od czego zaczynasz poniedziałkowy blok)
```

Dlaczego ten format: linia "Rytm" to Twoje metryki (szybki rzut oka, czy budżet błędów z lekcji 01 jest trzymany), sekcja retro utrwala wynik przeglądu, a "pierwszy krok" eliminuje rozruch z pustą kartką w poniedziałek. Całość wpisu ma zajmować 5–10 minut — jeśli zajmuje pół godziny, upraszczaj.

### Kiedy tego NIE robić (trade-offy)

- **Nie przenoś do repo rzeczy, które żyją gdzie indziej naturalnie.** Harmonogram bloków mieszka w DayChunks, nie w markdownie; repo trzyma treść i wyniki, nie kalendarz.
- **Nie buduj automatyzacji wokół repo na tym etapie** (CI, generatory, hooki). To prokrastynacja w przebraniu produktywności — wróci legitymacja do automatyzacji przy Capstone.
- **Nie commituj z pracy zawodowej niczego** — ani kodu, ani nazw klientów, ani szczegółów architektur, które są własnością pracodawcy.

## Praktyka

- [x] Zainicjuj git w katalogu planu (`git init`), jeśli jeszcze nie jest repozytorium; dodaj `.gitignore` pod .NET (np. `dotnet new gitignore`), bo w `exercises/` i folderach-wizytówkach pojawi się kod.
- [x] Utwórz strukturę: katalog `notes/` i `exercises/` (z plikiem `.gitkeep` lub pierwszą notatką, żeby git je widział) oraz plik `progress.md` z szablonem z tej lekcji.
- [x] Zrób pierwszy commit zgodnie z konwencją (np. `docs(m0): struktura repo nauki + progress.md`) i ustaw zdalne repo na GitHub.
- [x] Podejmij decyzję publiczne/prywatne — zapisz ją na razie w notatce; w lekcji 03 przeniesiesz ją do dziennika decyzji jako pełny wpis.
- [x] Podczas najbliższego przeglądu tygodnia wypełnij pierwszy wpis w `progress.md` (wpis za tydzień modułu 0).

## Artefakt

1. **Repozytorium nauki w git** — z ustaloną strukturą katalogów, `.gitignore`, zdalnym repo na GitHub i pierwszymi commitami w konwencji.
2. **`progress.md`** — z szablonem i pierwszym wpisem tygodniowym.

## Definition of Done

- [x] `git log` pokazuje co najmniej 3 commity z prawdziwej pracy, każdy w konwencji `typ(moduł): opis`.
- [x] Struktura `kurs/`, `notes/`, `exercises/`, `progress.md` istnieje w repo.
- [x] Repo ma skonfigurowany remote na GitHub i jest wypchnięte.
- [x] Decyzja publiczne/prywatne jest podjęta i zapisana (z uzasadnieniem).
- [x] `progress.md` zawiera pierwszy wypełniony wpis tygodniowy.

## Materiały

1. [Conventional Commits](https://www.conventionalcommits.org/) — specyfikacja konwencji commit message, baza dla uproszczonej konwencji z tej lekcji.
2. [GitHub Docs — "About READMEs" i profil GitHub](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes) — jak opisać repo, żeby pracowało na Ciebie u kogoś, kto widzi je pierwszy raz.
