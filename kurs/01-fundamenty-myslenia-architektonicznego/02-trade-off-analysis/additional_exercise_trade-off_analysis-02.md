**Problem:** Czas odpowiedzi odczytu grafiku ma zejść poniżej 500 ms p95 przy 200 RPS, bez psucia spójności (członek nie może zobaczyć zajęć, które właśnie odwołano).

**Kryteria (w kolejności priorytetu):** (1) p95 latencja (2) świeżość danych (jak bardzo nieaktualny grafik boli? założenie: staleness ≤ 5 s) (3) koszt operacyjny (4) złożoność

**Stan obecny:** p95 > 3 s przy 200 RPS → cel < 500 ms, ≥ 6×

| Kryterium | A: cache Redis | B: cache in memory | C: read replica bazy | D: denormalizacja bazy | E: materializacja widoku | F: CDN dla części statycznej | G: tylko indeks |
|---|---|---|---|---|---|---|---|
|p95 latencja| ++ odczyt na poziomie 1 ms (round-trip sieciowy) | ++ najszybszy odczyt na poziomie nanosekund|0 — nie skraca zapytania, tylko zdejmuje kontencję z primary; nie pomoże, jeśli wolny jest sam plan zapytania| ++ odczyt z jednej płaskiej tabeli| ++  odczyt po zmaterializowanym wyniku(MATERIALIZED VIEW)/indeksie(INDEXED VIEW)|0 — CDN przyspiesza ładowanie strony, ale endpoint zwracający grafik dalej robi to samo wolne zapytanie | ? — zależy od tego, dlaczego jest wolno. ZAŁOŻENIE: brak indeksu pod filtr (klub + data); jeśli tak, indeks daje rząd wielkości. DO WERYFIKACJI: plan zapytania (EXPLAIN / execution plan) — 30 minut pracy, zanim wydamy na cokolwiek innego 
|świeżość danych| + bez względu na ilość instancji cache jest zawsze zinwalidowany, jedna inwalidacja| -- przy wielu instancjach (rozjazd cache, łamie wymóg o odwołanych zajęciach)| − replication lag: przez czas lagu replika dalej pokazuje odwołane zajęcia ( replication lag w tej samej strefie to zwykle ~1 s)|+ (aktualizacja przy zapisie, brak okna, bo to ta sama transakcja)| − PG refresh cykliczny / + Azure SQL indexed view|0 grafiki statyczne nie dotyczy samych danych, a więc rozwiązanie rozwiąże przyspieszenie ładowania strony, ale nie zapewni świeżości danych| ++ — czytamy prosto ze źródła prawdy, staleness = 0 z definicji. Złoty standard, względem którego każda inna opcja coś oddaje|
|koszt operacyjny| - potrzeba utrzymania infrastruktury dla Redisa, czyli kolejna usługa do utrzymania | ++ zero infrastruktury| − kolejna usługa + koszt compute drugiej instancji bazy |+ zero nowych usług|+ zero nowych usług; koszt to dysk + narzut odświeżania|0/− usługa CDN + konfiguracja; koszt bieżący niski, ale nie zdejmuje obciążenia z bazy — dane grafiku dalej idą z bazy|++ — zero nowych usług |
|złożoność| 0 (wzorzec znany, ale dochodzi zależność od zewnętrznej usługi — nowy punkt awarii: co gdy Redis padnie? potrzebny fallback do bazy)| - sama w sobie znikoma dopóki nie pojawi się potrzeba wprowadzenia instancji cross-instance| − routing read/write + ryzyko read-your-own-writes | −/−− potrzeba pilnowania spójności, ryzyko trwałego rozjazdu| 0/- spójność utrzymuje silnik|− konfiguracja cache headers i purge dla statycznych assetów strony (JS/CSS/obrazy) — nie dotyka danych grafiku| ++ — jeden indeks (koszt: minimalnie wolniejsze zapisy)      |

**Odwracalność:** 
Wszystkie rozważane opcje poza D to two-way doors: G — DROP INDEX (minuty), A — wyłączenie cache flagą (godziny), E — DROP VIEW (godziny). D wsiąka w schemat i wszystkie ścieżki zapisu — wycofanie to tygodnie (częściowo one-way). Wniosek: decyzja jest tania w odwróceniu, więc lepiej szybko wdrożyć i zmierzyć, niż dłużej analizować.

**Rekomendacja**
Krok 1 — G: tylko indeks. Sprawdzamy plan zapytania (EXPLAIN, ~30 min); zakładamy brak indeksu pod filtr klub+data. Zakładamy indeks, mierzymy p95 pod obciążeniem 200 RPS. Jeśli p95 < 500 ms — koniec: cel osiągnięty zerowym kosztem operacyjnym i staleness = 0.
  
Krok 2 — jeśli p95 nadal > 500 ms: A, cache Redis z inwalidacją push przy każdej zmianie grafiku (odwołanie/dodanie zajęć). Świadomie płacę: nową usługą do utrzymania, kodem inwalidacji i oknem staleness do ~2 s — mieści się w założonym limicie ≤ 5 s. Dostaję: odczyt ~1 ms i zdjęcie ruchu odczytowego z bazy.
 
Odrzucone (w kolejności odpadania):
- B (cache in-memory): DQ — łamie constraint świeżości: przy wielu instancjach rozjazd cache daje staleness nieograniczony > 5 s.
- C (read replica): 0 na kryterium #1 — nie skraca zapytania, tylko zdejmuje kontencję; nie rozwiązuje postawionego problemu.
- F (CDN): 0 na kryterium #1 — endpoint grafiku dalej wykonuje to samo wolne zapytanie. Zostaje jako niezależne uzupełnienie dla statycznych assetów strony.
- D (denormalizacja): przegrywa złożonością (−/−−, ryzyko trwałego rozjazdu) i jako jedyna jest częściowo one-way — nie płacimy nieodwracalnością tam, gdzie odwracalne opcje dają ten sam efekt.
- E (indexed view): przegrywa wykonalnością. ZAŁOŻENIE: zapytanie grafiku wymaga LEFT JOIN/podzapytań, których indexed view w Azure SQL nie wspiera — DO WERYFIKACJI przy okazji kroku 1. Jeśli weryfikacja pokaże, że zapytanie da się przepisać pod ograniczenia indexed view — E wygrywa z A (ten sam efekt bez nowej usługi) i krok 2 zmienia zwycięzcę.
 
Trigger powrotu do decyzji: (a) zgłoszenia członków o nieaktualnym grafiku (inwalidacja zawodzi — staleness przekracza 5 s), (b) wzrost ruchu powyżej ~500 RPS, (c) koszty lub awaryjność Redis zaczynają przewyższać koszt alternatyw.