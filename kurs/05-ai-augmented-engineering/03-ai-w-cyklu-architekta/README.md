# Lekcja 03 — AI w cyklu pracy architekta: ADR-y, diagramy, trade-offy, legacy

## Cel lekcji

Po tej lekcji będziesz używał agenta nie tylko do pisania kodu, ale w pracy stricte architektonicznej: generowanie i krytyka ADR-ów, diagramy jako kod, analiza trade-offów z modelem jako "sparing partnerem" oraz przegląd istniejących systemów — z jasnymi granicami tego, czego do modelu wkleić nie wolno, i z technikami wykrywania, że wygenerowana analiza to gładki nonsens.

## Dlaczego to ważne

Kod to mniejszość pracy architekta. Większość to dokumenty decyzyjne, diagramy, analiza opcji i czytanie cudzych (często starych) systemów — i właśnie te czynności LLM-y wspierają zaskakująco dobrze, bo są czystą pracą na tekście i strukturze. Senior, który używa AI tylko do kodu, używa może jednej trzeciej narzędzia.

Jest jednak haczyk, dla którego ta lekcja istnieje: praca architektoniczna to dokładnie ten obszar, gdzie błędy AI są **najtrudniejsze do wykrycia**. Zły kod nie przejdzie testów; zły ADR przeczyta się świetnie i będzie kosztował zespół rok. Na rozmowie rekrutacyjnej umiejętność powiedzenia "używam AI do generowania opcji i kontrargumentów, ale decyzję i jej uzasadnienie piszę sam — i oto dlaczego" to sygnał dojrzałości, którego nie da się podrobić ogólnikami.

## Teoria

### Zasada porządkująca: AI generuje opcje, człowiek podejmuje decyzje

ADR (architecture decision record — krótki dokument: kontekst, decyzja, konsekwencje; znasz z modułu 1) ma wartość tylko wtedy, gdy zapisuje **faktyczny proces decyzyjny**. Naiwne użycie AI: "napisz mi ADR o wyborze Service Bus" → dostajesz dokument poprawny formalnie i pusty decyzyjnie, bo model nie zna Twojego kontekstu, budżetu, zespołu ani blizn. Taki ADR jest gorszy niż brak ADR-a — tworzy iluzję przemyślenia.

Właściwy workflow odwraca role — model pracuje tam, gdzie jest mocny (szerokość, generowanie wariantów), Ty tam, gdzie jest słaby (kontekst, wartościowanie, odpowiedzialność):

1. **Ty: kontekst.** Wklej do sesji wymagania, ograniczenia, skalę, istniejące decyzje (zanonimizowane — patrz sekcja o granicach). Im konkretniej, tym mniej generyczna odpowiedź.
2. **AI: opcje.** "Podaj 3–4 realne opcje rozwiązania X. Dla każdej: jak działa, koszty wprowadzenia i operowania, ryzyka, w jakim kontekście wygrywa." Model jest tu świetnym generatorem szerokości — przypomni opcję, o której nie pomyślałeś.
3. **AI: trade-offy i pytania.** "Jakie pytania powinienem zadać, zanim wybiorę? Czego brakuje w moim kontekście, żeby decyzja była odpowiedzialna?" — często cenniejsze niż same opcje.
4. **Ty: decyzja.** Wybierasz i — kluczowe — **własnymi słowami** piszesz sekcję "Decyzja" i "Uzasadnienie". Jeśli nie umiesz jej napisać bez modelu, nie podjąłeś decyzji, tylko ją zaimportowałeś.
5. **AI: krytyka draftu.** Gotowy ADR oddajesz modelowi do ataku: "znajdź słabe punkty tego uzasadnienia, ukryte założenia i scenariusze, w których ta decyzja okaże się błędna".

Zauważ symetrię z lekcją 01: tam spec → kod → review; tu kontekst → opcje → decyzja → krytyka. W obu przypadkach człowiek stoi na początku (intencja) i końcu (odpowiedzialność), agent w środku (objętość).

