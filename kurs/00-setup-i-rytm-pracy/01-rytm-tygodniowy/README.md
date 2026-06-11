# Lekcja 01 — Rytm tygodniowy: projektowanie trwałego systemu nauki

## Cel lekcji

Po tej lekcji będziesz miał zaprojektowany i wpisany w DayChunks konkretny tygodniowy rytm nauki (4× 2h deep work, 1× post, 3–4× komentowanie, 1× przegląd) oraz format cotygodniowego przeglądu, który pozwoli ten rytm korygować zamiast porzucać.

## Dlaczego to ważne

Plan Architect Track to 6–9 miesięcy pracy. Na rozmowach rekrutacyjnych nikt nie zapyta Cię o rytm tygodniowy — ale **wszystko, o co zapytają** (C4, ADR-y, outbox pattern, Azure, capstone), powstanie tylko wtedy, gdy ten rytm przetrwa. Architekt to także osoba, która projektuje procesy, nie tylko systemy. Pierwszym systemem, który zaprojektujesz w tym kursie, jest Twój własny tydzień — i potraktujesz go dokładnie tak, jak system produkcyjny: z monitoringiem (przegląd tygodnia), budżetem błędów (dopuszczalne odstępstwa) i mechanizmem recovery (co robić po zerwanym tygodniu).

Jest też powód czysto praktyczny: budujesz DayChunks — aplikację do planowania dnia blokami. Używanie własnego produktu do prowadzenia własnej nauki to dogfooding (praktyka używania własnego produktu na co dzień, żeby odczuć jego braki na własnej skórze) — i jednocześnie surowiec na posty LinkedIn.

## Teoria

### Problem: motywacja jako fundament planu

Naiwne podejście do długiego planu nauki wygląda tak: "jestem zmotywowany, będę się uczył codziennie po pracy". To projekt systemu, w którym pojedynczy komponent (motywacja) jest jednocześnie single point of failure (pojedynczy punkt awarii — element, którego usterka wyłącza cały system). Motywacja jest zasobem zmiennym: wysoka na starcie, spada przy pierwszym trudnym temacie, znika przy zmęczeniu, chorobie dziecka czy gorącym tygodniu w pracy.

Dlaczego to się psuje: decyzja "czy dziś się uczę" podejmowana codziennie od nowa kosztuje energię i codziennie może wypaść negatywnie. Po kilku negatywnych decyzjach z rzędu plan przestaje istnieć — nie przez jedną dużą porażkę, tylko przez serię małych "dziś nie".

Właściwy wzorzec: **rytm zamiast motywacji**. Rytm to decyzja podjęta raz ("uczę się w poniedziałek, wtorek, czwartek i piątek, 6:30–8:30"), która potem nie wymaga codziennego renegocjowania. Analogia z Twojego świata: to różnica między ręcznym wyzwalaniem joba a cronem. Cron nie pyta, czy ma ochotę — po prostu odpala się o zaplanowanej godzinie. Twoim zadaniem nie jest "chcieć się uczyć", tylko **nie odwoływać crona**.

Trade-off: rytm jest sztywny, życie nie. Dlatego rytm projektuje się z marginesem (o tym niżej, w "budżecie błędów"), a nie jako perfekcyjny harmonogram, który pęka przy pierwszym odstępstwie.

### Deep work: czym jest i dlaczego 2h, a nie 4×30 min

**Deep work** (głęboka praca) — termin spopularyzowany przez Cala Newporta — to praca w pełnym skupieniu, bez przerywników, nad zadaniem wymagającym wysiłku poznawczego. Przeciwieństwem jest shallow work (płytka praca): maile, Slack, "szybkie sprawdzenie LinkedIn".

Kluczowy mechanizm: **koszt przełączania kontekstu** (context switching). Znasz go z produkcji — wątek, który ciągle dostaje przerwania, nie kończy niczego. W mózgu działa to podobnie: po każdym rozproszeniu powrót do pełnego skupienia nad trudnym materiałem (np. modelem spójności w systemach rozproszonych) zajmuje 10–20 minut. Cztery bloki po 30 minut z przerwami to w praktyce 4× rozgrzewka i prawie zero pracy na pełnej głębokości. Jeden blok 2h to jedna rozgrzewka i ~100 minut realnej nauki. Dlatego plan zakłada 4× 2h, a nie "godzinka dziennie kiedy wyjdzie".

Jak chronić blok 2h — konkretne mechanizmy:

