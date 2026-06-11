# Lekcja 07 — Warsztat: ćwiczenia system design

## Cel lekcji

Po tej lekcji będziesz miał wytrenowany **powtarzalny framework podchodzenia do zadań system design** (wymagania → szacowanie → API → model danych → architektura → wąskie gardła) i trzy samodzielnie rozpisane projekty, w których użyłeś wszystkich narzędzi modułu: atrybutów jakościowych, trade-offów, C4, ADR-ów, bounded contexts i stylów.

## Dlaczego to ważne

System design interview to standardowy etap rekrutacji na role senior/staff/architect: dostajesz otwarte zadanie ("zaprojektuj X") i 45–60 minut. Ocenie nie podlega "poprawność" rozwiązania (nie ma jednego poprawnego) — ocenie podlega **proces**: czy zbierasz wymagania, zanim rysujesz; czy szacujesz, zanim dobierzesz technologię; czy nazywasz trade-offy; czy mówisz "why" do każdego klocka. Kandydaci wykładają się nie na braku wiedzy, tylko na braku **struktury**: skaczą od razu do ulubionej technologii i topią się w detalach jednego komponentu, gdy czas mija.

Druga funkcja tej lekcji: to **trening syntezy** całego modułu. Lekcje 01–06 dały klocki; tu po raz pierwszy składasz je pod presją pustej kartki. Definition of Done modułu ("dla dowolnego problemu: 2–3 opcje + trade-offy") testuje się dokładnie tutaj.

## Teoria

### Framework krokowy (45–60 minut rozmowy lub 2h własnej sesji)

Zasada nadrzędna: **mów głośno / pisz wszystko** — w rozmowie liczy się tok myślenia; w treningu solo notatka jest jedynym dowodem, że myślenie zaszło.

**Krok 1 — Wymagania (5–10 min).** Najpierw funkcjonalne: 3–5 kluczowych use case'ów — i ŚWIADOME cięcie zakresu ("projektuję rezerwację i odwołania; płatności i raporty oznaczam jako out of scope"). Potem niefunkcjonalne, czyli atrybuty jakościowe (lekcja 01): ilu użytkowników? jaki ruch i jego rozkład (szczyty!)? jaka tolerancja na niedostępność i na nieaktualne dane? Wybierz **2–3 atrybuty priorytetowe i powiedz, co poświęcasz**. Najczęstszy błąd całego formatu: pominięcie tego kroku. Zadanie "zaprojektuj X" jest celowo niedospecyfikowane — sprawdza, czy zauważysz.

**Krok 2 — Szacowanie (5 min, back-of-the-envelope).** Rząd wielkości, nie precyzja: użytkownicy → akcje/dzień → średni RPS (requests per second) → szczytowy RPS (zwykle 5–10× średni, w systemach z "otwarciem grafiku" i 50×) → rozmiar danych (rekordy × bajty × lata). Po co: liczby decydują o architekturze. 10 RPS = jedna baza i spokój; 10 000 RPS = cache, repliki, partycjonowanie. Bez szacowania każda dyskusja o skalowalności to teatr. Przydatne stałe: doba ~86 400 s; 1 mln akcji/dobę ≈ 12 RPS średnio; rekord "rezerwacja" ≈ setki bajtów, więc miliony rekordów to wciąż gigabajty — mieszczą się w jednej bazie.

**Krok 3 — API (5–10 min).** Nazwij główne operacje jako kontrakty (REST wystarczy): `POST /bookings`, `GET /classes/{id}/slots`, `DELETE /bookings/{id}`. Dla każdej: wejście, wyjście, błędy biznesowe (409 gdy brak miejsc?). API wymusza precyzję: "rezerwacja" przestaje być mgławicą, staje się operacją z parametrami. Tu też pada pytanie sync/async: co musi odpowiedzieć natychmiast, a co może być "przyjęte do realizacji" (202)?

**Krok 4 — Model danych (5–10 min).** Główne encje, klucze, relacje, kardynalności — plus dwa pytania architektoniczne: jaki magazyn (relacyjny? dokumentowy? cache?) i **co jest źródłem prawdy** dla każdej danej. Zwróć uwagę na dane gorące (stan wolnych miejsc — czytany tysiące razy, zmieniany często) vs zimne (historia rezerwacji) — często zasługują na różne traktowanie.