### AI jako sparing partner: każ modelowi argumentować przeciw sobie

LLM-y mają znaną skłonność do **przytakiwania** (sycophancy — model dostraja odpowiedź do widocznych preferencji rozmówcy, bo tak był trenowany na ludzkich ocenach). Spytasz "myślę o CQRS, dobry pomysł?" — dostaniesz entuzjazm i listę zalet. To czyni model bezwartościowym jako doradcę… chyba że jawnie odwrócisz mu rolę.

Techniki sparingu (działają, bo zmieniają zadanie z "oceń" na "wygeneruj określony typ treści"):

- **Atak na własną rekomendację:** "Zarekomendowałeś opcję B. Teraz przygotuj najmocniejszą możliwą argumentację, że B to błąd i powinienem wybrać A. Nie łagodź."
- **Steelman** (najmocniejsza uczciwa wersja cudzego stanowiska — przeciwieństwo strawmana): "Przedstaw najlepszą obronę monolitu w tym kontekście, jakby bronił go principal engineer z 20-letnim stażem."
- **Pre-mortem:** "Jest rok 2028, ta architektura okazała się porażką. Napisz post-mortem: co poszło nie tak i jakie sygnały zignorowaliśmy w 2026?"
- **Rada nadzorcza:** "Oceń ten projekt z trzech perspektyw: SRE dbającego o operowalność, CFO patrzącego na koszty, nowego developera, który ma to utrzymywać."

Ukryta korzyść: te sesje produkują gotową treść sekcji "Rozważane alternatywy" i "Konsekwencje" ADR-a — zwykle najsłabszych w ludzkich ADR-ach, bo autor po podjęciu decyzji nie ma już motywacji uczciwie opisywać odrzuconych opcji.

### Diagramy jako kod: generowane, wersjonowane, weryfikowane

**Diagrams as code** to zapisywanie diagramów w tekstowym DSL-u (np. **Mermaid** — renderowany natywnie przez GitHub w markdownie, albo **Structurizr DSL** dla modelu C4 z modułu 1) zamiast klikania w narzędziu graficznym. Dla pracy z AI to zmiana fundamentalna: tekst jest natywnym formatem modelu, więc agent może diagram **wygenerować z opisu lub wprost z kodu repo**, a Ty możesz go zdiffować w PR jak każdy plik.

Praktyczny przepływ: "przeczytaj solution i wygeneruj diagram C4 poziomu Container w Mermaid: kontenery, protokoły komunikacji, zewnętrzne zależności" → render → **weryfikacja**. I tu zasada, która odróżnia użycie profesjonalne od zabawy: diagram wygenerowany z kodu weryfikujesz **przeciw rzeczywistości** (czy każda strzałka odpowiada realnemu wywołaniu? czy czegoś nie brakuje — np. zależności ukrytej w konfiguracji?), bo model potrafi dorysować przepływ, który "powinien" istnieć w typowym systemie tego rodzaju, a w Twoim nie istnieje. Halucynacja na diagramie jest groźniejsza niż w kodzie — nie ma kompilatora, który ją złapie, a diagram C4 w dokumentacji ludzie traktują jak prawdę objawioną.

Trade-off względem ręcznego rysowania: tracisz pełną kontrolę nad layoutem (Mermaid rozmieszcza automatycznie, bywa brzydko przy >10 elementach), zyskujesz wersjonowanie, diff i odtwarzalność. Dla diagramów architektonicznych to wymiana niemal zawsze opłacalna; dla slajdu na zarząd — niekoniecznie.

### Przegląd legacy i cudzego kodu z agentem

Scenariusz z Twojej przyszłości jako architekta-konsultanta: dostajesz nieznany system i masz powiedzieć, co z nim jest nie tak. Agent z dostępem do repo skraca fazę rekonesansu z dni do godzin — pod warunkiem zadawania pytań **architektonicznych**, nie ogólnych:

