# Lekcja 05 — Partycjonowanie, kolejność zdarzeń i dead-letter handling

## Cel lekcji

Po tej lekcji będziesz umiał świadomie wybrać **klucz partycji**, wyjaśnić, dlaczego globalna kolejność zdarzeń w systemie rozproszonym nie istnieje (i jak żyć z kolejnością per klucz), oraz zaprojektować pełną obsługę **poison messages**: politykę retry, dead-letter queue i — co najważniejsze — **proces** wokół DLQ, nie tylko kolejkę. To lekcja, w której nazwiesz najwięcej swoich nocnych blizn z produkcji.

## Dlaczego to ważne

To są trzy tematy, którymi systemy kolejkowe budzą ludzi w nocy. Poison message zapętlający konsumenta, zdarzenia przetworzone w złej kolejności ("anulowanie przyszło przed utworzeniem"), jedna gorąca partycja dusząca cały przepływ — jeśli przepracowałeś lata przy kolejkach w Perfect Gym, znasz to wszystko z autopsji. Na rozmowach "jak zapewnisz kolejność zdarzeń?" i "co robisz z wiadomością, która zawsze się wywala?" to pytania-filtry: odpowiedź "DLQ" wystarcza juniorowi; architekt opisuje politykę retry przed DLQ i proces obsługi po niej.

## Teoria

### Partycjonowanie: po co i jak

**Partycjonowanie** (*partitioning*, w Kafce: partycje, w Service Bus: sesje/partitioned entities, w bazach: sharding) to podział strumienia danych na niezależne kawałki, żeby można je było przetwarzać **równolegle**. Jedna kolejka — jeden konsument na raz przetwarza w porządku = sufit przepustowości. Dziesięć partycji = dziesięciu konsumentów pracujących równolegle.

Do której partycji trafia wiadomość, decyduje **klucz partycji** (*partition key*): broker liczy `hash(klucz) % liczba_partycji`. Wszystkie wiadomości z tym samym kluczem trafiają do **tej samej partycji** — i to jest cała magia, bo wewnątrz partycji broker zachowuje kolejność.

**Jak wybrać klucz** — dwa kryteria, często w konflikcie:

1. **Kolejność**: klucz = tożsamość bytu, którego zdarzenia muszą być uporządkowane. Zdarzenia jednego zamówienia (`OrderId`) muszą iść po kolei → klucz = `OrderId`. Zdarzenia jednego klubowicza → `MemberId`.
2. **Równomierność rozkładu**: klucz musi mieć dużo wartości o w miarę równym ruchu. Klucz = `ClubId`, gdy jeden klub generuje 60% ruchu → **hot partition** (gorąca partycja): jedna partycja zapchana, konsument przy niej ledwo zipie, pozostałe się nudzą — a skalowanie liczby konsumentów nic nie daje, bo wąskim gardłem jest jedna partycja. Hot partition rozpoznasz po metrykach: lag (zaległość) rośnie na jednej partycji przy pustych pozostałych.

Reguła: **wybieraj najdrobniejszy klucz, który jeszcze zachowuje wymaganą kolejność**. Potrzebujesz kolejności per zamówienie? Kluczuj po `OrderId`, nie po `CustomerId` (klient może mieć wiele zamówień — niepotrzebnie sklejasz ich losy). Uwaga praktyczna: zmiana liczby partycji przetasowuje przypisanie kluczy — planuj liczbę partycji z zapasem od początku.

### Kolejność zdarzeń: globalna nie istnieje

Twarda prawda do wyartykułowania na głos: **w systemie rozproszonym nie ma globalnej kolejności zdarzeń**. Powody się kumulują: wielu producentów (zegary maszyn się różnią — *clock skew* — więc timestampy nie rozstrzygają), wiele partycji (broker porządkuje tylko wewnątrz partycji), retry (wiadomość wraca później, "przeskoczona" przez młodsze), konkurencyjni konsumenci (prefetch i równoległość mieszają porządek przetwarzania nawet z jednej kolejki).

Co realnie dostajesz: **kolejność per partycja**, czyli — przy dobrym kluczu — **kolejność per byt biznesowy**. I w 95% przypadków dokładnie tyle potrzeba: nie obchodzi Cię, czy zdarzenie zamówienia A wyprzedziło zdarzenie zamówienia B; obchodzi Cię, żeby `OrderCancelled` nie przetworzyło się przed `OrderPlaced` **tego samego** zamówienia. Po stronie konsumenta: w obrębie partycji/sesji przetwarzaj sekwencyjnie (jeden wątek na partycję), równoległość bierz z liczby partycji.

