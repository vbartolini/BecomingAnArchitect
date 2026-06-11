# Etap 01 — Koncept i wymagania: definicja produktu

## Cel lekcji

Po tym etapie masz spisaną definicję produktu capstone: domenę, wymagania funkcjonalne, wybrane atrybuty jakościowe i — co najważniejsze — **listę rzeczy, których świadomie NIE budujesz, z uzasadnieniem**. Powstaje dokument wymagań i pierwszy ADR projektu.

## Dlaczego to ważne

Najczęstszy sposób, w jaki umierają projekty portfolio: ktoś siada do kodu pierwszego dnia, po dwóch tygodniach ma pół systemu i zero dokumentacji, po czterech — porzucone repo. Drugi najczęstszy: projekt rośnie bez granic ("dodam jeszcze multi-tenancy... i prawdziwego Stripe'a...") i nigdy nie osiąga stanu "publikowalne".

Definicja zakresu to pierwsza decyzja architektoniczna projektu — i pierwsza rzecz, o którą zapyta dobry rozmówca na interview: *"dlaczego akurat to, a nie tamto?"*. Odpowiedź "nie zdążyłem" jest słaba. Odpowiedź "świadomie wyciąłem, bo nie demonstruje żadnej kompetencji, której nie demonstruje już X — oto ADR" — to jest język architekta. Sekcja "Trade-offs / czego tu nie ma" zaczyna się właśnie tutaj, nie na końcu projektu.

## Teoria

### Domena: rezerwacje lotów w tunelu aerodynamicznym

Wybieramy domenę, którą znasz z produkcji (Aerotunel) — celowo. Nie tracisz czasu na wymyślanie reguł biznesowych, a na rozmowie każdą decyzję możesz podeprzeć realną historią. Krótki słownik domeny (w repo trafi do `docs/domain.md`):

- **Slot** — okno czasowe lotu w komorze tunelu (np. 15 minut). Zasób ograniczony: w danym slocie lata jedna osoba (lub mała grupa) z jednym instruktorem.
- **Instruktor** — zasób ograniczony numer dwa. Slot bez dostępnego instruktora nie jest rezerwowalny.
- **Rezerwacja (booking)** — klient wybiera slot, system go **blokuje na czas opłacenia** (hold), po opłaceniu rezerwacja jest potwierdzona.
- **Płatność** — symulowana bramka: może się powieść, odmówić albo nie odpowiedzieć w czasie (timeout). Te trzy wyniki wystarczą, żeby zademonstrować sagę z kompensacją.
- **System zewnętrzny ("TunnelOps")** — symulowany system operacyjny tunelu (grafik obsługi, harmonogram dmuchaw). Po potwierdzeniu rezerwacji trzeba mu ją zgłosić; bywa niedostępny i bywa, że odpowiada śmieciami — idealny pretekst do retry + DLQ.

Dlaczego ta domena jest dobra dydaktycznie? Bo ma **naturalny konflikt o zasób ograniczony** (dwóch klientów chce ten sam slot) i **proces wieloetapowy z możliwością porażki w środku** (rezerwacja → płatność → potwierdzenie). To dokładnie warunki, w których wzorce z modułu 2 przestają być akademickie.

### Wymagania funkcjonalne (FR)

Spisz je w repo własnymi słowami; poniżej zestaw bazowy. Każde wymaganie ma istnieć **po coś** — w nawiasach kompetencja, którą demonstruje:

1. **FR-1 Przeglądanie dostępności** — klient widzi wolne sloty w wybranym dniu (proste query, tło dla reszty).
2. **FR-2 Rezerwacja z blokadą slotu** — utworzenie rezerwacji blokuje slot na N minut (hold z TTL). Dwóch klientów nie może zablokować tego samego slotu (kontrola współbieżności, unique constraint — lekcja 01 modułu 2 w wersji biznesowej).
3. **FR-3 Płatność symulowana** — po blokadzie klient "płaci"; bramka odpowiada: sukces / odmowa / timeout. Sukces potwierdza rezerwację. Odmowa lub brak płatności w oknie hold **zwalnia slot** (kompensacja — serce sagi).
4. **FR-4 Powiadomienia** — potwierdzenie i anulowanie generują powiadomienie (symulowane: log/zapis do tabeli wystarczy; liczy się przepływ przez zdarzenia, nie SMTP).
5. **FR-5 Integracja z TunnelOps** — potwierdzona rezerwacja jest zgłaszana do symulowanego systemu zewnętrznego, który bywa niedostępny (retry) i odsyła czasem nieprzetwarzalne odpowiedzi (poison message → DLQ).
6. **FR-6 Anulowanie rezerwacji** — klient anuluje potwierdzoną rezerwację; slot wraca do puli, TunnelOps dostaje korektę (drugi przepływ przez te same wzorce — pokazuje, że architektura się skaluje na nowe przypadki).

