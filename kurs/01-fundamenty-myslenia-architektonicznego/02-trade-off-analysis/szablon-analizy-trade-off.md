# Szablon analizy trade-off

> Wypracowany na ćwiczeniu 2 (cache grafiku). Wklej do nowego zadania i wypełnij — kolejność sekcji jest celowa.

**Stan obecny (baseline):** [gdzie jesteśmy — liczby, np. p95 > 3 s przy 200 RPS]
**Cel:** [dokąd mamy dojść — liczby, np. p95 < 500 ms → wymagane ≥ 6× przyspieszenie]

**Constrainty (twarde, nie podlegają wymianie):** [każdy z liczbą, np. staleness ≤ 5 s]
[jeśli liczba jest założeniem, a nie decyzją biznesu: `ZAŁOŻENIE: … — do potwierdzenia z biznesem`]

**Kryteria (w kolejności priorytetu):** (1) … (2) … (3) … (4) …

## Macierz

| Kryterium | A: … | B: … | … | Z: minimalna interwencja |
|---|---|---|---|---|
| … | | | | |

Zasady wypełniania:

- **Zawsze dodaj opcję minimalnej interwencji** — każda droższa opcja musi udowodnić, że jest od niej lepsza. Ciężar dowodu spoczywa na złożoności.
- **Skala względem baseline'u:** `++` rozwiązuje z zapasem · `+` wyraźnie lepiej · `0` jak baseline · `−` nowy koszt/regres · `−−` poważnie szkodzi · **DQ** łamie constraint → opcja odpada, plusy nie ratują.
- **Test komórki:** da się przeczytać jako „[ocena], bo [mechanizm], przy założeniu [założenie]". Nie umiesz napisać zdania → wpisz `ZAŁOŻENIE: … — DO WERYFIKACJI: [jak sprawdzić]` zamiast zgadywać.

## Odwracalność

[dla każdej opcji-finalisty: co kosztuje wycofanie? minuty / godziny / tygodnie; two-way vs one-way]
[wniosek: jak starannie trzeba decydować — two-way → wdróż i zmierz; one-way → analiza głębiej]

## Rekomendacja

Algorytm: (1) skreśl DQ → (2) zweryfikuj minimalną interwencję → (3) skreśl `0` na kryterium #1 → (4) ocalałych rozstrzygnij kolejnymi kryteriami wg priorytetu → (5) remis łamie odwracalność.

**Krok 1 — [najtańsza opcja]:** [jak weryfikujemy, ile to kosztuje, kiedy kończymy tutaj]

**Krok 2 — jeśli [warunek]: [zwycięzca].** **Świadomie płacę:** [czym — z liczbami]. **Dostaję:** [co].

**Odrzucone (w kolejności odpadania, każdy od WŁASNEJ słabości):**
- **[opcja]:** DQ — [który constraint i czym łamie]
- **[opcja]:** `0` na kryterium #1 — [dlaczego nie rozwiązuje problemu]
- **[opcja]:** [kryterium/odwracalność, które ją pogrążyło]
- **[opcja]:** [jeśli decydujący fakt to niezweryfikowane założenie — napisz to jawnie i co się zmieni, gdy upadnie]

**Trigger powrotu do decyzji:** [przyszłe zdarzenia, nie powtórzony warunek z kroków — np. sygnał spójności / skali / kosztu]
