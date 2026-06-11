# Lekcja 06 — Style architektoniczne: modular monolith vs microservices vs event-driven

## Cel lekcji

Po tej lekcji będziesz umiał opisać trzy style architektoniczne (modularny monolit, mikroserwisy, architektura zdarzeniowa) od deploymentu po komunikację między modułami, dobrać styl do kontekstu za pomocą macierzy decyzyjnej — i nazwać ukryte koszty mikroserwisów, o których nie mówią konferencyjne prezentacje, z distributed monolith jako antywzorcem na czele.

## Dlaczego to ważne

"Monolit czy mikroserwisy?" to najczęstsze pytanie architektoniczne ostatniej dekady — na rozmowach rekrutacyjnych, w decyzjach startowych projektów i w bolesnych retrospektywach. Odpowiedź dogmatyczna w każdą stronę dyskwalifikuje; odpowiedź przez trade-offy (lekcja 02) i bounded contexts (lekcja 05) wyróżnia. Ta lekcja spina cały moduł: styl architektoniczny to **decyzja jednokierunkowa** (one-way door) podejmowana względem **atrybutów jakościowych**, zasługująca na **ADR**, rysowana w **C4**, a jej granice wyznacza **mapa kontekstów**.

I blizna z produkcji: każdy, kto utrzymywał system rozproszony połączony kolejkami i integracjami, wie, że rozproszenie nie jest darmowe — debugowanie przez trzy systemy, wiadomości ginące między hopami, wersjonowanie kontraktów. Ta lekcja nadaje tym doświadczeniom nazwy i ramy.

### Najpierw słowo o pojęciu

**Styl architektoniczny** to ogólny kształt systemu: jak jest pocięty na części, jak części się komunikują i jak są wdrażane (deployowane). Styl ≠ wzorzec projektowy (wzorce działają wewnątrz, na poziomie klas/modułów) i styl ≠ technologia (mikroserwisy to nie "Kubernetes").

## Teoria

### Styl 1 — Modular monolith (monolit modularny)

**Czym jest:** cały system to **jedna deployowalna jednostka** (jeden proces, np. jedna aplikacja ASP.NET Core), ale wewnątrz podzielona na **moduły o twardych granicach** — najlepiej pokrywających się z bounded contexts z lekcji 05. Moduł = osobny projekt/assembly z publicznym interfejsem; inne moduły wolno wołać **tylko** przez ten interfejs, nigdy przez sięgnięcie do cudzych klas wewnętrznych czy cudzych tabel.

**Deployment:** jeden artefakt, jeden pipeline, jedna (zwykle) baza danych — choć dobrą praktyką jest **osobny schemat bazy per moduł** i zakaz JOIN-ów między schematami: to utrzymuje granice egzekwowalne i zostawia otwartą furtkę do przyszłego wydzielenia serwisu.

**Komunikacja między modułami:** wywołania w procesie (in-process) — przez publiczne interfejsy modułów albo wewnętrzne zdarzenia/mediator. Szybkie, transakcyjne, debugowalne F11 w debuggerze.

W .NET granice da się **egzekwować**, nie tylko deklarować: osobne projekty z `internal` jako domyślną widocznością, testy architektury (NetArchTest/ArchUnitNET: "moduł Rezerwacje nie referencuje wnętrza modułu Rozliczenia"), osobne `DbContext` per moduł.

**Dlaczego naiwny monolit się psuje** (i czemu dostał złą prasę): bez wymuszonych granic każdy moduł sięga wszędzie, po 5 latach wszystko zależy od wszystkiego — to **big ball of mud** (wielka kula błota), gdzie zmiana czegokolwiek psuje cokolwiek. Ale uwaga: to wina **braku granic**, nie monolitu. Modular monolith to monolit, który wziął z mikroserwisów dyscyplinę granic, a zostawił im koszty sieci.

**Trade-offy:** + prostota operacyjna (jeden deployment, jeden monitoring, brak sieci między modułami), + transakcje ACID przez cały przepływ za darmo, + tani refactoring przez granice (przesunięcie odpowiedzialności między modułami to PR, nie projekt migracyjny); − skalowanie tylko całością (nie da się skalować samego modułu grafiku na poranny szczyt — skalujesz wszystko), − jedna awaria pamięci/CPU kładzie całość, − jeden stack technologiczny, − przy wielu zespołach rośnie tarcie o wspólny pipeline i wspólne wydania.

### Styl 2 — Microservices (mikroserwisy)

**Czym jest:** system pocięty na **wiele małych, niezależnie deployowalnych serwisów**, z których każdy realizuje jeden obszar biznesowy (w dobrym wydaniu: jeden bounded context), ma **własną bazę danych** (zasada *database per service* — nikt nie czyta cudzych tabel) i może być rozwijany, wdrażany i skalowany **niezależnie** przez osobny zespół.

