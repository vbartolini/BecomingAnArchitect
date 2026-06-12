# Zasady pracy nad kursem "Architect Track"

Ten projekt to kurs samodzielnej nauki prowadzący do poziomu Solution Architect / Senior Engineer (architecture track), zbudowany na bazie `architect-track-plan.md`. Uczestnik ma 20+ lat doświadczenia w .NET, integracjach i systemach enterprise — ale materiał teoretyczny piszemy tak, **jakby tematu nie znał**.

## Struktura katalogów

```
kurs/
  NN-nazwa-modulu/          ← grupa tematyczna (moduł z planu)
    README.md               ← opis modułu: cel, kolejność lekcji, Definition of Done
    NN-nazwa-lekcji/        ← jedna lekcja = jeden folder
      README.md             ← pełna treść lekcji
```

- Numeracja `NN-` (dwucyfrowa) wyznacza kolejność przerabiania — zarówno modułów, jak i lekcji.
- Nazwy folderów: małe litery, kebab-case, bez polskich znaków (np. `03-model-c4`, `02-systemy-rozproszone`).
- Treść lekcji zawsze w `README.md` wewnątrz folderu lekcji. Dodatkowe pliki (kod ćwiczeń, diagramy, notatki) trafiają do tego samego folderu.
- **`messaging-patterns/` i `windtunnel-booking/` w korzeniu projektu to foldery-wizytówki** (artefakty publiczne w tym samym repozytorium, w całości EN — patrz polityka językowa). Nie są treścią kursu: nie stosuj w nich szablonu lekcji ani polskiej narracji. Commity dotyczące tych folderów pisz po angielsku — to część publicznego portfolio.

## Format lekcji (obowiązkowy szablon)

Każdy `README.md` lekcji zawiera sekcje w tej kolejności:

1. **Cel lekcji** — 1–3 zdania: co będziesz umiał po jej zakończeniu.
2. **Dlaczego to ważne** — kontekst architekta: gdzie ten temat pojawia się na rozmowach rekrutacyjnych i w realnych decyzjach.
3. **Teoria** — główna część. Wyjaśnienia od zera, z przykładami.
4. **Praktyka** — konkretne ćwiczenia do wykonania (checkboxy `- [ ]`).
5. **Artefakt** — co fizycznie powstaje po lekcji (kod, diagram, ADR, post). **Lekcja bez artefaktu się nie liczy.**
6. **Definition of Done** — jak sprawdzić, że lekcja jest naprawdę zaliczona.
7. **Materiały** — maksymalnie 2 źródła główne (dokumentacja, książka, paper). Nie listy 20 linków. Preferuj źródła pierwotne nad kursy wideo.

## Styl wyjaśniania (najważniejsza zasada)

- **Zakładaj, że czytelnik raczej NIE zna tematu.** Każdy termin fachowy (idempotency, saga, bounded context, PACELC…) definiuj przy pierwszym użyciu, prostym językiem, zanim go użyjesz w zdaniu.
- Wyjaśniaj schematem: **problem → naiwne rozwiązanie → dlaczego się psuje → właściwy wzorzec → trade-offy**. Architektura to wybory, nie dogmaty.
- Używaj analogii z życia codziennego tam, gdzie pojęcie jest abstrakcyjne (np. outbox pattern ≈ skrzynka nadawcza w urzędzie).
- Przykłady kodu w **C# / .NET**, przykłady chmurowe w **Azure** — to stack uczestnika.
- Tam gdzie się da, nawiązuj do doświadczenia uczestnika (Perfect Gym, Aerotunel, kolejki, integracje, DayChunks) — teoria ma nazywać blizny z produkcji.
- Język kursu: **polski**, z angielskimi terminami branżowymi w oryginale (nie tłumacz "circuit breaker" na "wyłącznik obwodu" — podaj angielski termin + polskie wyjaśnienie). Szczegóły: sekcja "Polityka językowa" poniżej.
- Każda lekcja ma sekcję trade-offów lub pytania kontrolne typu "kiedy tego NIE używać".

## Polityka językowa (dwie warstwy)

Zasada nadrzędna: **wizerunek profesjonalny budują artefakty publiczne, nie warsztat nauki**. Stąd dwie warstwy:

