# Lekcja 01 — Spec-driven development: specyfikacja zamiast promptu

## Cel lekcji

Po tej lekcji będziesz delegował zadania agentowi przez pisemną specyfikację (kontekst, cel, ograniczenia, definition of done, anty-cele) zamiast przez rozmowę ad-hoc — i będziesz miał własny szablon spec oraz plik sterujący (CLAUDE.md) dla repo, w którym pracujesz.

## Dlaczego to ważne

Używasz Claude Code codziennie, więc znasz ten scenariusz: prosisz o zmianę, agent robi coś *prawie* dobrego, doprecyzowujesz, agent poprawia i psuje coś innego, po czterech iteracjach masz wynik — i wrażenie, że sam napisałbyś to szybciej. To nie jest wada agenta. To wada **sposobu delegowania**.

Na rozmowie na architekta pytanie "jak używacie AI?" jest dziś standardem. Odpowiedź "piszę prompty" brzmi jak każdy. Odpowiedź "piszę specyfikacje z definition of done i anty-celami, utrzymuję pliki reguł per repo, a duże zadania tnę na etapy z punktami kontrolnymi" — brzmi jak ktoś, kto zaprojektował proces. Jest też głębszy powód: umiejętność pisania dobrych specyfikacji dla agenta to **dokładnie ta sama umiejętność**, co pisanie dobrych zadań dla zespołu. Architekt większość pracy wykonuje cudzymi rękami — ludzkimi lub agentowymi. Ta lekcja trenuje delegowanie na materiale, który masz pod ręką.

## Teoria

### Problem: prompt to rozmowa, spec to kontrakt

**Prompt ad-hoc** to wypowiedź w rozmowie: "dodaj retry do tego konsumera". Cała reszta — jaka polityka retry, co z poison message, czy logować, jakim wzorcem — zostaje w Twojej głowie. Agent musi te luki **zgadnąć**, a zgaduje na podstawie statystycznie typowego kodu, nie Twojego projektu. Każda błędnie zgadnięta luka to jedna iteracja poprawek.

**Specyfikacja** to kontrakt: dokument (zwykle plik `.md` w repo albo długa, ustrukturyzowana wiadomość), który domyka luki *przed* startem pracy. Różnica jest taka jak między "pogadajmy o integracji" a podpisanym interface contract między systemami: rozmowa jest tania i wieloznaczna, kontrakt kosztuje chwilę z góry i jest jednoznaczny.

Naiwne rozwiązanie problemu iteracji brzmi: "będę pisał dłuższe prompty". Dlaczego to się psuje: długi prompt bez struktury to nadal rozmowa — agent nie wie, co w nim jest wymaganiem twardym, co sugestią, a co kontekstem. Właściwy wzorzec to struktura, w której każda sekcja ma inną *moc wiążącą*.

Analogia, która powinna Ci być bliska: **spec dla agenta ≈ zadanie dla juniora**. Junior z taskiem "dodaj retry" przyjdzie pięć razy z pytaniami albo — gorzej — nie przyjdzie i zrobi po swojemu. Junior z taskiem zawierającym kontekst, kryteria akceptacji i wyraźne "czego NIE ruszać" dowiezie za pierwszym podejściem. Im lepsze zadanie, tym mniej iteracji. Agent jest jak bardzo szybki, bardzo oczytany junior **bez pamięci o Twoim projekcie i bez możliwości złapania Cię na korytarzu** — więc spec musi zawierać wszystko, co junior wiedziałby z onboardingu.

Trade-off jest oczywisty i trzeba go nazwać uczciwie: spec kosztuje 10–20 minut z góry. Rachunek się opłaca, gdy zadanie jest większe niż trywialne (patrz: "kiedy tego NIE robić" na końcu).

### Anatomia dobrej specyfikacji dla agenta

Pięć sekcji, każda odpowiada na inne pytanie:

1. **Kontekst** — *w jakim świecie pracujesz*. Co to za repo/moduł, jakie konwencje obowiązują, jakie decyzje już zapadły i są nienegocjowalne ("używamy EF Core, outbox już istnieje w `Infrastructure/Outbox`, broker to RabbitMQ"). Bez kontekstu agent odtworzy "statystycznie typowy" projekt, nie Twój.
2. **Cel** — *co ma być prawdą po zakończeniu*, opisane od strony zachowania, nie implementacji. "Konsument `PaymentRequestedConsumer` ma przetwarzać duplikaty komunikatów bez podwójnego obciążenia" jest lepsze niż "dodaj tabelę dedup", bo zostawia agentowi przestrzeń na dobrą implementację, a Tobie daje kryterium oceny.
3. **Ograniczenia** — *jakimi ścieżkami wolno iść*. Technologie, wzorce, granice zmian ("nie dodawaj nowych pakietów NuGet", "zmiany tylko w projekcie `Payments.Worker`", "styl błędów: Result pattern, nie wyjątki kontrolne").
4. **Definition of done** — *po czym poznasz, że skończone*. Lista sprawdzalna mechanicznie: "testy X, Y przechodzą", "`docker compose up` + skrypt symulacji duplikatów pokazuje jedno obciążenie", "README zaktualizowane o sekcję dedup". To jest sekcja, która najbardziej redukuje iteracje — agent sam weryfikuje pracę przed oddaniem.
5. **Anty-cele** — *czego NIE robić*. Najbardziej niedoceniana sekcja. Agenci mają silną skłonność do "ulepszania przy okazji": refactoring sąsiedniego kodu, dodanie warstwy abstrakcji, wymiana biblioteki na "lepszą". Anty-cele to jawne zakazy: "NIE refaktoruj istniejących konsumerów", "NIE wprowadzaj generycznego frameworka dedup — to jedno miejsce", "NIE zmieniaj formatu logów". Każdy anty-cel to jeden przyszły rollback mniej.

