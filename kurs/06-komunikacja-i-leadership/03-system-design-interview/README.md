# Lekcja 03 — System design interview: format, framework, trening

## Cel lekcji

Po tej lekcji będziesz znał format system design interview od podszewki — czego naprawdę szuka interviewer, jak prowadzić rozmowę krok po kroku w 45 minut — i będziesz miał zaplanowany harmonogram 5–6 mock interviews (rozmów próbnych), w tym z Claude jako interviewerem.

## Dlaczego to ważne

System design interview to standardowy etap rekrutacji na role Senior/Staff/Architect — w big techu obowiązkowy, w polskich firmach produktowych i software house'ach coraz częstszy. Paradoks Twojej sytuacji: masz 20+ lat realnego projektowania systemów (Perfect Gym, Aerotunel, integracje, kolejki), a mimo to możesz oblać ten format — bo on **nie mierzy, czy umiesz projektować systemy, tylko czy umiesz projektować system na głos, na tablicy, w 45 minut, z nieznajomym**. To osobna umiejętność, w pełni trenowalna. Dobra wiadomość: kandydaci z prawdziwymi bliznami produkcyjnymi, którzy opanują format, miażdżą kandydatów znających tylko podręcznik — Twoje "u nas to się wywaliło, bo…" jest nie do podrobienia. Cała wiedza merytoryczna jest już za Tobą (moduły 1–3); ta lekcja dokłada formę.

## Teoria

### Czym jest ten format

**System design interview** to 45–60-minutowa rozmowa, w której dostajesz celowo mgliste zadanie ("zaprojektuj Twittera", "zaprojektuj system powiadomień") i projektujesz rozwiązanie na żywo — na tablicy, w narzędziu typu Excalidraw/Miro (zdalnie) albo we współdzielonym dokumencie — rozmawiając z interviewerem przez cały czas. Nie piszesz kodu. Nie ma jednej poprawnej odpowiedzi — to nie test, to **symulacja współpracy z architektem**: interviewer udaje stakeholdera/kolegę z zespołu i patrzy, jak się z Tobą projektuje.

Czego interviewer naprawdę szuka (w tej kolejności):

1. **Proces myślenia** — czy zbierasz wymagania przed projektowaniem, czy strukturyzujesz problem, czy idziesz od ogółu do szczegółu. Kandydat z gorszym finalnym diagramem, ale czystym procesem, wygrywa z kandydatem, który "zgadł" dobry diagram.
2. **Trade-offy** — czy każdy wybór uzasadniasz i znasz jego koszt ("biorę eventual consistency, bo X; płacę za to Y"). Zdania zaczynające się od "to zależy od…" + konkretne kryterium to muzyka dla uszu interviewera.
3. **Komunikacja** — czy mówisz, co myślisz (thinking out loud), czy zadajesz pytania, czy reagujesz na podpowiedzi. Milczący kandydat jest nieoceniany, czyli oceniony źle.
4. **Głębia w wybranym miejscu** — nie musisz znać wszystkiego; musisz umieć zejść głęboko tam, gdzie interviewer poprosi (albo tam, gdzie sam zaproponujesz — np. w messaging, Twojej najmocniejszej działce).

### Framework krokowy

Naiwne podejście: usłyszeć zadanie i zacząć rysować mikroserwisy. Dlaczego się psuje: projektujesz system na wymagania, których nie znasz — a interviewer celowo dał mgliste zadanie, żeby sprawdzić, czy o nie zapytasz. Właściwy wzorzec to stały framework — kolejność kroków, którą wykonujesz **zawsze**, niezależnie od zadania:

**Krok 1: Clarify requirements (wyjaśnienie wymagań, ~5 min).** Zadajesz pytania zanim cokolwiek narysujesz. Funkcjonalne: *kto jest użytkownikiem? jakie są 3 kluczowe operacje? co jest poza zakresem?* Niefunkcjonalne: *ilu użytkowników? jaki ruch (zapisy vs odczyty)? jakie wymagania na opóźnienia i dostępność? czy dane mogą być chwilowo niespójne?* Na koniec **powtórz głośno ustalony zakres** — to Twój kontrakt na resztę rozmowy.