1. **Warsztat — to repo** (`kurs/`, `notes/`, `exercises/`, `progress.md`): polski-first. Narracja po polsku, terminy fachowe zawsze w oryginale angielskim. Nazwy folderów: polski opis + angielskie terms of art (np. `01-gwarancje-dostarczania-i-idempotency`); folder może być w całości angielski tylko wtedy, gdy cała nazwa jest terminem fachowym (np. `07-observability`). To celowa konwencja — nie "misz-masz" i nie wolno jej "naprawiać" przez tłumaczenie terminów na polski ani rename'owanie warsztatu na angielski.
2. **Artefakty publiczne — foldery-wizytówki** (`messaging-patterns/` z modułu 2, `windtunnel-booking/` — capstone z modułu 7; oba w tym samym repozytorium): **w całości po angielsku** — README, ADR-y, diagramy, runbooki, komentarze w kodzie oraz komunikaty commitów dotyczących tych folderów. To wizytówki dla rekruterów i międzynarodowej publiczności; tu mieszanka językowa faktycznie szkodzi.
3. **Posty na LinkedIn**: jeden język narracji w obrębie jednego posta — polski (gdy target to polska sieć) albo angielski (zasięg międzynarodowy); terminy fachowe zawsze w oryginale.

Korzyść uboczna warstwy 2: przepisanie ADR-a czy diagramu z polskich notatek na angielski artefakt to dodatkowa runda utrwalenia materiału w języku, w którym uczestnik będzie go bronił na rozmowach (system design interview odbywa się po angielsku).

Przy generowaniu nowych lekcji: jeśli lekcja każe wytworzyć artefakt publiczny (repo, ADR do repo publicznego, diagram do Featured), wskaż wprost, że powstaje po angielsku.

## Zasady merytoryczne

- Każdy moduł kończy się artefaktem nadającym się na LinkedIn (post, diagram do Featured, repo).
- Weryfikuj aktualność technologii — plan pisany w czerwcu 2026; przy nazwach usług Azure i wersjach .NET sprawdzaj bieżący stan, nie cytuj z pamięci starych wersji.
- Kolejność modułów: 0 → 1 → 2 → 3 → 4 (lekki) → 5 (częściowo równolegle z 2–4) → 6 → Capstone. Capstone wymaga ukończonych modułów 2–3.
- Postęp odnotowuj w `progress.md` (jeśli istnieje) po każdym ukończonym tygodniu.

## Fiszki Anki (materiał lokalny — nigdy w repozytorium)

Fiszki Anki i wszystko, co służy do ich wytworzenia, to prywatny materiał do nauki — **nie trafia do repozytorium**. `.gitignore` ignoruje `*.apkg` oraz każdy folder o nazwie `anki/`. Nigdy nie dodawaj tych plików siłą (`git add -f`) i nie umieszczaj pytań/fiszek poza strukturą opisaną niżej, bo wyciekną do gita.

Struktura i sposób generowania (wzorzec: `kurs/01-fundamenty-myslenia-architektonicznego/anki/`):

- Całość mieszka w `kurs/NN-modul/anki/`: generatory (`generate_anki.py` — talia bazowa z fiszkami w kodzie, `generate_szczegolowe.py` — talia szczegółowa), paczki zbiorcze `.apkg` i README.
- **Źródła pytań szczegółowych**: `anki/zrodla/<folder-lekcji>.md` — nagłówek `### ` = pytanie, treść pod nim = odpowiedź (akapity, `**pogrubienia**`, listy `- `; bez tabel i nagłówków w odpowiedziach). Plików `.md` z pytaniami nie zostawiaj w folderach lekcji.
- Generator szczegółowy zapisuje oprócz paczki zbiorczej **paczkę per lekcja do folderu lekcji**, nazwaną tak samo jak folder: `NN-lekcja/<NN-lekcja>.apkg` (dzięki regule `*.apkg` te pliki też są ignorowane).
- **Nazwy talii**: `Architect Track::L<poziom><lekcja> - <nazwa lekcji>`, np. `L0101 - Atrybuty jakościowe` (L = lekcja, pierwsza para cyfr = poziom/moduł, druga = numer lekcji). Bez pośrednich poziomów typu "M1 Szczegółowe" — talia bazowa i szczegółowa zasilają tę samą talię lekcji.
- **GUID-y kart**: liczone z treści pytania, w rozdzielnych namespace'ach per paczka (np. `m1-szczegolowe::<pytanie>`), żeby ponowny import aktualizował karty zamiast tworzyć duplikaty.
- Ponieważ folder `anki/` jest poza gitem, nie ma kopii zapasowej w repozytorium — nie usuwaj źródeł w `zrodla/` przy porządkach.

## Jak rozbudowywać kurs (instrukcja dla Claude Code)

- Nową lekcję dodawaj jako kolejny folder `NN-...` w odpowiednim module, według szablonu powyżej.
- Nie zmieniaj numeracji istniejących lekcji — dopisuj na końcu modułu.
- Przy aktualizacji treści zachowaj checkboxy: odhaczone (`- [x]`) oznaczają wykonaną pracę uczestnika i nie wolno ich resetować.
- `architect-track-plan.md` to źródło prawdy dla zakresu; kurs w `kurs/` to jego rozwinięcie dydaktyczne.
