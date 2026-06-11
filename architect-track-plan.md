# Architect Track — plan nauki i kariery

> Cel: w 6–9 miesięcy zbudować kompetencje, portfolio i widoczność na poziomie Solution Architect / Senior Engineer (architecture track), bazując na 20+ latach doświadczenia w .NET, integracjach i systemach enterprise.
>
> Zasada przewodnia: **każdy moduł kończy się artefaktem** (kod, diagram, ADR lub post na LinkedIn). Nauka bez artefaktu się nie liczy.

<!-- INSTRUKCJA DLA CLAUDE CODE:
Ten plik to szkielet. Rozbudowuj go modułami:
- każdy moduł rozwiń do osobnego pliku modules/NN-nazwa.md z planem tygodniowym,
- sekcje oznaczone [TODO: ...] wypełnij konkretami,
- zachowaj konwencję: Cel / Tematy / Praktyka / Artefakty / Definition of Done,
- przy doborze materiałów preferuj źródła pierwotne (dokumentacja, książki, papery) nad kursy wideo.
-->

## Status

- [ ] Moduł 0 — Setup i rytm pracy
- [ ] Moduł 1 — Fundamenty myślenia architektonicznego
- [ ] Moduł 2 — Distributed systems (pogłębienie)
- [ ] Moduł 3 — Cloud-native na Azure
- [ ] Moduł 4 — Nowoczesny stack .NET
- [ ] Moduł 5 — AI-augmented engineering
- [ ] Moduł 6 — Komunikacja architektoniczna i leadership
- [ ] Capstone — projekt referencyjny
- [ ] Równolegle (ciągłe) — LinkedIn i widoczność

---

## Moduł 0 — Setup i rytm pracy (1 tydzień)

**Cel:** ustawić powtarzalny tygodniowy rytm, który przetrwa do końca planu.

**Rytm tygodniowy (propozycja, dostosuj):**
- 4×/tydz. — 2h głębokiej nauki/praktyki (rano, jako pierwszy "chunk" w DayChunks)
- 1×/tydz. — pisanie posta LinkedIn (60–90 min)
- 3–4×/tydz. — 15 min komentowania na LinkedIn
- 1×/tydz. — przegląd tygodnia: co działa, co wyciąć

**Artefakty:**
- [ ] szablon w DayChunks z blokami nauki
- [ ] to repo zainicjowane w git (plan + notatki + kod ćwiczeń w jednym miejscu)
- [ ] plik `notes/decisions-log.md` — własny dziennik decyzji (trenujesz format ADR na sobie)

---

## Moduł 1 — Fundamenty myślenia architektonicznego (3–4 tygodnie)

**Cel:** przejść mentalnie z "jak to zaimplementować" na "jakie są opcje, trade-offy i konsekwencje".

**Tematy:**
- atrybuty jakościowe (scalability, availability, maintainability, security) i ich konflikt
- trade-off analysis — architektura jako wybór, czego NIE robić (masz to już w narracji DayChunks)
- C4 model — diagramy: context, container, component
- ADR (Architecture Decision Records) — format, kiedy pisać, przykłady
- DDD strategiczne: bounded contexts, context mapping (bez przesadnego zagłębiania w taktyczne)
- style architektoniczne: modular monolith vs microservices vs event-driven — kiedy który
- [TODO: dobrać 1 książkę bazową — kandydaci: "Fundamentals of Software Architecture" (Richards/Ford), "Software Architecture: The Hard Parts"]

**Praktyka:**
- [ ] narysuj C4 (poziom 1 i 2) dla systemu z Perfect Gym, którego byłeś ownerem — z pamięci, zanonimizowany
- [ ] napisz 3 ADR-y wstecz dla decyzji z własnej kariery (w tym: "DayChunks bez backendu")
- [ ] [TODO: 2–3 ćwiczenia system design — np. zaprojektuj system rezerwacji, system powiadomień]

**Artefakty:**
- [ ] diagram C4 → kandydat do Featured na LinkedIn
- [ ] post: "ADR — najtańsze narzędzie architekta, którego prawie nikt nie używa"

**Definition of Done:** umiesz dla dowolnego problemu wymienić 2–3 opcje architektoniczne i uczciwie nazwać trade-offy każdej.

---

## Moduł 2 — Distributed systems: pogłębienie (4–5 tygodni)

**Cel:** zamienić praktyczne doświadczenie (kolejki, integracje, produkcja) w usystematyzowaną, nazwaną wiedzę — żeby na rozmowie mówić językiem teorii popartym bliznami z produkcji.

