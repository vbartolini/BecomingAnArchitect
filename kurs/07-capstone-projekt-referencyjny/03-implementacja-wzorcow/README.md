# Etap 03 — Implementacja wzorców: przepływ end-to-end

## Cel lekcji

Po tym etapie (2–3 tygodnie) masz działający lokalnie system: rezerwacja → płatność → potwierdzenie → integracja zewnętrzna, z kompletem wzorców z modułu 2 w użyciu — outbox, idempotent consumers, saga z kompensacją, retry + DLQ — oraz testami (w tym Testcontainers), które **udowadniają** odporność na duplikaty, timeouty, poison messages i restart workera.

## Dlaczego to ważne

To jest serce capstone — i moment, w którym wiedza z modułu 2 przestaje być "przerobiłem lekcję", a staje się "mam to w publicznym repo z testami". Różnica dla recenzenta jest fundamentalna: każdy może napisać w CV "znam outbox pattern"; mało kto może pokazać test, który zabija proces w środku sagi i asercją sprawdza, że slot został zwolniony.

Druga rzecz: ten etap trenuje dyscyplinę zakresu. 2–3 tygodnie to mało. Pokusa "jeszcze tylko dopieszczę abstrakcję" zabija projekty — dlatego etap ma tygodniowe punkty kontrolne. Jeśli punkt kontrolny nie jest zielony, tniesz zakres według reguł poniżej, a nie przedłużasz tydzień.

## Teoria

Wzorce znasz z modułu 2 — tu mapujemy je na konkretne miejsca w systemie i wskazujemy pułapki specyficzne dla tej domeny.

### Przepływ end-to-end (mapa terenu)

```
Klient → [Booking API] POST /bookings
           └─ TX: rezerwacja(Pending) + blokada slotu (unique constraint) + outbox(BookingCreated)
[Worker/outbox publisher] → Service Bus: BookingCreated
[Worker/saga] start sagi → żądanie płatności (symulowana bramka)
   ├─ sukces  → TX: rezerwacja(Confirmed) + outbox(BookingConfirmed)
   ├─ odmowa  → KOMPENSACJA: TX: rezerwacja(Cancelled) + zwolnienie slotu + outbox(BookingCancelled)
   └─ timeout → retry wg polityki → po wyczerpaniu: kompensacja jak wyżej
[Worker/notifications] BookingConfirmed/Cancelled → zapis powiadomienia (idempotentnie)
[Worker/tunnelops]     BookingConfirmed → HTTP do TunnelOps (retry; poison → DLQ)
```

### Outbox przy publikacji zdarzeń bookingu

Wzorzec z modułu 2, lekcja 02. Tu kluczowy punkt: **blokada slotu i wpis do outboxa muszą być w jednej transakcji** — to dokładnie problem dual write. Pułapka domenowa: hold ma TTL; wygaśnięcie holdu też jest zmianą stanu i też publikuje zdarzenie przez outbox (procesem tła skanującym wygasłe holdy), inaczej wygaśnięcie "znika" dla reszty systemu. Publisher outboxa: kolejność per rezerwacja (porządkuj po pozycji w tabeli), oznaczanie wysłanych, odporność na własny restart (publikacja jest at-least-once → odbiorcy i tak są idempotentni).

### Idempotent consumers — wszędzie, bez wyjątku

Każdy handler w Workerze ma tabelę dedup z unique constraint (moduł 2, lekcja 01). W tej domenie duplikaty bolą konkretnie: podwójne `PaymentSucceeded` to podwójne potwierdzenie i potencjalnie dwa zgłoszenia do TunnelOps; podwójne `BookingCancelled` to próba podwójnego zwolnienia slotu. Dobra wiadomość: część operacji da się zaprojektować naturalnie idempotentnie (`UPDATE ... WHERE Status = 'Pending'` — przejście stanu zamiast inkrementu) i wtedy tabela dedup jest drugą linią obrony, nie jedyną.

### Saga rezerwacji z kompensacją

Orkiestrator (ADR-004) trzyma stan w tabeli `BookingSagas`: identyfikator rezerwacji, bieżący krok, deadline płatności, licznik prób. Zasady, które trzeba przemyśleć przed kodem:

