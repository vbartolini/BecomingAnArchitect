# Etap 05 — Dokumentacja i publikacja: repo jako wizytówka

## Cel lekcji

Po tym etapie capstone jest opublikowany i pracuje na Ciebie: README przechodzi "test 5 minut" recenzenta, link do wizytówki wisi w Featured na LinkedIn, post podsumowujący jest opublikowany, a Ty umiesz opowiedzieć o projekcie na rozmowie rekrutacyjnej jako war story o decyzjach — nie jako listę technologii.

## Dlaczego to ważne

Brutalna prawda o projektach portfolio: **nikt nie czyta kodu**. Hiring manager poświęci Ci 3–5 minut, z czego 80% spędzi w README. Jeśli README nie sprzeda projektu, najlepsza saga świata nie istnieje. Cały etap 05 to optymalizacja pod jednego użytkownika: zmęczonego recenzenta z dziesiątką innych kandydatur w kolejce.

Druga prawda: repo bez dystrybucji to drzewo przewracające się w pustym lesie. Featured na LinkedIn, post i (opcjonalnie) artykuł to nie marketing dla marketingu — to kanały, którymi projekt dociera do ludzi podejmujących decyzje o zatrudnieniu. Robisz to raz, porządnie, i działa miesiącami.

## Teoria

### Anatomia świetnego README projektu referencyjnego

Kolejność sekcji odpowiada kolejności, w jakiej recenzent traci cierpliwość — najważniejsze rzeczy idą pierwsze:

1. **Elevator pitch (2–3 zdania + ewentualnie 1 linia "tech")**. Co to jest, co demonstruje, na czym działa. Wzór: *"Event-driven booking system for a wind tunnel (time slots, instructors, simulated payments). Demonstrates production-grade messaging patterns — transactional outbox, idempotent consumers, orchestrated saga with compensation, retry + dead-lettering — deployed to Azure Container Apps with full IaC (Bicep) and end-to-end distributed tracing."* Bez "this is my learning project" — to projekt referencyjny, nie zeszyt ćwiczeń.
2. **Diagram C4 poziom 2** — od razu pod pitchem, jako obraz w README. Recenzent-architekt rozumie system z diagramu szybciej niż z dziesięciu akapitów. Pod spodem link do poziomu 1 i źródeł diagramów.
3. **"Architecture decisions"** — tabela ADR-ów z jednozdaniowym streszczeniem decyzji i linkiem do pliku: *ADR-002: Modular monolith, deployed as API + worker — module boundaries allow extracting services without re-architecting.* Sam fakt prowadzenia ADR-ów jest sygnałem; streszczenia pozwalają recenzentowi "skonsumować" decyzje bez otwierania plików.
4. **"Trade-offs — what this project deliberately does NOT have"** — Twój znak firmowy. Lista NO-features z ADR-001 + odrzucone wzorce (CQRS/ES) + rezygnacje jakościowe (multi-region, skala) — każde z uzasadnieniem i ścieżką rozbudowy. To sekcja, która odróżnia architekta od entuzjasty: entuzjasta chwali się tym, co wbudował; architekt potrafi obronić to, czego nie wbudował.
5. **"Failure scenarios"** — krótka tabela scenariuszy z etapu 03 (duplikat, timeout, poison, restart workera): co się dzieje, który test to udowadnia (link do pliku testu). Mało które repo to ma — a to najmocniejszy dowód kompetencji w całym projekcie.
6. **"Run locally in 5 minutes"** — wymagania (Docker, .NET), `docker compose up`, `dotnet run`, przykładowe żądania (plik `.http` lub `curl`), w tym jedno wywołanie demonstrujące awarię (np. wymuszony timeout płatności). Obietnica 5 minut musi być prawdą — przetestujesz to w praktyce poniżej.
7. **"Deploy to Azure"** — jeden akapit + link do `infra/`; sekcja "Running costs" z etapu 04.
8. **Stopka**: licencja, link do Twojego LinkedIn.

Plus elementy "higieniczne": badge CI u góry, screenshot trace'a/dashboardu przy sekcji observability, spis treści jeśli README przekracza ~2 ekrany.

Cały szkielet w pigułce (kolejność = priorytet recenzenta):

```markdown
# windtunnel-booking            [CI badge]
> Elevator pitch (2–3 zdania)

![C4 Container diagram](docs/images/c4-l2.png)

## Architecture decisions       → tabela ADR-ów z linkami
## Trade-offs — what this project deliberately does NOT have
## Failure scenarios            → tabela: scenariusz → zachowanie → test
## Run locally in 5 minutes
## Deploy to Azure              → infra/, Running costs
## License · Author
```