**Krok 5 — Architektura wysokopoziomowa (10–15 min).** Teraz dopiero rysujesz — i to jest po prostu **C4 poziom 2** (lekcja 03): aktorzy, kontenery, magazyny, strzałki z czasownikami. Dobierz styl świadomie (lekcja 06) i powiedz dlaczego. Dla każdego niestandardowego klocka (kolejka, cache) — jedno zdanie uzasadnienia przez wymaganie z kroku 1. Klocek bez uzasadnienia to dług na dopytkę "why?".

**Krok 6 — Wąskie gardła, awarie, trade-offy (10–15 min).** Sam atakujesz własny projekt, zanim zrobi to rozmówca: co pada pierwsze przy 10× ruchu? co się dzieje, gdy umrze baza / broker / zewnętrzny dostawca? gdzie jest wyścig (dwóch ludzi, ostatnie miejsce)? gdzie świadomie wybrałeś eventual consistency i co użytkownik z tego zobaczy? Zamknij podsumowaniem: "głównymi trade-offami tego projektu są…". To krok, który odróżnia seniora — junior broni projektu, senior zna jego słabości z góry.

### Jak trenować solo (bez rozmówcy)

- **Timeboxuj** (90–120 min na ćwiczenie, z podziałem na kroki jak wyżej) — bez limitu czasu nie trenujesz selekcji, a selekcja jest połową umiejętności.
- **Pisz/rysuj wszystko** — artefaktem jest notatka + diagram, jak z prawdziwej rozmowy.
- **Odegraj dopytki:** po skończeniu zadaj sobie 3 pytania "why not X?" i 2 pytania "co jak 10× ruch / co jak padnie Y?" — i odpowiedz pisemnie.
- **Po 1–2 dniach wróć i zrecenzuj** własną pracę checklistą z frameworku: który krok pominąłeś? (Prawie zawsze: szacowanie albo cięcie zakresu.)

Pułapki do unikania: zaczynanie od technologii ("użyję Kafki" zanim padło słowo o ruchu); projektowanie pod skalę Google przy 12 RPS (overengineering to TEŻ błąd — nazwij go: YAGNI); milczące pominięcie awarii zależności zewnętrznych; równomierne 60 minut na wszystko zamiast świadomych proporcji.

---

### Ćwiczenie A — System rezerwacji zajęć grupowych (twój teren — zacznij od niego)

**Brief:** Sieć 200 klubów fitness. Członkowie rezerwują miejsca na zajęcia grupowe (joga 18:00, 20 miejsc) przez aplikację mobilną i web. Grafik na kolejny tydzień otwiera się w poniedziałek o 6:00. Lista rezerwowa: gdy ktoś odwoła, pierwszy z listy dostaje miejsce. No-show (rezerwacja bez przyjścia) ma konsekwencje (np. blokada na 7 dni).

**Wymagania niefunkcjonalne (przyjmij):** 400 tys. aktywnych członków; poniedziałek 6:00–6:15 = ekstremalny szczyt (setki tysięcy prób w kwadrans); rezerwacja musi być definitywna (członek od razu wie: mam / nie mam / jestem 3. na liście rezerwowej); overbooking niedopuszczalny.

**Pytania naprowadzające:**
- Policz szczytowy RPS dla poniedziałku 6:00. Czy odczyt grafiku i zapis rezerwacji mają ten sam rozkład ruchu? (Podpowiedź: odczytów jest 10–100× więcej — co z tego wynika dla cache?)
- Dwóch członków klika ostatnie wolne miejsce w tej samej milisekundzie. Gdzie dokładnie rozstrzygasz ten wyścig: blokada w bazie? constraint + obsługa konfliktu? kolejka serializująca zapisy na zajęcia? Rozpisz 2 opcje i trade-offy (latencja vs prostota vs throughput).
- Czy lista rezerwowa i kary za no-show muszą działać synchronicznie? Co naturalnie wychodzi za broker (lekcja 06, etap 2)?
- Po czym poznasz, że problem szczytu rozwiązywać cache'em+replikami, a po czym, że kolejkowaniem zapisu ("poczekalnia" jak w sklepach biletowych)? Kiedy drugie podejście psuje wymaganie "definitywności"?

