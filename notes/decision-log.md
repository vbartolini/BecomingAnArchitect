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
