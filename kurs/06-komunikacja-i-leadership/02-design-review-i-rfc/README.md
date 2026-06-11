# Lekcja 02 — Design review i RFC: architektura jako proces zespołowy

## Cel lekcji

Po tej lekcji będziesz umiał napisać dokument RFC (Request for Comments) od zera, poprowadzić design review tak, żeby zespół naprawdę dyskutował (a nie przytakiwał), oraz kulturalnie i skutecznie zgłaszać uwagi do cudzych projektów i przyjmować krytykę własnych.

## Dlaczego to ważne

Architektura podejmowana w pojedynkę ma dwa tryby awarii: albo decyzja jest błędna i nikt jej nie zatrzymał, albo jest dobra i nikt jej nie rozumie — więc zespół wykonuje ją bez przekonania, po cichu obchodząc ją w kodzie. Dojrzałe organizacje (Google, Amazon, Stripe, ale też coraz częściej polskie software house'y) traktują architekturę jako **proces pisemny i zespołowy**: propozycja → dokument → komentarze → decyzja. Na rozmowach na role Lead/Architect pytanie "jak u Ciebie wyglądał proces podejmowania decyzji architektonicznych?" pada niemal zawsze — i odpowiedź "ja decydowałem, bo znałem system najlepiej" jest odpowiedzią seniora-wykonawcy, nie architekta.

Dla Ciebie jest tu też wątek osobisty: jako wieloletni owner systemu Perfect Gym byłeś w pozycji, w której Twoje zdanie domyślnie wygrywało. To wygodne i niebezpieczne — ta lekcja uczy, jak używać autorytetu tak, żeby nie zabijał dyskusji. A praktycznie: RFC, które tu napiszesz, będzie pierwszym dokumentem Twojego Capstone.

## Teoria

### Problem: decyzje podejmowane na spotkaniach i w głowach

Naiwny proces decyzyjny w zespołach wygląda tak: ktoś rzuca problem na spotkaniu, godzina dyskusji, najgłośniejszy lub najstarszy stażem "wygrywa", ktoś coś notuje (albo nie). Dlaczego to się psuje:

- **Decyzja zapada w czasie rzeczywistym** — przewagę mają szybcy mówcy, nie najlepsze argumenty. Osoby potrzebujące czasu na przemyślenie (często najlepsi inżynierowie) milczą.
- **Brak zapisu** — pół roku później nikt nie pamięta, *dlaczego* tak zdecydowano, więc decyzję się podważa albo bezwiednie łamie.
- **Brak udziału nieobecnych** — w zespole rozproszonym lub z różnymi strefami czasowymi część ludzi jest strukturalnie wykluczona.

Właściwy wzorzec: **decyzja jako dokument** + dyskusja asynchroniczna + spotkanie tylko do rozstrzygnięcia spornych punktów. I tu wchodzą dwa narzędzia: RFC i design review.

### RFC od zera: czym jest i jak wygląda

**RFC (Request for Comments)** — dosłownie "prośba o komentarze" — to dokument opisujący **proponowaną** (jeszcze niepodjętą) decyzję lub projekt, rozesłany do zespołu z jawnym zaproszeniem do krytyki. Nazwa pochodzi z lat 60. od dokumentów, którymi twórcy internetu uzgadniali protokoły (RFC 791 to specyfikacja IP) — duch jest ten sam: "oto moja propozycja, rozstrzelajcie ją, zanim ją zbudujemy". W firmach spotkasz też nazwy: design doc (Google), one-pager/six-pager (Amazon), tech spec — różnice są kosmetyczne.

Struktura RFC (minimalna, praktyczna):

```
# RFC-NNN: Tytuł (czasownik + przedmiot, np. "Wprowadzenie outboxa w serwisie rezerwacji")
- Status: Draft | In Review | Accepted | Rejected | Superseded
- Autor, data, deadline na komentarze (np. +5 dni roboczych)

## Summary — 3–5 zdań: co proponujesz i dlaczego (piramida z lekcji 01!)
## Kontekst i problem — co się dzieje, dlaczego status quo nie wystarcza; liczby, jeśli są
## Proponowane rozwiązanie — opis + diagram; na tyle szczegółowo, żeby dało się krytykować
## Rozważane alternatywy — 2–3 opcje z powodami odrzucenia (sekcja, po której poznaje się dobre RFC)
## Trade-offy i ryzyka — co świadomie pogarszamy, co może pójść źle, jak mitygujemy
## Open questions — pytania, na które autor NIE zna odpowiedzi (jawne zaproszenie do dyskusji)
```

Cykl życia RFC: **Draft** (piszesz) → **In Review** (zespół komentuje, zwykle 3–7 dni) → rozstrzygnięcie spornych wątków (komentarze lub krótkie spotkanie) → **Accepted / Rejected** → po akceptacji kluczowa decyzja trafia do ADR-a. Status **Superseded** oznacza "zastąpiony nowszym dokumentem" — RFC się nie kasuje, bo historia odrzuconych pomysłów jest cenna ("już to rozważaliśmy w 2024, oto dlaczego odpadło").

### RFC vs ADR — kiedy które

Znasz już ADR-y z modułu 1. Rozróżnienie jest proste i warto je umieć powiedzieć jednym zdaniem na rozmowie:

| | RFC | ADR |
|---|---|---|
| Czym jest | **propozycja do dyskusji** | **zapis podjętej decyzji** |
| Czas powstania | przed decyzją | w momencie decyzji / tuż po |
| Ton | "proponuję, co myślicie?" | "zdecydowaliśmy, oto dlaczego" |
| Długość | strony (z alternatywami i analizą) | zwięzły (kontekst → decyzja → konsekwencje) |
| Żyje | przez okres review, potem zamrożony | tak długo, jak decyzja obowiązuje |

Typowy przepływ: problem → RFC → dyskusja → decyzja → **ADR podsumowujący wynik** (często z linkiem do RFC). Małe decyzje mogą iść od razu do ADR-a bez RFC; duże lub kontrowersyjne zasługują na pełny cykl. Praktyczna heurystyka: jeśli decyzja jest droga w odwróceniu albo dotyka pracy więcej niż jednego zespołu/obszaru — pisz RFC.

### Design review: jak je prowadzić, żeby nie zabić dyskusji

**Design review** to przegląd projektu (designu) przed implementacją — odpowiednik code review, tylko o poziom wcześniej i taniej: poprawka w dokumencie kosztuje godzinę, poprawka w zbudowanym systemie — tygodnie. Review może być w pełni asynchroniczne (komentarze do dokumentu) albo hybrydowe (async + 30–60 min spotkania na sporne punkty).

Najtrudniejsza część nie jest techniczna, tylko społeczna — zwłaszcza gdy prowadzisz review jako najbardziej doświadczona osoba na sali. Zjawisko do nazwania: **HiPPO** (Highest Paid Person's Opinion) — dyskusja samoczynnie układa się pod zdanie najwyższego rangą/stażem. Gdy 20-letni weteran mówi "ja bym to zrobił przez kolejkę", dyskusja umiera, nawet jeśli intencją było luźne głośne myślenie. Twoje opinie ważą więcej, niż chcesz — musisz to aktywnie kompensować.

Zasady prowadzącego:

1. **Pytania zamiast wyroków.** Zamiast "to się nie przeskaluje" → "jak to się zachowa przy 10× ruchu? policzmy". Zamiast "źle, tu trzeba outbox" → "co się stanie, gdy zapis do bazy się uda, a publikacja zdarzenia nie?". Pytanie prowadzi autora do wniosku — i autor ten wniosek *posiada*; wyrok prowadzi do obrony albo kapitulacji.
2. **Mów ostatni.** Jako senior/prowadzący wstrzymaj własną opinię, aż wypowiedzą się inni — szczególnie młodsi. Gdy zaczniesz od swojej, resztę usłyszysz już tylko jako warianty Twojej.
3. **Wyciągaj milczących wprost**, po imieniu i konkretnie: "Kasia, Ty ostatnio walczyłaś z tym modułem — widzisz tu coś, co nas ugryzie?". Ogólne "czy ktoś ma uwagi?" generuje ciszę.
4. **Oddziel rundę zrozumienia od rundy krytyki.** Najpierw pytania doprecyzowujące ("czy dobrze rozumiem, że…"), dopiero potem ocena. Połowa sporów na review to spory o różne wyobrażenia tego samego dokumentu.
5. **Nazwij wynik.** Review bez konkluzji to strata czasu. Kończ jawnie: co zaakceptowane, co do poprawy, co eskalujemy, kto decyduje o nierozstrzygniętych punktach i do kiedy.

### Asynchroniczny review w zespole rozproszonym

W zespole rozproszonym (zdalnym, wielostrefowym) spotkanie jest drogie, więc ciężar przenosi się na pracę pisemną:

- **Dokument w miejscu z komentarzami do akapitów** (Google Docs, Notion, pull request z plikiem `.md` w repo — komentarze do linii). RFC w repo ma zaletę: historia zmian i review w tym samym narzędziu co kod.
- **Jawny deadline na komentarze** ("review do piątku EOD") — bez deadline'u review trwa wiecznie. Brak komentarza do deadline'u = zgoda (zasada **lazy consensus**: milczenie oznacza akceptację; trzeba ją ogłosić z góry, inaczej milczenie znaczy "nie przeczytałem").
- **Autor odpowiada na każdy komentarz** — choćby słowem "przyjęte, poprawione w sekcji X" albo "nie zgadzam się, bo Y". Komentarze bez odpowiedzi uczą ludzi, że komentowanie nie ma sensu.
- **Spotkanie tylko dla wątków, w których wymiana komentarzy przekroczyła ~3 rundy** — to sygnał, że pisemnie się nie dogadacie.

### Jak zgłaszać uwagi i jak przyjmować krytykę

Zgłaszanie uwag do cudzego projektu — trzy składniki dobrego komentarza:

1. **Konkretność**: wskaż miejsce i scenariusz, nie wrażenie. ❌ "ta integracja wygląda krucho" → ✅ "sekcja 4: gdy system płatności odpowie po timeout, a my już zwróciliśmy błąd użytkownikowi — kto wycofuje rezerwację?".
2. **Intencja**: zaznacz wagę komentarza. Konwencja: **blocking** (nie akceptuję bez rozwiązania) vs **non-blocking / nit** (drobiazg, do rozważenia). Bez tego autor traktuje wszystkie 20 uwag jako równie ważne i tonie.
3. **Alternatywa**: krytyka bez propozycji to zgłoszenie problemu; krytyka z propozycją to wkład. Nie zawsze masz gotowe rozwiązanie — wtedy powiedz to wprost: "nie wiem, jak to rozwiązać, ale ten scenariusz musi być obsłużony".

Przyjmowanie krytyki własnego projektu — trudniejsza połowa, szczególnie po latach bycia "tym, który wie najlepiej":

- **Oddziel siebie od dokumentu.** Komentarz "ten design ma dziurę" nie znaczy "jesteś słabym inżynierem". Pomaga sztuczka językowa: mów o projekcie "ten design", nie "mój design".
- **Najpierw doprecyzuj, potem odpowiadaj.** Odruch obrony każe odpowiadać natychmiast; zamiast tego: "rozumiem, że martwi Cię scenariusz X — czy dobrze czytam?". Połowa krytyki rozmywa się przy doprecyzowaniu, a reszta staje się konkretna.
- **Publicznie przyznawaj trafienia.** "Masz rację, nie pomyślałem o tym — poprawiam" napisane na forum zespołu to najtańsza inwestycja w kulturę review: pokazuje, że krytykowanie jest bezpieczne i skuteczne, skoro działa nawet wobec najstarszego stażem.
- **Miej kryterium rozstrzygania.** Gdy po wymianie argumentów dalej się nie zgadzacie: kto decyduje? (owner obszaru / architekt / głosowanie / eskalacja). Brak reguły = decyduje wytrwałość, a to najgorsza możliwa reguła. Przydaje się zasada **disagree and commit**: mam inne zdanie, zgłosiłem je, decyzja zapadła inna — wspieram ją w pełni, bez sabotowania "a nie mówiłem".

### Kiedy tego NIE stosować (trade-offy)

- **RFC dla każdej decyzji = paraliż.** Wybór nazwy klasy czy biblioteki do mapowania nie potrzebuje tygodnia review. Proces pisemny rezerwuj dla decyzji drogich w odwróceniu lub przecinających granice zespołów.
- **Podczas pożaru nie pisze się RFC.** Awaria produkcyjna o 3 w nocy wymaga decyzji w minutach — wtedy decyduje incident commander, a dokumentem jest post-mortem *po* fakcie, nie propozycja *przed*.
- **Konsensus nie jest celem samym w sobie.** Celem jest dobra decyzja podjęta świadomie i zapisana. Dążenie do 100% zgody wszystkich produkuje rozwiązania kompromisowe, które nie cieszą nikogo — stąd właśnie "disagree and commit".

## Praktyka

- [ ] Wybierz **jedną realną decyzję do podjęcia w nadchodzącym Capstone** (dobre kandydatki: orchestration vs choreography dla sagi rezerwacji; Azure Service Bus vs Event Hubs jako broker; jedna baza vs database-per-service). Wybierz taką, co do której *naprawdę* masz wątpliwości.
- [ ] Napisz pełne RFC według struktury z teorii: Summary, kontekst, propozycja, **minimum 2 alternatywy z powodami odrzucenia**, trade-offy i ryzyka, minimum 2 open questions. Zapisz jako `rfc-001-<temat>.md` w folderze tej lekcji (później skopiujesz do repo Capstone).
- [ ] Ustaw status `In Review` i przeprowadź rundę review: jeśli masz znajomego inżyniera — poproś o komentarze z deadline'em; jeśli nie — użyj Claude jako reviewera (prompt: *"Jesteś doświadczonym architektem robiącym design review poniższego RFC. Zgłoś 5–8 uwag: każda z lokalizacją w dokumencie, oznaczeniem blocking/non-blocking i — gdzie umiesz — alternatywą. Zadawaj pytania o scenariusze awarii, skalowanie i koszty. Nie chwal."*).
- [ ] Odpowiedz pisemnie na **każdy** komentarz (przyjęte + zmiana / odrzucone + powód). Zaktualizuj dokument i status na `Accepted` lub `Rejected`.
- [ ] Wynik zapisz dodatkowo jako krótki ADR (kontekst → decyzja → konsekwencje) — masz wtedy na żywo przećwiczoną parę RFC → ADR.
- [ ] Pytanie kontrolne (odpowiedz pisemnie w 3 zdaniach): czym różni się RFC od ADR-a i dlaczego zespołowi potrzebne są oba?

## Artefakt

1. **RFC dla decyzji w Capstone** (`rfc-001-<temat>.md`) — z pełnym cyklem: draft → review z komentarzami i Twoimi odpowiedziami → status końcowy. Pierwszy dokument Twojego Capstone.
2. **ADR podsumowujący decyzję** z linkiem do RFC.

## Definition of Done

- [ ] RFC istnieje, ma wszystkie sekcje z szablonu, w tym minimum 2 odrzucone alternatywy i minimum 2 open questions.
- [ ] RFC przeszedł co najmniej jedną rundę komentarzy (człowiek lub Claude) i każdy komentarz ma Twoją pisemną odpowiedź.
- [ ] Status RFC jest rozstrzygnięty (`Accepted`/`Rejected`) i istnieje ADR z wynikiem.
- [ ] Umiesz powiedzieć jednym zdaniem różnicę RFC vs ADR i wymienić z pamięci 3 zasady prowadzenia design review tak, by nie zabić dyskusji autorytetem.
- [ ] Co najmniej jeden komentarz z review faktycznie zmienił treść dokumentu — jeśli żaden nie zmienił, review było fasadowe; powtórz z ostrzejszym reviewerem.

## Materiały

1. [Design Docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/) — Malte Ubl; najlepszy zwięzły opis kultury dokumentów projektowych: struktura, cykl życia, kiedy pisać, a kiedy nie.
2. [Scaling Engineering Teams via RFCs](https://blog.pragmaticengineer.com/scaling-engineering-teams-via-writing-things-down-rfcs/) — Gergely Orosz; praktyka RFC w zespołach rozproszonych, z szablonem i pułapkami procesu.
