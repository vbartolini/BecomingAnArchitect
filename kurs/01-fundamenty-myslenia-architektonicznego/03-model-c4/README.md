# Lekcja 03 — Model C4: diagramy, które ludzie naprawdę czytają

## Cel lekcji

Po tej lekcji będziesz umiał narysować czytelny diagram architektury w modelu C4: pewnie na poziomach 1 (Context) i 2 (Container), świadomie na poziomie 3 (Component), i będziesz wiedział, dlaczego poziom 4 (Code) prawie zawsze się pomija. Wybierzesz też narzędzie, w którym będziesz diagramować przez resztę kursu.

## Dlaczego to ważne

Architektura, której nie da się narysować na jednej kartce, nie jest zrozumiana — nawet przez autora. Diagramy to podstawowy nośnik komunikacji architekta: z zespołem, z biznesem, z rekruterem.

Na rozmowach system design **rysujesz cały czas** — i kandydaci bez wytrenowanej konwencji rysują chmurki połączone kreskami, gdzie nie wiadomo, czy prostokąt to serwer, proces, moduł czy zespół. C4 daje ci gotową, rozpoznawalną konwencję: rekruter-architekt widzi diagram C4 i od razu wie, że rozmawia z kimś, kto diagramował zawodowo. Dodatkowo: artefakt tej lekcji idzie do sekcji Featured na LinkedIn — to twoja wizytówka architektoniczna.

## Teoria

### Po co w ogóle diagramy (i dlaczego UML umarł w praktyce)

Problem: nowa osoba wchodzi do projektu i pyta "jak to działa?". Naiwne rozwiązanie nr 1: "poczytaj kod". Kod ma milion linii i mówi *jak*, nie *dlaczego* i nie *co z czym gada* — strukturę systemu widać z kodu tak, jak plan miasta z poziomu chodnika. Naiwne rozwiązanie nr 2: "narysujmy wszystko w UML-u".

**UML** (Unified Modeling Language) to ustandaryzowany w latach 90. język diagramów (klas, sekwencji, wdrożenia — kilkanaście typów). W praktyce przemysłowej umarł, z kilku powodów:

- **Wymagał wiedzy, żeby go czytać.** Romb vs trójkąt vs wypełniony romb na końcu strzałki niosły znaczenie, którego biznes (i połowa developerów) nie znała. Diagram, który trzeba dekodować, nie komunikuje.
- **Modelował głównie kod (klasy), a kod się zmienia codziennie** — diagramy klas dezaktualizowały się w tydzień i stawały się dokumentacją-kłamstwem, gorszą niż brak dokumentacji.
- **Ciężkie narzędzia i kultura big design up front** — Rational Rose, Enterprise Architect, generowanie kodu z modelu. Agile to zmiótł, ale wylał dziecko z kąpielą: zamiast "lżejsze diagramy" zostało "żadne diagramy" albo chmurki ad hoc na whiteboardzie.

Wniosek Simona Browna (autora C4): nie potrzebujemy 14 typów diagramów. Potrzebujemy **map w kilku skalach powiększenia** — jak Google Maps: najpierw kraj, potem miasto, potem ulica. I prostej notacji: pudełka + strzałki + **etykiety tekstowe, które robią całą robotę**.

### Model C4 — cztery poziomy zoomu

**C4** = **Context, Container, Component, Code** — cztery poziomy szczegółowości, od ogółu do detalu. Trzy pojęcia bazowe, definiowane od zera:

- **Software System** (system) — najwyższy poziom: coś, co dostarcza wartość użytkownikom jako całość. "Platforma do zarządzania siłowniami". Z perspektywy użytkownika — jedna rzecz.
- **Container** (kontener) — **osobno uruchamialna/deployowalna jednostka**, w której wykonuje się kod lub żyją dane: aplikacja webowa, API, aplikacja mobilna, baza danych, broker kolejek, funkcja serverless. UWAGA na fałszywego przyjaciela: **to NIE jest kontener Dockera** (choć często kontener C4 jeździ w kontenerze Dockera). Test: "czy to się osobno uruchamia/wdraża i komunikuje z resztą przez sieć/protokół?" — tak → kontener.
- **Component** (komponent) — logiczny moduł **wewnątrz** kontenera: grupa klas za wspólnym interfejsem (np. `BookingService`, `PaymentGateway`, `AccessControlModule`). Komponenty nie deployują się osobno.

Notacja jest celowo banalna i samoopisująca się — każdy element ma **trzy linijki tekstu**: nazwa, typ/technologia w nawiasie, jedno zdanie odpowiedzialności. Każda strzałka ma etykietę z czasownikiem ("wysyła rezerwacje do", "odczytuje grafik z") i opcjonalnie protokołem [HTTPS/JSON, AMQP]. Diagram bez etykiet na strzałkach to nie diagram, tylko abstrakcyjna grafika.

### Poziom 1 — System Context (kontekst systemu)

