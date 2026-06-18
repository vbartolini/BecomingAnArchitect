# Lekcja 02 — Trade-off analysis: architektura jako wybór, czego NIE robić

## Cel lekcji

Po tej lekcji będziesz umiał przeprowadzić **pisemną analizę trade-offów**: rozpisać 2–3 opcje rozwiązania, ocenić je w macierzy względem priorytetowych atrybutów, ocenić koszt odwracalności decyzji (one-way vs two-way door) i sformułować rekomendację, która uczciwie nazywa, co tracimy.

## Dlaczego to ważne

To jest **najważniejsza lekcja całego modułu** — wszystko inne (C4, ADR, DDD, style) to narzędzia obsługujące tę jedną umiejętność.

Na rozmowach na role architektoniczne najczęstsze pytanie pogłębiające brzmi: *"why not X?"*. Zaproponowałeś mikroserwisy — "a czemu nie monolit?". Wybrałeś kolejkę — "a czemu nie synchroniczne API?". Kandydat-developer broni swojego wyboru. Kandydat-architekt mówi: "rozważałem trzy opcje, oto dlaczego ta wygrała **w tym kontekście** i co by musiało się zmienić, żebym wybrał inną". Ta druga odpowiedź jest nie do podrobienia bez treningu.

W realnej pracy analiza trade-offów na piśmie ma drugą funkcję: **chroni cię rok później**. Gdy decyzja okaże się bolesna (a część zawsze się okaże), różnica między "podjęliśmy ją świadomie, znając ten koszt" a "jakoś tak wyszło" to różnica między zaufaniem zespołu a jego utratą. Masz to w narracji DayChunks od początku: aplikacja bez backendu to dokładnie taki świadomy wybór, czego NIE robić.

## Teoria

### Pierwsze prawo architektury

Richards i Ford otwierają *Fundamentals of Software Architecture* dwoma prawami:

> **Pierwsze prawo:** *Everything in software architecture is a trade-off.* — Wszystko w architekturze oprogramowania jest trade-offem. A jeśli wydaje ci się, że znalazłeś coś, co nim nie jest — to znaczy, że jeszcze nie zidentyfikowałeś trade-offu.
>
> **Drugie prawo:** *Why is more important than how.* — "Dlaczego" jest ważniejsze niż "jak".

**Trade-off** (kompromis, wymiana) to sytuacja, w której nie da się mieć obu rzeczy naraz i wybór jednej oznacza rezygnację z części drugiej. W architekturze nie istnieją rozwiązania lepsze "w ogóle" — istnieją rozwiązania lepsze **dla konkretnego kontekstu**: zestawu priorytetowych atrybutów (lekcja 01), zespołu, budżetu, horyzontu czasowego.

Konsekwencja praktyczna, którą warto sobie wytatuować: jeśli na pytanie architektoniczne odpowiadasz natychmiast i bez "to zależy" — prawdopodobnie odpowiadasz jako developer, z pamięci mięśniowej, a nie jako architekt. Poprawna pierwsza odpowiedź architekta na większość pytań brzmi *"it depends"* — pod warunkiem, że zaraz potem potrafisz powiedzieć, **od czego dokładnie zależy**.

### Architektura jako wybór, czego NIE robić

Każda decyzja architektoniczna jest jednocześnie decyzją o rezygnacji:

- Wybierasz spójność silną → rezygnujesz z części dostępności i wydajności.
- Wybierasz mikroserwisy → rezygnujesz z prostoty debugowania, transakcji obejmujących całość i taniego refactoringu między granicami.
- Wybierasz "DayChunks bez backendu" → rezygnujesz z synchronizacji między urządzeniami, analityki po stronie serwera i części modeli monetyzacji — w zamian za zerowy koszt utrzymania, prywatność danych i brak całych klas awarii.

Niedojrzała architektura poznaje się po tym, że lista rezygnacji jest pusta. Dojrzała — po tym, że rezygnacje są **wypisane wprost, zanim ktoś odkryje je na produkcji**.

### Proces: jak prowadzić analizę trade-offów na piśmie

Dlaczego na piśmie? Bo w głowie każdy z nas oszukuje: faworyzuje opcję, która przyszła pierwsza (anchoring), albo tę, którą zna najlepiej. Pisanie wymusza symetrię — każda opcja dostaje te same rubryki.

Krok po kroku:

**1. Sformułuj problem bez wbudowanego rozwiązania.** "Jak wdrożyć Redis do cache'owania grafiku?" to nie jest problem — to rozwiązanie udające problem. Problem brzmi: "odczyt grafiku zajęć nie wytrzymuje porannego szczytu (p95 > 3 s przy 200 RPS)". Z tak postawionym problemem opcjami są: cache, read replica bazy, denormalizacja, CDN dla części statycznej… Z tamtym pierwszym — tylko Redis.

**2. Ustal kryteria PRZED wypisaniem opcji.** Kryteria to priorytetowe atrybuty jakościowe z lekcji 01 plus realia: koszt, czas dostarczenia, kompetencje zespołu, koszt operacyjny (kto to będzie utrzymywał o 3:00 w nocy). Kolejność jest ważna: kryteria ustalone *po* wypisaniu opcji mają tendencję do bycia dopasowanymi pod ulubieńca.

