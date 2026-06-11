# Lekcja 02 — Code review pracy agenta: gdzie AI systematycznie błądzi

## Cel lekcji

Po tej lekcji będziesz robił review kodu pisanego przez agenta według konkretnej checklisty celującej w **typowe klasy błędów AI** (a nie te, które łapie review ludzkiego kodu) — i zaczniesz prowadzić log eksperymentów, który stanie się Twoim surowcem na posty i konkretami na rozmowy rekrutacyjne.

## Dlaczego to ważne

Najczęstszy błąd seniorów pracujących z agentami nie polega na zbyt małym zaufaniu, tylko na **zaufaniu skalibrowanym do ludzi**. Po 20 latach robienia review masz wytrenowaną intuicję, gdzie człowiek się myli: literówki, off-by-one, niedoczytany wymóg, lenistwo w testach. Agent myli się **inaczej** — pisze kod, który wygląda na napisany przez sumiennego mid-developera, kompiluje się, przechodzi testy (które sam napisał) i zawiera błędy klasy, której ludzkie review prawie nie spotyka: wywołania API, które nie istnieją, albo perfekcyjną obsługę przypadku, który nie może wystąpić, przy braku obsługi tego, który wystąpi w piątek wieczorem.

Na rozmowie na architekta to pytanie pada wprost: "AI pisze coraz więcej kodu — jak zapewniacie jakość?". Odpowiedź z konkretną taksonomią błędów i checklistą to odpowiedź architekta. Jest też zasada nadrzędna, którą warto umieć wypowiedzieć: **agent pisze, człowiek odpowiada**. Odpowiedzialność architektoniczna jest niedelegowalna — możesz delegować pisanie, nie możesz delegować odpowiedzialności za to, co weszło do systemu. Dokładnie tak, jak lead nie przestaje odpowiadać za produkcję dlatego, że kod napisał ktoś z zespołu.

## Teoria

### Dlaczego agent błądzi inaczej niż człowiek

LLM (large language model) generuje kod jako **najbardziej prawdopodobną kontynuację** w danym kontekście — nie wykonuje go w głowie, nie ma modelu Twojej produkcji, nie poniesie konsekwencji buga. Z tego jednego mechanizmu wynikają wszystkie klasy błędów poniżej: model ciąży ku temu, co *statystycznie typowe* w kodzie, na którym był trenowany, a Twój system jest specyficzny. Tam, gdzie typowość i specyfika się rozjeżdżają, powstaje błąd — ubrany w pewny siebie, dobrze sformatowany kod. To kluczowa różnica wobec juniora: junior, gdy nie wie, pisze kod *wyglądający niepewnie* albo pyta. Agent, gdy nie wie, pisze kod wyglądający dokładnie tak samo przekonująco jak wtedy, gdy wie.

### Typowe klasy błędów agenta — taksonomia

**1. Nadmierna ogólność i przeinżynierowanie.** Prosisz o deduplikację w jednym konsumerze, dostajesz `IDeduplicationStrategy`, fabrykę, trzy implementacje i opcje konfiguracyjne "na przyszłość". Mechanizm: w danych treningowych "dobry kod" często znaczy "wzorcowy kod z tutoriali", a tutoriale demonstrują wzorce. Skutek architektoniczny: przyrost powierzchni do utrzymania bez przyrostu wartości — czyli dokładnie to, z czym jako architekt walczysz u ludzi, tylko produkowane szybciej.

**2. Ignorowanie istniejących konwencji repo.** Repo używa Result pattern — agent rzuca wyjątki. Repo ma ustalony układ folderów — agent tworzy własny. Mechanizm: konwencje repo są w kontekście słabym sygnałem na tle miliardów linii "typowego" kodu z treningu. Mitygacja częściowa: CLAUDE.md i spec (lekcja 01); reszta zostaje na review. To błąd pozornie kosmetyczny, ale architektonicznie groźny: erozja konwencji to erozja czytelności systemu, po jednym sprincie z agentem bez review repo wygląda jak pisane przez pięć niekomunikujących się zespołów.

**3. Halucynowane API.** **Halucynacja** to generowanie treści brzmiącej wiarygodnie, lecz nieprawdziwej — w kodzie: metoda `ServiceBusClient.SendWithRetryAsync(...)`, która nie istnieje, pakiet NuGet o niemal poprawnej nazwie, flaga konfiguracyjna z innej wersji biblioteki. Mechanizm: model uzupełnia API tak, jak *powinno* wyglądać według wzorców nazewniczych. W kompilowanym .NET masz przewagę — większość halucynacji łapie kompilator. Groźniejsze są te, które przechodzą: poprawna składniowo metoda o **innej semantyce** niż agent zakłada (np. timeout, który nie obejmuje retry), wartości magic stringów, nazwy nagłówków, schematy JSON zewnętrznych API.