### Porządki przed publikacją

- **Język.** Cały folder capstone po angielsku — README, ADR-y, runbooki, komentarze w kodzie, komunikaty commitów go dotyczących. Przeczesz pozostałości polskich notatek z warsztatu: TODO w kodzie, teksty seedów/danych testowych, opisy w diagramach. Mieszanka językowa w publicznej wizytówce to dokładnie ten sygnał niechlujności, którego cały etap 05 ma uniknąć.
- **Sekrety.** Przeskanuj **całą historię**, nie tylko bieżący stan (`gitleaks` lub podobny skaner; GitHub push protection włączone). Sekret w historii commitów = sekret opublikowany. Jeśli coś znajdziesz: najprościej i najuczciwiej — unieważnij sekret u źródła (rotacja); przepisywanie historii to ostateczność.
- **Historia commitów.** Recenzent, który zajrzy w `git log`, ma zobaczyć inżyniera: komunikaty mówiące "co i po co" (nie "fix", "wip"). Nie przepisuj historii kosmetycznie na siłę — autentyczna, czytelna historia przyrostu projektu jest wiarygodniejsza niż jeden commit "initial". Jeśli historia jest naprawdę wstydliwa, świadomie zdecyduj (i już tak zostaw).
- **Licencja** — plik `LICENSE` (MIT to bezpieczny wybór dla portfolio; brak licencji = formalnie nikt nie może kodu użyć, co wygląda na niedopatrzenie).
- **Drobiazgi, które robią pierwsze wrażenie**: opis i topics repo na GitHubie (wypełnione = repo wygląda na utrzymywane), zielone CI, brak zakomentowanych zwałów kodu, `docs/` spójne z rzeczywistością (diagram zgodny z kodem!).
- **Nazwa folderu wizytówki** — teraz jest moment na finalną: krótka, opisowa, bez "test"/"demo2" (np. `windtunnel-booking` — nazwa jest częścią linku, który wisi w Featured).

### Publikacja: Featured, post, artykuł

**Featured na LinkedIn.** Dodaj link do folderu capstone do sekcji Featured z własnym opisem (nie zostawiaj surowego linku; GitHub renderuje README folderu, więc link działa jak strona projektu): 2 zdania pitchu + "see the Trade-offs section". Zrób porządek w Featured przy okazji: capstone na pierwszym miejscu, wizytówkę messagingową z modułu 2 możesz zostawić niżej albo usunąć — Featured to gablota, nie archiwum. Sprawdź też górę głównego README repo: angielska sekcja z linkami do wizytówek (lekcja 00/02) ma kierować prosto do capstone.

**Post podsumowujący** — najpierw decyzja językowa: **jeden język narracji w obrębie posta** — polski, jeśli Twoja sieć jest głównie polska, angielski, jeśli celujesz w zasięg międzynarodowy (terminy fachowe w oryginale w obu wariantach). Struktura, która działa (rytm znasz z modułu 0/5):
1. *Hak*: konkretna decyzja lub liczba, nie ogłoszenie. Źle: "Ukończyłem swój projekt!". Dobrze: "Mój system rezerwacji nie ma prawdziwych płatności. Celowo — i to była najlepsza decyzja architektoniczna w projekcie."
2. *Kontekst* (2–3 zdania): co zbudowałeś i po co (dowód kompetencji, domena z realnego doświadczenia).
3. *Mięso*: 3 decyzje z trade-offami w punktach (np. modular monolith zamiast mikroserwisów; symulacja zamiast Stripe = deterministyczne testowanie awarii; orchestration zamiast choreography — i co to kosztowało).
4. *Lekcja/puenta*: jedno zdanie o tym, czego projekt nauczył Cię o cięciu zakresu.
5. *CTA* (call to action): link do repo + zaproszenie do dyskusji ("która z tych decyzji budzi Wasz sprzeciw?").

**Artykuł techniczny (opcjonalny, dev.to / blog).** Nie streszczaj całego projektu — weź **jeden wątek z głębią**, np. "Saga compensation in practice: what happens when payment times out mid-booking" albo "Propagating trace context through a transactional outbox". Struktura: problem (z realnym kontekstem) → naiwne rozwiązanie i dlaczego pęka → właściwe rozwiązanie z kodem z repo → trade-offy → link do całości. 1200–2000 słów. Artykuł linkujesz potem z README i z LinkedIn — wszystko wskazuje na wszystko.