Tyle. Sześć wymagań. Jeśli czujesz pokusę dodania siódmego — najpierw odpowiedz pisemnie: *jaką kompetencję demonstruje, której nie demonstruje żadne z powyższych?*

Mapowanie wymagań na kompetencje (ta tabela trafi też do dokumentu wymagań — recenzent zobaczy, że nic tu nie jest przypadkowe):

| FR | Główny wzorzec / kompetencja | Moduł kursu |
|---|---|---|
| FR-1 | proste query, tło dla reszty | — |
| FR-2 | kontrola współbieżności (unique constraint), outbox | 2 (lekcje 01–02) |
| FR-3 | saga z kompensacją, timeout jako zdarzenie | 2 (lekcja 03) |
| FR-4 | komunikacja przez zdarzenia, idempotent consumer | 2 (lekcja 01) |
| FR-5 | retry, poison message, DLQ | 2 (lekcje 05–06) |
| FR-6 | dowód, że architektura przyjmuje nowe przypadki tanio | 1 (style, granice modułów) |

Każde wymaganie spisuj z **kryterium akceptacji** (acceptance criterion — sprawdzalny warunek "skończone"). Przykład zapisu FR-2:

> **FR-2 Rezerwacja z blokadą slotu.** Klient tworzy rezerwację na wolny slot; slot zostaje zablokowany na 15 minut.
> *Kryteria akceptacji:* (a) dwa równoległe żądania na ten sam slot → dokładnie jeden hold, drugie żądanie dostaje 409; (b) hold nieopłacony przez 15 minut wygasa i slot wraca do puli; (c) utworzenie holdu publikuje zdarzenie `BookingCreated`.

### Wymagania niefunkcjonalne: które atrybuty wybieramy

Z modułu 1 wiesz, że atrybutów jakościowych nie da się mieć wszystkich naraz — wybór i rezygnacja to decyzja. Dla capstone:

**Wybieramy (i będziemy mierzalnie demonstrować):**
- **Reliability / fault tolerance** — system przeżywa duplikaty, timeouty, awarie systemu zewnętrznego i restart workera w środku procesu. To główny "produkt" repo.
- **Observability** — każdy przepływ ma distributed trace end-to-end; awarie widać w dashboardzie, nie w `Console.WriteLine`.
- **Deployability** — całość wdrażalna jedną komendą z IaC; lokalnie startuje w 5 minut.
- **Cost-awareness** — środowisko demo policzone i zoptymalizowane (scale-to-zero); koszt jest atrybutem architektury (moduł 3, lekcja 06).

**Świadomie rezygnujemy (zapisz to w dokumencie wymagań!):**
- **Scalability ponad demo** — nie projektujemy pod 10k rezerwacji/s. Wzorce, które stosujemy, skalują się koncepcyjnie — i to wystarczy; udowadnianie tego benchmarkami to inny projekt.
- **High availability multi-region** — jedno środowisko, jeden region. DR opisujemy w trade-offach jako "co bym zrobił, gdyby" — nie wdrażamy.
- **Security ponad rozsądne minimum** — managed identities i brak sekretów w repo: tak. Pełny threat modeling, WAF, pen-testy: nie.

### NO-features: cięcia zakresu jako decyzje architektoniczne

To jest sekcja, którą recenzent repo zapamięta. Każde cięcie zapisz z uzasadnieniem — wzorzec zapisu: *czego nie ma → dlaczego nie → co by się zmieniło, gdyby było potrzebne*.

- **Bez prawdziwych płatności.** Integracja ze Stripe/PayU demonstruje czytanie cudzej dokumentacji, nie architekturę. Symulowana bramka daje pełną kontrolę nad scenariuszami awaryjnymi (timeout na żądanie!), których u prawdziwego dostawcy nie wywołasz deterministycznie. Gdyby było potrzebne: bramka jest za interfejsem, podmiana to jedna implementacja + sekrety w Key Vault.
- **Bez multi-tenancy.** Jeden tunel, jeden operator. Multi-tenant dotyka partycjonowania danych, izolacji i rozliczeń — osobny, duży temat, który rozmyłby przekaz repo. Gdyby było potrzebne: TenantId w modelu + strategia izolacji (ADR do napisania wtedy).
- **Bez UI ponad minimum.** REST API + plik `.http` / Swagger + ewentualnie jedna strona statusu. Recenzent-architekt ocenia przepływy i decyzje, nie CSS. Frontend to inny zawód i inne portfolio.
- **Bez CQRS z osobnym read store, bez event sourcingu.** Kuszące "do kompletu", ale: zwiększają złożoność o rząd wielkości, a żadne wymaganie ich nie uzasadnia. Wpisanie tego do trade-offów ("rozważyłem i odrzuciłem, bo...") jest cenniejsze niż wdrożenie.
- **Bez mikroserwisów jako osobnych repo/deploymentów per moduł.** Decyzja zapadnie formalnie w etapie 02 — tu tylko zaznaczamy: liczba wdrażalnych jednostek to decyzja, nie odruch.