**Deployment:** każdy serwis to osobny artefakt i pipeline; w praktyce kontenery + orkiestrator (Kubernetes/Azure Container Apps), do tego load balancery, service discovery, centralne logowanie.

**Komunikacja:** przez sieć — synchronicznie (HTTP/gRPC: proste, ale wiąże dostępności serwisów ze sobą) lub asynchronicznie (broker komunikatów: luźniejsze wiązanie, ale złożoność — patrz styl 3). Tu wchodzi cała fizyka systemów rozproszonych: sieć zawodzi, timeouty, retry, idempotencja (odporność operacji na powtórzenie — pełne rozwinięcie w module 2).

**Po co w ogóle:** głównym uzasadnieniem NIE jest wydajność, tylko **niezależność zespołów i wdrożeń** (50 developerów w jednym monolicie depcze sobie po piętach; 8 zespołów × własny serwis wdraża bez koordynacji) oraz **niezależne skalowanie i izolacja awarii** (moduł raportów może umrzeć, bramki dalej wpuszczają).

**Koszty, o których nikt nie mówi na konferencjach:**

1. **Podatek operacyjny:** każdy serwis to pipeline, monitoring, alerty, certyfikaty, sekrety, wersjonowanie. 20 serwisów = 20× ta lista + infrastruktura wspólna (orkiestrator, service mesh, tracing). Potrzebujesz dojrzałego DevOps **zanim** zaczniesz, nie potem.
2. **Koniec transakcji:** rezerwacja + płatność + powiadomienie w trzech serwisach = brak wspólnej transakcji ACID. Wchodzą sagi, kompensacje, eventual consistency — każda z tych rzeczy to tygodnie pracy i nowe klasy błędów (moduł 2 jest w połowie o tym).
3. **Debugowanie i obserwowalność:** stack trace zamienia się w wędrówkę po logach pięciu serwisów; bez distributed tracing (lekcja 01) jesteś ślepy. "Gdzie zginęła wiadomość?" — znasz to pytanie i wiesz, ile kosztuje odpowiedź.
4. **Wersjonowanie kontraktów:** zmiana API serwisu wymaga koordynacji z każdym konsumentem; "niezależne wdrożenia" są niezależne tylko przy zdyscyplinowanej kompatybilności wstecznej.
5. **Koszt pomyłki w granicach:** jeśli potniesz źle (a na początku projektu wiesz o domenie najmniej!), przesunięcie odpowiedzialności między serwisami to migracja danych, zmiana kontraktów i tygodnie pracy — versus jeden PR w monolicie. Dlatego mikroserwisy wymagają **dojrzałej znajomości domeny**.

**Distributed monolith — antywzorzec:** najgorszy możliwy wynik — system pocięty na serwisy, które **nie są niezależne**: serwis A synchronicznie woła B, B woła C, wszystkie trzeba wdrażać razem (lockstep), wspólna baza albo łańcuch wywołań tak ciasny, że awaria jednego kładzie wszystkie. Dostajesz **wszystkie koszty rozproszenia (sieć, debugging, operacje) i żadnej korzyści (ani niezależnych wdrożeń, ani izolacji awarii)**. Powstaje zwykle z cięcia po warstwach technicznych zamiast po bounded contexts, albo z "mikroserwisów, bo CV/konferencja". Test diagnostyczny: czy możesz wdrożyć jeden serwis w poniedziałek, nie dotykając pozostałych i niczego nie psując? Nie? Masz distributed monolith.

### Styl 3 — Event-driven architecture (architektura zdarzeniowa)

**Czym jest:** styl, w którym komponenty komunikują się przez **zdarzenia** (ang. *event* — komunikat o fakcie, który już zaszedł: `ZajeciaOdwolane`, `UmowaAktywowana`), publikowane do **brokera komunikatów** (RabbitMQ, Azure Service Bus, Kafka). Nadawca **nie wie i nie dba**, kto słucha; konsumenci subskrybują i reagują niezależnie. Porównaj z poleceniem (*command*): "wyślij e-mail" to rozkaz do konkretnego odbiorcy; "zajęcia odwołane" to fakt — co z nim zrobią inni, to ich sprawa. Ta zmiana kierunku zależności jest sercem stylu.

**Deployment/komunikacja:** EDA to styl **komunikacji**, ortogonalny do poprzednich — zdarzeniowo mogą rozmawiać moduły wewnątrz monolitu (in-process events), a mikroserwisy mogą gadać synchronicznym HTTP. W praktyce: producent + broker + niezależni konsumenci (worker services), często jako uzupełnienie stylu bazowego, nie zamiast niego.

