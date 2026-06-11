# Moduł 2 — Systemy rozproszone: pogłębienie

**Czas trwania:** 4–5 tygodni
**Pozycja w kursie:** po module 1 (fundamenty myślenia architektonicznego). Ten moduł jest fundamentem pod moduł 3 (Azure) i pod Capstone.

## Cel modułu

Zamienić Twoje praktyczne doświadczenie z produkcji — kolejki, integracje, nocne awarie z Perfect Gym i Aerotunel — w usystematyzowaną, **nazwaną** wiedzę. Po tym module o tym samym zdarzeniu, które kiedyś opisywałeś jako "wiadomość przyszła dwa razy i rozjechały się dane", powiesz: "at-least-once delivery bez idempotent consumera — klasyczny brak deduplikacji po stronie odbiorcy". To dokładnie ta różnica, którą słychać na rozmowie na poziom architekta.

Większość treści tego modułu prawdopodobnie **już przeżyłeś** — tylko nikt Ci nie powiedział, jak się te rzeczy nazywają i dlaczego działają (albo nie działają). Teoria ma tu nazywać blizny, nie dodawać nowych.

## Lekcje (przerabiaj po kolei)

| Lekcja | Temat | Artefakt |
|---|---|---|
| [01](01-gwarancje-dostarczania-i-idempotency/) | Gwarancje dostarczania i idempotency — dlaczego "exactly-once" to mit | idempotent consumer w C# + notatka |
| [02](02-outbox-inbox-transactional-messaging/) | Outbox, inbox i problem dual write | implementacja outboxa w .NET + EF Core |
| [03](03-sagi-orchestration-vs-choreography/) | Sagi: transakcje rozproszone bez 2PC | diagram + szkic sagi rezerwacji z kompensacją |
| [04](04-spojnosc-cap-pacelc/) | Spójność: strong vs eventual, CAP i PACELC uczciwie | tabela decyzyjna modeli spójności + post |
| [05](05-partycjonowanie-kolejnosc-dead-letter/) | Partycjonowanie, kolejność zdarzeń, poison messages i DLQ | runbook obsługi DLQ + post "production engineering" |
| [06](06-odpornosc-retry-circuit-breaker-backpressure/) | Odporność: timeouts, retry, circuit breaker, backpressure | pipeline odporności w Polly / Microsoft.Extensions.Resilience |
| [07](07-observability/) | Observability: logi, metryki RED/USE, distributed tracing | instrumentacja OpenTelemetry w .NET |
| [08](08-mini-projekt-messagingowy/) | Mini-projekt: repo "messaging-patterns" spinające cały moduł | repo → kandydat do Featured na LinkedIn |

## Rozkład na tygodnie

| Tydzień | Lekcje | Akcent |
|---|---|---|
| 1 | 01 + 02 | Fundament niezawodności: gwarancje dostarczania, idempotency, outbox. To serce modułu — nie spiesz się. |
| 2 | 03 + 04 | Koordynacja i spójność: sagi, CAP/PACELC. Więcej teorii, mniej kodu — dobry tydzień na pierwszy post. |
| 3 | 05 + 06 | Realia produkcji: partycje, kolejność, DLQ, retry, circuit breaker. Tu nazwiesz najwięcej własnych blizn. |
| 4 | 07 + start 08 | Observability + inicjalizacja repo mini-projektu (broker w Dockerze, szkielet usług). |
| 5 | 08 | Dokończenie mini-projektu: saga, DLQ, tracing, symulacje awarii, README. |

Jeśli mieścisz się w 4 tygodniach — lekcję 07 i start projektu zacznij już w tygodniu 3. Jeśli potrzebujesz więcej — tnij zakres mini-projektu (jest rozpisany na poziomy), nie tnij lekcji 01–02.

## Materiał bazowy modułu (max 2 źródła)

1. **"Designing Data-Intensive Applications"** — Martin Kleppmann. Biblia tego modułu. Czytaj równolegle z lekcjami (mapowanie rozdziałów jest w lekcjach). Nie musisz przeczytać całości — rozdziały 5, 7, 8, 9, 11 wystarczą.
2. **[microservices.io](https://microservices.io/patterns/)** — Chris Richardson. Katalog wzorców (saga, outbox, CQRS…) z jednolitą strukturą problem/rozwiązanie. Idealny jako szybka referencja i źródło nazw, którymi mówi się na rozmowach.

## Artefakty modułu

- [ ] Repo **messaging-patterns** w .NET (lekcja 08) — działający przepływ zamówienie → płatność → powiadomienie z outboxem, idempotencją, sagą, retry+DLQ i tracingiem. **Kandydat do Featured na LinkedIn**, później baza pod Capstone.
- [ ] Seria 2–3 postów "production engineering" na LinkedIn, np.:
  - "Czego nauczył mnie poison message o 3 w nocy" (lekcja 05),
  - "Exactly-once delivery nie istnieje — i co z tym robić" (lekcja 01),
  - "Dual write: bug, którego pewnie masz w produkcji i o tym nie wiesz" (lekcja 02).
- [ ] Mniejsze artefakty per lekcja (kod, diagramy, runbook) — szczegóły w lekcjach.

## Definition of Done modułu

Moduł 2 jest zaliczony, gdy **wszystkie** poniższe są prawdą:

- [ ] Potrafisz zaprojektować niezawodny przepływ komunikatów end-to-end (producent → broker → konsument → kolejna usługa) i **wyjaśnić każdą decyzję**: dlaczego at-least-once, dlaczego outbox, dlaczego ten klucz partycji, jaka polityka retry i co trafia do DLQ.
- [ ] Umiesz na tablicy, bez przygotowania, wytłumaczyć: idempotency, outbox, sagę z kompensacją, różnicę orchestration/choreography, co naprawdę mówi CAP i czym jest PACELC.
- [ ] Repo messaging-patterns działa lokalnie (jedno `docker compose up` + `dotnet run`), przeżywa zasymulowane duplikaty, out-of-order i poison message — i ma README, które to udowadnia.
- [ ] Opublikowane minimum 2 posty z serii "production engineering".
- [ ] Dla każdego wzorca z modułu umiesz odpowiedzieć na pytanie: **"kiedy tego NIE używać?"**.

Po zaliczeniu: odhacz Moduł 2 w `architect-track-plan.md`, zaktualizuj `progress.md` i przejdź do modułu 3 — tam ten sam system trafi na Azure.