**3. Wypisz 2–4 realne opcje — w tym zawsze opcję "nie robić nic" lub "zrobić najprościej".** Status quo to też opcja i często wygrywa. Jeśli masz tylko jedną opcję, nie podejmujesz decyzji — wykonujesz odruch. Jeśli masz siedem, nie zrobiłeś preselekcji i utopisz analizę.

**4. Wypełnij macierz opcji.** Dla każdej pary (opcja × kryterium) — krótka ocena z uzasadnieniem. Skala prosta: ++ / + / 0 / − / −− albo słownie. **Nie sumuj punktów mechanicznie** — macierz służy do zobaczenia profilu każdej opcji i wykrycia dyskwalifikatorów (jedno "−−" na krytycznym kryterium bije pięć "++" gdzie indziej). Suma punktów to iluzja obiektywności; decyzję i tak podejmuje człowiek, macierz ma tylko zmusić go do spojrzenia wszystkim kosztom w oczy.

**5. Oceń odwracalność (o tym za chwilę) i sformułuj rekomendację** w formacie: *wybieram X, świadomie płacąc Y; wrócimy do tej decyzji, jeśli Z*. Część "jeśli Z" to **warunki rewizji** — z góry nazwane sygnały, że kontekst się zmienił (np. "jeśli liczba klubów przekroczy 200" albo "jeśli pojawi się wymaganie pracy offline").

### One-way doors vs two-way doors — koszt odwracalności

Pojęcie spopularyzowane przez Jeffa Bezosa, w architekturze bezcenne:

- **Two-way door** (drzwi dwukierunkowe) — decyzja **odwracalna**: jak się nie spodoba, wracasz, koszt powrotu mały. Przykłady: biblioteka do logowania, format konfiguracji, większość wyborów wewnątrz jednego modułu.
- **One-way door** (drzwi jednokierunkowe) — decyzja **praktycznie nieodwracalna**: powrót kosztuje miesiące albo jest niemożliwy. Przykłady: wybór głównej bazy danych po roku produkcji, publiczny kontrakt API używany przez integratorów (znasz z Perfect Gym: raz opublikowanego API, na którym wiszą zewnętrzne systemy, nie cofniesz), model multi-tenancy (osobna baza per klub vs wspólna z kolumną TenantId), granice między serwisami.

Reguła decyzyjna:
- **Two-way door → decyduj szybko, płytko, najlepiej na niskim szczeblu.** Analiza na 3 strony dla wyboru biblioteki JSON to marnotrawstwo gorsze niż zły wybór.
- **One-way door → zwolnij.** Pełna macierz, ADR (lekcja 04), recenzja drugiej osoby, czasem prototyp (spike) zanim klamka zapadnie.

Najczęstszy błąd organizacji: **odwrotna alokacja uwagi** — tygodnie sporów o formatowanie kodu i nazewnictwo (two-way), a wybór szyny komunikacji między serwisami "jakoś wyszedł" na daily (one-way).

Druga technika z tym związana: **opóźniaj nieodwracalne do last responsible moment** (ostatniego odpowiedzialnego momentu) — punktu, po którym brak decyzji sam staje się decyzją (i to zwykle najgorszą). Nie wcześniej, bo wiedza rośnie z czasem; nie później, bo opcje wygasają. Oraz: czasem warto **zapłacić za zamianę one-way w two-way** — np. wzorzec strangler fig (stopniowa wymiana systemu kawałek po kawałku zamiast big-bang rewrite) jest dokładnie tym: kupowaniem odwracalności.

### Przykład przeprowadzonej analizy

Realistyczny problem z twojego podwórka.

**Problem:** System dla sieci siłowni musi powiadamiać członków o odwołanych zajęciach (trener chory → 30 osób ma się dowiedzieć w minuty, nie godziny). Dziś: recepcja dzwoni ręcznie. Skala: 300 klubów, ~50 odwołań dziennie w całej sieci, szczytowo 200 po śnieżycy.

**Kryteria (w kolejności priorytetu):** (1) niezawodność dostarczenia — członek, który przyjedzie na odwołane zajęcia, to realna strata zaufania; (2) czas dostarczenia < 5 min; (3) koszt utrzymania przez mały zespół; (4) rozszerzalność o kolejne kanały (push, SMS, e-mail).

**Opcje:**

