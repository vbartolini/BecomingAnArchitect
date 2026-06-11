# Capstone — projekt referencyjny

**Czas trwania:** 4–6 tygodni
**Pozycja w kursie:** po modułach 2 (systemy rozproszone) i 3 (Azure). Moduły 4 (nowoczesny stack .NET) i 6 (komunikacja architektoniczna) wspierają etapy 3 i 5, ale nie blokują startu.

## Cel modułu

Jeden projekt-wizytówka (folder `windtunnel-booking/`), który jest **dowodem wszystkich kompetencji z tego kursu naraz**: myślenia trade-offami (moduł 1), wzorców systemów rozproszonych (moduł 2), wdrożenia cloud-native na Azure (moduł 3) i umiejętności komunikowania architektury (moduł 6). Link do niego trafia do sekcji Featured na LinkedIn i staje się główną osią rozmów rekrutacyjnych: zamiast opowiadać "robiłem kiedyś coś podobnego", otwierasz link i pokazujesz.

Koncept: **event-driven system rezerwacji lotów w tunelu aerodynamicznym** — celowo blisko Twojego doświadczenia z Aerotunel. Na rozmowie to podwójna amunicja: kod w repo plus prawdziwe historie z produkcji o tej samej domenie. Booking ze slotami czasowymi, płatności (symulowane), powiadomienia, integracja z symulowanym systemem zewnętrznym sterowania tunelem.

Najważniejsza zmiana perspektywy względem mini-projektu z modułu 2: tam celem było **nauczyć się wzorców**. Tutaj celem jest **przekonać obcego człowieka w 5 minut, że umiesz myśleć jak architekt**. To inna optymalizacja — dokumentacja, ADR-y i sekcja "Trade-offs" są tu równie ważne jak kod.

**Język: capstone powstaje w całości po angielsku** — README, ADR-y, diagramy, komentarze w kodzie, komunikaty commitów. To artefakt publiczny (polityka językowa kursu), a jego recenzentem może być ktoś spoza Polski. Bonus: przepisując decyzje z polskich notatek warsztatu na angielskie ADR-y, ćwiczysz dokładnie ten język, w którym będziesz ich bronił na rozmowach.

## Etapy (przerabiaj po kolei)

| Etap | Temat | Artefakt |
|---|---|---|
| [01](01-koncept-i-wymagania/) | Koncept i wymagania — definicja produktu i świadome cięcia zakresu | dokument wymagań + ADR-001 "zakres i nie-zakres" |
| [02](02-architektura-i-backlog/) | Architektura i backlog — projekt przed kodem | diagramy C4 (L1–L2) + 3–5 ADR-ów + backlog na epiki |
| [03](03-implementacja-wzorcow/) | Implementacja wzorców — przepływ end-to-end (2–3 tygodnie) | działający system: outbox, idempotency, saga, DLQ, testy |
| [04](04-wdrozenie-i-observability/) | Wdrożenie i observability — Azure z Bicep | działające środowisko demo + tracing + sekcja kosztów |
| [05](05-dokumentacja-i-publikacja/) | Dokumentacja i publikacja — repo jako wizytówka | README "rekruterskie" + Featured + post (+ opcjonalnie artykuł) |

## Harmonogram

Wariant bazowy — **6 tygodni** (rytm z modułu 0: ~8h głębokiej pracy tygodniowo):

| Tydzień | Etapy | Akcent |
|---|---|---|
| 1 | 01 + 02 | Wymagania, cięcia zakresu, C4, ADR-y, backlog. Zero kodu produkcyjnego — i to jest celowe. |
| 2 | 03 (E1+E2) | Szkielet repo + CI, moduł booking z outboxem. |
| 3 | 03 (E3) | Płatność symulowana + saga z kompensacją. Najtrudniejszy tydzień. |
| 4 | 03 (E4) | Powiadomienia, integracja zewnętrzna, retry + DLQ, testy scenariuszy awaryjnych. |
| 5 | 04 | Bicep, wdrożenie na Azure, tracing end-to-end, dashboard, koszty. |
| 6 | 05 | README, porządki, publikacja: Featured + post (+ artykuł). |