**Pytanie, na które odpowiada:** kto i co używa naszego systemu oraz z jakimi innymi systemami on rozmawia? Nasz system to **jedno pudełko w środku**; dookoła ludzie (aktorzy) i systemy zewnętrzne. Zero technologii w środku — ten diagram ma rozumieć prezes.

Przykład — system rezerwacji lotów w tunelu aerodynamicznym (twój Aerotunel, zanonimizowany):

```
[Klient indywidualny]              [Pracownik recepcji]            [Instruktor]
   (osoba)                              (osoba)                      (osoba)
      │ rezerwuje loty,                    │ zarządza grafikiem,        │ przegląda
      │ płaci online                       │ obsługuje walk-in          │ swój grafik
      ▼                                    ▼                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              System Rezerwacji Tunelu Aerodynamicznego                  │
│   Umożliwia rezerwację i opłacenie lotów, zarządza grafikiem            │
│   slotów, instruktorów i voucherami                                     │
└─────────────────────────────────────────────────────────────────────────┘
      │ pobiera płatności [HTTPS]      │ wysyła e-maile         │ wystawia
      ▼                                ▼  potwierdzeń           ▼  faktury
[Bramka płatności]               [Dostawca e-mail]        [System księgowy]
 (system zewn.)                   (system zewn.)           (system zewn.)
```

Typowe błędy poziomu 1: wrzucanie do środka baz danych i serwisów (to poziom 2!), pomijanie aktorów-ludzi, brak czasowników na strzałkach.

### Poziom 2 — Container (kontenery)

**Pytanie:** z jakich osobno działających klocków składa się system i jak się komunikują? Robimy zoom-in w to jedno pudełko z poziomu 1. Aktorzy i systemy zewnętrzne zostają na obrzeżach (kontekst się nie zmienia, zmienia się powiększenie). Tu **pojawiają się technologie**.

Ten sam system, poziom 2:

```
[Klient] ─── używa [HTTPS] ───►  ┌──────────────────────────┐
                                 │ Aplikacja webowa (SPA)   │
                                 │ [React]                  │
                                 │ rezerwacja i płatność    │
                                 └──────────┬───────────────┘
                                            │ wywołuje [HTTPS/JSON]
[Recepcja] ── używa ──►  ┌──────────────┐   ▼
                         │ Panel        │  ┌───────────────────────────┐
                         │ recepcji     │─►│ API rezerwacji            │
                         │ [Blazor]     │  │ [ASP.NET Core]            │
                         └──────────────┘  │ logika rezerwacji,        │
                                           │ grafiku i voucherów       │
                                           └───┬──────────┬────────────┘
                       odczyt/zapis [EF Core]  │          │ publikuje zdarzenia [AMQP]
                                               ▼          ▼
                                    ┌───────────────┐  ┌──────────────────┐
                                    │ Baza danych   │  │ Broker kolejek   │
                                    │ [SQL Server]  │  │ [RabbitMQ]       │
                                    └───────────────┘  └──────┬───────────┘
                                                              │ konsumuje
                                                              ▼
                                                   ┌─────────────────────────┐
                                                   │ Worker powiadomień      │
                                                   │ [.NET Worker Service]   │──► [Dostawca e-mail]
                                                   │ e-maile, SMS-y, retry   │──► [Bramka płatności: webhooki]
                                                   └─────────────────────────┘
```

Czytelnik poziomu 2 (nowy developer, DevOps, architekt z innego zespołu) widzi w 60 sekund: ile jest deployowalnych jednostek, gdzie żyją dane, co jest synchroniczne (HTTP), a co asynchroniczne (kolejka). **Poziomy 1 i 2 to 90% wartości całego C4** — i to one są wymagane w artefakcie tej lekcji.

### Poziom 3 — Component (krócej)

Zoom w **jeden wybrany kontener** (zwykle główne API): jakie moduły logiczne w nim żyją i jak na siebie wołają. Np. wewnątrz "API rezerwacji": `BookingController` → `BookingService` (zasady rezerwacji, konflikt slotów) → `SlotRepository`; obok `VoucherModule`, `PricingModule`, `NotificationPublisher`.

Rysuj poziom 3 tylko, gdy: kontener jest duży i wchodzą do niego nowi ludzie, albo dyskutujesz o granicach modułów (przyda się w lekcji 05 i 06 — komponenty poziomu 3 często pokrywają się z bounded contexts). Nie rysuj "dla kompletu": poziom 3 dezaktualizuje się szybciej niż 1–2, a każdy nieaktualny diagram pracuje przeciwko tobie.

### Poziom 4 — Code (kiedy pomijać: prawie zawsze)

Diagram klas dla jednego komponentu. Brown sam mówi: pomijaj — IDE wygeneruje to na żądanie z aktualnego kodu, a ręcznie rysowany poziom 4 to powrót do grzechu UML-a (dokumentacja-kłamstwo po pierwszym refactoringu). Wyjątek: chwilowy szkic złożonego algorytmu na potrzeby jednej dyskusji, do wyrzucenia po niej.