**Dlaczego bywa świetne:** odwołanie zajęć → dziś e-mail; za pół roku dochodzi push i zwolnienie miejsca z listy rezerwowej — **dokładamy konsumentów, nie dotykając producenta**. Plus naturalna amortyzacja szczytów (kolejka buforuje) i izolacja awarii (dostawca e-maili leży → zdarzenia czekają, reszta systemu żyje).

**Trade-offy i blizny:** − przepływ logiki przestaje być widoczny w kodzie ("co się dzieje po rezerwacji?" — nie odpowie żaden plik, tylko mapa subskrypcji), − eventual consistency z definicji (UI może nie widzieć skutku od razu), − duplikaty i kolejność komunikatów (broker zwykle gwarantuje *at-least-once* — "co najmniej raz" — więc konsument MUSI być idempotentny), − dead-letter queue wymaga procesu i właściciela, bo nieoglądana zamienia się w cmentarz utraconych operacji (lekcja 02), − debugowanie wymaga korelacji (correlation id przez cały przepływ). Wiesz to wszystko z produkcji — tu tylko dostaje nazwy: at-least-once, idempotent consumer, DLQ.

### Ścieżka ewolucji: monolit → moduły → serwisy

Najważniejsza praktyczna rada lekcji: **styl to nie ślub na całe życie, to etap** — pod warunkiem, że pilnujesz granic od początku.

1. **Etap 1 — monolit (modularny od pierwszego dnia):** zaczynasz z jedną deployowalną jednostką, ale granice modułów = bounded contexts, osobne schematy bazy, komunikacja przez interfejsy/zdarzenia in-process. Koszt dyscypliny: mały. Wartość: opcja na przyszłość.
2. **Etap 2 — monolit z wyniesionymi workerami:** rzeczy asynchroniczne (powiadomienia, raporty, integracje) wychodzą za broker do osobnych workerów. Nadal jeden "rdzeń", ale szczyty i awarie integracji przestają dotykać core'u. Wiele systemów powinno się tu **zatrzymać na zawsze**.
3. **Etap 3 — wydzielanie serwisów tam, gdzie boli:** gdy KONKRETNY moduł ma inne potrzeby skalowania (grafik w poranny szczyt), inne tempo zmian albo dedykowany zespół — wycinasz go wzdłuż istniejącej granicy (wzorzec **strangler fig** z lekcji 02: stopniowo, nie big-bang). Zdarzenia, którymi moduł już rozmawiał in-process, zamieniasz na zdarzenia przez broker — kontrakt często zostaje.

Ewolucja w tę stronę jest tania, **jeśli granice istniały**. W odwrotną stronę (scalanie źle pociętych mikroserwisów) — koszmarna. Stąd reguła praktyczna (zgodna z "monolith first" Fowlera): **domyślną odpowiedzią jest modularny monolit; mikroserwisy trzeba uzasadnić** konkretną potrzebą (skala zespołów, niezależność wdrożeń, izolacja), a nie modą.

### Macierz decyzyjna

| Kryterium | Modular monolith | Microservices | Event-driven (jako warstwa komunikacji) |
|---|---|---|---|
| Wielkość zespołu | 1–2 zespoły: **idealny** | uzasadniony od ~3+ zespołów z własnymi obszarami | neutralne; wymaga kompetencji messagingowych |
| Znajomość domeny | niska/średnia: bezpieczny (granice tanio poprawisz) | wymaga dojrzałej (złe cięcie = bardzo drogie) | neutralne |
| Niezależne wdrożenia | − wszyscy wydają razem | ++ główna korzyść | + konsumenci niezależni |
| Skalowanie wybiórcze | − tylko całość | ++ per serwis | + per konsument; kolejka amortyzuje szczyty |
| Spójność danych | ++ ACID w procesie | −− sagi, eventual consistency | −− eventual consistency z definicji |
| Debugowanie / obserwowalność | ++ jeden proces, stack trace | −− tracing rozproszony obowiązkowy | − korelacja zdarzeń, "niewidzialny" przepływ |
| Koszt operacyjny (DevOps) | + niski | −− wysoki, płatny z góry | − broker + DLQ + monitoring |
| Izolacja awarii | − ograniczona | ++ | ++ dla przepływów async |
| Rozszerzalność o nowych odbiorców procesu | 0 | 0/+ | ++ nowy konsument bez zmian u producenta |
| Odwracalność wyboru | ++ (przy granicach: droga do serwisów otwarta) | − one-way door | + (zdarzenia in-process → broker to mała zmiana) |