- "Zmapuj granice modułów i wszystkie zależności przekraczające te granice. Gdzie zależności idą w złym kierunku (infrastruktura → domena)?"
- "Znajdź miejsca, gdzie ten sam koncept biznesowy jest zamodelowany wielokrotnie i niespójnie."
- "Gdzie są operacje 'zapisz + opublikuj/wyślij' bez transakcji (kandydaci na dual write)? Gdzie brakuje timeoutów na I/O?"
- "Które fragmenty wyglądają na pisane w innym stylu/epoce niż reszta — i co to sugeruje o historii systemu?"

Reguła weryfikacji: każde **twierdzenie** agenta o kodzie traktuj jako hipotezę z obowiązkiem wskazania dowodu — "podaj plik i linie, z których to wynika". Agent czytający duże repo skanuje wybiórczo (context window — nie widzi wszystkiego naraz) i potrafi generalizować z trzech plików na całość. Hipoteza + wskazany dowód + Twoje 2 minuty na sprawdzenie = solidny wniosek. Hipoteza bez dowodu = szum.

### Granice: czego nie wolno wkleić do modelu

Model hostowany to **zewnętrzny procesor danych**. Zanim cokolwiek wkleisz, obowiązuje Cię to samo myślenie, co przy każdej integracji z third-party:

- **Nigdy:** dane osobowe klientów, sekrety (klucze, connection stringi, certyfikaty), kod i dokumenty objęte NDA lub własnością klienta bez jego zgody, dane finansowe/medyczne. To nie jest kwestia zaufania do dostawcy, tylko zgodności (RODO, umowy) i higieny: sekret wklejony do czatu jest skompromitowany z definicji — jak sekret zacommitowany do repo.
- **Sprawdź zanim założysz:** plan/umowa ma znaczenie. Plany konsumenckie często domyślnie używają danych do treningu; plany enterprise i API zwykle nie — ale to musisz **zweryfikować w aktualnych warunkach swojego dostawcy**, nie założyć. W firmie: czy istnieje polityka AI? Jeśli nie ma — to jest temat, który architekt podnosi, a nie obchodzi.
- **Technika anonimizacji:** problem architektoniczny prawie zawsze da się przedstawić bez danych wrażliwych — zmień nazwy domenowe na neutralne, liczby na rzędy wielkości, usuń identyfikatory. "System rezerwacji o ruchu X rps z integracją płatności" wystarcza modelowi w 95% przypadków równie dobrze jak prawdziwe szczegóły.

### Jak rozpoznać gładki nonsens

Analiza wygenerowana przez LLM ma zawsze tę samą fakturę: pewny ton, ładna struktura, poprawny żargon — **niezależnie od tego, czy jest trafna**. Forma przestaje być sygnałem jakości (u ludzi bełkot zwykle *wygląda* na bełkot; tu nie). Testy, które warto zrobić nawykiem:

1. **Test podstawienia:** czy ta analiza pasowałaby do dowolnego systemu tej klasy? Jeśli po zamianie nazw na inny projekt nadal "pasuje" — to horoskop, nie analiza.
2. **Test konkretu:** czy każde twierdzenie wskazuje dowód (plik, liczba, mechanizm)? "Może powodować problemy ze skalowalnością" bez mechanizmu *jak* — do kosza albo do drążenia.
3. **Test odwrócenia:** poproś o równie mocną argumentację za tezą przeciwną. Jeśli przychodzi równie gładko i przekonująco — model nie ma podstaw, tylko płynność; decyzyjnie obie wersje są bezwartościowe.
4. **Test wiedzy świeżej:** twierdzenia o wersjach, limitach usług, cennikach — model może cytować stan sprzed lat (knowledge cutoff — data graniczna wiedzy treningowej). Wszystko, co zależy od "dziś", weryfikuj w dokumentacji.

### Kiedy tego NIE robić (trade-offy)