1. **Pora: rano, jako pierwszy chunk dnia.** Rano są dwie przewagi: (a) nikt jeszcze niczego od Ciebie nie chce — brak meetingów, brak pilnych wiadomości; (b) decyzyjność jest najwyższa, zanim dzień ją zużyje. Blok "po pracy" przegrywa z dziennym zmęczeniem niemal zawsze.
2. **Telefon poza zasięgiem ręki**, komunikatory zamknięte, powiadomienia systemowe wyciszone. Nie "będę silny" — tylko fizyczne usunięcie pokusy. To różnica między walidacją a uniemożliwieniem błędnego wejścia: projektuj środowisko, nie testuj siły woli.
3. **Zdefiniowane wejście do bloku.** Blok zaczyna się od otwarcia konkretnej lekcji w repo, nie od "zastanowię się, co dziś robić". Decyzję "co robię jutro rano" podejmuj poprzedniego dnia (ostatnie 2 minuty bloku: zapisz, od czego zaczniesz). To eliminuje najdroższy moment — rozruch z pustą kartką.
4. **Zdefiniowane wyjście.** Blok kończy się commitem do repo nauki (lekcja 02) — nawet jeśli to tylko notatki. Commit to Twój "acknowledgement" — potwierdzenie, że wiadomość została przetworzona.

### Jak zaprojektować tydzień

Rytm z planu rozpisany na elementy:

| Aktywność | Częstotliwość | Czas | Charakter |
|---|---|---|---|
| Głęboka nauka/praktyka | 4×/tydz. | 2h, rano | deep work — nietykalne |
| Post na LinkedIn | 1×/tydz. | 60–90 min | praca twórcza, też wymaga skupienia |
| Komentowanie na LinkedIn | 3–4×/tydz. | 15 min | shallow — może być w dowolnej szczelinie dnia |
| Przegląd tygodnia | 1×/tydz. | 20–30 min | meta-praca, najlepiej stały termin (np. piątek po bloku lub niedziela wieczór) |

Zasady projektowe:

- **Bloki deep work przypisz do konkretnych dni i godzin.** "4 razy w tygodniu" bez dat to nie plan, to życzenie. Przykład: pn/wt/czw/pt 6:30–8:30. Środa wolna — celowo, jako bufor (patrz niżej).
- **Post pisz w stałym slocie**, np. sobota rano. Temat posta wychodzi z tego, czego uczyłeś się w tygodniu — nie wymyślaj z powietrza. Pod koniec każdego bloku nauki zapisuj jednym zdaniem "kandydata na post".
- **Komentowanie to celowo shallow work** — wrzucaj je w naturalne przerwy (kawa, kolejka), nigdy w blok deep work.
- **Przegląd tygodnia musi mieć stały termin**, bo to jedyny mechanizm samonaprawy całego systemu. Bez niego rytm degraduje się po cichu.

W DayChunks odwzoruj to jako szablon tygodnia: bloki nauki jako pierwszy chunk dnia, post i przegląd jako osobne, nazwane chunki.

### Cotygodniowy przegląd — format retro

**Retro** (retrospektywa) to znana Ci z pracy zespołowej praktyka: regularne spotkanie, na którym zespół ocenia swój proces i go koryguje. Tu robisz retro jednoosobowe. Format minimalny — trzy pytania, 20–30 minut, wynik zapisany (w `progress.md`, lekcja 02):

1. **Co działa?** — które elementy rytmu wydarzyły się zgodnie z planem. Nazwij je wprost; to chroni przed wrażeniem "nic nie działa", gdy zawiódł jeden element z pięciu.
2. **Co nie zadziałało i dlaczego?** — bez samobiczowania, jak przy post-mortem po incydencie: szukasz przyczyny systemowej, nie winnego. "Nie wstałem w czwartek" → przyczyna: położyłem się o 1:00 → korekta dotyczy wieczora środy, nie poranka czwartku.
3. **Co wycinam lub zmieniam w przyszłym tygodniu?** — maksymalnie 1–2 korekty naraz. Zmiana wielu parametrów jednocześnie uniemożliwia ocenę, która zadziałała — dokładnie jak przy debugowaniu.

Dodatkowo raz na przegląd odpowiedz na pytanie kontrolne: **czy w tym tygodniu powstał artefakt?** (commit, notatka, diagram, post). Tydzień bez artefaktu to sygnał ostrzegawczy, nawet jeśli "czytałeś dużo".

### Dlaczego plany nauki umierają w 3. tygodniu — i przeciwdziałanie

Typowe mechanizmy porażki i ich mitygacje:

1. **Plan zaprojektowany na wersję siebie z dnia największej motywacji.** 6×/tydz. po 3h działa tydzień. Mitygacja: planuj na swój przeciętny tydzień, nie najlepszy. 4× 2h z wolną środą to plan z marginesem.
2. **Zasada "wszystko albo nic".** Jedno zerwane rano → "tydzień stracony" → "plan stracony". Mitygacja: **budżet błędów** — jak error budget w SRE (umowna ilość awarii, która nie uruchamia alarmu): 3 z 4 bloków w tygodniu = tydzień zaliczony. Dopiero drugi tydzień z rzędu poniżej budżetu jest incydentem do analizy na przeglądzie.
3. **Brak mechanizmu recovery.** Plany nie umierają od opuszczenia bloku — umierają od braku procedury powrotu. Mitygacja: ustal regułę z góry: *zerwany blok się nie odrabia* (doklejanie zaległości lawinowo przeciąża kolejne dni — jak retry storm, gdy zaległe komunikaty zapychają kolejkę). Wracasz po prostu do najbliższego zaplanowanego bloku.
4. **Nauka bez widocznego postępu.** Po 3 tygodniach czytania "nic nie widać" i sens znika. Mitygacja: artefakty (zasada całego kursu) + `progress.md` + historia commitów. Postęp ma być mierzalny i oglądalny.
5. **Cichy rozrost zakresu (scope creep).** "Przy okazji nauczę się jeszcze Rusta i Kubernetesa". Mitygacja: przegląd tygodnia ma pytanie "co wycinam" — nie "co dokładam".

### Kiedy tego NIE stosować (trade-offy)

- **Sztywny poranny rytm nie jest dogmatem.** Jeśli po 2–3 tygodniach dane z przeglądów pokazują, że poranki systematycznie nie działają w Twoim życiu rodzinnym — zmień porę. Zmieniaj na podstawie danych z retro, nie chwilowego nastroju; i zmieniaj porę, nie zasadę "stały blok".
- **Deep work nie jest potrzebny do wszystkiego.** Komentowanie na LinkedIn czy aktualizacja `progress.md` to shallow work — pilnowanie "pełnego skupienia" przy nich to przerost formy. Chroń skupienie tam, gdzie jest drogie: nauka i pisanie.
- **Tydzień adaptacji jest częścią planu.** Pierwszy tydzień rytmu będzie nierówny — to dane kalibracyjne, nie porażka.

## Praktyka

- [x] Zaprojektuj swój tydzień na papierze/w notatce: przypisz 4 bloki deep work do konkretnych dni i godzin, wyznacz slot na post, slot na przegląd i dzień buforowy.
- [ ] Utwórz w DayChunks szablon tygodnia z tymi blokami: blok nauki jako pierwszy chunk dnia (4×), chunk "post LinkedIn" (1×), chunk "przegląd tygodnia" (1×), chunki "komentowanie 15 min" (3–4×).
- [x] Zdefiniuj swój budżet błędów i regułę recovery (np. "3/4 bloków = tydzień OK; zerwanego bloku nie odrabiam") — zapisz je, trafią potem do dziennika decyzji (lekcja 03).
- [x] Przeprowadź pierwszy blok 2h według zasad ochrony (rano, telefon poza zasięgiem, wejście od konkretnej lekcji, wyjście commitem) — materiałem może być lekcja 02 tego modułu.
- [x] Na koniec tygodnia przeprowadź pierwszy przegląd według formatu trzech pytań i zapisz wynik.

## Artefakt

1. **Szablon bloków nauki w DayChunks** — powtarzalny tydzień gotowy do klonowania na kolejne tygodnie.
2. **Zapisany format przeglądu tygodnia** (trzy pytania + pytanie o artefakt) — jako notatka w repo nauki, do użycia co tydzień.

## Definition of Done

- [ ] Szablon tygodnia istnieje w DayChunks i ma wszystkie elementy rytmu (4× nauka, 1× post, 3–4× komentowanie, 1× przegląd).
- [ ] Każdy blok deep work ma przypisany konkretny dzień i godzinę — nie "4 razy w tygodniu".
- [x] Masz zapisany budżet błędów i regułę powrotu po zerwanym bloku.
- [x] Co najmniej jeden blok 2h odbył się naprawdę i zakończył commitem.
- [x] Pierwszy przegląd tygodnia odbył się i jego wynik jest zapisany.

## Materiały

1. Cal Newport, *Deep Work* (2016) — źródło pojęcia i praktyk ochrony skupienia; wystarczy część I (argument) i wybrane reguły z części II.
2. James Clear, *Atomic Habits* (2018) — mechanika nawyków: dlaczego systemy biją cele i jak projektować środowisko zamiast polegać na silnej woli.
