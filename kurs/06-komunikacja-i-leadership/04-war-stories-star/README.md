# Lekcja 04 — War stories w formacie STAR: 20 lat doświadczenia w 3 historiach

## Cel lekcji

Po tej lekcji będziesz miał trzy dopracowane "war stories" (historie z pola walki — opowieści o realnych, trudnych sytuacjach zawodowych) w formacie STAR, każdą w wersji 2-minutowej i 5-minutowej, z mierzalnymi rezultatami — gotowe do opowiedzenia na rozmowie rekrutacyjnej bez kartki i bez zacinania się.

## Dlaczego to ważne

Obok system design interview (lekcja 03) drugim filarem rekrutacji na role Senior/Lead/Architect jest **behavioral interview** (rozmowa behawioralna): pytania typu *"opowiedz o sytuacji, gdy musiałeś podjąć trudną decyzję techniczną pod presją"*, *"opowiedz o największej awarii, w której brałeś udział"*, *"jak przekonałeś zespół do swojego rozwiązania?"*. Założenie tego formatu: przeszłe zachowanie najlepiej przewiduje przyszłe — dlatego rekruter nie pyta "jak *byś* postąpił", tylko "jak *postąpiłeś*".

Twój paradoks jest odwrotny niż u większości kandydatów: nie brakuje Ci materiału — masz go za dużo. 20 lat Perfect Gym, Aerotunelu, integracji i nocnych awarii to setki potencjalnych historii. Nieprzygotowany kandydat z takim bagażem na pytanie o awarię zaczyna opowiadać trzy historie naraz, gubi wątek, pomija rezultat i kończy po 10 minutach w połowie zdania. Przygotowanie polega na **selekcji i szlifie**: 3 historie dopracowane do połysku pokrywają 80% pytań behawioralnych, bo jedną dobrą historię można obracać pod różne kompetencje. To także główny artefakt całego modułu 6.

## Teoria

### Format STAR od zera

**STAR** to akronim struktury opowiadania o doświadczeniu: **S**ituation, **T**ask, **A**ction, **R**esult. To nie korporacyjny wymysł, tylko rozwiązanie realnego problemu komunikacyjnego — nieustrukturyzowane opowieści o pracy zwykle są albo samym kontekstem bez puenty, albo samą puentą bez kontekstu.

1. **Situation (sytuacja)** — scena i stawka w 2–3 zdaniach: gdzie, kiedy, co się działo, dlaczego to było ważne. *"Perfect Gym, system obsługujący kilkaset klubów fitness; byłem ownerem systemu. W szczycie sezonu zaczęły do nas spływać zgłoszenia o podwójnych obciążeniach klientów."*
2. **Task (zadanie)** — co konkretnie było **Twoją** odpowiedzialnością w tej sytuacji, w 1–2 zdaniach: *"Moim zadaniem było znaleźć przyczynę i zaproponować rozwiązanie, które nie zatrzyma sprzedaży w sezonie."* Task odróżnia "byłem przy tym" od "to było moje".
3. **Action (działanie)** — co zrobiłeś, krok po kroku, 60–70% całej historii. Kluczowa zasada: **"ja", nie "my"**. Rekruter ocenia Ciebie; "zrobiliśmy analizę" nie mówi nic o Twoim wkładzie. Mów "przeanalizowałem logi i znalazłem…", "zaproponowałem zespołowi…", "przekonałem biznes, że…" — a rolę zespołu oddaj uczciwie tam, gdzie była ("dwóch dev-ów z mojego zespołu zaimplementowało X, ja w tym czasie…").
4. **Result (rezultat)** — co się zmieniło, najlepiej z liczbą: *"Podwójne obciążenia spadły do zera; reklamacje tego typu zniknęły z supportu — wcześniej ~20 miesięcznie. Wzorzec wdrożyliśmy potem w dwóch kolejnych integracjach."* Plus — na poziom architekta niemal obowiązkowo — **wniosek/lekcja**: czego Cię to nauczyło, co dziś zrobiłbyś inaczej. Czasem mówi się o formacie STAR(L) — L jak learning.

Dlaczego rekruterzy oczekują tego formatu: po pierwsze, daje porównywalność między kandydatami (każdy oceniany po tym samym szkielecie); po drugie, jest odporny na ściemę — w historii ze strukturą S→T→A→R łatwo dopytać o każdy element ("a co dokładnie *Ty* zrobiłeś?", "skąd ta liczba?"), a zmyślona historia sypie się przy drugim pytaniu pogłębiającym. Prawdziwa historia, dobrze ustrukturyzowana, przy dopytywaniu tylko zyskuje.