**Co robić, gdy konsument mimo wszystko dostaje out-of-order** (bo dostanie — choćby po retry):

- **Wersjonowanie stanu**: byt ma numer wersji/sekwencji; zdarzenie z wersją ≤ ostatnio zastosowanej → zignoruj (to uogólniona idempotencja z lekcji 01: nie tylko "czy już widziałem", ale "czy to nie jest starsze niż mój stan").
- **Bufor i dosztukowanie** (*resequencer*): jeśli przyszła wersja N+2, a czekasz na N+1 — odłóż na chwilę. Działa, ale dodaje opóźnienie i stan do zarządzania; stosuj oszczędnie.
- **Projekt zdarzeń odporny na kolejność**: zdarzenia niosące pełny stan ("status zamówienia = Cancelled, wersja 7") zamiast delt ("zmień status") — wtedy wystarczy "ostatnia wersja wygrywa".
- **Reguły biznesowe tolerujące odwrotność**: `OrderCancelled` przed `OrderPlaced`? Zapisz "tombstone" anulowania i gdy przyjdzie `OrderPlaced` — od razu anuluj. Brzmi dziwnie, bywa najprostsze.

### Poison messages i dead-letter queue

**Poison message** (wiadomość trująca) to wiadomość, której przetworzenie **zawsze** kończy się błędem: zdeformowany JSON, payload łamiący założenia kodu, odwołanie do rekordu, którego nie ma, zdarzenie ze starej wersji kontraktu. Cechą definiującą jest **deterministyczność błędu** — retry nic nie zmieni.

Naiwne zachowanie (i domyślne w wielu konfiguracjach): błąd → nack → wiadomość wraca na początek kolejki → błąd → … Konsument mieli w kółko tę samą wiadomość, **blokując wszystko za nią** (head-of-line blocking), paląc CPU i zalewając logi. Jeśli kiedyś o 3 w nocy patrzyłeś na kolejkę, która "stoi", choć konsument "działa na 100% CPU" — to był ten scenariusz.

**Dead-letter queue (DLQ)** — kolejka umarłych listów (termin z poczty: przegródka na listy niemożliwe do doręczenia). Boczna kolejka, do której broker lub konsument odkłada wiadomości, których nie udało się przetworzyć po wyczerpaniu prób. Wiadomość nie ginie (to ważne — at-least-once dalej obowiązuje!), ale przestaje blokować przepływ. RabbitMQ: dead letter exchange; Azure Service Bus: wbudowana sub-kolejka `$DeadLetterQueue` per kolejka, z powodem (`DeadLetterReason`) w nagłówkach.

**Polityka retry przed DLQ** — klucz to odróżnienie dwóch klas błędów:

- **Przejściowe** (*transient*): timeout bazy, deadlock, chwilowy brak sieci, 503 z zależności → **retry ma sens**, z odstępami (backoff — szczegóły w lekcji 06).
- **Trwałe** (*permanent*): deserializacja, walidacja, naruszenie reguły biznesowej → **retry to strata czasu i opóźnianie nieuniknionego** → od razu do DLQ.

Rozsądny domyślny pipeline: wyjątek klasyfikowany jako trwały → natychmiast DLQ z powodem; przejściowy → np. 3 próby z rosnącym odstępem (często przez *delayed redelivery* — wiadomość wraca na kolejkę z opóźnieniem, nie blokując innych) → po wyczerpaniu → DLQ. Limit prób musi być **skończony i mały** — "retry forever" to poison loop w przebraniu.

### DLQ to proces, nie kolejka

Najczęstszy grzech produkcyjny: DLQ skonfigurowana, odhaczona… i nigdy nie oglądana. DLQ bez procesu to **cmentarz danych** — wiadomości tam to utracone operacje biznesowe (niezaksięgowane wejście, niewysłane powiadomienie), tylko rozłożone w czasie. Dojrzała obsługa DLQ:

1. **Monitoring i alert**: metryka "liczba wiadomości w DLQ" z alertem przy > 0 (lub progu) — DLQ niepusta to incydent o niskim priorytecie, nie folklor.
2. **Diagnoza**: wiadomość w DLQ musi nieść kontekst — powód, stack trace / identyfikator błędu, liczbę prób, correlation id (lekcja 07) — żeby dało się ustalić przyczynę bez archeologii.
3. **Decyzja per wiadomość/grupa**: **redrive** (zwróć na kolejkę główną po naprawie buga/danych — broker lub własne narzędzie), **napraw i wyślij ponownie** (np. uzupełnij brakujący rekord), albo **odrzuć świadomie** (z decyzją biznesową i zapisem dlaczego).
4. **Pętla doskonalenia**: każda kategoria wiadomości w DLQ to wskazówka — brak walidacji u producenta, luka w kontrakcie, brakujący przypadek w kodzie. DLQ to darmowy backlog jakości.

