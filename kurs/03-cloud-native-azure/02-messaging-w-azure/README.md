# Lekcja 02 — Messaging w Azure: Service Bus, Event Grid, Event Hubs, Storage Queues

## Cel lekcji

Po tej lekcji rozróżniasz cztery usługi messagingowe Azure, wiesz, do jakiej klasy problemów każda służy, i umiesz zmapować wzorce z modułu 2 (outbox, saga, DLQ, retry) na konkretne mechanizmy tych usług. Efekt: dla danego przepływu komunikatów wskazujesz usługę (lub kombinację) z uzasadnieniem.

## Dlaczego to ważne

Azure ma cztery usługi, które z daleka wyglądają jak "kolejka" — i wybór złej to klasyczny błąd, który odkrywa się na produkcji ("dlaczego Event Grid nie trzyma kolejności?", "dlaczego Storage Queue nie ma topiców?"). Pytanie "Service Bus czy Event Hubs?" to standard rozmów na role architektoniczne w stacku Microsoft. Masz przewagę: moduł 2 dał ci język wzorców (at-least-once, DLQ, ordering, idempotencja) — ta lekcja tylko przykleja do tych pojęć nazwy usług i ich ograniczenia. Większość kandydatów idzie odwrotnie: zna nazwy, nie rozumie wzorców.

## Teoria

### Najpierw rozróżnienie fundamentalne: messages vs events vs streams

Trzy klasy problemów, które łatwo pomylić, bo wszystkie polegają na "wysyłaniu czegoś":

- **Message (komunikat/polecenie):** nadawca **oczekuje przetworzenia**. "Zrealizuj zamówienie #123". Zguba = strata biznesowa. Potrzebujesz: gwarancji dostarczenia, retry, DLQ, czasem kolejności i transakcji. → **Service Bus** (lub Storage Queues w wersji minimum).
- **Event (zdarzenie/fakt):** nadawca **ogłasza, że coś się stało** i nie obchodzi go, kto słucha. "Blob został wgrany", "zamówienie utworzone". Konsumentów może być zero lub dziesięciu. Potrzebujesz: taniego rozgłaszania, filtrowania, push do subskrybentów. → **Event Grid**.
- **Stream (strumień):** **ciągły, masowy** napływ danych — telemetria, clickstream, logi. Liczy się przepustowość i możliwość ponownego odczytu, nie pojedyncza wiadomość. → **Event Hubs**.

Na rozmowie samo rozpoczęcie odpowiedzi od tego podziału ("najpierw ustalmy, czy to command, event czy stream") ustawia cię od razu na poziomie architekta.

### Azure Service Bus — enterprise broker, twój domyślny wybór dla komunikatów

**Co to jest:** pełnoprawny message broker (klasa RabbitMQ / IBM MQ) jako usługa zarządzana. Wszystkie wzorce z modułu 2 mają tu natywne wsparcie:

- **Queues** — point-to-point: jeden logiczny odbiorca, konkurujący konsumenci (competing consumers) skalują przetwarzanie. Tryb odbioru *peek-lock*: wiadomość jest blokowana, nie usuwana; consumer przetwarza i wywołuje `Complete` (sukces) albo `Abandon` (wraca do kolejki). To jest mechanizm at-least-once delivery, który w module 2 obsługiwałeś idempotencją — ona nadal jest twoja.
- **Topics + subscriptions** — pub/sub: publikujesz na topic, każda subskrypcja dostaje własną kopię, z **filtrami** (SQL-owe warunki na właściwościach wiadomości). Jeden topic `orders`, subskrypcje `billing`, `shipping`, `analytics` — każda z innym filtrem.
- **DLQ wbudowany** — każda kolejka i subskrypcja ma automatyczny dead-letter queue. Po przekroczeniu `MaxDeliveryCount` (licznik nieudanych odbiorów) wiadomość trafia do DLQ z powodem. To, co w module 2 budowałeś/symulowałeś ręcznie, tu dostajesz w cenie — twoim zadaniem pozostaje **proces obsługi DLQ** (monitoring, triage, redrive), bo tego żadna usługa za ciebie nie zrobi.
- **Sesje (sessions)** — gwarancja kolejności FIFO w obrębie klucza: wiadomości z tym samym `SessionId` (np. ID zamówienia) trafiają do jednego consumera, po kolei. Rozwiązanie problemu out-of-order z modułu 2 — kolejność per encja, bez rezygnacji z równoległości między encjami.
- **Transakcje** — atomowe grupowanie operacji na brokerze (np. complete wiadomości przychodzącej + send dwóch wychodzących). Uwaga: transakcja obejmuje **tylko Service Bus** — nie połączysz jej z transakcją SQL. Dlatego outbox pattern nadal jest potrzebny (o tym niżej).
- Ponadto: scheduled messages (dostarczenie w przyszłości), duplicate detection (deduplikacja po `MessageId` w oknie czasowym — wspiera idempotencję po stronie brokera), auto-forwarding.