### Jak opowiadać o projekcie na rozmowie

Capstone wchodzi do Twojego zestawu war stories (moduł 6) jako historia w formacie STAR (Situation–Task–Action–Result), z jedną różnicą: tu sam byłeś sponsorem, architektem i zespołem, więc akcent kładziesz na **decyzje i ich obronę**:

- **Situation/Task**: "Chciałem mieć jeden publiczny dowód kompetencji z systemów rozproszonych i Azure. Wybrałem domenę, którą znam z produkcji — rezerwacje tunelu aerodynamicznego — bo mogłem przenieść realne reguły i realne tryby awarii."
- **Action**: 2–3 decyzje z trade-offami (te same, co w poście — spójność narracji to zaleta, nie lenistwo). Przygotuj się na drążenie: "a co by się stało, gdyby płatność jednak pobrała pieniądze przy timeout?" — masz to przemyślane w etapie 03.
- **Result**: działający system z testami scenariuszy awaryjnych + link. Najmocniejsze zdanie na koniec: **"najwięcej nauczyło mnie to, czego zdecydowałem się nie budować"** — i płynne przejście do sekcji Trade-offs.

Przećwicz wersję 5-minutową (pełna) i 90-sekundową (zaczepka, po której rozmówca sam dopyta) — nagraj się, jak w module 6.

## Praktyka

- [ ] Napisz README według anatomii powyżej (sekcje 1–8); diagramy i screenshoty z etapów 02 i 04 na miejscu.
- [ ] Test "5 minut na zimno": daj README osobie technicznej (albo odegraj recenzenta po dniu przerwy) — czy po 5 minutach umie powiedzieć, co system robi i jakie 2 decyzje podjąłeś? Popraw, co zgrzytało.
- [ ] Test "fresh clone": sklonuj repo do nowego katalogu na czystej maszynie/profilu i wykonaj własną instrukcję "run locally" z zegarkiem. Każde potknięcie = poprawka w README.
- [ ] Porządki: skan sekretów w całej historii, licencja, opis + topics na GitHubie, przegląd `git log`, finalna nazwa repo.
- [ ] Opublikuj: link do folderu capstone w Featured (z opisem), post podsumowujący wg struktury (hak → kontekst → 3 decyzje → puenta → CTA).
- [ ] Opcjonalnie: artykuł na dev.to o jednym wątku z głębią; podlinkuj z README i profilu.
- [ ] Przygotuj i nagraj opowieść o projekcie: wersja 5 min i 90 s; dopisz do zestawu war stories z modułu 6.
- [ ] Zaktualizuj `architect-track-plan.md` (odhacz Capstone) i `progress.md`.

## Artefakt

1. Opublikowana wizytówka capstone z README "rekruterskim" — podlinkowana w Featured na LinkedIn,
2. post podsumowujący na LinkedIn,
3. (opcjonalnie) artykuł techniczny,
4. nagrana opowieść o projekcie (5 min + 90 s) w repo nauki / notatkach do rozmów.

## Definition of Done

- [ ] "Test 5 minut" z README modułu capstone przechodzi w całości — punkt po punkcie, bez taryfy ulgowej.
- [ ] Fresh clone → działający system lokalnie w ≤ 5 minut, według samego README.
- [ ] Wizytówka w Featured, post opublikowany; w ciągu tygodnia odpowiadasz na każdy komentarz pod postem (dystrybucja to też praca).
- [ ] Opowieść 90-sekundowa wychodzi płynnie, bez notatek, i kończy się trade-offami, nie listą technologii.
- [ ] Pytanie kontrolne: gdybyś miał dziś usunąć z repo jedną rzecz, a dodać inną — co i dlaczego? (Jeśli nie masz odpowiedzi, nie przemyślałeś projektu do końca; jeśli masz — to świetne ostatnie zdanie na rozmowę.)

## Materiały

1. **Moduł 6 — Komunikacja architektoniczna** (war stories w formacie STAR, narracja na system design interview) — capstone jest głównym materiałem źródłowym tych historii.
2. **Moduł 1, lekcja 02 — Trade-off analysis** (`kurs/01-fundamenty-myslenia-architektonicznego/02-trade-off-analysis/`) — język, którym pisana jest sekcja "Trade-offs" w README.