**Tematy:**
- gwarancje dostarczania: at-least-once, at-most-once, effectively-once; idempotency
- outbox pattern, inbox pattern, transactional messaging
- sagi i koordynacja procesów (orchestration vs choreography)
- spójność: strong vs eventual; CAP/PACELC w praktyce, nie w teorii
- partycjonowanie, kolejność zdarzeń, dead-letter handling (→ masz historie z Perfect Gym)
- backpressure, retry policies, circuit breakers, timeouts — budżety błędów
- observability: distributed tracing, korelacja, metryki RED/USE
- [TODO: dobrać materiał bazowy — kandydaci: "Designing Data-Intensive Applications" (Kleppmann), microservices.io]

**Praktyka:**
- [ ] zaimplementuj outbox pattern w .NET na czystym przykładzie (repo ćwiczeniowe)
- [ ] zasymuluj i obsłuż: duplikaty wiadomości, wiadomości out-of-order, poison message
- [ ] [TODO: rozpisać mini-projekt messagingowy — wejdzie potem do Capstone]

**Artefakty:**
- [ ] seria 2–3 postów "production engineering" (np. "Czego nauczył mnie poison message o 3 w nocy")
- [ ] repo z wzorcami messagingowymi — kandydat do Featured

**Definition of Done:** potrafisz zaprojektować niezawodny przepływ komunikatów end-to-end i wyjaśnić każdą decyzję.

---

## Moduł 3 — Cloud-native na Azure (4 tygodnie)

**Cel:** przełożyć wiedzę o systemach rozproszonych na konkretne usługi i decyzje kosztowe w Azure.

**Tematy:**
- compute: Container Apps vs AKS vs Functions vs App Service — macierz decyzyjna
- messaging: Service Bus vs Event Grid vs Event Hubs — kiedy który
- dane: SQL Database, Cosmos DB — modele spójności w praktyce
- sieć i bezpieczeństwo: identity (Entra ID, managed identities), Key Vault, private endpoints
- IaC: Bicep lub Terraform — podstawy wystarczające do projektu referencyjnego
- koszty jako atrybut architektury (FinOps w pigułce) — silny temat na rozmowach
- [TODO: zdecydować, czy celować w certyfikat AZ-305 (Azure Solutions Architect) — duża wartość sygnalizacyjna przy zmianie pracy; jeśli tak, rozpisać przygotowanie]

**Praktyka:**
- [ ] wdróż aplikację z modułu 2 na Azure (Container Apps + Service Bus) z IaC
- [ ] skonfiguruj pełną observability (Application Insights, distributed tracing)
- [ ] policz i udokumentuj koszty trzech wariantów architektury tej samej aplikacji

**Artefakty:**
- [ ] post: porównanie wariantów compute/messaging z realnymi kosztami
- [ ] diagram architektury wdrożenia → Featured

**Definition of Done:** umiesz uzasadnić wybór usług Azure dla danego problemu, włącznie z kosztami.

---

## Moduł 4 — Nowoczesny stack .NET (2–3 tygodnie, lekki)

**Cel:** domknąć luki względem aktualnego ekosystemu — bez kursowania od zera, masz fundament.

**Tematy:**
- [TODO: zweryfikować aktualny stan ekosystemu na dziś — co jest bieżącym LTS .NET, status .NET Aspire, aktualne praktyki minimal APIs — i wpisać konkrety]
- architektura aplikacji: modular monolith w praktyce, vertical slice architecture
- testy: testcontainers, testy architektury (ArchUnit-style), contract testing
- wydajność: profilowanie, EF Core pod kątem perf, source generators — przegląd

**Praktyka:**
- [ ] zrefaktoryzuj jedno ćwiczenie z modułu 2 do vertical slices + pełne testy

**Artefakty:**
- [ ] post: "Co realnie zmieniło się w .NET przez ostatnie lata — okiem kogoś, kto pamięta WebForms"

**Definition of Done:** Twój kod ćwiczeniowy wygląda jak współczesny, idiomatyczny .NET.

---

## Moduł 5 — AI-augmented engineering (3–4 tygodnie, częściowo równolegle z 2–4)

**Cel:** udokumentować i pogłębić to, co już robisz (Claude Code przy DayChunks) — to Twój wyróżnik na rynku, mało który 20-letni senior to ma.

**Tematy:**
- spec-driven development: pisanie specyfikacji/planów dla agenta zamiast promptów ad-hoc
- code review pracy agenta — gdzie AI systematycznie błądzi w architekturze
- AI w cyklu architekta: generowanie ADR-ów, diagramów, analiz trade-offów — i ich weryfikacja
- podstawy budowania funkcji AI w produktach: wywołania LLM z .NET, RAG w zarysie, koszty i latencja jako trade-off
- [TODO: rozpisać konkretne eksperymenty na bazie codziennej pracy z Claude Code]