**Krok 2: Back-of-envelope estimation (szacowanie "na kopercie", ~5 min).** Zgrubne rachunki rzędów wielkości: QPS (queries per second — zapytania na sekundę), wolumen danych, przepustowość. Przykład: 10 mln użytkowników dziennie × 10 akcji = 100 mln żądań/dobę ≈ ~1200 QPS średnio, szczyt ~3–5×. Cel nie jest księgowy — chodzi o decyzje: 1200 QPS to nie jest skala wymagająca egzotyki; 500 TB danych rocznie to jest argument za partycjonowaniem. Licz głośno i zaokrąglaj brutalnie (sekund w dobie jest "około 100 tysięcy").

**Krok 3: API design (~5 min).** Kilka kluczowych endpointów/operacji: nazwa, parametry, odpowiedź. Np. `POST /bookings {resourceId, slot, userId} → 201 {bookingId}`. To zmusza do skonkretyzowania, czym system w ogóle jest, i daje słownik do dalszej rozmowy.

**Krok 4: Data model (~5 min).** Główne encje, relacje, klucze; wybór bazy z uzasadnieniem (relacyjna vs dokumentowa vs key-value) — wybór ma wynikać z wzorców dostępu z kroków 1–2, nie z mody.

**Krok 5: High-level design (~10 min).** Diagram pudełek: klienci → load balancer → usługi → bazy/kolejki/cache. Poziom C4-kontenerów (moduł 1). Przeprowadź główny przepływ przez diagram ("rezerwacja wchodzi tu, walidacja tu, zapis tu, zdarzenie leci tam"). Diagram ma działać dla wymagań z kroku 1 — skalowanie przyjdzie później.

**Krok 6: Deep dive (~10 min).** Zejście w głąb 1–2 komponentów. Często interviewer wskazuje miejsce; jeśli nie — zaproponuj sam, najlepiej tam, gdzie jesteś mocny: "najciekawszy problem widzę w niezawodnym dostarczeniu powiadomień — mogę rozrysować idempotencję i DLQ?". Tu spłacają się moduły 2–3.

**Krok 7: Bottlenecks i scaling (~5 min).** Sam wskaż wąskie gardła i pojedyncze punkty awarii: co pada pierwsze przy 10× ruchu? gdzie cache, gdzie repliki odczytu, gdzie partycjonowanie? co się dzieje, gdy padnie komponent X? Zakończenie z własnej inicjatywy zdaniem "co bym poprawił, mając więcej czasu" zostawia świetne wrażenie.

### Zarządzanie czasem (45 minut)

Typowy podział: ~5 min small talk i treść zadania, ~35 min projektowanie, ~5 min Twoje pytania do firmy. W tych 35 minutach kroki 1–4 to maksymalnie 15 minut — najczęstszy błąd czasowy to utknięcie w wymaganiach i estymacjach, przez co high-level design powstaje w panice. Trzymaj prosty budżet: **15 min fundamenty (kroki 1–4), 10 min high-level, 10 min deep dive + bottlenecki**. Kontroluj czas jawnie i na głos: "zostało nam ~15 minut — proponuję wejść głębiej w serwis powiadomień, chyba że wolisz inny obszar?". Zarządzanie czasem rozmowy to też sygnał seniority.

### Typowe zadania — krótka charakterystyka