Praktyczna heurystyka kompletności: przeczytaj spec i zapytaj siebie "czy senior, który nigdy nie widział tego repo, dowiózłby to bez jednego pytania do mnie?". Jeśli nie — luka, którą znajdziesz, to luka, w którą wpadnie agent.

Przykład — kompletny spec realnego rozmiaru (zauważ: ~20 linii, nie elaborat):

```markdown
# Spec: deduplikacja w PaymentRequestedConsumer

## Kontekst
Repo messaging-patterns. RabbitMQ daje at-least-once delivery — duplikaty
PaymentRequested zdarzają się przy redelivery. Persystencja: EF Core + Postgres.
Konwencje repo opisane w CLAUDE.md (Result pattern, jeden handler = jeden plik).

## Cel
Ponowne dostarczenie tego samego PaymentRequested (ten sam MessageId)
nie skutkuje drugim obciążeniem. Dotyczy też dwóch duplikatów RÓWNOLEGLE.

## Ograniczenia
- Zmiany tylko w Payments.Worker; dedup oparty o MessageId + unique constraint.
- Bez nowych pakietów NuGet. Migracja przez `dotnet ef migrations add`.

## Definition of done
- Test: dwukrotne przetworzenie tego samego komunikatu → jedno obciążenie.
- Test: dwa równoległe przetworzenia → jedno obciążenie (bez wyjątku u klienta).
- `docker compose up` + scripts/simulate-duplicates.sh pokazuje poprawny wynik.

## Anty-cele
- NIE buduj generycznego mechanizmu dedup dla wszystkich konsumerów.
- NIE refaktoruj istniejących handlerów ani formatu logów.
```

### Plany wieloetapowe i punkty kontrolne

Spec działa świetnie dla zadań na 1–3 godziny pracy agenta. Dla większych (np. "zaimplementuj sagę rezerwacji z kompensacją") pojedynczy spec pęka: za dużo decyzji naraz, a błąd w kroku 1 zatruwa kroki 2–5, zanim go zobaczysz.

Wzorzec: **plan wieloetapowy z punktami kontrolnymi** (checkpoint). Najpierw każesz agentowi *zaplanować*, nie kodować: "rozpisz implementację na etapy, każdy etap ma być osobno weryfikowalny; nie pisz jeszcze kodu". Plan czytasz i poprawiasz — to jest moment, w którym łapiesz błędne założenia tanio (zmiana zdania w planie kosztuje zero, zmiana w kodzie kosztuje godziny). Potem wykonujesz etapy pojedynczo, a po każdym jest checkpoint: testy przechodzą, Ty robisz przegląd, dopiero wtedy "jedziemy z etapem 2".

Powinno Ci to coś przypominać: to są **kamienie milowe i quality gates** z normalnego prowadzenia projektu. Nic nowego — nowe jest tylko to, że cykl trwa godziny, nie tygodnie. Drugi powód techniczny: agent ma **context window** (okno kontekstu — ograniczoną ilość tekstu, którą model "widzi" naraz; o tym szczegółowo w lekcji 04). W bardzo długiej sesji wcześniejsze ustalenia wypadają z okna lub są kompresowane i agent zaczyna "zapominać" reguły z początku rozmowy. Etapy + spec w pliku (a nie w historii czatu) są na to odporne: każdą nową sesję zaczynasz od "przeczytaj `spec.md` i `plan.md`, jesteśmy na etapie 3".

### Pliki sterujące: CLAUDE.md jako architektura współpracy z agentem

Część kontekstu jest stała dla całego repo: konwencje nazewnicze, struktura katalogów, komendy budowania i testów, decyzje architektoniczne, rzeczy zakazane. Wklejanie tego do każdego spec to duplikacja — a Ty wiesz, co duplikacja robi z utrzymaniem.

Rozwiązanie: **plik sterujący** — w Claude Code to `CLAUDE.md` w korzeniu repo (inne narzędzia mają odpowiedniki, np. pliki reguł projektu), czytany automatycznie na starcie każdej sesji. Traktuj go jak **onboarding doc dla nieskończonego strumienia nowych członków zespołu**: każda sesja agenta to "nowy pracownik pierwszego dnia", a CLAUDE.md to wszystko, co musi wiedzieć, zanim dotknie kodu.

Co dobry CLAUDE.md zawiera (i czego nie):