| Kryterium | A: wysyłka synchroniczna z procesu API (pętla po członkach, SMTP/SMS w request handlerze) | B: kolejka + osobny worker powiadomień | C: gotowy SaaS (np. Twilio/SendGrid + ich orkiestracja) |
|---|---|---|---|
| Niezawodność dostarczenia | −− brak retry; restart appki w trakcie pętli = część osób bez SMS-a i **nie wiemy która** | ++ retry, dead-letter queue, persystencja | + wysoka, ale zależna od zewnętrznego dostawcy i jego SLA |
| Czas dostarczenia | + natychmiast (póki działa) | + sekundy opóźnienia, bez znaczenia przy celu 5 min | + porównywalnie |
| Koszt utrzymania | ++ zero nowej infrastruktury | − kolejka + worker + monitoring DLQ do utrzymania | 0 mało kodu, ale vendor billing, limity, integracja |
| Rozszerzalność o kanały | −− każdy kanał wydłuża request; timeouts | ++ nowy konsument kolejki = nowy kanał | + zależnie od oferty vendora |
| Odwracalność | two-way (łatwo uciec) | two-way/one-way (kontrakt komunikatu zaczyna żyć własnym życiem) | częściowo one-way (uzależnienie od vendora, koszty wyjścia) |

**Rekomendacja:** Opcja B. Świadomie płacimy: nowy element infrastruktury i konieczność monitorowania kolejki (w tym dead-letter queue — kolejki "niedoręczalnych", o której wiesz z produkcji, że nieoglądana zamienia się w cmentarz). Opcja A odpada na dyskwalifikatorze: cicha utrata powiadomień bez śladu uderza w kryterium nr 1. Opcja C wraca do rozważenia, **jeśli** dojdzie wymaganie wielokanałowej orkiestracji z preferencjami użytkownika — wtedy budowanie tego in-house przestaje się spinać. To jest warunek rewizji.

Zauważ, czego ta analiza NIE robi: nie udaje, że B jest "najlepsze". Mówi: B jest najmniej złe dla tych kryteriów, a oto rachunek.

### Kiedy NIE robić formalnej analizy

- Decyzja jest two-way door i dotyczy jednego modułu → decyduj w 5 minut, zapisz jedno zdanie w dzienniku decyzji (moduł 0), idź dalej.
- Brakuje danych, a da się je tanio zdobyć → najpierw spike/prototyp, potem analiza. Macierz wypełniona zgadywaniem to teatr.
- Decyzja już zapadła politycznie i analiza ma ją tylko "ubrać" → nie firmuj tego. Napisz uczciwie: "decyzja odgórna, oto jej konsekwencje" (to też jest ADR — zobacz lekcja 04).

Uwaga na **analysis paralysis**: jeśli analiza two-way door trwa dłużej niż wdrożenie i ewentualne wycofanie opcji — analiza stała się droższa od błędu.

## Praktyka

- [x] Weź jedną dużą decyzję techniczną ze swojej przeszłości (np. wybór sposobu integracji w Perfect Gym, architektura rezerwacji w Aerotunelu, "DayChunks bez backendu") i **zrekonstruuj analizę, której wtedy nie spisano**: problem, kryteria, min. 3 opcje (w tym ta odrzucona i ta wybrana), macierz, rekomendacja. Pisz tak, jakbyś przekonywał ówczesnego siebie.
- [x] Dla każdej opcji z macierzy zaklasyfikuj: one-way czy two-way door? Czy ówczesna staranność była proporcjonalna do odwracalności?
- [x] Wypisz 5 decyzji czekających cię w DayChunks (np. sposób przechowywania danych, model synchronizacji, framework UI) i poukładaj je na osi odwracalności. Zaznacz, które zasługują na pełną analizę, a które na decyzję w 5 minut.
- [ ] Pytanie kontrolne (odpowiedz pisemnie, 3–4 zdania): dlaczego mechaniczne sumowanie punktów w macierzy opcji jest pułapką?

## Artefakt

Plik `analiza-trade-off-01.md` w repo nauki: pełna pisemna analiza z pierwszego ćwiczenia (problem → kryteria → macierz → rekomendacja z rezygnacjami i warunkami rewizji). Plus plik `daychunks-decyzje-odwracalnosc.md` z listą decyzji na osi one-way/two-way. Pierwsza analiza będzie też wsadem do lekcji 04 (przepiszesz ją jako ADR).

## Definition of Done

- [ ] Umiesz wyrecytować i zilustrować własnym przykładem oba prawa architektury Richardsa/Forda.
- [x] Twoja macierz ma minimum 3 opcje (w tym "nie robić nic"/"najprościej") i kryteria ustalone przed opcjami.
- [x] W rekomendacji jest wprost nazwane: co poświęcamy i jaki jest warunek rewizji decyzji.
- [ ] Umiesz dla dowolnej świeżej decyzji powiedzieć w 30 sekund: one-way czy two-way door i ile staranności jej się należy.
- [ ] Test z lustra: opowiedz na głos analizę z artefaktu w 3 minuty, kończąc zdaniem "wybrałem X, płacąc Y". Jeśli "Y" brzmi pusto — wróć do macierzy.

## Materiały

1. **"Fundamentals of Software Architecture"** (Richards, Ford) — rozdz. 1 (prawa architektury) i rozdz. o architecture decisions; to źródło "everything is a trade-off".
2. **"Software Architecture: The Hard Parts"** (Ford, Richards, Sadalage, Dehghani) — rozdz. 1 i ostatni (trade-off analysis w praktyce); na razie tylko te fragmenty, pełna lektura wraca w module 2.