Szkic stanu sagi (kolumny, nie gotowy kod — model dopasuj do swoich slice'ów):

```sql
CREATE TABLE BookingSagas (
    BookingId       UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    CurrentStep     NVARCHAR(50)  NOT NULL,  -- AwaitingPayment / Confirming / Compensating / Completed / Failed
    PaymentDeadline DATETIME2     NOT NULL,  -- koniec okna hold
    PaymentAttempts INT           NOT NULL DEFAULT 0,
    UpdatedAt       DATETIME2     NOT NULL
);
```

Zasady, których ten szkic pilnuje:

- **Stan zapisuj PRZED wykonaniem kroku** (intencja), nie tylko po — inaczej restart workera między wykonaniem a zapisem gubi wiedzę o tym, co już się stało. Razem z idempotencją kroków daje to bezpieczne wznowienie.
- **Kompensacja to nie rollback.** Zwolnienie slotu po nieudanej płatności nie cofa historii — rezerwacja ma stan `Cancelled` z powodem, a slot wraca do puli. Ślad zostaje (i bardzo dobrze: to materiał na audyt i na rozmowę o różnicy saga vs transakcja).
- **Timeout płatności to zdarzenie pierwszej klasy**, nie wyjątek. Bramka, która nie odpowiedziała, mogła pieniądze pobrać — w realnym systemie kompensacją byłby refund, u nas: zapisz tę świadomość w komentarzu/ADR i kompensuj zwolnieniem slotu. Na rozmowie to złoto: "symulacja pozwoliła mi zademonstrować przypadek, którego z prawdziwą bramką nie wywołam deterministycznie".

### Retry + DLQ dla integracji z TunnelOps

Moduł 2, lekcje 05–06, w praktyce: wywołanie HTTP do TunnelOps opakowane w politykę retry z backoffem (Microsoft.Extensions.Resilience / Polly) — ale retry tylko dla błędów przejściowych (5xx, timeout). Odpowiedź nieprzetwarzalna (TunnelOps odsyła śmieci albo nasz komunikat jest uszkodzony) to **poison message**: retry nic nie da, wiadomość po N nieudanych dostarczeniach trafia do DLQ Service Busa. Do DLQ dołącz **runbook** (`docs/runbooks/dlq.md`): jak obejrzeć wiadomość, jak rozpoznać przyczynę, jak bezpiecznie ponowić (resubmit) — to artefakt, który odróżnia "wiem, że DLQ istnieje" od "umiem z nim żyć na produkcji".

TunnelOps simulator musi umieć na żądanie (nagłówek/konfiguracja): odpowiadać poprawnie, zwracać 500, wisieć ponad timeout, odsyłać body-śmieci. Deterministyczne awarie to cała wartość symulatora.

### Vertical slices i testy

Organizacja kodu zgodnie z modułem 4: **vertical slice** = jeden przypadek użycia (np. `CreateBooking`) trzyma razem endpoint, handler, walidację i dostęp do danych — zamiast warstw poziomych rozrzucających jedną funkcję po pięciu projektach. Dla recenzenta: otwiera folder `Booking/CreateBooking/` i widzi całość.

Testy — piramida praktyczna:
- **Jednostkowe**: logika domenowa (przejścia stanów rezerwacji, decyzje sagi) — szybkie, bez infrastruktury.
- **Integracyjne z Testcontainers**: prawdziwy SQL Server (i broker, jeśli ADR-005 wybrał kontenerowy) w kontenerach podnoszonych przez testy. To tu żyją testy scenariuszy awaryjnych — one są wizytówką repo.

### Scenariusze do zademonstrowania i przetestowania

Każdy z poniższych = test automatyczny + (dla 2–3 najlepszych) opis w README z instrukcją ręcznej reprodukcji:

1. **Duplikat płatności** — `PaymentSucceeded` dostarczone dwukrotnie (sekwencyjnie i równolegle): jedna zmiana stanu, jedno powiadomienie, jedno zgłoszenie do TunnelOps. Gwarancją unique constraint, nie `if`.
2. **Timeout płatności** — bramka nie odpowiada: saga po wyczerpaniu polityki kompensuje, slot wraca do puli (asercja: slot da się zarezerwować ponownie), rezerwacja `Cancelled` z powodem `PaymentTimeout`.
3. **Poison message z TunnelOps** — odpowiedź-śmieć: brak nieskończonego retry, wiadomość w DLQ, pozostałe wiadomości przetwarzają się dalej (poison nie blokuje kolejki), zdarzenie widoczne w logach z korelacją.
4. **Restart workera w środku sagi** — zabij proces po udanej płatności, a przed potwierdzeniem (w teście: przerwij przetwarzanie; manualnie: `docker kill`); po starcie saga wznawia się ze stanu w bazie i kończy przepływ — bez podwójnych skutków.
5. *(bonus, jeśli jest czas)* **Wyścig o slot** — dwa równoległe POST na ten sam slot: dokładnie jeden hold, drugi klient dostaje czytelny błąd 409.

## Praktyka — punkty kontrolne tygodniowe

### Tydzień A (E1 + E2): szkielet, CI, booking z outboxem

- [ ] Solution: moduły booking/payments/notifications + Worker + TunnelOps simulator; `docker compose up` podnosi SQL + broker; CI buduje i odpala testy, badge w README.
- [ ] FR-1 + FR-2 działają: rezerwacja tworzy hold (unique constraint przetestowany testem równoległym), zdarzenie wychodzi przez outbox.
- [ ] Pierwszy idempotent consumer + pierwszy test Testcontainers (duplikat sekwencyjny i równoległy) zielony.
- [ ] **Punkt kontrolny:** przepływ "rezerwacja → zdarzenie skonsumowane" działa end-to-end lokalnie. Jeśli nie — w tygodniu B zacznij od domknięcia, kosztem bonusowego scenariusza 5.

### Tydzień B (E3): płatność i saga

- [ ] Symulowana bramka: sukces / odmowa / timeout sterowane konfiguracją.
- [ ] Orkiestrator sagi ze stanem w bazie; happy path: hold → płatność → `Confirmed`.
- [ ] Kompensacja: odmowa i timeout zwalniają slot (scenariusz 2 zielony); wygasanie holdów działa.
- [ ] Test restartu workera w środku sagi (scenariusz 4) zielony.
- [ ] **Punkt kontrolny:** wszystkie ścieżki sagi pokryte testami. Jeśli nie — wytnij FR-6 (anulowanie) z tygodnia C; saga jest ważniejsza.

### Tydzień C (E4): powiadomienia, TunnelOps, DLQ

- [ ] Powiadomienia generowane ze zdarzeń, idempotentnie.
- [ ] Integracja z TunnelOps: retry z backoffem dla 5xx/timeout (scenariusze sterowane symulatorem).
- [ ] Poison message trafia do DLQ i nie blokuje kolejki (scenariusz 3 zielony); runbook `docs/runbooks/dlq.md` napisany.
- [ ] FR-6 anulowanie (jeśli nie wycięte w tygodniu B) — przez te same wzorce.
- [ ] **Punkt kontrolny / DoD etapu:** sekcja Definition of Done poniżej w całości zielona.

W wariancie 4-tygodniowym (2 tygodnie na etap): tydzień A bez zmian; tygodnie B+C łączysz, tnąc: FR-6, scenariusz 5, powiadomienia redukujesz do jednego typu. **Nie tnij**: sagi z kompensacją, testów scenariuszy 1–4, runbooka DLQ.

## Artefakt

Działający system w repo: kod (vertical slices), `docker compose up` + `dotnet run` podnosi całość lokalnie, zestaw testów z Testcontainers obejmujący scenariusze 1–4, runbook DLQ, zielone CI.

## Definition of Done

- [ ] Pełny przepływ działa lokalnie: rezerwacja → płatność → potwierdzenie → powiadomienie → zgłoszenie do TunnelOps.
- [ ] Scenariusze 1–4 mają zielone testy automatyczne; umiesz każdy zademonstrować też ręcznie i wskazać w kodzie element dający gwarancję (transakcja, unique constraint, stan sagi).
- [ ] Żaden handler nie jest nie-idempotentny; każda publikacja zdarzenia idzie przez outbox.
- [ ] CI zielone na głównym branchu, czas świeżego startu lokalnego ≤ ~5 minut.
- [ ] Pytanie kontrolne (odpowiedz pisemnie w notatce, użyjesz w etapie 05): które z zastosowanych wzorców byłyby przerostem formy w systemie o 10× mniejszych wymaganiach niezawodności — i po czym to poznać?

## Materiały

1. **Moduł 2, lekcje 01–06** (`kurs/02-systemy-rozproszone/`) — pełna teoria każdego wzorca używanego w tym etapie; wracaj punktowo, gdy implementacja zaskrzypi.
2. **[Testcontainers for .NET](https://dotnet.testcontainers.org/)** — dokumentacja biblioteki do testów integracyjnych na prawdziwych kontenerach (SQL Server, broker).
