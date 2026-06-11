# Dziennik decyzji (ADR-lite)

Rejestr moich decyzji o konsekwencjach dłuższych niż tydzień, w formacie
kontekst → decyzja → konsekwencje. Zasady:

- wpisy są **niemutowalne** — zmiana zdania = nowy wpis z adnotacją "unieważnia D-NNN",
- najnowsze wpisy **na górze**,
- próg wejścia: decyzja ma konsekwencje **dłuższe niż tydzień**,
- wpis powstaje w momencie decyzji (max do najbliższego przeglądu tygodnia), zajmuje 5–10 minut.

Szablon wpisu:

```markdown
## D-NNN: Krótki tytuł decyzji (RRRR-MM-DD)

**Kontekst:** 2–5 zdań. Sytuacja, ograniczenia, siły. Opis świata,
nie uzasadnienie wyboru.

**Decyzja:** 1–2 zdania w trybie oznajmującym. Robię X (zamiast Y, bo ...).

**Konsekwencje:**
- (+) co zyskuję
- (–) co mnie to kosztuje / co staje się trudniejsze
- (?) czego nie wiem — co zweryfikuje czas (opcjonalnie)
```

---

## D-003: Rezygnacja z dwóch repoztoriów public/private (2026-06-11)

**Kontekst:** Lekcja 02 zakłada publiczne/prywatne repozytorium..

**Decyzja:** Będzie jedno repozytorium, w którym będą przechowywane wszystkie informacje.

**Konsekwencje:**
- (+) pełną chronologię
- (+) brak zamieszania z potrzebą commitowania do różnych repozytoriów.
- (+) możliwość skupienia się na pracy, a nie na separaci, która w kontekście nauki jest po prostu niepotrzebnym kosztem.
- (?) czego nie wiem — czy w przyszłości moje notatki, błędy będą postrzegane jako negatywy podczas rozpatrywania mojej kandydatury

## D-002: Budżet błędów rytmu nauki — 2 z 4 bloków, bez odrabiania, przegląd nietykalny (2026-06-11)

**Kontekst:** Rytm z lekcji 01 zakłada 4 bloki deep work po 2h tygodniowo, plus post, komentowanie i przegląd tygodnia. Równolegle rozwijam DayChunks — w tygodniu startowym dobudowanie szablonu tygodnia pochłonęło 16h (D-001)
i ta praca będzie kontynuowana. Budżet błędów ma być zaprojektowany na mój przeciętny tydzień (praca, rodzina, DayChunks), nie na tydzień największej motywacji — zbyt ambitny próg uruchomi mechanizm "wszystko albo nic", przed którym ostrzega lekcja.

**Decyzja:** Tydzień jest zaliczony przy **2 z 4 bloków deep work** (zamiast 3/4 z lekcji, bo na etapie równoległej pracy nad DayChunks ostrzejszy próg byłby progiem na najlepszy tydzień, nie przeciętny). Reguły towarzyszące: **zerwanego bloku nie odrabiam** — wracam do najbliższego zaplanowanego; budżet obejmuje tylko bloki deep work, a **przegląd tygodnia odbywa się zawsze** — to mechanizm samonaprawy i jedyny element bez budżetu. Dopiero drugi tydzień z rzędu poniżej budżetu to incydent do analizy na przeglądzie.

**Konsekwencje:**
- (+) plan przeżyje gorsze tygodnie — jeden zły tydzień nie uruchamia  spirali "plan stracony"
- (+) brak odrabiania = brak lawiny zaległości przeciążającej kolejne dni
- (+) nietykalny przegląd gwarantuje, że nawet słaby tydzień kończy się pętlą korekty, a nie cichym dryfem
- (–) próg 2/4 to połowa zakładanego tempa — jeśli stanie się normą, a nie podłogą, kurs planowany na 6–9 miesięcy rozciągnie się do 12 i więcej
- (–) ryzyko kotwiczenia: budżet błędów łatwo zamienia się w cel ("2 bloki zrobione, jestem OK") — na przeglądzie liczę faktyczne bloki,  nie tylko zaliczenie progu
- (?) czy 2/4 to kalibracja na czas prac nad DayChunks, czy stałe tempo — zweryfikuję po 4 tygodniach; jeśli regularnie wychodzą 3–4 bloki, nowy wpis podniesie próg (unieważni D-002)

## D-001: Dobudowuję brakującą funkcję w DayChunks zamiast obchodzić jej brak (2026-06-11)

**Kontekst:** Prowadzę naukę z kursu Architect Track według rytmu z modułu 0, a bloki nauki planuję w DayChunks — własnej aplikacji do planowania dnia chunkami, którą rozwijam po godzinach. Lekcja 01 wymaga powtarzalnego szablonu tygodnia z blokami nauki, a okazało się, że DayChunks nie oferuje funkcji szablonu tygodnia. 

**Decyzja:** Buduję brakującą funkcję w DayChunks (zamiast obejścia lub zmiany narzędzia, bo dogfooding to świadoma strategia — kurs właśnie ujawnił realny brak w produkcie, a obejście oznaczałoby cotygodniowy ręczny koszt bez trwałej wartości). 
Nauki nie wstrzymuję: funkcję będę dobudowywał równolegle i kontynuuję moduł 0 (lekcja 02 — repo nauki — ukończona).

**Konsekwencje:**
- (+) funkcja powstała z realnej potrzeby, nie z wyobrażonej — najlepszy  możliwy sygnał produktowy
- (+) gotowa historia na post LinkedIn: "własny kurs ujawnił brak we  własnym produkcie" (dogfooding w praktyce)
- (+) cotygodniowe planowanie bloków przestaje mieć ręczny koszt
- (–) 16 godzin poszło w DayChunks zamiast w naukę — rytm tygodnia  ucierpiał w tygodniu startowym
- (–) precedens: każdy kolejny brak w DayChunks będzie kusił, żeby "tylko dobudować" — muszę pilnować progu (buduję tylko to, co blokuje rytm nauki, reszta idzie do backlogu)
- (?) czy funkcja w obecnym kształcie przyda się innym użytkownikom? Tak, ale w tym momencie jest ona dla mnie istotna, a dla innych może stanowić wartość dodaną lub zachętę do rozpoczęcia korzystania z narzędzia.
