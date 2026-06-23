# Trade-off analysis — dodatkowe ćwiczenia

Zestaw scenariuszy do samodzielnego treningu metody z lekcji 02. Każde ćwiczenie to **problem bez wbudowanego rozwiązania** — Twoim zadaniem jest przejść pełną procedurę:

1. Przeformułuj problem tak, by NIE zawierał rozwiązania (jeśli trzeba).
2. Ustal kryteria (priorytetowe atrybuty + realia) **przed** wypisaniem opcji.
3. Wypisz 2–4 realne opcje, w tym „nie robić nic" / „najprościej".
4. Wypełnij macierz opcja × kryterium (skala `++ / + / 0 / − / −−`, bez sumowania punktów).
5. Oceń odwracalność każdej opcji (one-way / two-way, pamiętaj o asymetrii kierunku i o tym, że to *zobowiązania*, nie dane, robią one-way door).
6. Rekomendacja w formacie: *wybieram X, świadomie płacąc Y; wrócę do tego, jeśli Z*.

Scenariusze ułożone od najbardziej two-way (decyzje lekkie) do twardych one-way doorów. Pod każdym jest blok **Podpowiedź** — zajrzyj do niego dopiero PO swojej analizie, żeby się sprawdzić, nie żeby się podeprzeć.

---

## Ćwiczenie 1 — Dostęp do danych w nowym module (two-way)

**Kontekst:** Dopisujesz nowy moduł raportowy w aplikacji .NET. Zespół spiera się, czym czytać dane z bazy.

**Problem:** Warstwa dostępu do danych w module raportowym ma być wydajna przy kilku ciężkich zapytaniach analitycznych i jednocześnie łatwa w utrzymaniu przez resztę zespołu.

**Wskazówki do kryteriów:** wydajność zapytań read-heavy, czytelność/utrzymywalność, kompetencje zespołu, spójność z resztą kodu.

<details><summary>Podpowiedź</summary>

Opcje typu EF Core / Dapper / raw ADO.NET. To klasyczny **two-way door** — wszystko w jednym module, zmiana kosztuje godziny, nie miesiące. Ćwiczenie ma Cię nauczyć rozpoznawać, że tu *nie* należy się trzystronicowa analiza (analysis paralysis). Puenta: proporcjonalność staranności do odwracalności.
</details>

---

## Ćwiczenie 2 — Cache grafiku zajęć pod poranny szczyt (two-way, ale z pułapką)

**Kontekst:** Perfect Gym. Odczyt grafiku zajęć nie wytrzymuje porannego szczytu: p95 > 3 s przy 200 RPS.

**Problem:** (już dobrze postawiony — zwróć uwagę, że to problem, nie rozwiązanie) Czas odpowiedzi odczytu grafiku ma zejść poniżej 500 ms p95 przy 200 RPS, bez psucia spójności (członek nie może zobaczyć zajęć, które właśnie odwołano).

**Wskazówki do kryteriów:** p95 latencja, świeżość danych (jak bardzo nieaktualny grafik boli?), koszt operacyjny, złożoność.

<details><summary>Podpowiedź</summary>

Opcje: cache (Redis / in-memory), read replica bazy, denormalizacja/materializacja widoku, CDN dla części statycznej, „nie robić nic + indeks". Większość to two-way. Pułapka: **invalidacja cache** to ukryty trade-off ze spójnością (członek widzi odwołane zajęcia). Tu trenujesz przeformułowanie „wdróżmy Redis" → problem wydajnościowy z wieloma opcjami.
</details>

---

## Ćwiczenie 3 — Współbieżne rezerwacje slotu w tunelu (two-way technicznie, krytyczne biznesowo)

**Kontekst:** Aerotunel. Slot ma ograniczoną liczbę miejsc; przy szczycie dwie osoby potrafią zarezerwować ostatnie miejsce „równocześnie".

**Problem:** System rezerwacji nie może sprzedać więcej miejsc na slot, niż jest dostępnych, nawet przy równoczesnych żądaniach — przy zachowaniu dobrego UX (nie blokować użytkownika na sekundy).

**Wskazówki do kryteriów:** poprawność (zero overbookingu), latencja/UX, przepustowość w szczycie, złożoność implementacji.

<details><summary>Podpowiedź</summary>