Wariant skrócony — **4 tygodnie**: etapy 01+02 w 3–4 dni (masz już wprawę z modułów 1–3), etap 03 w 2 tygodnie z cięciem zakresu opisanym w lekcji 03 (poziom "minimum demonstrowalne"), etapy 04+05 łącznie w tydzień. Zasada cięcia: **tnij liczbę funkcji, nigdy jakość wzorców i dokumentacji** — repo z dwoma przepływami zrobionymi wzorcowo bije repo z pięcioma zrobionymi byle jak.

Jeśli coś się sypie czasowo: etap 05 jest **nienegocjowalny**. Nieopublikowane repo nie istnieje — tak samo jak architektura, która jest tylko w czyjejś głowie.

## Definition of Done całego capstone — test 5 minut

Wyobraź sobie recenzenta: hiring manager albo architekt robiący screening przed rozmową. Ma 5 minut i Twój link. Capstone jest zaliczony, gdy w te 5 minut zobaczy **wszystko** poniższe:

**Pierwsze 30 sekund (góra README):**
- [ ] Elevator pitch: 2–3 zdania — co to jest, jakie wzorce demonstruje, na czym działa.
- [ ] Diagram C4 (poziom 2) widoczny bez scrollowania w nieskończoność.
- [ ] Badge CI — zielony build na głównym branchu.

**Minuty 1–3 (scrollowanie README):**
- [ ] Sekcja **"Architecture decisions"** linkująca ADR-y w repo (minimum 5, w jednolitym formacie z modułu 1).
- [ ] Sekcja **"Trade-offs — czego tu świadomie NIE ma i dlaczego"** — Twój znak firmowy. To ona odróżnia to repo od tysięcy todo-listów z mikroserwisami.
- [ ] Sekcja "How to run locally" — obietnica uruchomienia w ~5 minut (`docker compose up` + `dotnet run`), która jest prawdą.
- [ ] Sekcja kosztów Azure — miesięczny koszt środowiska demo i jak go zminimalizowano.

**Minuty 3–5 (wejście w kod):**
- [ ] Struktura katalogów mówi sama za siebie: moduły domenowe, `infra/` z Bicep, `docs/adr/`, testy.
- [ ] Wzorce z modułu 2 widoczne w kodzie: outbox, idempotent consumer, saga z kompensacją, obsługa DLQ — z testami, które je udowadniają (w tym scenariusze awaryjne: duplikat, timeout, poison message, restart workera).
- [ ] Historia commitów czytelna (sensowne komunikaty po angielsku, bez "fix", "wip", "asdf"), zero sekretów w historii, licencja w repo.

**Poza repo:**
- [ ] Wizytówka (link do folderu capstone) wisi w Featured na LinkedIn.
- [ ] Post podsumowujący opublikowany.
- [ ] Umiesz opowiedzieć o projekcie w 5 minut bez patrzenia w notatki: problem → decyzje → trade-offy → co bym zrobił inaczej.

## Artefakty modułu

- [ ] **Folder-wizytówka capstone (`windtunnel-booking/`)** — w całości po angielsku, podlinkowany w Featured na LinkedIn. Zastępuje lub uzupełnia wizytówkę messaging-patterns z modułu 2 jako główny dowód kompetencji.
- [ ] **Post podsumowujący na LinkedIn** — struktura rozpisana w etapie 05.
- [ ] Opcjonalnie: **artykuł techniczny** (dev.to / blog) o jednym wątku z projektu — np. saga z kompensacją w praktyce albo "czego świadomie nie zbudowałem".
- [ ] Komplet ADR-ów i diagramów C4 w repo — używalne potem jako materiał na rozmowy (moduł 6).

## Materiały przekrojowe modułu

Capstone nie wprowadza nowej teorii — spina istniejącą. Główne odniesienia to Twoje własne moduły:

1. **Moduł 2 — Systemy rozproszone** (`kurs/02-systemy-rozproszone/`) — wzorce, które tu implementujesz: outbox, idempotency, saga, DLQ, observability.
2. **Moduł 3 — Cloud-native na Azure** (`kurs/03-cloud-native-azure/`) — macierze decyzyjne, Bicep, koszty; capstone powtarza ten przepływ na nowym systemie, już bez taryfy ulgowej.

Po zaliczeniu: odhacz Capstone w `architect-track-plan.md`, zaktualizuj `progress.md` — i zacznij używać tego repo tam, gdzie miało trafić: w aplikacjach i na rozmowach.