- **Tak:** komendy (build, test, run — dokładne, działające), mapę struktury repo, konwencje ("handlery w `Application/Handlers`, jeden plik = jeden handler"), twarde zakazy ("nigdy nie commituj bez przejścia testów", "nie dotykaj migracji ręcznie"), odsyłacze do ADR-ów.
- **Nie:** rzeczy zmienne per zadanie (to idzie do spec), długie traktaty filozoficzne (agent gubi sygnał w szumie), kopie dokumentacji frameworków (agent je zna lepiej z treningu).

To jest dosłownie praca architektoniczna: projektujesz **interfejs między człowiekiem a agentem** i utrzymujesz go jak każdy inny kontrakt — gdy agent popełnia ten sam błąd drugi raz, poprawka idzie do CLAUDE.md, nie do czatu. Mechanizm identyczny jak post-mortem → runbook: incydent ma zostawiać trwały ślad w systemie, nie w pamięci operatora.

### Kiedy tego NIE robić (trade-offy)

- **Zadania trywialne i jednorazowe.** "Zmień nazwę pola i popraw użycia" nie potrzebuje spec — koszt kontraktu przekracza koszt iteracji. Heurystyka: spec pisz, gdy spodziewasz się >30 minut pracy agenta albo gdy zadanie dotyka logiki, za którą odpowiadasz.
- **Eksploracja.** Gdy sam nie wiesz, czego chcesz ("jak można by podejść do..."), rozmowa ad-hoc jest *właściwym* narzędziem — spec pisany przy mglistym celu to fałszywa precyzja. Najpierw eksploruj czatem, potem skondensuj wnioski w spec.
- **Przeinżynierowany szablon.** Spec z 15 sekcjami, którego nikt nie wypełnia, umiera jak każdy proces-zombie. Pięć sekcji z tej lekcji to maksimum; dla średnich zadań wystarczą cel + ograniczenia + DoD + anty-cele w 15 linijkach.

## Praktyka

Ćwiczenie wykonujesz na **realnym zadaniu z repo messaging-patterns** (moduł 2, lekcja 08) — wybierz coś, co i tak masz do zrobienia, np. idempotent consumer, obsługę DLQ albo symulator duplikatów.

- [ ] Wybierz dwa porównywalne zadania z repo (podobny rozmiar, ~1–2h pracy). Zadanie A zrób z agentem **ad-hoc**: jedna prośba na czacie, doprecyzowuj w trakcie, jak zwykle. Zanotuj: liczba iteracji, czas łączny, co musiałeś poprawić ręcznie.
- [ ] Dla zadania B napisz **spec** według pięciu sekcji (kontekst, cel, ograniczenia, definition of done, anty-cele) — zanim otworzysz sesję agenta. Limit: 20 minut na spec.
- [ ] Wykonaj zadanie B przez spec (wklej/wskaż plik spec jako całe zadanie). Zanotuj te same metryki co dla A.
- [ ] Napisz `CLAUDE.md` dla repo messaging-patterns: komendy, struktura, konwencje, zakazy. Maksymalnie 60 linii — zwięzłość to feature.
- [ ] Spisz porównanie A vs B (pół strony: liczby + 3 obserwacje) — to gotowy szkic posta "Spec zamiast promptu: jeden tydzień, dwa podejścia".
- [ ] Wyciągnij ze spec zadania B **szablon wielokrotnego użytku** (`spec-template.md` w repo nauki) i używaj go we wszystkich kolejnych zadaniach modułów 2–4.

## Artefakt

1. **`spec-template.md`** — Twój szablon specyfikacji (5 sekcji) w repo nauki.
2. **`CLAUDE.md`** w repo messaging-patterns — działający plik sterujący.
3. **Notatka porównawcza ad-hoc vs spec** z liczbami — surowiec na post i pierwszy wpis do logu eksperymentów (lekcja 02).

## Definition of Done

- [ ] Oba zadania (A i B) zostały realnie wykonane i masz zmierzone: iteracje, czas, zakres ręcznych poprawek.
- [ ] `spec-template.md` istnieje i zawiera wszystkie 5 sekcji z krótką instrukcją wypełniania.
- [ ] `CLAUDE.md` w repo messagingowym istnieje, a nowa sesja agenta po jego wprowadzeniu przestrzega co najmniej jednej konwencji, którą wcześniej łamała.
- [ ] Umiesz w 2 minuty wyjaśnić różnicę prompt vs spec i uzasadnić sekcję anty-celów przykładem z własnej praktyki.
- [ ] Kolejne zadanie delegowane agentowi (już poza ćwiczeniem) zrobiłeś przez spec — nawyk, nie jednorazowy eksperyment.

## Materiały

1. [Anthropic — Claude Code: Best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices) — źródło pierwotne o plikach CLAUDE.md, planowaniu przed kodowaniem i workflow z agentem.
2. [GitHub — Spec Kit](https://github.com/github/spec-kit) — otwarte narzędzie i metodyka spec-driven development; nawet jeśli nie użyjesz narzędzia, struktura ich specyfikacji to dobry punkt odniesienia dla własnego szablonu.
