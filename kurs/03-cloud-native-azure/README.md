# Moduł 3 — Cloud-native na Azure

**Czas trwania:** 4 tygodnie
**Pozycja w kursie:** po module 2 (systemy rozproszone). Repo messagingowe z modułu 2 jest wejściem do projektu kończącego ten moduł.

## Cel modułu

Przełożyć wiedzę o systemach rozproszonych — outbox, saga, DLQ, observability — na **konkretne usługi Azure i konkretne decyzje kosztowe**. Po module 2 wiesz *jak* projektować niezawodny przepływ komunikatów. Po module 3 masz umieć powiedzieć: "ten przepływ wdrażam na Container Apps + Service Bus + Azure SQL, bo X, a kosztuje to mniej więcej Y — i oto alternatywy, które odrzuciłem".

To jest dokładnie język, którym mówi Solution Architect na rozmowie rekrutacyjnej: nie "znam Azure", tylko "umiem uzasadnić wybór usługi, włącznie z kosztami i trade-offami".

## Założenie wyjściowe

Moduł zakłada, że **nie znasz Azure w głębi** — każda usługa jest przedstawiona od zera: co robi, jak się rozlicza, kiedy jej użyć, a kiedy nie. Znasz za to systemy rozproszone z praktyki, więc lekcje stale mapują usługi na wzorce z modułu 2 ("Service Bus to miejsce, gdzie naturalnie żyje twój DLQ").

