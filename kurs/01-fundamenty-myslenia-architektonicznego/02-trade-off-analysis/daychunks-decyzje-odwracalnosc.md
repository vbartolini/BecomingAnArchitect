 # DayChunks — decyzje na osi odwracalności

**Kontekst:** DayChunks jest przed startem — jedna osoba utrzymuje, brak realnych użytkowników, dane trzymane lokalnie (localStorage / PWA). Na tym etapie większość drzwi jest jeszcze dwukierunkowa; one-way doory pojawią się dopiero, gdy będę miał userów i ich dane. Ta lista ma pokazać, której decyzji należy się pełna analiza, a którą biorę w 5 minut — żeby staranność była proporcjonalna do odwracalności.

## Tabela decyzji

| # | Decyzja | One-way / two-way | Dlaczego (co narasta przy wyjściu) | Należna staranność |
|---|---------|-------------------|------------------------------------|--------------------|
| 1 | Umożliwienie eksportu/importu szablonów dnia | two-way (dźwignia odwracalności) | Sama w sobie odwracalna (kod). To ona *czyni odwracalnymi* #2 i #5: dzięki eksportowi JSON dane userów wychodzą z localStorage do bazy i z powrotem bez strat. | Niskie ryzyko, ale **zdecyduj wcześnie i na TAK** — tania polisa, która rozbraja najgroźniejsze one-way doory (#2, #5). |
| 2 | Zmiana modelu danych związana z potrzebą wprowadzenia planowania całego tygodnia - szablon tygodniowy | two-way teraz → częściowo one-way po zapisaniu danych userów | Zmiana schematu to nie kod, to dane. W localStorage nie ma centralnej bazy — migrację trzeba wykonać u każdego usera osobno (przy starcie appki, z wersjonowaniem schematu). Im więcej userów z danymi, tym droższe wyjście. | **Pełna analiza teraz** — last responsible moment. Przemyśleć schemat (dzień jako atom, tydzień = kolekcja dni) ZANIM pojawią się userzy z danymi. |
| 3 | Zbieranie danych o liczbie pobrań strony z unikalnych adresów | technicznie two-way (licznik się wyłącza), ale częściowo one-way w warstwie privacy/zaufania | Śledzenie unikalnych adresów wraca w teren cookies/RODO — łamie obietnicę z kryterium nr 2 analizy głównej („bez logowania, cookies i zgód"). Raz dodany banner zgód i pozycja „jednak śledzimy" trudno cofnąć bez utraty wizerunku prywatnej appki. | **Pełna analiza** — dotyka rdzenia pozycjonowania. Najpierw alternatywy privacy-friendly (agregaty serwerowe bez ciasteczek, bez identyfikacji jednostki). |
| 4 | Wprowadzenie kolorystycznych szablonów dla wyglądu strony | two-way | Czysty UI, jeden moduł. Zero wpływu na dane userów i zobowiązania — dodaj/usuń bez kosztu. | **5 minut**, niski szczebel. Modelowy two-way door — żadnej macierzy. |
| 5 | Utworzenie wersji serwerowej w celu ułatwienia korzystania z aplikacji na wielu urządzeniach | two-way dla danych, ale **silnie one-way dla zobowiązań** | Serwer + konta + sync = stajesz się administratorem danych osobowych (RODO: zgody, prawo do usunięcia), masz infrę do utrzymania 24/7, userzy zależą od dostępności. Wyjście = migracja userów, demontaż infry, złamanie privacy-posture. To dokładnie opcja A z `analiza-trade-off-01.md`. | **Pełna macierz + ADR.** Opóźnić do last responsible moment (realna potrzeba multi-device). Dzięki #1 wejście/wyjście mniej bolesne. |

## Oś odwracalności (od najłatwiej odwracalnych do najtrudniej)

```
two-way ─────────────────────────────────────────────► one-way
  [#4]   [#1]   [#2 przed startem]   [#3 privacy]   [#2 po danych]   [#5]
```

## Wnioski

- **Decyzja w 5 minut:** #4 (kolory) — two-way, jeden moduł, zero wpływu na dane.
- **Pełna macierz / ADR:** #5 (wersja serwerowa — logika już rozpisana w analizie głównej), #3 (statystyki — rusza privacy-posture) oraz #2 *teraz, przed startem* (model danych, póki nie ma userów z danymi do migracji).
- **#1 to dźwignia, nie punkt na osi** — zdecyduj o eksporcie/imporcie wcześnie, bo tanim kosztem przesuwa #2 i #5 ku two-way. To „zapłać za zamianę one-way w two-way" z lekcji.
- **Ryzyko odwrotnej alokacji uwagi:** najłatwiej przegadać godziny o #4 (kolory), a #5 i #3 przepuścić „jakoś tak" — choć to one są one-way. Uwagę alokuj odwrotnie do tego, co kusi.