**Na co zwrócić uwagę przy autorecenzji:** czy wybrałeś spójność czy dostępność dla zapisu rezerwacji i czy powiedziałeś to wprost (lekcja 01); czy stan "wolne miejsca" ma jedno źródło prawdy; czy poniedziałkowy szczyt przeliczyłeś, a nie "obsłużyłem skalowalnością"; bonus: porównaj z tym, jak robił to system, który znasz z produkcji — co dziś zaprojektowałbyś inaczej i czemu?

### Ćwiczenie B — System powiadomień multi-channel

**Brief:** Wewnętrzna "platforma powiadomień" dla firmy typu Perfect Gym: inne systemy (rezerwacje, umowy, płatności) zlecają powiadomienia, platforma dostarcza je kanałami: e-mail, SMS, push, in-app. Użytkownik ma preferencje (np. "marketing tylko e-mail, operacyjne wszystkie kanały"). SMS kosztuje pieniądze. Dostawcy kanałów bywają niedostępni.

**Wymagania niefunkcjonalne (przyjmij):** 2 mln powiadomień/dobę, szczyty po masowych zdarzeniach (odwołanie zajęć w 200 klubach po awarii = 100 tys. powiadomień w minutę); powiadomienie operacyjne ≤ 5 min; **żadne nie może zginąć bez śladu**; duplikat SMS-a = realny koszt i irytacja.

**Pytania naprowadzające:**
- Jaki jest kontrakt wejściowy? Systemy zlecające mówią "wyślij SMS o treści X" (command) czy "stało się Y" (event), a platforma sama decyduje o kanałach wg preferencji? Rozpisz trade-off (sprzężenie, kto zna preferencje, kto zna szablony treści — lekcja 05: gdzie jest granica kontekstu?).
- Dostawca SMS dostarczył, ale odpowiedź HTTP do ciebie nie doszła (timeout). Retry wyśle SMS drugi raz. Jak ograniczasz duplikaty? (Słowa-klucze do drążenia: idempotency key, at-least-once vs at-most-once — wybierz świadomie PER KANAŁ: czy e-mail i SMS zasługują na tę samą gwarancję?)
- Gdzie żyje stan "powiadomienie N: zlecone → zrenderowane → wysłane → dostarczone/odbite"? Jak zrealizujesz "żadne nie ginie bez śladu" i co będzie twoim DLQ-procesem (lekcja 06)?
- Jak izolujesz wolnego/martwego dostawcę SMS, żeby nie zatkał e-maili? (Podpowiedź: osobne kolejki/pule per kanał; do nazwania: backpressure.)
- Priorytety: masowy marketing nie może opóźnić operacyjnego "twoje zajęcia za godzinę odwołano". Jak?

**Na co zwrócić uwagę:** to ćwiczenie jest naturalnie event-driven — sprawdź, czy zastosowałeś lekcję 06 świadomie (i czy COŚ zostało synchroniczne, np. API przyjęcia zlecenia); czy rozdzieliłeś orkiestrację (decyzja: co, komu, którym kanałem) od dostarczania (adapter per dostawca — lekcja 05: to są ACL do vendorów); czy status powiadomienia da się odpytać (obserwowalność).

### Ćwiczenie C — Skracacz URL-i (klasyk rozmów)

**Brief:** Publiczny serwis à la bit.ly: `POST` z długim URL-em zwraca krótki (`https://krt.li/x7Yp2A`); `GET` krótkiego przekierowuje (301/302) na długi. Opcjonalnie: własne aliasy, data wygaśnięcia, licznik kliknięć.

**Wymagania niefunkcjonalne (przyjmij):** 100 mln nowych URL-i/rok; odczyty:zapisy = 100:1; redirect p99 < 50 ms globalnie; dostępność redirectu ważniejsza niż świeżość licznika kliknięć.

**Pytania naprowadzające:**
- Policz: zapisy RPS, odczyty RPS, ile znaków potrzebuje krótki kod przy alfabecie base62 (62 znaki), żeby pomieścić lata działania bez kolizji? Ile TB danych po 5 latach (URL ≈ 500 B)?
- Generowanie kodu: hash długiego URL-a vs licznik globalny zakodowany base62 vs pre-generowana pula kluczy. Rozpisz trade-offy każdej opcji (kolizje? przewidywalność/wyliczalność kodów? wąskie gardło licznika w skali horyzontalnej?).
- Ścieżka odczytu robi 99% ruchu i ma budżet 50 ms: co cache'ujesz, gdzie (in-memory? Redis? CDN?) i co z **cache miss** oraz nieistniejącym kodem? (Do drążenia: jak złośliwy ruch o losowe kody omija cache i bije w bazę — czym się bronisz?)
- 301 (permanent, przeglądarka zapamięta i więcej nie zapyta) vs 302 (temporary, każde kliknięcie przechodzi przez ciebie): który wybierasz i co to robi z licznikiem kliknięć? To czysty trade-off performance vs funkcja — nazwij go.
- Licznik kliknięć przy 10 tys. RPS odczytów: UPDATE na wiersz przy każdym kliknięciu? (Podpowiedź: zliczanie asynchroniczne/wsadowe; połącz z deklaracją "świeżość licznika nieważna" z wymagań.)