Typowe proporcje czasowe: S 15%, T 10%, A 60%, R 15%. Najczęstsza wada surowych historii: 70% sytuacji ("a jeszcze musisz wiedzieć, że ten system…"), 10% działania, zero rezultatu.

### Jakich historii potrzebuje kandydat na architekta

Pytania behawioralne na role architektoniczne krążą wokół kilku kompetencji. Dobierz historie tak, żeby je pokryć:

| Kompetencja | Typowe pytanie | Co musi być w historii |
|---|---|---|
| **Decyzja pod presją** | "Opowiedz o trudnej decyzji technicznej przy niepełnych danych / braku czasu" | realne ograniczenia (czas, budżet), rozważone opcje, kryterium wyboru, świadome ryzyko |
| **Trade-off kosztowy** | "Opowiedz, jak pogodziłeś jakość techniczną z wymaganiami biznesu" | jawnie nazwany koszt obu opcji, rozmowa z biznesem (lekcja 01!), decyzja "wystarczająco dobra zamiast idealnej" |
| **Awaria produkcyjna i wnioski** | "Opowiedz o największej awarii — co się stało i czego się nauczyłeś" | przebieg diagnozy, Twoje działanie w stresie, root cause, **zmiana systemowa po** (nie tylko "naprawiliśmy") |
| **Wpływ bez formalnej władzy** | "Jak przekonałeś zespół/innych do rozwiązania, gdy nie mogłeś go narzucić?" | argumentacja zamiast eskalacji, wysłuchanie kontrargumentów, czyjaś zmiana zdania (może Twoja!) |

Jedna dobra historia często pokrywa 2–3 kompetencje naraz — historia o nocnej awarii z poison message (komunikat, którego konsument nie umie przetworzyć i który blokuje kolejkę) to jednocześnie "awaria i wnioski" oraz "decyzja pod presją". Dlatego 3 dopracowane historie + świadomość, którą kompetencję którą historią pokrywasz, wystarcza na całą rozmowę behawioralną.

### Jak zamienić 20 lat w 3 historie: kryteria wyboru

Zrób najpierw **inwentarz**: wypisz 8–10 kandydatek jednym zdaniem każda (awarie, migracje, trudne integracje, spory architektoniczne, decyzje, których skutki widziałeś latami). Potem przefiltruj kryteriami:

1. **Byłeś sprawcą, nie świadkiem.** Musi istnieć wyraźne "ja zrobiłem/zdecydowałem/przekonałem". Historia, w której głównie obserwowałeś, odpada — choćby była spektakularna.
2. **Jest konflikt lub trudność.** "Zaprojektowałem system i działał" to nie historia. Historia wymaga przeszkody: awarii, presji, sprzeciwu, niewiadomej.
3. **Rezultat da się zmierzyć lub konkretnie opisać.** Jeśli nie pamiętasz żadnej liczby ani namacalnego skutku — historia będzie słaba przy dopytywaniu.
4. **Jest lekcja na poziomie architektury**, nie tylko "naprawiłem buga": zmiana wzorca, procesu, sposobu podejmowania decyzji.
5. **Umiesz ją opowiedzieć bez łamania NDA** i bez obsmarowywania byłych pracodawców (rekruterzy to wychwytują i to czerwona flaga — o byłych firmach mów rzeczowo, krytykuj systemy i procesy, nie ludzi).

Twoje naturalne kandydatki (potraktuj jako punkt startowy inwentarza): historia z **Perfect Gym** (wieloletnie ownership dużego systemu — dobra pod "trade-off kosztowy" lub "wpływ bez władzy"), historia z **Aerotunelu** (system rezerwacji, integracje — dobra pod "decyzja pod presją"), historia o **poison message / nocnej awarii** (dobra pod "awaria i wnioski"). Razem pokrywają wszystkie cztery kompetencje z tabeli.

### Szlifowanie: liczby, rezultaty, dwie długości

**Liczby.** Surowa historia mówi "było dużo błędów, po zmianie było lepiej". Szlif polega na odtworzeniu rzędów wielkości: ile zgłoszeń, ile godzin przestoju, ilu klientów dotkniętych, o ile spadło X po zmianie. Po latach nie pamiętasz dokładnie? **Szacuj uczciwie i mów, że szacujesz**: "z pamięci: rząd wielkości 20–30 zgłoszeń miesięcznie, po wdrożeniu — pojedyncze w kwartał". Zaokrąglony szacunek podany pewnie jest w porządku; pseudoprecyzja ("siedemnaście zgłoszeń") wymyślona na poczekaniu — nie, bo posypie się przy dopytaniu. Rezultaty nie zawsze są liczbowe — "wzorzec stał się standardem w kolejnych integracjach" albo "biznes zaczął nas zapraszać do decyzji wcześniej" to też mocne rezultaty.