Opcje: optimistic concurrency (wersjonowanie wiersza), pessimistic lock, serializacja przez kolejkę, „reservation hold" z TTL. Sama mechanika jest two-way (zmienisz strategię w kodzie), ale **dyskwalifikator** — jakiekolwiek „−−" na poprawności — bije resztę. Ćwiczenie uczy wykrywania dyskwalifikatora, nie sumowania punktów.
</details>

---

## Ćwiczenie 4 — Wybór szyny komunikatów w Azure (two-way/one-way — kontrakt zaczyna żyć)

**Kontekst:** Nowy system rozbijasz na kilka usług, które muszą komunikować się asynchronicznie. Stack: Azure.

**Problem:** Usługi mają wymieniać zdarzenia asynchronicznie z gwarancją dostarczenia i możliwością retry, przy koszcie operacyjnym do udźwignięcia przez mały zespół.

**Wskazówki do kryteriów:** gwarancje dostarczenia (at-least-once?), koszt i obsługa operacyjna, dojrzałość/lock-in, możliwość lokalnego testowania.

<details><summary>Podpowiedź</summary>

Opcje: Azure Service Bus, Storage Queue, Event Grid, self-hosted RabbitMQ na kontenerze, „na razie wywołania HTTP". Sam wybór brokera bywa two-way, ale **kontrakt komunikatu** (schemat zdarzenia), na którym zawisną konsumenci, dryfuje w stronę one-way — to on jest prawdziwym one-way doorem, nie logo dostawcy. Zauważ to w kolumnie odwracalności.
</details>

---

## Ćwiczenie 5 — Build vs buy: uwierzytelnianie (częściowo one-way)

**Kontekst:** Startujesz produkt B2C z kontami użytkowników.

**Problem:** Użytkownicy mają mieć bezpieczne logowanie (hasło + social + MFA w przyszłości) bez budowania i utrzymywania całej maszynerii bezpieczeństwa własnymi siłami — albo świadomie z jej budową, jeśli to się spina.

**Wskazówki do kryteriów:** bezpieczeństwo/zgodność, czas dostarczenia, koszt (stały vs per-MAU), kontrola i lock-in, koszt migracji userów później.

<details><summary>Podpowiedź</summary>

Opcje: Microsoft Entra External ID (d. Azure AD B2C), Auth0, własny IdentityServer/ASP.NET Identity, „na razie magic link". To **częściowo one-way**: gdy masz już bazę kont i tożsamości userów, migracja do innego dostawcy jest bolesna (hasła nie wyeksportujesz). Asymetria: wejście tanie, wyjście drogie. Porównaj z opcją A z `analiza-trade-off-01.md`.
</details>

---

## Ćwiczenie 6 — Model multi-tenancy dla sieci klubów (twardy one-way)

**Kontekst:** Budujesz SaaS dla sieci siłowni. Każda sieć = tenant; docelowo setki tenantów, część z wymaganiami compliance (osobna lokalizacja danych).

**Problem:** Dane różnych sieci mają być izolowane (bezpieczeństwo + compliance) przy rozsądnym koszcie na tenant i możliwości obsługi setek tenantów przez mały zespół.

**Wskazówki do kryteriów:** izolacja/compliance, koszt na tenant, operacyjna skalowalność (migracje, backupy ×N), elastyczność per-tenant.

<details><summary>Podpowiedź</summary>

Opcje: baza-per-tenant, wspólna baza + kolumna `TenantId`, schema-per-tenant, model hybrydowy (większość shared, VIP osobno). To **twardy one-way door** — zmiana modelu po roku produkcji to migracja wszystkich danych i przepisanie warstwy dostępu. Dokładnie typ decyzji, której należy się pełna macierz + ADR + recenzja + ewentualny spike. Ćwicz tu „zwolnij przy one-way".
</details>

---

## Jak sprawdzić, że ćwiczenie zaliczone

- Masz min. 3 opcje, w tym najprostszą / „nie robić nic".
- Kryteria spisane przed opcjami.
- W macierzy widać *profil* każdej opcji i ewentualny dyskwalifikator — nie sumujesz punktów.
- Każda opcja ma ocenę odwracalności z uzasadnieniem (co narasta przy wyjściu).
- Rekomendacja kończy się zdaniem „wybieram X, płacąc Y, wrócę do tego, jeśli Z".
- Test z lustra: opowiadasz analizę na głos w 3 minuty.