Potrzebujesz subskrypcji Azure. Na start wystarczy darmowe konto (free tier + kredyt startowy) — zweryfikuj aktualne warunki na [azure.microsoft.com/free](https://azure.microsoft.com/free). Ustaw budżet z alertem **w pierwszym tygodniu** (lekcja 06 wyjaśnia jak; zrób to wcześniej, na zapas, nawet "na ślepo" przez portal).

## Lekcje (przerabiaj po kolei)

| Lekcja | Temat | Artefakt |
|---|---|---|
| [01](01-compute-macierz-decyzyjna/) | Compute w Azure — macierz decyzyjna | macierz decyzyjna compute + ADR dla własnej aplikacji |
| [02](02-messaging-w-azure/) | Messaging — Service Bus vs Event Grid vs Event Hubs vs Storage Queues | macierz messaging + mapowanie wzorców z modułu 2 |
| [03](03-dane-sql-cosmos/) | Dane — Azure SQL Database vs Cosmos DB | notatka decyzyjna SQL/Cosmos + tabela poziomów spójności |
| [04](04-tozsamosc-i-bezpieczenstwo/) | Tożsamość i bezpieczeństwo — Entra ID, managed identities, Key Vault | diagram przepływu tożsamości "zero connection stringów" |
| [05](05-iac-bicep/) | Infrastructure as Code — Bicep | działający szablon Bicep (Container App + Service Bus + SQL) |
| [06](06-koszty-finops/) | Koszty jako atrybut architektury — FinOps | porównanie kosztów 3 wariantów architektury (→ post) |
| [07](07-wdrozenie-projektu/) | Projekt: wdrożenie aplikacji z modułu 2 na Azure | działające wdrożenie + diagram architektury (→ Featured) |

## Rozkład na 4 tygodnie

| Tydzień | Lekcje | Akcent |
|---|---|---|
| 1 | 01 + 02 | fundament decyzyjny: compute i messaging — to dwie osie każdej architektury w tym kursie |
| 2 | 03 + 04 | dane i tożsamość; pod koniec tygodnia masz komplet "klocków" do projektu |
| 3 | 05 + start 07 | IaC i pierwsze wdrożenie infrastruktury z Bicep; aplikacja zaczyna działać w chmurze |
| 4 | 06 + finisz 07 | koszty trzech wariantów, pełna observability, artefakty na LinkedIn, decyzja AZ-305 |

Lekcje 01–06 są celowo "gęste teoretycznie", a 07 to czysta praktyka — jeśli w tygodniu 3–4 zabraknie czasu, tnij głębię ćwiczeń w 01–06, ale **nie tnij projektu 07**. To on jest dowodem dla rynku.

## Artefakty modułu

- [ ] post na LinkedIn: porównanie wariantów compute/messaging z realnymi kosztami (lekcja 06)
- [ ] diagram architektury wdrożenia → sekcja Featured na LinkedIn (lekcja 07)
- [ ] repo z modułu 2 rozszerzone o katalog `infra/` (Bicep) i działające wdrożenie na Azure (lekcje 05 + 07)
- [ ] komplet macierzy decyzyjnych (compute, messaging, dane) w notatkach — twoja ściągawka na rozmowy

## Definition of Done modułu

Moduł 3 jest zaliczony, gdy **wszystkie** poniższe są prawdą:

- [ ] Dla dowolnego z trzech problemów (API webowe, przetwarzanie kolejkowe, strumień zdarzeń) umiesz w 2 minuty wskazać usługi Azure, uzasadnić wybór i podać rząd wielkości kosztu.
- [ ] Aplikacja messagingowa z modułu 2 działa na Azure (Container Apps + Service Bus + SQL), wdrożona w 100% z Bicep, bez ani jednego connection stringa w konfiguracji.
- [ ] Distributed trace przechodzi end-to-end: producer → Service Bus → consumer → baza, widoczny w Application Insights.
- [ ] Porównanie kosztów trzech wariantów architektury jest policzone, udokumentowane i opublikowane jako post.
- [ ] Diagram architektury wdrożenia wisi w Featured.
- [ ] Decyzja w sprawie AZ-305 jest podjęta i zapisana w `notes/decisions-log.md`.

## Decyzja: AZ-305 — tak czy nie?

Pod koniec modułu podejmij świadomą decyzję, czy celować w certyfikat **AZ-305: Designing Microsoft Azure Infrastructure Solutions** (daje tytuł *Azure Solutions Architect Expert*). To otwarta pozycja z planu — nie odkładaj jej w nieskończoność, zapisz decyzję jako wpis w dzienniku decyzji.

**Co to jest.** Egzamin projektowy (design), nie implementacyjny: scenariusze "klient potrzebuje X, wybierz architekturę" z obszarów: identity/governance, dane, business continuity (HA/DR), infrastruktura (compute, sieć, messaging). Czyli dokładnie to, co ćwiczysz w tym module — tylko szerzej (sieci, hybrid, migracje, governance na poziomie enterprise).

**Warunek wstępny.** Tytuł Expert wymaga wcześniejszego zdania **AZ-104 (Azure Administrator)** — egzaminu operacyjnego (portal, CLI, zarządzanie zasobami). Realny koszt decyzji "tak" to więc **dwa egzaminy** albo solidne nadrobienie wiedzy administratorskiej, której ten kurs celowo nie pogłębia. Zweryfikuj aktualne wymagania na Microsoft Learn — zasady certyfikacji bywają zmieniane.

**Argumenty za:**
- Silny sygnał przy zmianie pracy — szczególnie w firmach konsultingowych i enterprise, gdzie certyfikaty są filtrem rekrutacyjnym i wymogiem partnerstw Microsoft.
- Przy 20+ latach doświadczenia i świeżo przerobionych modułach 2–3 koszt przygotowania jest niższy niż dla typowego kandydata — większość materiału już znasz lub właśnie poznajesz.
- Struktura egzaminu wymusza domknięcie luk (sieci, DR, governance), których samodzielny kurs naturalnie nie pokrywa.

**Argumenty przeciw:**
- Czas. AZ-104 + AZ-305 to realnie 6–10 tygodni przygotowań obok pracy — czas zabrany capstone'owi i budowaniu widocznego portfolio, które dla wielu pracodawców (zwłaszcza produktowych) waży więcej niż certyfikat.
- Wartość sygnalizacyjna zależy od targetu: software house'y i konsulting → wysoka; firmy produktowe i startupy → umiarkowana (tam liczy się repo, posty, rozmowa systemowa).
- Certyfikat bez projektu w portfolio jest słabszym sygnałem niż projekt bez certyfikatu.

**Kryteria decyzji (odpowiedz pisemnie na każde):**
1. Czy firmy, do których realnie aplikujesz, wymieniają certyfikaty Azure w ogłoszeniach? (Sprawdź 10 ogłoszeń, na które byś aplikował.)
2. Czy po module 3 czujesz, że egzamin byłby "domknięciem", czy "nauką od nowa"? Zrób darmowy practice assessment AZ-305 na Microsoft Learn — wynik powie ci więcej niż przeczucie.
3. Czy masz budżet czasowy: capstone (moduły 6–7) + certyfikacja równolegle, czy to "albo–albo"?

**Rekomendacja domyślna:** jeśli wynik practice assessment ≥ ~60% i celujesz w konsulting/enterprise — tak, rozpisz przygotowanie po capstone. W przeciwnym razie — nie teraz; wróć do tematu po pierwszych rozmowach rekrutacyjnych, gdy rynek powie ci, czy certyfikat jest twoim wąskim gardłem.

## Materiały przekrojowe modułu

- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/) — wzorce, referencyjne architektury, przewodniki wyboru usług; główne źródło całego modułu.
- [Microsoft Learn — ścieżka AZ-305](https://learn.microsoft.com/credentials/certifications/azure-solutions-architect/) — nawet bez decyzji o certyfikacie, sylabus jest dobrą mapą kompletności wiedzy architekta Azure.