Jak czytać: nie sumuj kolumn (lekcja 02!) — znajdź swoje 2–3 krytyczne wiersze (z atrybutów jakościowych, lekcja 01) i patrz na dyskwalifikatory. Przykłady: system dla sieci siłowni, wiele zespołów, bramka wymaga izolacji od reszty → mikroserwisy/wydzielone serwisy mają uzasadnienie przynajmniej dla kontroli dostępu. DayChunks, jedna osoba, zero ops-budżetu → modularny monolit (a w wariancie bez backendu — po prostu dobrze zmodularyzowana aplikacja lokalna); każdy inny wybór byłby autosabotażem.

### Kiedy NIE… (pytania kontrolne stylu)

- NIE mikroserwisy, gdy: zespół < ~10 osób, domena słabo poznana, brak dojrzałego CI/CD i monitoringu, głównym argumentem jest "skalowalność", której nikt nie policzył, albo CV-driven development.
- NIE event-driven, gdy: przepływ jest z natury synchroniczny i użytkownik czeka na wynik (rezerwacja MUSI od razu powiedzieć "masz miejsce/nie masz"), zespół nie ma nawyków idempotencji i monitorowania DLQ, albo prosty request-response załatwia sprawę.
- NIE "modular monolith" jako wymówka, gdy granic nikt nie egzekwuje — bez testów architektury i dyscypliny code review to tylko big ball of mud z lepszym PR-em.

## Praktyka

- [ ] Opisz system, który znasz najlepiej (Perfect Gym lub Aerotunel), językiem tej lekcji: jaki to był styl (czysty? hybryda?), gdzie przebiegały granice deploymentu, która komunikacja była sync, a która async. Czy miał cechy distributed monolith? Po czym to poznajesz (test z poniedziałkowym deployem)?
- [ ] **Ćwiczenie główne:** dla systemu "platforma dla sieci siłowni" z twojej mapy kontekstów (lekcja 05) przeprowadź pisemną analizę stylu wg frameworku z lekcji 02: kryteria z atrybutów (lekcja 01) → 3 opcje (modular monolith / monolit + workery / mikroserwisy per kontekst) → macierz → rekomendacja z rezygnacjami i warunkami rewizji ("wydzielamy kontrolę dostępu, jeśli…").
- [ ] Napisz ADR "Styl architektoniczny DayChunks" (do `docs/adr/`): rekomendacja stylu + uzasadnienie + co najmniej 2 negatywne konsekwencje. Powiąż z ADR-em "bez backendu" z lekcji 04.
- [ ] Naszkicuj plan ewolucji (3 etapy) dla systemu rezerwacji tunelu: co jest monolitem dziś, co wychodzi za broker w etapie 2, co i POD JAKIM WARUNKIEM staje się serwisem w etapie 3.
- [ ] Pytanie kontrolne (pisemnie, 5 zdań): dlaczego distributed monolith jest gorszy zarówno od monolitu, jak i od mikroserwisów — i jaki pojedynczy błąd projektowy najczęściej do niego prowadzi?

## Artefakt

Plik `analiza-stylu-gym-platform.md` (pełna analiza z macierzą i rekomendacją) + ADR o stylu DayChunks w `docs/adr/` + szkic planu ewolucji. Razem z mapą kontekstów i C4 masz komplet materiałów, z którego łatwo wyjdzie post LinkedIn typu "Dlaczego NIE wybrałem mikroserwisów" — zanotuj szkic, publikacja wedle kalendarza postów.

## Definition of Done

- [ ] Umiesz opisać każdy z trzech stylów w 2 minuty: jednostka deploymentu, sposób komunikacji, dla kogo, główny koszt.
- [ ] Umiesz wymienić z pamięci min. 4 ukryte koszty mikroserwisów i objaśnić distributed monolith wraz z testem diagnostycznym.
- [ ] Umiesz opowiedzieć ścieżkę ewolucji monolit → moduły → serwisy i wskazać, co musi być prawdą od etapu 1, żeby etap 3 był tani.
- [ ] Twoja analiza stylu ma nazwane rezygnacje i warunki rewizji; ADR DayChunks ma negatywne konsekwencje.
- [ ] Test modułu działa na tej lekcji: na pytanie "monolit czy mikroserwisy dla X?" odpowiadasz "to zależy od…" i w 3 minuty dowozisz kryteria, opcje i rekomendację.

## Materiały

1. **"Fundamentals of Software Architecture"** (Richards, Ford) — rozdziały o stylach (część II): zwłaszcza microservices, event-driven i (w 2. wydaniu) modular monolith; czytaj z macierzą z tej lekcji obok i porównuj oceny.
2. **Martin Fowler, "MonolithFirst"** + **"Microservice Prerequisites"** (martinfowler.com) — dwa krótkie, klasyczne teksty o tym, kiedy mikroserwisy mają sens; razem ~20 minut lektury.
