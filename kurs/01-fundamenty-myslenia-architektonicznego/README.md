# Moduł 1 — Fundamenty myślenia architektonicznego

**Czas trwania:** 3–4 tygodnie
**Pozycja w kursie:** po module 0 (rytm musi już działać). Ten moduł jest fundamentem dla wszystkich kolejnych — pojęcia wprowadzone tutaj (trade-off, atrybut jakościowy, bounded context, ADR, C4) wracają w każdym następnym module.

## Cel modułu

Przejść mentalnie z pytania **"jak to zaimplementować?"** na pytanie **"jakie są opcje, trade-offy i konsekwencje?"**.

To jest najważniejsza zmiana na drodze od senior developera do architekta. Developer dostaje problem i szuka rozwiązania. Architekt dostaje problem i szuka **przestrzeni rozwiązań**: wymienia 2–3 realne opcje, nazywa koszty każdej i świadomie wybiera, wiedząc, z czego rezygnuje. Po 20+ latach w .NET masz w głowie setki gotowych rozwiązań — ten moduł uczy zatrzymywać się chwilę PRZED sięgnięciem po pierwsze z nich.

Drugi cel, równie ważny: nauczyć się **utrwalać myślenie architektoniczne na piśmie** (diagramy C4, ADR-y, analizy trade-offów). Architektura, która istnieje tylko w czyjejś głowie, nie istnieje — nie da się jej zrecenzować, przekazać ani obronić na rozmowie rekrutacyjnej.

## Lekcje (przerabiaj po kolei)

| Lekcja | Temat | Artefakt |
|---|---|---|
| [01](01-atrybuty-jakosciowe/) | Atrybuty jakościowe — "-ilities" i ich konflikty | tabela top-3 atrybutów dla Perfect Gym i DayChunks, z uzasadnieniem |
| [02](02-trade-off-analysis/) | Trade-off analysis — architektura jako wybór, czego NIE robić | pisemna analiza trade-offów realnej decyzji (macierz opcji) |
| [03](03-model-c4/) | Model C4 — diagramy, które ludzie naprawdę czytają | diagram C4 (poziom 1 i 2) systemu klasy Perfect Gym, zanonimizowany |
| [04](04-adr-architecture-decision-records/) | ADR — Architecture Decision Records | 3 ADR-y napisane wstecz dla decyzji z własnej kariery |
| [05](05-ddd-strategiczne/) | DDD strategiczne — bounded contexts i context mapping | mapa kontekstów systemu dla siłowni |
| [06](06-style-architektoniczne/) | Style architektoniczne — monolit modularny, mikroserwisy, event-driven | macierz decyzyjna stylów + rekomendacja dla DayChunks |
| [07](07-cwiczenia-system-design/) | Warsztat system design — framework i 3 ćwiczenia | 3 rozpisane projekty (notatki + diagramy) |

## Sugerowany rozkład tygodniowy

Rytm z modułu 0: 4 bloki głębokiej pracy po 2h tygodniowo, czyli ok. 8h/tydzień.

| Tydzień | Lekcje | Uwagi |
|---|---|---|
| 1 | 01 + 02 | Fundament pojęciowy: atrybuty jakościowe i trade-offy. Lekcja 02 jest sercem modułu — nie skracaj jej. |
| 2 | 03 + 04 | Narzędzia zapisu: C4 i ADR. Tu powstają oba artefakty LinkedIn (diagram + materiał na post). |
| 3 | 05 + 06 | DDD strategiczne i style architektoniczne. Lekcja 06 spina wszystko z lekcji 01–05. |
| 4 | 07 + domknięcie | Warsztat system design (3 ćwiczenia = 3 bloki) + publikacja posta o ADR, przegląd Definition of Done. |

Jeśli idziesz szybciej i zamkniesz moduł w 3 tygodnie — dobrze. Jeśli któreś ćwiczenie z lekcji 07 wymaga dodatkowego bloku — też dobrze. Nie przechodź dalej z niezrobionymi artefaktami: w tym module artefakty SĄ nauką.

## Artefakty modułu (na zewnątrz)

Każdy moduł kończy się czymś, co nadaje się na LinkedIn. W tym module:

- [ ] **Diagram C4 (poziom 1–2) do sekcji Featured na LinkedIn** — zanonimizowany diagram systemu klasy "platforma dla siłowni" albo system rezerwacji tunelu. Powstaje w lekcji 03, doszlifowany w tygodniu 2–3. To pierwszy publiczny dowód, że myślisz i komunikujesz jak architekt.
- [ ] **Post: "ADR — najtańsze narzędzie architekta, którego prawie nikt nie używa"** — materiał powstaje naturalnie w lekcji 04 (3 ADR-y napisane wstecz dają konkretne, osobiste przykłady). Publikacja w tygodniu 3 lub 4.

Plus artefakty wewnętrzne (do repo nauki): tabela atrybutów, analiza trade-offów, 3 ADR-y, mapa kontekstów, macierz stylów, notatki z 3 ćwiczeń system design.

## Definition of Done modułu

Moduł 1 jest zaliczony, gdy **wszystkie** poniższe są prawdą:

- [ ] **Test główny:** dla dowolnego problemu architektonicznego (np. "jak powiadamiać klientów o odwołanych zajęciach?") umiesz w 10 minut wymienić **2–3 opcje architektoniczne i uczciwie nazwać trade-offy każdej** — bez zaglądania do notatek. Sprawdź na 2–3 problemach z własnej przeszłości zawodowej.
- [ ] Wszystkie 7 lekcji ma wykonane ćwiczenia i wytworzone artefakty (są w repo nauki).
- [ ] Diagram C4 wisi w Featured na LinkedIn.
- [ ] Post o ADR jest opublikowany.
- [ ] Umiesz z pamięci wyjaśnić komuś nietechnicznemu: czym jest trade-off architektoniczny, czym jest bounded context i po co pisze się ADR-y. (Wyjaśnienie "komuś nietechnicznemu" to test zrozumienia, nie formalność.)
- [ ] `progress.md` ma wpisy za każdy tydzień modułu.

## Książka bazowa modułu

**"Fundamentals of Software Architecture"** (Mark Richards, Neal Ford, O'Reilly) — to kręgosłup tego modułu. Nie musisz czytać całej od deski do deski: lekcje wskazują konkretne rozdziały. Uzupełniająco (głównie do lekcji 02 i 06): **"Software Architecture: The Hard Parts"** tych samych autorów — sięgaj po nią punktowo, pełna lektura wraca w module 2.