**Model rozliczeń:** tiery Basic (tylko kolejki, do zabawy), **Standard** (topics, płatność za operacje — tani start), **Premium** (dedykowane Messaging Units, stała cena, przewidywalna latencja, VNet/private endpoints — produkcja enterprise). Standard wystarcza do nauki i małej produkcji; granicę opłacalności Premium policz w lekcji 06.

**Kiedy nie:** masowe strumienie telemetrii (miliony zdarzeń/s — za drogi i nie do tego służy) i trywialne scenariusze, gdzie wystarczy Storage Queue.

### Azure Event Grid — reaktywny eventing, push i filtrowanie

**Co to jest:** ruter zdarzeń. Nie przechowuje wiadomości do odbioru jak kolejka — **pcha** (push) zdarzenia do subskrybentów: webhooków, Functions, kolejek Service Bus/Storage. Dwa źródła zdarzeń: **systemowe** (zdarzenia samego Azure: blob wgrany, zasób utworzony, deployment zakończony) i **własne** (custom topics — twoje zdarzenia domenowe).

- **Filtrowanie** na poziomie subskrypcji: po typie zdarzenia, prefiksie/sufiksie subject, wartościach pól. Subskrybent dostaje tylko to, co go interesuje.
- **Retry z backoff** do każdego subskrybenta niezależnie + dead-lettering do Storage (opcjonalny, włączasz sam).
- Model **pay-per-event** — przy małym wolumenie koszt pomijalny.

**Czym NIE jest:** kolejką roboczą. Zdarzenie ma być małe ("co się stało + referencja"), a dostarczanie nie daje narzędzi przetwarzania (peek-lock, sesje, transakcje). Klasyczna kompozycja: **Event Grid → kolejka Service Bus → consumer** — Grid rozgłasza i filtruje, kolejka daje buforowanie i semantykę przetwarzania.

**Kiedy używać:** reakcja na zdarzenia infrastruktury Azure (to się praktycznie nie ma alternatywy), luźne powiadamianie wielu odbiorców, integracje webhookowe. **Kiedy nie:** gdy zguba zdarzenia boli biznesowo, a odbiorca potrzebuje kontroli tempa przetwarzania — wtedy komunikat na Service Bus.

### Azure Event Hubs — strumienie; "Kafka w wersji zarządzanej"

**Co to jest:** platforma do ingestii strumieni zdarzeń o dużej przepustowości. Jeśli kojarzysz Kafkę choćby z opisu — Event Hubs to ten sam model (i wystawia **endpoint zgodny z protokołem Kafki**, więc aplikacje kafkowe podłączysz bez zmian kodu):