**Na co zwrócić uwagę:** to ćwiczenie testuje głównie kroki 2 i 6 (szacowanie i wąskie gardła) — jeśli nie policzyłeś liczb, zrobiłeś je źle; sprawdź, czy ścieżka odczytu i zapisu dostały RÓŻNE potraktowanie (to esencja zadania); uwaga na overengineering — przy tych liczbach NIE potrzebujesz egzotycznych baz, i powiedzenie tego wprost jest punktowane.

## Praktyka

- [ ] Przepisz framework (6 kroków + czasy) własnymi słowami na jedną kartkę-ściągę; trzymaj ją przy każdym ćwiczeniu i przyszłych rozmowach.
- [ ] **Ćwiczenie A** — pełna sesja 120 min wg frameworku: notatki kroków 1–4, diagram C4-L2, sekcja wąskich gardeł, minimum 1 ADR dla najtrudniejszej decyzji (wyścig o ostatnie miejsce).
- [ ] **Ćwiczenie B** — pełna sesja 120 min, jak wyżej (ADR: gwarancje dostarczenia per kanał).
- [ ] **Ćwiczenie C** — sesja 90 min, jak wyżej (tu szczególnie: kompletne szacowanie).
- [ ] Po każdym ćwiczeniu: autorecenzja z opóźnieniem 1–2 dni (checklista frameworku + sekcja "na co zwrócić uwagę") + pisemne odpowiedzi na 3× "why not X?" i 2× "co jak 10×/awaria?".
- [ ] Na koniec: wróć do Definition of Done modułu i przetestuj się na świeżym problemie spoza listy (np. "system kolejkowania do recepcji" albo dowolny z codziennej pracy): 10 minut, 2–3 opcje, trade-offy, na głos.

## Artefakt

Folder `system-design/` w repo nauki z trzema podfolderami (`a-rezerwacje/`, `b-powiadomienia/`, `c-url-shortener/`), każdy zawiera: notatki kroków 1–6, diagram (plik źródłowy + obraz), ADR-y decyzji kluczowych, autorecenzję. To portfolio procesu — przed prawdziwymi rozmowami (moduł 6 i dalej) wrócisz tu i zobaczysz progres; wybrane fragmenty mogą zasilić posty LinkedIn.

## Definition of Done

- [ ] Umiesz wyrecytować 6 kroków frameworku z orientacyjnymi czasami i wyjaśnić, czemu kolejność jest taka, a nie inna (czemu szacowanie przed technologią, czemu API przed diagramem).
- [ ] Trzy ćwiczenia wykonane w timeboxie, z kompletem artefaktów i autorecenzją; w każdym z nich kroki 1 (cięcie zakresu + atrybuty) i 2 (liczby) są wykonane, nie pominięte.
- [ ] W każdym projekcie potrafisz wskazać: główny trade-off, pierwszy element, który padnie przy 10× ruchu, i decyzję, którą podjąłbyś inaczej przy zmienionych wymaganiach.
- [ ] Test końcowy modułu zaliczony: świeży problem → 2–3 opcje z uczciwymi trade-offami w ≤ 10 minut, bez notatek.

## Materiały

1. **"System Design Interview – An Insider's Guide"** (Alex Xu, vol. 1) — kanoniczny trening formatu; po własnym wykonaniu ćwiczenia C porównaj ze stosownym rozdziałem (kolejność ważna: najpierw sam, potem książka — inaczej odbierasz sobie trening).
2. **"Fundamentals of Software Architecture"** (Richards, Ford) — rozdziały o architecture katas; sami autorzy zalecają katas jako podstawową metodę treningu myślenia architektonicznego — ta lekcja jest dokładnie tym.