| Zadanie | Co naprawdę testuje | Sedno |
|---|---|---|
| **URL shortener** (skracacz linków, np. bit.ly) | klasyka na rozgrzewkę: estymacje, hashowanie, read-heavy | generowanie krótkich, unikalnych kluczy; stosunek odczytów do zapisów ~100:1 → cache; przekierowanie 301 vs 302 (cache'owalność vs statystyki) |
| **Notification system** (system powiadomień: push/SMS/e-mail) | messaging — Twój teren | fan-out przez kolejki, integracje z zewnętrznymi dostawcami (rate limity, awarie), retry + idempotencja (nikt nie chce 5 takich samych SMS-ów), preferencje użytkownika |
| **Booking system** (rezerwacje: kino, lekarz, hotel) | spójność i współbieżność — Twój teren z Aerotunelu | dwóch klientów, jedno miejsce: blokada pesymistyczna vs optymistyczna, rezerwacja tymczasowa z TTL (time-to-live — czas życia wpisu), saga płatności, overbooking jako decyzja biznesowa |
| **Chat** (komunikator, np. WhatsApp) | komunikacja real-time, stan połączeń | WebSocket (trwałe dwukierunkowe połączenie) vs polling, dostarczanie do odbiorcy offline, kolejność wiadomości, statusy odczytania, grupy = fan-out |
| **News feed** (strumień aktualności, np. Facebook/LinkedIn) | klasyk trade-offu push vs pull | fan-out on write (budujemy feed przy publikacji — szybki odczyt, drogi zapis) vs fan-out on read (składamy przy odczycie — odwrotnie); hybryda dla celebrytów z milionami obserwujących; cache + paginacja |

Wzorce się powtarzają: niemal każde zadanie sprowadza się do kombinacji load balancer + cache + kolejka + partycjonowana baza + decyzja o spójności. Po 3–4 przerobionych zadaniach zaczniesz to widzieć.

### Najczęstsze błędy kandydatów

1. **Skok do rozwiązania bez wymagań.** Rysowanie Kafki w drugiej minucie. Dla interviewera to czerwona flaga: tak samo będziesz pracował z prawdziwymi stakeholderami.
2. **Milczenie.** Myślisz świetnie, ale w ciszy — interviewer nie ocenia myśli, ocenia to, co słyszy. Trenuj thinking out loud: "rozważam A albo B; A daje X, ale kosztuje Y… biorę B, bo wymaganie Z".
3. **Brak trade-offów.** "Użyję NoSQL" bez "zamiast czego i za jaką cenę". Każda decyzja bez kosztu brzmi jak zaklęcie z bloga.
4. **Ignorowanie podpowiedzi.** Pytanie interviewera "a co, jeśli ten serwis padnie?" to nie ciekawość — to wskazanie, gdzie masz iść. Kandydaci zbywający hinty jednym zdaniem i wracający do swojego planu przepalają największą szansę rozmowy.
5. **Over-engineering na starcie.** Mikroserwisy, multi-region i CQRS dla 1000 użytkowników. Zacznij prosto, skaluj, gdy estymacje tego wymagają — i powiedz to wprost: "na tę skalę wystarczy monolit z repliką; powiem, co zmienię przy 100×".
6. **Brak domknięcia.** Rozmowa urywa się w połowie deep dive'u. Pilnuj zegara, zostaw 3 minuty na podsumowanie: co zaprojektowaliśmy, jakie decyzje, co dalej.

### Jak ćwiczyć — solo i z partnerem

**Solo:** weź zadanie, ustaw timer 35 min, rysuj w Excalidraw i **mów na głos do pustego pokoju** (tak, to dziwne; tak, to działa — mówienie jest połową treningu). Po sesji porównaj z przerobionym rozwiązaniem (np. z książki Alexa Xu) — nie żeby skopiować, tylko sprawdzić, jakie wymagania i bottlenecki pominąłeś.

**Z partnerem:** znajdź drugiego inżyniera (niekoniecznie lepszego — zadawanie pytań jako interviewer to też świetny trening) i umówcie wymianę: tydzień Ty projektujesz, tydzień on.

**Z Claude jako interviewerem:** pełnowartościowy mock dostępny od ręki. Gotowy prompt:

> Jesteś doświadczonym interviewerem prowadzącym system design interview na poziom Senior/Architect. Zadanie: [np. "zaprojektuj system powiadomień dla aplikacji z 5 mln użytkowników"]. Zasady: (1) podajesz tylko treść zadania — wymagania ujawniasz wyłącznie, gdy o nie zapytam; (2) prowadzisz rozmowę etapami, nie podajesz rozwiązań, najwyżej krótkie hinty, gdy utknę na dobre; (3) zadajesz pytania pogłębiające o trade-offy, scenariusze awarii i skalowanie — jak prawdziwy interviewer; (4) pilnujesz czasu: po ~35 min wymiany prosisz o podsumowanie; (5) na końcu dajesz szczerą ocenę w formacie: mocne strony / słabe strony / werdykt (hire na jaki poziom / no-hire) / 3 rzeczy do poprawy następnym razem. Nie bądź miły kosztem szczerości. Zaczynamy — podaj zadanie.

Diagram rysuj równolegle w Excalidraw, a do czatu opisuj decyzje słowami — wymuszona werbalizacja to dokładnie ten trening, którego potrzebujesz. Wariant mocniejszy: rozmowa głosowa z Claude (aplikacja mobilna) — ćwiczysz pełne mówienie. Po mocku poproś dodatkowo: "wskaż 3 momenty, w których powinienem był zadać pytanie, a nie zadałem".

### Kiedy ten format NIE mierzy tego, co trzeba (trade-offy)

- Format premiuje szybkość i pewność siebie — realna architektura premiuje namysł i research. Nie wyciągaj z mocków wniosku, że "architekt musi decydować w 45 minut".
- Zadania big-techowe (miliardy użytkowników) bywają oderwane od firm, do których realnie aplikujesz — na rozmowie w polskiej firmie produktowej "booking system na 50k użytkowników" jest bardziej prawdopodobny niż "zaprojektuj YouTube". Trenuj oba rejestry skali.
- Twoja przewaga — prawdziwe historie — działa tylko dozowana: jedno zdanie blizny ("w Aerotunelu ten dokładnie scenariusz położył nam integrację, dlatego…") wzmacnia; pięciominutowa anegdota zjada budżet czasu. Pełne historie zostaw na rozmowę behawioralną (lekcja 04).

## Praktyka

Harmonogram 5–6 mocków — element **ciągły** modułu: pierwszy mock w tym tygodniu, kolejne co ~tydzień (dobrze współgrają z pracą nad Capstone). Po każdym mocku zapisz 3 wnioski w notatce `mock-NN.md` w folderze tej lekcji i przeczytaj wnioski z poprzedniego mocka *przed* kolejnym.

- [ ] Przygotowanie (tydzień 2 modułu): rozpisz framework krokowy z pamięci na jednej kartce (ściąga), przeczytaj 1 przerobione zadanie u Alexa Xu, ustaw Excalidraw.
- [ ] **Mock 1 (tydzień 2 modułu): URL shortener, solo z timerem** — rozgrzewka, cel: przejść wszystkie kroki frameworku w 35 min, mówiąc na głos.
- [ ] **Mock 2 (+1 tydzień): notification system, Claude jako interviewer** (prompt z teorii) — Twój mocny teren; cel: deep dive w retry/idempotencję/DLQ + pełna ocena od Claude.
- [ ] **Mock 3 (+2 tygodnie): booking system, Claude jako interviewer** — teren Aerotunelu; cel: współbieżność i spójność + wpleść jedną bliznę produkcyjną w jednym zdaniu.
- [ ] **Mock 4 (+3 tygodnie): news feed, Claude lub partner** — teren celowo obcy; cel: fan-out push vs pull, radzenie sobie poza strefą komfortu.
- [ ] **Mock 5 (+4 tygodnie): chat, najlepiej z żywym partnerem** (jeśli brak — Claude głosowo); cel: real-time, WebSockety, kolejność wiadomości.
- [ ] **Mock 6 (+5 tygodni, opcjonalny-zalecany): powtórka najsłabszego zadania** z mocków 1–5, z naciskiem na wnioski z notatek.
- [ ] Wpisz wszystkie mocki z datami do DayChunks/kalendarza **już dziś** — mock niewpisany to mock, który się nie odbędzie.

## Artefakt

1. **Ściąga frameworku** (1 kartka/plik `framework.md`): kroki, budżet czasowy, pytania do wymagań, liczby do estymacji.
2. **Notatki z mocków** (`mock-01.md` … `mock-06.md`): zadanie, diagram (eksport z Excalidraw), 3 wnioski, ocena interviewera.
3. **Harmonogram mocków wpisany do DayChunks** z konkretnymi datami.

## Definition of Done

- [ ] Wymieniasz z pamięci wszystkie kroki frameworku wraz z budżetem czasowym każdego.
- [ ] Odbyły się co najmniej 2 z zaplanowanych mocków (pozostałe mają daty w kalendarzu) — moduł domykasz po dwóch, serię kończysz w kolejnych tygodniach.
- [ ] Każdy odbyty mock ma notatkę z 3 wnioskami i diagramem.
- [ ] Dla każdego z 5 typowych zadań umiesz powiedzieć jednym zdaniem, "o co w nim naprawdę chodzi".
- [ ] W co najmniej jednym mocku świadomie użyłeś prawdziwej historii produkcyjnej jako jednozdaniowego argumentu za decyzją.

## Materiały

1. Alex Xu, *System Design Interview – An Insider's Guide* (vol. 1) — framework + kilkanaście przerobionych zadań (jest tam URL shortener, notification system, news feed i chat); czytaj po jednym rozdziale przed odpowiadającym mockiem.
2. [System Design Primer](https://github.com/donnemartin/system-design-primer) — darmowe repo-kompendium: estymacje, wzorce skalowania, zadania z rozwiązaniami; używaj jako referencji po mockach, nie jako lektury od deski do deski.
