# Postęp kursu Architect Track

> Aktualizuj po każdym tygodniu (rytuał z modułu 00: przegląd tygodnia). Jedna sekcja = jeden tydzień. Nie kasuj starych wpisów — to dziennik, nie dashboard.

## Status modułów

- [ ] 00 — Setup i rytm pracy
- [ ] 01 — Fundamenty myślenia architektonicznego
- [ ] 02 — Systemy rozproszone
- [ ] 03 — Cloud-native na Azure
- [ ] 04 — Nowoczesny stack .NET
- [ ] 05 — AI-augmented engineering (równolegle z 2–4)
- [ ] 06 — Komunikacja i leadership
- [ ] 07 — Capstone

## Szablon wpisu tygodniowego

```markdown
## Tydzień N (RRRR-MM-DD – RRRR-MM-DD)

**Moduł / lekcje:** ...
**Zrobione:**
- ...

**Artefakty:** (kod / diagram / ADR / post — linki)
- ...

**LinkedIn:** post: tak/nie | komentarze: N | nowe kontakty: N

**Przegląd tygodnia:**
- Co działało: ...
- Co wyciąć / zmienić: ...
- Czy w tym tygodniu podjąłem decyzję wartą wpisu do decision-log?
```

---

<!-- Wpisy tygodniowe dodawaj poniżej, najnowszy na górze. -->

**Moduł / lekcje:** 00 — Setup i rytm pracy, lekcje 01–03 (2026-06-09 - 2026-06-11)

  **Zrobione:**
  - zaprojektowany rytm tygodnia: 4× 2h rano, post, komentowanie, przegląd
  - repo nauki w git: struktura katalogów, .gitignore, konwencja commitów, progress.md
  - dziennik decyzji `notes/decision-log.md` z wpisami D-001 (dobudowa funkcji
    w DayChunks) i D-002 (budżet błędów 2/4)
  - pytanie o nowe decyzje dodane do formatu przeglądu tygodnia
  - [N] z 4 bloków deep work odbyło się naprawdę → budżet 2/4: [zaliczony / nie]
  - poza planem: ~16h pracy nad funkcją szablonu tygodnia w DayChunks

  **Artefakty:** (kod / diagram / ADR / post — linki)
  - `notes/decision-log.md` — D-001, D-002
  - historia commitów w konwencji `typ(moduł)` (4 commity)
  - szablon tygodnia w DayChunks — **nie powstał**: zablokowany przez brak
    funkcjonalności w produkcie, w budowie (patrz D-001)

  **LinkedIn:** post: nie | komentarze: 0 | nowe kontakty: 0

  **Przegląd tygodnia:**
  - Co działało: tempo przerabiania treści (moduł 0 w 3 dni); commit po każdym
    bloku jako zdefiniowane wyjście
  - Co nie zadziałało i dlaczego: nie dało się domknąć Definition of Done modułu —
    szablon tygodnia wymaga funkcji, której DayChunks nie ma; przyczyna systemowa:
    plan założył możliwość produktu bez jej weryfikacji, koszt ujawnił się jako 16h
    poza budżetem nauki
  - Co wyciąć / zmienić: do czasu ukończenia funkcji planuję tydzień w DayChunks
    ręcznie (rytm nie czeka na narzędzie); praca nad funkcją idzie poza blokami
    deep work
  - Nowe decyzje: tak — D-001 i D-002 (oba zapisane)