# Pomysły na posty LinkedIn

Lista tematów do przerobienia na posty. Jeden pomysł = jeden blok. Po publikacji oznacz `[x]` i wklej link.

---

## 1. One-way vs two-way doors — czego architekt NIE robi

- [x] Status: pomysł
- **Źródło:** Moduł 1, lekcja 02 (Trade-off analysis); artefakty `analiza-trade-off-01.md` + `daychunks-decyzje-odwracalnosc.md`
- **Język:** do decyzji (polski dla sieci PL / angielski dla zasięgu — patrz polityka językowa)

**Teza:** Najczęstszy błąd zespołów to *odwrotna alokacja uwagi* — tygodnie sporów o rzeczy odwracalne (formatowanie, kolory), a decyzje nieodwracalne (baza, kontrakt API, wersja serwerowa) „jakoś wychodzą" na daily.

**Hak (możliwe otwarcia):**
- „Spędziliśmy godzinę na kolorach motywu. Decyzję o backendzie podjęliśmy w 5 minut. Dokładnie odwrotnie, niż powinniśmy."
- „Pierwsze prawo architektury Richardsa/Forda: *everything is a trade-off*. Drugie, ważniejsze pytanie: czy te drzwi się jeszcze otworzą?"

**Oś narracji:**
1. Two-way door (odwracalna, decyduj szybko) vs one-way door (nieodwracalna, zwolnij).
2. Klucz: to nie *dane* czynią decyzję one-way — dane prawie zawsze się przemigruje. One-way robią z decyzji *zobowiązania* (RODO, konta, publiczny kontrakt) i ich asymetria (B→A tanio, A→B drogo).
3. Realny przykład: DayChunks bez backendu jako świadomy wybór, czego NIE robić — wersja serwerowa to one-way (RODO + infra), a eksport/import JSON to dźwignia, która *kupuje* odwracalność.
4. Puenta: dojrzałą architekturę poznaje się po tym, że lista rezygnacji jest wypisana wprost, zanim ktoś odkryje ją na produkcji.

**Artefakt do dołączenia:** tabela decyzji na osi odwracalności (z `daychunks-decyzje-odwracalnosc.md`) jako grafika.

**Do przemyślenia:** czy łączyć z wątkiem „it depends" (poprawna pierwsza odpowiedź architekta), czy zostawić na osobny post.

Link: https://www.linkedin.com/posts/bartosz-kanak-0a436a124_we-spent-an-hour-discussing-theme-colors-activity-7473279734722887680-skZH?utm_source=share&utm_medium=member_desktop&rcm=ACoAAB6qTfwBMdzMvn7qLEziKcntH3FxdBAOoUk