**4. Zbyt optymistyczna obsługa błędów.** Happy path wypieszczony; ścieżki awarii — `catch (Exception ex) { _logger.LogError(...); }` i jedziemy dalej, połknięty wyjątek, brak rozróżnienia błędów przejściowych (transient — warto powtórzyć) od trwałych (retry tylko pogorszy), brak timeoutów na wywołaniach sieciowych. Mechanizm: w danych treningowych obsługa błędów jest najczęściej szczątkowa, bo przykłady kodu ilustrują działanie, nie awarie. Po module 2 wiesz, że system definiuje się przez zachowanie w awarii — agent domyślnie projektuje system, który definiuje się przez happy path.

**5. Brak idempotencji i edge'ów współbieżności.** Agent napisze konsumera, który działa poprawnie dla jednego komunikatu dostarczonego raz. Duplikaty (at-least-once delivery), dwa wątki w tej samej sekcji, check-then-act bez blokady, brak transakcji wokół "zapisz i opublikuj" (dual write!) — te przypadki w kodzie agenta domyślnie **nie istnieją**, bo nie ma ich w typowym kodzie przykładowym. To najdroższa klasa błędów: niewidoczna w testach jednostkowych, na lokalnym środowisku i w demo; widoczna o 3 w nocy pod obciążeniem. Dlatego review pracy agenta w systemach rozproszonych to przede wszystkim review *tej* klasy.

**6. Testy testujące implementację, nie zachowanie.** Agent poproszony o testy ma dwie skłonności: (a) pisze testy *do kodu, który właśnie napisał* — więc testy utrwalają jego błędne założenia zamiast je wykrywać; (b) testuje strukturę zamiast kontraktu: mock na każdą zależność i asercje "metoda X została wywołana raz" zamiast "stan końcowy jest poprawny". Taki test przechodzi, daje zielony pasek i zero informacji — a przy refactoringu pęka, choć zachowanie się nie zmieniło. Mitygacja: kryteria testowe definiuj **w spec, przed kodem** (to jest sekcja definition of done z lekcji 01), i każ testować zachowanie obserwowalne z zewnątrz.

### Checklist review pracy agenta

Konkretna lista — przejdź ją przy każdym niebanalnym kawałku kodu od agenta. Kolejność celowa: od najtańszych sprawdzeń do najdroższych.

```
□ ZAKRES    Czy zmiana robi tylko to, o co prosiłem? Każdy plik dotknięty
            "przy okazji" → wyjaśnij albo wycofaj.
□ KONWENCJE Czy kod wygląda, jakby napisał go autor reszty repo?
            (nazewnictwo, struktura, styl błędów, logowanie)
□ API       Czy każde wywołanie zewnętrznego API naprawdę istnieje i znaczy to,
            co kod zakłada? Niezweryfikowane → otwórz dokumentację (nie pytaj
            o to agenta — sędzia nie może być oskarżonym).
□ ABSTRAKCJE Czy każdy interfejs/warstwa/opcja ma dziś drugiego klienta?
            Nie ma → do usunięcia (YAGNI).
□ AWARIE    Co się stanie, gdy: zależność nie odpowiada / odpowiada po 30 s /
            zwraca 500 / zwraca bzdurę? Czy błędy transient i trwałe są
            rozróżnione? Gdzie są timeouty?
□ WSPÓŁBIEŻNOŚĆ I DUPLIKATY  Co przy dwóch równoległych wykonaniach?
            Co przy ponownym dostarczeniu tego samego komunikatu?
            Czy "zapisz + opublikuj" jest atomowe?
□ TESTY     Czy testy opisują zachowanie (wejście → obserwowalny skutek),
            czy strukturę (wywołano metodę)? Czy istnieje test na przypadek
            brzegowy, który sam uważam za najgroźniejszy?
□ BEZPIECZEŃSTWO  Sekrety w kodzie? Dane wejściowe walidowane? Nowe pakiety
            NuGet — czy znam i akceptuję każdy?
□ ROZUMIEM  Czy umiem wyjaśnić każdą linię? Linia, której nie rozumiem,
            nie wchodzi. (To jest test niedelegowalności odpowiedzialności.)
```

Dwie zasady użycia. Po pierwsze, **review skaluj do ryzyka**: skrypt pomocniczy w repo nauki — szybki rzut oka; kod ścieżki płatności — pełna checklista plus uruchomienie z symulacją awarii. Po drugie, **wnioski z review wracają do systemu**: powtarzający się błąd → nowa reguła w CLAUDE.md lub nowy anty-cel w szablonie spec. Review, które nie zostawia śladu w procesie, to koszt; review, które poprawia spec, to inwestycja.

### Log eksperymentów: pamięć Twojego procesu

Bez zapisków za trzy miesiące będziesz "miał wrażenie", że agent radzi sobie z X, a wykłada na Y. Wrażenia nie nadają się ani na posty, ani na rozmowę rekrutacyjną — liczby i przykłady tak. **Log eksperymentów** to plik (`ai-log.md` w repo nauki), do którego po każdej istotnej delegacji dopisujesz wpis według szablonu:

```markdown
## 2026-06-15 — idempotent consumer w messaging-patterns
**Delegowałem:** implementacja dedup w PaymentRequestedConsumer (spec, 14 linii)
**Wyszło:** logika dedup poprawna za 1. podejściem; testy zachowań OK
**Poprawiałem:** (1) wymyślona metoda CreateIfNotExistsAsync na moim repozytorium
  → halucynacja API; (2) check-then-act bez unique constraint — race przy
  równoległych duplikatach → klasa: współbieżność
**Wniosek/zmiana procesu:** do CLAUDE.md dopisana reguła o unique constraints
  przy dedup; do checklisty nic — pozycja WSPÓŁBIEŻNOŚĆ złapała
**Czas:** spec 15 min + agent 25 min + review 20 min (vs moja estymata solo: 2,5 h)
```

Pięć pól, 2–4 minuty na wpis. Pole "Poprawiałem" taguj klasami błędów z taksonomii — po 10 wpisach zobaczysz swój rzeczywisty rozkład (u większości dominują #2 i #5) i będziesz wiedział, gdzie wzmacniać spec, a gdzie review. To jest dokładnie ten sam mechanizm, co metryki w observability: bez pomiaru optymalizujesz na ślepo. A każdy wpis to gotowy akapit posta z serii "honest takes" — posty pisane z logu mają konkrety, których nie da się wymyślić.

### Kiedy tego NIE robić (trade-offy)

- **Pełna checklista do wszystkiego** zabija przepustowość — całą oszczędność z delegowania oddasz w review. Skaluj głębokość do ryzyka i nieodwracalności zmiany, jak przy ludzkich PR-ach.
- **Zero zaufania = zero sensu.** Jeśli każdą linię i tak piszesz w głowie od nowa, delegacja jest teatrem. Celem jest skalibrowane zaufanie: pełna kontrola na architekturze i ścieżkach awarii, swoboda agenta w boilerplate.
- **Log nie może być pamiętnikiem.** Wpisy po 2 strony umrą w drugim tygodniu. Pięć pól, twardy limit — jak przy retro: format, który przeżyje zmęczenie, bije format idealny.

## Praktyka

- [ ] Przepisz checklistę z tej lekcji do własnego pliku (`ai-review-checklist.md`) i **dostosuj**: usuń/dodaj pozycje na bazie własnych doświadczeń z Claude Code przy DayChunks. Minimum jedna pozycja musi być Twoja.
- [ ] Zrób pełne review wg checklisty dla najbliższego zadania delegowanego agentowi w repo messaging-patterns (idealnie: kod z lekcji 02–05 modułu 2). Zapisz, które pozycje coś złapały.
- [ ] Celowo sprowokuj klasę #5: poproś agenta (bez podpowiedzi w spec) o konsumera komunikatu zapisującego coś do bazy. Sprawdź, czy obsłużył duplikaty i równoległość. Wynik → log.
- [ ] Załóż `ai-log.md` z szablonem wpisu i dodaj pierwsze 3 wpisy (jednym z nich może być porównanie A/B z lekcji 01).
- [ ] Napisz i opublikuj pierwszy post serii "AI-augmented engineering — honest takes" na bazie najlepszego wpisu z logu (np. o znalezionej halucynacji API albo o brakującej idempotencji).

## Artefakt

1. **`ai-review-checklist.md`** — Twoja checklista review pracy agenta (wersjonowana — będzie ewoluować do końca kursu).
2. **`ai-log.md`** — log eksperymentów z szablonem i ≥3 wpisami; prowadzony dalej przez cały kurs.
3. **Pierwszy post** z serii "honest takes" — opublikowany.

## Definition of Done

- [ ] Checklista istnieje, ma ≥1 pozycję autorską i została użyta na realnym kodzie agenta co najmniej dwa razy.
- [ ] Umiesz wymienić z pamięci 6 klas błędów agenta i dla każdej podać przykład **z własnej praktyki** (to jest dokładnie odpowiedź na pytanie rekrutacyjne "gdzie AI błądzi?").
- [ ] Log ma ≥3 wpisy z tagami klas błędów i co najmniej jedna obserwacja z logu zmieniła Twój proces (wpis w CLAUDE.md, spec-template lub checkliście).
- [ ] Pierwszy post serii opublikowany.
- [ ] Potrafisz uzasadnić zdanie "odpowiedzialność architektoniczna jest niedelegowalna" w 60 sekund, z przykładem.

## Materiały

1. [Google — "How to do a code review" (eng-practices)](https://google.github.io/eng-practices/review/reviewer/) — kanon rzemiosła review; czytaj z pytaniem "które z tych praktyk zmieniają znaczenie, gdy autorem jest agent?".
2. [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — przegląd ryzyk pracy z LLM; na tę lekcję wystarczy sekcja o nadmiernym poleganiu na outputach (overreliance), reszta wróci w lekcji 04.