- **Decyzje, których nie umiesz ocenić.** Sparing partner działa, gdy potrafisz rozsądzić, kto wygrał rundę. W dziedzinie, w której jesteś zielony, model Cię nie nauczy odpowiedzialnej decyzji — najpierw fundament (moduły 1–3), potem sparing.
- **ADR-y "taśmowo".** Wygenerowanie 15 ADR-ów w wieczór, żeby "mieć dokumentację", produkuje szum, który zabija zaufanie do całego rejestru decyzji. ADR ma być śladem myślenia, nie wypełniaczem.
- **Gdy kontekst jest tajny i nie da się zanonimizować** — wtedy model publiczny odpada w całości; opcje on-premise/private endpoint to temat architektoniczny sam w sobie (wróci przy lekcji 04).

## Praktyka

Ćwiczenia celowo pracują na artefaktach z modułów 1–3 — nie wymyślaj nowych przykładów.

- [ ] Weź jedną realną decyzję z modułu 2 lub 3 (np. wybór brokera, orchestration vs choreography w sadze, SQL vs Cosmos) i przeprowadź pełny workflow ADR: Twój kontekst → opcje od AI → sparing (atak na rekomendację + pre-mortem) → **Twoja** decyzja → krytyka draftu przez AI. Zapisz gotowy ADR obok pozostałych.
- [ ] Każ agentowi wygenerować diagram C4 (poziom Container) repo messaging-patterns w Mermaid wprost z kodu. Zweryfikuj każdą strzałkę przeciw kodowi; błędy/braki zanotuj w `ai-log.md` z klasą błędu.
- [ ] Przeprowadź przegląd legacy: weź najstarszy własny kod, jaki masz pod ręką (albo starszy moduł DayChunks), i zadaj agentowi 4 pytania architektoniczne z tej lekcji, egzekwując "wskaż plik i linie". Oceń: ile hipotez się obroniło?
- [ ] Wykonaj test odwrócenia na dowolnej rekomendacji AI z tego tygodnia i opisz wynik jednym akapitem — kandydat na post.
- [ ] Spisz w `ai-workflow.md` sekcję "AI w pracy architektonicznej": workflow ADR, zasady diagramów, granice danych (3–5 twardych reguł), testy na nonsens.

## Artefakt

1. **ADR wytworzony workflow z tej lekcji** — w repo, obok ADR-ów z modułów 1–3, z dopiskiem w sekcji kontekstu, że opcje generowano z AI.
2. **Diagram C4/Mermaid repo messagingowego** — zweryfikowany, zacommitowany (nadaje się do README repo → Featured).
3. **Sekcja "AI w pracy architektonicznej" w `ai-workflow.md`** — w tym Twoje twarde reguły danych.

## Definition of Done

- [ ] ADR istnieje i sekcja "Decyzja/Uzasadnienie" jest napisana Twoimi słowami — umiesz jej bronić bez zaglądania do sesji z modelem.
- [ ] Sekcja "Rozważane alternatywy" tego ADR-a jest mocniejsza niż w Twoich ADR-ach z modułu 1 (porównaj — to mierzalny efekt sparingu).
- [ ] Diagram po weryfikacji wisi w repo; wiesz, ile i jakich błędów model popełnił przy generowaniu.
- [ ] Masz spisane granice danych i stosujesz je odruchowo (test: ostatnie 5 sesji z agentem — zero wklejonych sekretów/danych klienckich).
- [ ] Umiesz opowiedzieć na rozmowie technikę sparing partnera z konkretnym przykładem, w którym zmieniła Twoją decyzję albo wzmocniła uzasadnienie.

## Materiały

1. [Architecture Decision Records — adr.github.io](https://adr.github.io/) — kanoniczne źródło formatów i praktyk ADR; odśwież przed ćwiczeniem workflow.
2. [Dokumentacja Mermaid](https://mermaid.js.org/) — składnia diagramów (flowchart, sequence, C4 — sprawdź bieżący stan wsparcia C4); wystarczy przejrzeć typy diagramów, resztę wygeneruje agent, a Ty masz rozumieć, co czytasz.