**Dwie długości.** Każdą historię przygotuj w dwóch wersjach:

- **2-minutowa** — domyślna odpowiedź na pytanie behawioralne: S i T po 2–3 zdania, A jako 3–4 najważniejsze kroki, R z jedną liczbą i jedną lekcją. Kończysz i **milkniesz** — dopytanie to dobry znak, zaproszenie do zejścia głębiej, a nie sygnał, że powiedziałeś za mało.
- **5-minutowa** — na prośbę "opowiedz więcej" albo na pytania otwarte typu "opowiedz o najciekawszym projekcie": więcej kontekstu, rozważone alternatywy, jak wyglądała komunikacja z ludźmi, pełniejsza lekcja.

Wersję 2-minutową buduj przez **cięcie wersji 5-minutowej**, nie przez osobne pisanie — wtedy obie opowiadają to samo i przejście między nimi w trakcie rozmowy jest naturalne. I nie ucz się tekstu na pamięć słowo w słowo (recytacja brzmi jak recytacja i pęka przy pierwszym wtrąceniu) — zapamiętaj **szkielet**: 5–7 punktów-haseł na historię.

### Przykład szlifu: przed i po

Surowa odpowiedź (tak brzmi nieprzygotowany weteran — wszystko prawda, zero struktury):

> "Mieliśmy kiedyś taki problem z kolejką, że w nocy coś się zablokowało. To był taki system, gdzie były integracje z różnymi partnerami, i tam była kolejka, i w sumie to długo nie wiedzieliśmy, o co chodzi. No i w końcu się okazało, że jedna wiadomość była uszkodzona i wszystko stało. Naprawiliśmy to i potem już działało. No i od tamtej pory wiedzieliśmy, że trzeba uważać na takie rzeczy."

Ta sama historia po szlifie STAR (wersja 2-minutowa, szkielet):

> **S:** "System rezerwacji z integracjami do partnerów zewnętrznych, komunikacja przez kolejkę. Ok. 2 w nocy przestały przechodzić wszystkie rezerwacje — pełna blokada przepływu w środku sezonu." **T:** "Byłem osobą odpowiedzialną za tę integrację — diagnoza i przywrócenie ruchu były na mnie." **A:** "Zacząłem od metryk kolejki: rosła, konsument żył, ale kręcił się w kółko na jednym komunikacie. Zidentyfikowałem go — uszkodzony payload od partnera, którego nasz parser nie obsługiwał, a polityka retry bez limitu w nieskończoność ponawiała ten sam komunikat, blokując resztę. Doraźnie: wyciąłem komunikat ręcznie i ruch ruszył po ~40 minutach przestoju. Systemowo: w kolejnym tygodniu zaproponowałem i wdrożyłem limit ponowień z dead-letter queue, czyli osobną kolejką na komunikaty nie do przetworzenia, plus alert na pierwszy komunikat w DLQ." **R:** "Ten typ blokady już nigdy się nie powtórzył — pojedyncze zatrute komunikaty lądowały w DLQ i były obsługiwane rano, bez nocnych telefonów. Lekcja, którą stosuję do dziś: retry bez limitu to nie odporność, tylko odroczona awaria — i każdy konsument dostaje u mnie DLQ od pierwszego dnia."

Różnica nie polega na ubarwieniu — fakty są te same. Różnica to: stawka w pierwszym zdaniu, wyraźne "ja", przebieg diagnozy zamiast "w końcu się okazało", rozdzielenie fix doraźny / zmiana systemowa, mierzalny rezultat i lekcja, która pokazuje poziom architekta, nie operatora.

**Nagranie i odsłuch.** Najtańsze i najskuteczniejsze narzędzie szlifu: nagraj się telefonem, opowiadając historię, i odsłuchaj. Będzie nieprzyjemnie — u wszystkich jest. Słuchaj jak interviewer i odhaczaj: Czy w pierwszych 30 sekundach wiadomo, o czym jest historia i jaka jest stawka? Czy słychać "ja", czy samo "my"? Czy rezultat padł, z liczbą? Ile razy "yyy", dygresje, cofanie się ("a, jeszcze zapomniałem powiedzieć, że…")? Czy zmieściłeś się w czasie? Drugie nagranie po poprawkach jest zwykle skokowo lepsze.

### Kiedy STAR przeszkadza (trade-offy)