- **Partycje** — strumień jest dzielony na partycje; zdarzenia z tym samym kluczem partycji lądują w tej samej partycji **w kolejności**. To samo partycjonowanie, które znasz z lekcji modułu 2 — klucz dobierasz tak, by zachować kolejność tam, gdzie jej potrzebujesz, i rozłożyć ruch równomiernie (uwaga na hot partitions).
- **Konsument NIE usuwa zdarzeń.** To dziennik (log): zdarzenia żyją przez okres retencji (godziny–dni), a każdy consumer pamięta swój **offset** (pozycję odczytu) — checkpointing robisz np. do Blob Storage. Konsekwencja: możesz **cofnąć się i przeczytać ponownie** (replay) — rzecz niemożliwa w klasycznej kolejce, bezcenna przy event sourcingu i naprawianiu bugów konsumenta.
- **Consumer groups** — niezależne "widoki" strumienia: analityka czyta w swoim tempie, alerting w swoim, każdy ze swoim offsetem.
- **Brak DLQ i per-message retry** — to nie ten model. Poison event obsługujesz we własnym kodzie (np. odkładając go na Service Bus).

**Model rozliczeń:** za przepustowość — Throughput Units (Standard) / Processing Units (Premium), plus retencja. Jest też tier Dedicated dla ekstremalnych wolumenów. **Kiedy używać:** telemetria, IoT, clickstream, pipeline'y analityczne, event sourcing na dużą skalę. **Kiedy nie:** zwykłe komunikaty biznesowe o wolumenie tysięcy/dzień — Service Bus jest prostszy i tańszy.

### Azure Storage Queues — kiedy wystarczą

Najprostsza kolejka, część konta Storage: put/get/delete, visibility timeout (analogia peek-lock), bardzo tania, ogromna pojemność. **Nie ma:** topics/pub-sub, sesji/FIFO, transakcji, natywnego DLQ (jest licznik `DequeueCount` — DLQ budujesz sam), zaawansowanego filtrowania. **Wystarczą, gdy:** jeden producer, jeden typ consumera, kolejność nieistotna, idempotencja załatwia duplikaty, liczy się minimalny koszt — np. kolejka zadań w tle dla jednej aplikacji. Gdy zaczynasz dobudowywać do nich pub/sub albo DLQ ręcznie — to sygnał, że czas na Service Bus.

### Mapowanie wzorców z modułu 2 na usługi

| Wzorzec z modułu 2 | Gdzie żyje w Azure |
|---|---|
| Outbox | **bez zmian, w twoim kodzie + SQL** — żadna usługa Azure go nie zastępuje, bo problem (atomowość zapisu do bazy i publikacji) leży między bazą a brokerem. Relay czyta outbox i publikuje na Service Bus. Duplicate detection w Service Bus dodatkowo łagodzi skutki ponownej publikacji |
| Idempotent consumer | nadal twój kod; Service Bus daje at-least-once (peek-lock), więc idempotencja pozostaje obowiązkowa |
| DLQ / poison message | Service Bus: wbudowany, z `MaxDeliveryCount` i powodem dead-letteringu; Event Grid: opcjonalny dead-letter do Storage; Event Hubs/Storage Queues: budujesz sam |
| Saga / process manager | logika w twoim kodzie (lub frameworku typu MassTransit/NServiceBus); komunikacja kroków przez kolejki/topics Service Bus; timeout sagi → scheduled messages |
| Ordering per encja | Service Bus sessions albo partition key w Event Hubs |
| Partycjonowanie | Event Hubs partitions (jawne) / Service Bus sessions (logiczne) |
| Retry z backoff | polityki retry w SDK + `MaxDeliveryCount`; Event Grid: wbudowany retry push |
| Replay / event sourcing | tylko Event Hubs (offset + retencja); kolejki nie umieją replay |

### Macierz decyzyjna

