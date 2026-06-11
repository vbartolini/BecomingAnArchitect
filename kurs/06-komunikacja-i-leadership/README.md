# Moduł 6 — Komunikacja architektoniczna i leadership

**Czas trwania:** 2 tygodnie pracy skupionej + elementy ciągłe (mock interviews, szlifowanie historii) rozłożone na kolejne tygodnie
**Pozycja w kursie:** po modułach 1–4 (możesz zacząć równolegle z końcówką modułu 5). Ten moduł bezpośrednio przygotowuje Cię do Capstone (lekcja 02 — RFC) i do rozmów rekrutacyjnych (lekcje 03–04).

## Cel modułu

Architektem się jest wtedy, gdy **inni rozumieją Twoje decyzje** — nie wtedy, gdy masz rację. Możesz znać na pamięć outbox pattern, CAP i model C4, ale jeśli zarząd nie rozumie, po co chcesz 3 tygodnie na refaktoring, PM nie wie, co dostanie i kiedy, a zespół wykonuje Twoje decyzje bez przekonania — to nie jesteś jeszcze architektem, tylko bardzo dobrym inżynierem. Ten moduł trenuje cztery konkretne umiejętności komunikacyjne:

1. **Mówienie do różnych odbiorców** — ta sama decyzja opowiedziana zarządowi, PM-owi i zespołowi to trzy różne historie.
2. **Architektura jako proces zespołowy** — design review i RFC, czyli jak podejmować decyzje *z* ludźmi, a nie *za* ludzi.
3. **System design interview** — format, w którym rekrutuje się na role Senior/Lead/Architect; trenowalny jak każdy inny.
4. **War stories w formacie STAR** — zamiana 20 lat doświadczenia (Perfect Gym, Aerotunel, nocne awarie) w dopracowane historie pod rozmowy behawioralne.

W odróżnieniu od modułów 2–3 tu prawie nie ma nowej wiedzy technicznej. Jest za to trening — i to taki, który wymaga mówienia na głos, nagrywania się i pisania dokumentów. To bywa niewygodne. Właśnie dlatego większość inżynierów tego nie robi — i właśnie dlatego to działa.

## Lekcje (przerabiaj po kolei)

| Lekcja | Temat | Artefakt |
|---|---|---|
| [01](01-trzy-wersje-tej-samej-historii/) | Trzy wersje tej samej historii — zarząd / PM / zespół | jedna decyzja architektoniczna spisana w 3 wersjach |
| [02](02-design-review-i-rfc/) | Design review i RFC — architektura jako proces zespołowy | RFC dla decyzji w nadchodzącym Capstone |
| [03](03-system-design-interview/) | System design interview — format, framework, trening | notatki + harmonogram 5–6 mock interviews (ciągłe) |
| [04](04-war-stories-star/) | War stories w formacie STAR | 3 dopracowane historie + nagranie 5-minutowe |

## Rozkład na 2 tygodnie + elementy ciągłe

| Tydzień | Lekcje | Akcent |
|---|---|---|
| 1 | 01 + 02 | Komunikacja pisemna i do odbiorców: trzy wersje historii, RFC pod Capstone. RFC z lekcji 02 wykorzystasz wprost na starcie Capstone. |
| 2 | 03 + 04 | Rozmowy rekrutacyjne: poznaj format system design interview, przeprowadź pierwszy mock, spisz i nagraj 3 war stories. |
| ciągłe | 03 (mocki) + 04 (szlif) | 5–6 mock system design interviews rozłożonych na kolejne tygodnie (harmonogram w lekcji 03) — najlepiej równolegle z Capstone. Raz na 2 tygodnie odśwież jedną war story na głos. |

Elementy ciągłe są częścią modułu — moduł formalnie "domykasz" po 2 tygodniach, ale mocki i powtórki historii planujesz z góry i wpisujesz w DayChunks jak każdy inny rytm (moduł 0). Umiejętność mówienia zanika szybciej niż wiedza: war story nieopowiadana przez 2 miesiące przestaje być płynna.

## Materiał bazowy modułu (max 2 źródła)

1. **Alex Xu, *System Design Interview – An Insider's Guide* (vol. 1)** — kanoniczne źródło do lekcji 03: framework krokowy i przerobione zadania (URL shortener, notification system, news feed…). Czytaj rozdziały równolegle z mockami, nie wszystko naraz.
2. **Barbara Minto, *The Pyramid Principle*** — źródło techniki piramidy z lekcji 01 (wniosek najpierw, szczegóły na żądanie). Wystarczy część I; reszta to rozwinięcia.

## Artefakty modułu

- [ ] **3 "war stories" w formacie STAR** (lekcja 04) — główny artefakt modułu: spisane (wersja 2-min i 5-min), z liczbami w rezultatach, gotowe pod rozmowy rekrutacyjne. Kandydaci: Perfect Gym, Aerotunel, poison message / nocna awaria.
- [ ] Jedna decyzja architektoniczna z Twojej kariery spisana w 3 wersjach: zarząd / PM / zespół (lekcja 01) — wersja "do zarządu" to gotowy szkielet posta na LinkedIn.
- [ ] RFC dla jednej decyzji w nadchodzącym Capstone (lekcja 02) — trafi do repo Capstone obok ADR-ów.
- [ ] Nagranie: jedna war story opowiedziana w 5 minut (lekcja 04) + notatki z krytycznego odsłuchu.
- [ ] Harmonogram i notatki z 5–6 mock system design interviews (lekcja 03) — checkboxy odhaczasz w kolejnych tygodniach.

## Definition of Done modułu

Moduł 6 jest zaliczony, gdy **wszystkie** poniższe są prawdą:

- [ ] Masz spisaną jedną decyzję architektoniczną w 3 wersjach i potrafisz wyjaśnić, czym różnią się te wersje i dlaczego (nie tylko "krótsza/dłuższa").
- [ ] RFC dla decyzji w Capstone istnieje, przeszedł przez co najmniej jedną rundę komentarzy (człowiek lub Claude w roli reviewera) i ma zapisany status.
- [ ] Umiesz z pamięci wymienić kroki frameworku system design interview i masz za sobą **co najmniej 2 z 5–6** zaplanowanych mocków (reszta wpisana do kalendarza/DayChunks z datami).
- [ ] 3 war stories w formacie STAR są spisane, każda ma w rezultacie konkretną liczbę lub mierzalny efekt, i istnieje wersja 2-minutowa i 5-minutowa każdej.
- [ ] Nagrałeś się opowiadając jedną historię w ~5 minut, odsłuchałeś i wprowadziłeś poprawki po odsłuchu.
- [ ] Test końcowy: opowiedz dowolną war story na głos, bez kartki, mieszcząc się w 2 minutach — i odpowiedz na jedno pytanie pogłębiające (np. "co byś dziś zrobił inaczej?") bez zawahania.

Po zaliczeniu: odhacz Moduł 6 w `architect-track-plan.md`, zaktualizuj `progress.md` i przejdź do Capstone — RFC z lekcji 02 to jego pierwszy dokument. Mocki z lekcji 03 prowadź dalej równolegle z Capstone.