- **STAR to struktura odpowiedzi na pytanie behawioralne, nie sposób mówienia w ogóle.** Odpowiadanie STAR-em na "jakiej bazy użyliście?" brzmi absurdalnie. Format włączasz, gdy pytanie zaczyna się od "opowiedz o sytuacji, gdy…".
- **Recytowany STAR brzmi jak robot.** Sygnalizowanie sekcji ("a teraz Action…") zabija naturalność — struktura ma być szkieletem pod skórą, niewidocznym z zewnątrz.
- **Nie naginaj historii do formatu kosztem prawdy.** Jeśli rezultat był częściowy albo lekcja wzięła się z porażki — mów tak, jak było. Historia "podjąłem złą decyzję, oto co mnie kosztowała i co od tej pory robię inaczej" bywa najsilniejszą kartą seniora; wymaga tylko domkniętej, przemyślanej lekcji.

## Praktyka

- [ ] Zrób inwentarz: wypisz 8–10 historii-kandydatek z całej kariery, po jednym zdaniu. Zacznij od trzech naturalnych: Perfect Gym, Aerotunel, poison message / nocna awaria — i dopisz resztę.
- [ ] Przefiltruj inwentarz pięcioma kryteriami z teorii i wybierz **3 historie** tak, żeby razem pokrywały wszystkie 4 kompetencje z tabeli. Zapisz przy każdej, które kompetencje pokrywa.
- [ ] Spisz każdą historię w pełnym STAR (wersja 5-minutowa, ok. 1 strona): z jawnym podziałem na S/T/A/R, z "ja" w Action, z minimum jedną liczbą lub mierzalnym skutkiem w Result i z lekcją na końcu.
- [ ] Brakujące liczby odtwórz: poszukaj w starych mailach/notatkach/pamięci rzędów wielkości; gdzie się nie da — zapisz uczciwy szacunek z zaznaczeniem "szacunek".
- [ ] Z każdej wersji 5-minutowej wytnij wersję 2-minutową (szkielet 5–7 haseł). Sprawdź na głos z timerem, że naprawdę mieści się w ~2 minutach.
- [ ] **Nagraj się**, opowiadając jedną wybraną historię w wersji 5-minutowej (samo audio wystarczy). Odsłuchaj z checklistą z teorii i zapisz 3 poprawki. Nagraj drugi raz po poprawkach.
- [ ] Test krzyżowy: poproś Claude o odegranie rekrutera (prompt: *"Zadaj mi pytanie behawioralne na rolę architekta, wysłuchaj odpowiedzi [wklejam tekst], a potem zadaj 3 pytania pogłębiające, jakie zadałby wnikliwy rekruter, i oceń moją historię: struktura STAR, widoczność mojego wkładu, siła rezultatu"*). Przejdź tak co najmniej jedną historię.

## Artefakt

**Plik `war-stories.md`** w folderze tej lekcji (główny artefakt całego modułu 6): 3 historie, każda z (a) wersją 5-minutową w pełnym STAR, (b) szkieletem wersji 2-minutowej, (c) listą kompetencji, które pokrywa, (d) liczbami/skutkami w Result. Plus **nagranie audio** jednej historii (wersja po poprawkach) i notatka z odsłuchu.

## Definition of Done

- [ ] `war-stories.md` zawiera 3 historie w pełnym formacie STAR; razem pokrywają wszystkie 4 kompetencje (decyzja pod presją, trade-off kosztowy, awaria i wnioski, wpływ bez władzy).
- [ ] Każdy Result zawiera liczbę lub konkretny mierzalny skutek; szacunki są oznaczone jako szacunki.
- [ ] Każda historia ma wersję 2-minutową i 5-minutową, sprawdzone z timerem na głos.
- [ ] Istnieje nagranie jednej historii **po** rundzie poprawek z odsłuchu, plus notatka z 3 wprowadzonymi poprawkami.
- [ ] Co najmniej jedna historia przeszła test pytań pogłębiających (Claude jako rekruter lub żywy człowiek) i nie posypała się przy "a co dokładnie Ty zrobiłeś?" oraz "skąd ta liczba?".
- [ ] Test końcowy: opowiadasz dowolną z trzech historii z pamięci (ze szkieletu, nie z tekstu), w 2 minuty, zaczynając w 5 sekund od usłyszenia pytania.

## Materiały

1. [Amazon — In-person interview: Behavioral questions i metoda STAR](https://www.amazon.jobs/en/landing_pages/in-person-interview) — opis formatu u źródła, czyli w firmie, która zbudowała na nim cały proces rekrutacyjny (Leadership Principles + STAR).
2. Gayle Laakmann McDowell, *Cracking the Coding Interview* — rozdział o behavioral questions: tabela "projekt × kompetencja" do inwentaryzacji historii (reszta książki dotyczy innych etapów — nie musisz jej czytać).