| Wymiar | Storage Queues | Service Bus | Event Grid | Event Hubs |
|---|---|---|---|---|
| Klasa problemu | prosta kolejka zadań | komunikaty/polecenia biznesowe | rozgłaszanie zdarzeń (push) | strumienie wysokiego wolumenu |
| Model konsumpcji | pull, delete po przetworzeniu | pull, peek-lock | push do subskrybentów | pull po offsecie, bez usuwania |
| Pub/sub | nie | topics + subscriptions + filtry SQL | tak, filtrowanie wbudowane | consumer groups (każda czyta całość) |
| Kolejność | nie | sessions (FIFO per klucz) | nie | per partycja |
| DLQ | ręcznie | wbudowany | opcjonalny (do Storage) | brak (ręcznie) |
| Replay | nie | nie | nie | **tak** |
| Rozliczanie | za operacje (grosze) | za operacje / Messaging Units | za zdarzenie | za przepustowość |
| Analogia | — | RabbitMQ | — (ruter zdarzeń) | Kafka |

### Typowe kombinacje (tak wyglądają realne architektury)

1. **Container Apps + Service Bus (queues/topics) + KEDA** — rdzeń systemu biznesowego; KEDA skaluje consumerów po głębokości kolejki. To architektura projektu z lekcji 07.
2. **Event Grid → Service Bus queue → consumer** — reaktywność i filtrowanie Grida + niezawodne przetwarzanie kolejki (np. obróbka wgranych blobów).
3. **Event Hubs → procesor → Service Bus** — telemetria wpada strumieniem; wykryte sytuacje wymagające akcji stają się komunikatami biznesowymi.
4. **Functions + Storage Queue** — minimalistyczny background processing dla jednej aplikacji, koszt bliski zera.

## Praktyka

- [ ] Dla 6 scenariuszy wybierz usługę i jedno zdanie uzasadnienia: (a) faktura PDF do wygenerowania po zamówieniu, (b) 50k zdarzeń telemetrii/s z urządzeń, (c) powiadom 4 systemy o nowym kliencie, (d) reakcja na upload pliku do Blob Storage, (e) nightly batch jednej aplikacji, (f) kroki sagi rezerwacji.
- [ ] Utwórz namespace Service Bus (Standard) i z poziomu .NET (`Azure.Messaging.ServiceBus`): wyślij/odbierz z kolejki w trybie peek-lock; doprowadź wiadomość do DLQ (rzucaj wyjątek aż przekroczysz `MaxDeliveryCount`); odczytaj ją z DLQ i obejrzyj powód; utwórz topic z 2 subskrypcjami z różnymi filtrami i sprawdź routing.
- [ ] Przetestuj sesje: wyślij 10 wiadomości z dwoma różnymi `SessionId` i potwierdź kolejność per sesja przy 2 równoległych consumerach.
- [ ] Wróć do diagramu przepływu z modułu 2 i podpisz każdą strzałkę usługą Azure. Zapisz mapowanie w notatkach.
- [ ] **Usuń resource group po ćwiczeniach** (Standard nalicza za operacje, ale porządek to nawyk z lekcji 06).

## Artefakt

1. **Macierz decyzyjna messaging** + tabela mapowania wzorców z modułu 2 w notatkach.
2. **Zaktualizowany diagram** aplikacji z modułu 2 z konkretnymi usługami Azure — wejście do projektu w lekcji 07 i materiał na post ("4 usługi Azure, które wyglądają jak kolejka — jak wybrać").

## Definition of Done

- [ ] Rozróżniasz message/event/stream i przypisujesz do nich usługi bez zaglądania do notatek.
- [ ] Widziałeś w działaniu: peek-lock, DLQ z powodem, filtry subskrypcji, sesje — kod ćwiczeń jest w repo.
- [ ] Umiesz wyjaśnić, dlaczego Service Bus **nie zastępuje** outbox patternu (transakcja brokera ≠ transakcja z bazą).
- [ ] Diagram aplikacji z usługami Azure jest zacommitowany.

## Materiały

- [Azure Architecture Center — Asynchronous messaging options](https://learn.microsoft.com/azure/architecture/guide/technology-choices/messaging) — oficjalne porównanie; skonfrontuj z macierzą z lekcji.
- [Microsoft Learn — Service Bus queues, topics, and subscriptions](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-queues-topics-subscriptions) — Service Bus będzie rdzeniem projektu, znaj go najgłębiej z całej czwórki.