### Narzędzia

- **Structurizr** (narzędzie Browna) — **diagrams-as-code**: opisujesz model w tekstowym DSL-u, narzędzie renderuje wszystkie poziomy z **jednego modelu** (zmiana nazwy kontenera poprawia się wszędzie). Diagram żyje w repo, ma diff w pull requestach. Wariant lekki: Structurizr Lite (darmowy, lokalny, Docker).
- **Inne diagrams-as-code:** PlantUML z biblioteką C4-PlantUML, Mermaid (ma natywny typ `C4Context` — i renderuje się w README na GitHubie, co czyni go najtańszym startem). Trade-off vs Structurizr: każdy diagram to osobny plik, spójność między poziomami pilnujesz ręcznie.
- **draw.io / diagrams.net** — klikanie zamiast kodu; szybkie i ładne na jednorazowe diagramy (np. ten na LinkedIn), ale brak diffów i wspólnego modelu. Dobre na start, słabe na "living documentation".

Rekomendacja kursowa: artefakt tej lekcji zrób w czym chcesz (draw.io jest OK — liczy się czytelność), ale do końca modułu spróbuj Structurizr DSL lub Mermaid, bo diagrams-as-code wraca w capstone.

### Kiedy NIE używać C4

Diagram przepływu procesu w czasie (co dzieje się krok po kroku przy rezerwacji) → diagram sekwencji, nie C4. Infrastruktura/deployment (które kontenery na którym środowisku Azure) → C4 ma osobny "deployment diagram", wróci w module 3. Modelowanie domeny biznesowej → event storming / mapa kontekstów (lekcja 05). C4 to mapa **struktury statycznej** — nie wciskaj w nią wszystkiego.

## Praktyka

- [ ] Przeczytaj c4model.com (sekcje: Abstractions, Container diagram, Notation) — to godzina lektury, źródło pierwotne.
- [ ] **Ćwiczenie główne:** narysuj z pamięci C4 **poziom 1 i poziom 2** dla systemu klasy Perfect Gym, **zanonimizowane** (nazwy własne klientów/integracji zastąp rodzajowymi: "system kontroli dostępu", "bramka płatności", "platforma fitness-agregator"). Uwzględnij aktorów (członek klubu, recepcja, manager klubu, centrala sieci) i integracje, które pamiętasz (płatności, kontrola dostępu/bramki, e-mail/SMS, księgowość, agregatory typu multisport).
- [ ] Zrób przegląd jakości własnego diagramu checklistą: każde pudełko ma nazwę + typ/technologię + zdanie odpowiedzialności? każda strzałka ma czasownik? poziom 1 nie zawiera technologii? poziom 2 pokazuje wszystkie magazyny danych i kolejki? legenda istnieje?
- [ ] Pokaż diagram jednej osobie (niekoniecznie technicznej) i poproś, by opowiedziała, jak działa system. Każde miejsce, gdzie się zacina, to twój błąd diagramu, nie jej.
- [ ] Pytanie kontrolne: wymień 2 powody, dla których nie rysujemy poziomu 4, i jedną sytuację, w której poziom 3 jest jednak wart wysiłku.

## Artefakt

Dwa diagramy (poziom 1 i 2) systemu klasy "platforma dla siłowni", zanonimizowane, w repo nauki (`c4-gym-platform-context.png` / `...-container.png` + plik źródłowy). **Wersję dopracowaną graficznie wrzuć do sekcji Featured na LinkedIn** z 2–3 zdaniami opisu (czego dotyczy, czemu C4) — to artefakt modułu. Jeśli użyłeś Mermaid/Structurizr — źródło DSL też do repo.

## Definition of Done

- [ ] Umiesz z pamięci wymienić 4 poziomy C4 i powiedzieć jednym zdaniem, na jakie pytanie odpowiada każdy i kto jest jego czytelnikiem.
- [ ] Umiesz wyjaśnić różnicę kontener C4 vs kontener Dockera oraz kontener vs komponent.
- [ ] Diagramy poziomu 1 i 2 przeszły checklistę jakości i test opowiedzenia przez drugą osobę.
- [ ] Diagram wisi w Featured na LinkedIn.
- [ ] Umiesz uzasadnić, dlaczego UML-owe diagramy klas jako dokumentacja architektury przegrały — i czym C4 różni się *systemowo* (skale zoomu, notacja tekstowa), a nie tylko kosmetycznie.

## Materiały

1. **c4model.com** (Simon Brown) — oficjalna, darmowa i zwięzła dokumentacja modelu; źródło pierwotne, przeczytaj w całości.
2. **Structurizr DSL** — dokumentacja + przykłady (docs.structurizr.com/dsl) — gdy zdecydujesz się na diagrams-as-code; zacznij od gotowego przykładu i przerób go na swój system.