**Praktyka:**
- [ ] prowadź log eksperymentów: co delegujesz, co wychodzi, co poprawiasz (surowiec na posty)
- [ ] zbuduj jeden mały feature AI end-to-end (może w przyszłej wersji DayChunks lub osobno)

**Artefakty:**
- [ ] seria postów "AI-augmented engineering — honest takes" (1–2/mies., długoterminowo)

**Definition of Done:** masz udokumentowany, powtarzalny workflow pracy z agentem i umiesz o nim opowiedzieć na rozmowie z konkretami.

---

## Moduł 6 — Komunikacja architektoniczna i leadership (2 tygodnie + ciągłe)

**Cel:** architektem się jest wtedy, gdy inni rozumieją Twoje decyzje. Trening komunikacji.

**Tematy:**
- prezentowanie architektury różnym odbiorcom: zarząd / PM / zespół — trzy wersje tej samej historii
- prowadzenie design review i architektura jako proces zespołowy (RFC, ADR w zespole)
- przygotowanie do rozmów rekrutacyjnych: system design interview — format, typowe zadania, narracja
- [TODO: rozpisać 5–6 ćwiczeń system design interview z harmonogramem]

**Praktyka:**
- [ ] opowiedz (nagraj się) historię architektury z Aerotunel/Perfect Gym w 5 minut — wersja "na rozmowę"
- [ ] przeprowadź 2–3 mock system design interviews [TODO: znaleźć partnera lub użyć Claude jako interviewera]

**Artefakty:**
- [ ] gotowe 3 "war stories" w formacie STAR pod rozmowy rekrutacyjne

---

## Capstone — projekt referencyjny (4–6 tygodni, po modułach 2–3)

**Cel:** jedno repo, które jest dowodem wszystkiego powyżej. To, co pokazujesz w Featured i na rozmowach.

**Koncept (do decyzji):** event-driven system rezerwacji (nawiązuje do doświadczenia z Aerotunel) — booking, płatności (symulowane), powiadomienia, integracja z systemem zewnętrznym (symulowanym).

**Wymagania:**
- README z diagramami C4 i sekcją "Architecture decisions" (ADR-y w repo)
- outbox, idempotency, saga, dead-lettering — wzorce z modułu 2 w użyciu
- wdrażalne na Azure z IaC, z observability
- sekcja "Trade-offs" — czego świadomie NIE ma i dlaczego (Twój znak firmowy po poście o DayChunks)
- [TODO: rozpisać backlog projektu na epiki i tygodnie]

**Artefakty:**
- [ ] repo → Featured (zastępuje/uzupełnia wszystko wcześniejsze)
- [ ] post podsumowujący + ewentualnie artykuł techniczny (dev.to / blog)

---

## Równolegle (ciągłe) — LinkedIn i widoczność

> Szczegółowy plan LinkedIn jest osobno (profil zrobiony: headline, About, baner, Featured w trakcie). Tu tylko rytm.

- [ ] 1 post/tydzień — tematy wychodzą z modułów (każdy moduł = 2–4 posty)
- [ ] 3–4 komentarze/tydzień u architektów, principal engineerów, ludzi od distributed systems
- [ ] 5–10 nowych kontaktów technicznych/tydzień (nie HR)
- [ ] raz w miesiącu: przegląd profilu — co zaktualizować po zakończonym module
- [ ] od modułu 3: zacznij aplikować na role Senior/Lead z "architecture exposure" — nie czekaj na "gotowość"

## Kamienie milowe

| Kiedy | Co jest prawdą |
|---|---|
| koniec mies. 1 | rytm działa, pierwsze C4 i ADR-y istnieją, 4+ posty opublikowane |
| koniec mies. 3 | moduły 1–2 zamknięte, repo messagingowe w Featured, sieć rośnie |
| koniec mies. 5 | Azure + capstone w toku, aktywne aplikowanie, pierwsze rozmowy |
| koniec mies. 6–9 | capstone w Featured, [TODO: decyzja ws. AZ-305], oferta na stole |

<!-- TODO globalne dla Claude Code:
1. Zweryfikuj aktualność wszystkich technologii i nazw usług (plan pisany 2026-06).
2. Rozbij moduły na pliki tygodniowe z konkretnymi zadaniami dziennymi.
3. Do każdego modułu dobierz max 2 źródła główne — nie listy 20 linków.
4. Dodaj plik progress.md aktualizowany po każdym tygodniu.
-->