Zauważ schemat: **brak czegoś + uzasadnienie + ścieżka rozbudowy = dojrzałość architektoniczna**. Brak czegoś bez słowa = niedoróbka. Różnica jest wyłącznie w dokumentacji.

### ADR-001: zakres i nie-zakres projektu

Pierwszy ADR projektu nie dotyczy technologii — dotyczy **granic**. Format znasz z modułu 1 (lekcja 04): Kontekst → Decyzja → Konsekwencje. W kontekście: cel repo (dowód kompetencji, nie produkt komercyjny) i budżet czasowy (4–6 tygodni). W decyzji: lista FR i lista NO-features. W konsekwencjach: co zyskujemy (publikowalność w terminie, czytelny przekaz) i co tracimy (repo nie jest "produkcyjnym" systemem i nie udaje, że jest).

Ten ADR będzie potem linkowany z sekcji "Trade-offs" w README — piszesz go raz, procentuje wszędzie.

## Praktyka

- [ ] Zacznij pracę w folderze-wizytówce `windtunnel-booking/` (już istnieje w repo nauki, ze stubem README; nazwa robocza może się zmienić przed publikacją — etap 05). Struktura startowa: `docs/`, `docs/adr/`, `README.md` z jednym akapitem "work in progress". Wszystko w tym folderze — łącznie z commitami — po angielsku.
- [ ] Napisz `docs/domain.md` — słownik domeny (slot, hold, rezerwacja, instruktor, TunnelOps) + 2–3 reguły biznesowe (np. "hold wygasa po 15 minutach", "slot wymaga przypisanego instruktora").
- [ ] Napisz `docs/requirements.md` — wymagania FR-1…FR-6 (własnymi słowami, możesz modyfikować) + sekcja atrybutów jakościowych: 4 wybrane, 3 odrzucone, każde z jednym zdaniem uzasadnienia.
- [ ] Napisz sekcję NO-features w `docs/requirements.md` — minimum 4 pozycje w formacie: czego nie ma / dlaczego / ścieżka rozbudowy.
- [ ] Napisz `docs/adr/ADR-001-project-scope.md` — po angielsku, jak cała dokumentacja tego repo (polityka językowa kursu: capstone to artefakt publiczny).
- [ ] Test kontrolny: przeczytaj wymagania i odpowiedz na pytanie "czy potrafię wskazać, który wzorzec z modułu 2 demonstruje każde FR?". Jeśli jakieś FR nie demonstruje niczego — wytnij je albo uzasadnij inaczej.

## Artefakt

W repo capstone istnieją i są scommitowane:
1. `docs/domain.md` — słownik domeny,
2. `docs/requirements.md` — FR + atrybuty jakościowe + NO-features,
3. `docs/adr/ADR-001-zakres-projektu.md` — pierwszy ADR.

## Definition of Done

- [ ] Wszystkie trzy dokumenty są w repo i przeszłyby test "obcego czytelnika": osoba spoza projektu zrozumie, co system robi i czego celowo nie robi.
- [ ] Każde NO-feature ma uzasadnienie i ścieżkę rozbudowy — żadne nie brzmi jak wymówka.
- [ ] Decyzja o języku dokumentacji repo (polski/angielski) podjęta i zapisana.
- [ ] Umiesz w 60 sekund opowiedzieć, co budujesz i czego nie budujesz — bez zaglądania do notatek.

## Materiały

1. **Moduł 1, lekcja 01 — Atrybuty jakościowe** (`kurs/01-fundamenty-myslenia-architektonicznego/01-atrybuty-jakosciowe/`) — tu wracasz po metodę wyboru i nazywania atrybutów.
2. **Moduł 1, lekcja 04 — ADR** (`kurs/01-fundamenty-myslenia-architektonicznego/04-adr-architecture-decision-records/`) — format ADR, którego używasz w całym capstone.