Pytanie kontrolne do każdego designu: *"kto, kiedy i czym przegląda DLQ — i jak wygląda redrive?"*. Brak odpowiedzi = brak obsługi błędów, tylko z dodatkowym krokiem.

### Trade-offy i "kiedy tego NIE używać"

- **Więcej partycji ≠ lepiej**: partycje kosztują (zasoby brokera, ogony latencji, rebalansowanie konsumentów) i utrudniają operacje globalne. Dobieraj do realnej przepustowości z zapasem, nie "na zaś" ×100.
- **Kolejność jest droga**: wymóg kolejności tam, gdzie biznesowo zbędna, zabija równoległość (skrajność: globalna kolejność = jedna partycja = jeden konsument). Zawsze pytaj: "czy te dwa zdarzenia naprawdę muszą być uporządkowane względem siebie?".
- **DLQ nie zastępuje idempotencji ani walidacji**: to siatka bezpieczeństwa na końcu, nie strategia jakości. Jeśli do DLQ regularnie spada duży procent ruchu — problem jest wcześniej.
- Automatyczny, bezmyślny redrive całej DLQ co godzinę = poison loop w zwolnionym tempie. Redrive po naprawie przyczyny, nie zamiast niej.

## Praktyka

- [ ] Dla przepływu zamówienie → płatność → powiadomienie (Twój mini-projekt z lekcji 08) wybierz klucze partycji dla każdego strumienia zdarzeń i uzasadnij pisemnie (jakiej kolejności wymagam? jaki rozkład ma klucz? co byłoby hot partition?).
- [ ] Rozpisz tabelę klasyfikacji błędów dla jednego ze swoich realnych konsumentów: 6–10 typów wyjątków × przejściowy/trwały × polityka (ile prób, jaki odstęp, kiedy DLQ).
- [ ] Napisz **runbook obsługi DLQ** (1 strona): alert → jak obejrzeć wiadomość i powód → drzewo decyzyjne (redrive / napraw / odrzuć) → jak wykonać redrive → co zanotować po incydencie. Pisz tak, żeby ktoś obudzony o 3 w nocy mógł z tego skorzystać — to jest test jakości runbooka.
- [ ] Spisz z pamięci jeden prawdziwy incydent "poison message / kolejka stoi" z Twojej kariery: objawy → fałszywe tropy → przyczyna → fix → czego brakowało w systemie. To surowiec na post.

## Artefakt

Folder `05-partycjonowanie-kolejnosc-dead-letter/` w repo nauki: uzasadnienie kluczy partycji, tabela klasyfikacji błędów, runbook DLQ (wejdzie żywcem do repo mini-projektu) oraz **post "Czego nauczył mnie poison message o 3 w nocy"** — flagowy post serii "production engineering" (struktura: scena z nocy → co to jest poison message → naiwna obsługa i czemu pali system → retry policy + DLQ → "DLQ to proces, nie kolejka"). Opublikuj w tym tygodniu.

## Definition of Done

- [ ] Umiesz wyjaśnić, czemu globalna kolejność nie istnieje, i podać minimum dwie strategie obsługi out-of-order u konsumenta.
- [ ] Dla dowolnego strumienia zdarzeń umiesz w minutę zaproponować klucz partycji i wskazać, kiedy stanie się hot partition.
- [ ] Twoja polityka retry rozróżnia błędy przejściowe od trwałych i ma skończony limit prób; umiesz uzasadnić każdy parametr.
- [ ] Runbook DLQ istnieje i odpowiada na pytanie "kto, kiedy, czym i co potem".
- [ ] Post o poison message opublikowany.

## Materiały

1. **"Designing Data-Intensive Applications"**, M. Kleppmann — rozdz. 6 ("Partitioning") + fragmenty rozdz. 11 o porządkowaniu zdarzeń.
2. **Dokumentacja Azure Service Bus — [Dead-letter queues](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-dead-letter-queues)** — konkretny, dobrze opisany model DLQ z powodami i redrive; przyda się wprost w module 3.
