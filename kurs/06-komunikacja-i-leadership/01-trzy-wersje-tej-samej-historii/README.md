# Lekcja 01 — Trzy wersje tej samej historii: zarząd, PM, zespół

## Cel lekcji

Po tej lekcji będziesz umiał opowiedzieć tę samą decyzję architektoniczną na trzy sposoby — do zarządu (pieniądze, ryzyko, czas), do PM-a (zakres, trade-offy, terminy) i do zespołu (szczegóły techniczne i uzasadnienie) — oraz świadomie wybierać, od czego zacząć i co pominąć.

## Dlaczego to ważne

To jest najczęstszy powód, dla którego świetni inżynierowie nie dostają roli architekta: **mówią do wszystkich tak samo, czyli tak jak do innych inżynierów**. Architekt w typowym tygodniu rozmawia z trzema różnymi światami: z zarządem o budżecie na migrację, z PM-em o tym, co wypadnie z roadmapy przez refaktoring, z zespołem o tym, dlaczego idziemy w outbox pattern, a nie w distributed transaction. Każdy z tych światów mierzy wartość czym innym — i każdemu trzeba opowiedzieć tę samą prawdę innym językiem, bez kłamania i bez infantylizowania.

Na rozmowach rekrutacyjnych na role Senior/Lead/Architect to pytanie pada wprost: *"Opowiedz, jak przekonałeś biznes do decyzji technicznej"* albo *"Jak tłumaczysz dług techniczny nietechnicznym interesariuszom?"*. Znasz to z własnej kariery — jako owner systemu Perfect Gym musiałeś tłumaczyć te same decyzje i programistom, i ludziom od biznesu. Ta lekcja nazywa i porządkuje to, co robiłeś intuicyjnie.

## Teoria

### Problem: jedna historia, trzech odbiorców, trzy różne pytania w głowach

Wyobraź sobie decyzję z Twojej przeszłości: system rezerwacji (Aerotunel) gubił albo dublował rezerwacje przy awariach integracji, więc zdecydowałeś się przebudować komunikację na kolejki z gwarancjami dostarczania. Teraz masz o tym opowiedzieć trzem osobom. Każda z nich, słuchając Cię, w głowie odpowiada sobie na **inne pytanie**:

| Odbiorca | Pytanie w głowie | Miara wartości | Czego NIE chce słuchać |
|---|---|---|---|
| Zarząd / sponsor | "Ile to kosztuje, co ryzykujemy, kiedy zobaczymy efekt?" | pieniądze, ryzyko, czas | technikaliów — jakichkolwiek |
| PM / product | "Co to zmienia w roadmapie? Co dostanę, czego nie dostanę, co kiedy?" | zakres, terminy, trade-offy | szczegółów implementacji |
| Zespół | "Dlaczego tak, a nie inaczej? Jak to wpłynie na mój kod?" | jakość decyzji, spójność, wykonalność | ogólników i sloganów |

Naiwne rozwiązanie: jedna prezentacja dla wszystkich, "od ogółu do szczegółu", 40 slajdów. Dlaczego się psuje: zarząd wyłącza się na slajdzie 3 (diagram kolejek), PM nie znajduje odpowiedzi na pytanie o roadmapę, a zespół czuje, że techniczna część była spłycona "pod biznes". Wszyscy wyszli, nikt nie podjął decyzji.

Właściwy wzorzec: **trzy wersje tej samej historii**. Nie trzy różne prawdy — ta sama decyzja, te same fakty, ale inna selekcja, inna kolejność i inny język. Analogia: lekarz mówi pacjentowi "ma pan zapalenie, antybiotyk przez 7 dni, wraca pan do pracy w przyszłym tygodniu", a koledze po fachu poda nazwę bakterii, antybiogram i dawkowanie. Obie wersje są prawdziwe; różni je to, **czego odbiorca potrzebuje do podjęcia swojej decyzji**.

### Technika piramidy: wniosek najpierw, szczegóły na żądanie

**Pyramid principle** (zasada piramidy, Barbara Minto, McKinsey) to technika strukturyzowania komunikatu: zaczynasz od **wniosku/rekomendacji**, potem podajesz 2–3 argumenty go wspierające, a dopiero pod nimi — szczegóły i dane. Szczegóły ujawniasz *na żądanie*, gdy ktoś dopyta.

Inżynierowie domyślnie komunikują odwrotnie — **chronologicznie i dedukcyjnie**: "najpierw kontekst, potem analiza, potem opcje, na końcu wniosek". Tak pisze się papers i dokumentację, ale dla decydenta to tortura: przez 15 minut nie wie, do czego zmierzasz, i nie wie, czego ma słuchać uważnie.

Porównaj dwa otwarcia tej samej rozmowy z zarządem:

> ❌ "Nasz system komunikuje się z systemem płatności synchronicznie po HTTP. Gdy płatności mają przeciążenie, nasze wywołania dostają timeout, a retry powoduje czasem podwójne obciążenie klienta, ponieważ…"

> ✅ "Rekomenduję 6 tygodni pracy zespołu na przebudowę integracji z płatnościami. Powód: obecne rozwiązanie raz na kilka tygodni podwójnie obciąża klientów — to zwroty, reklamacje i realne ryzyko reputacyjne. Po zmianie ten typ błędu znika konstrukcyjnie. Mogę opowiedzieć szczegóły, jeśli chcecie."

Druga wersja w 20 sekund daje decydentowi wszystko: **co** rekomendujesz (koszt: 6 tygodni), **dlaczego** (ryzyko biznesowe) i **co dostanie** (klasa błędów znika). Reszta to szczegóły na żądanie. To samo działa w mailach (pierwszy akapit = wniosek), w RFC (sekcja "Summary" na górze) i na rozmowie rekrutacyjnej (najpierw rezultat, potem droga — zobaczysz to w lekcji 04 przy STAR).

Ważne: piramida **nie zwalnia z posiadania szczegółów**. Wręcz przeciwnie — musisz mieć całą piramidę w głowie, bo pierwsze pytanie pogłębiające ("a skąd te 6 tygodni?") zjeżdża piętro niżej. Piramida to kolejność podawania, nie redukcja wiedzy.

### Typowe błędy inżynierów mówiących do biznesu

1. **Zaczynanie od "jak" zamiast od "po co".** "Wdrożymy outbox pattern z brokerem komunikatów" — zarząd słyszy szum. Zacznij od skutku: "wyeliminujemy klasę błędów, która kosztuje nas X reklamacji miesięcznie". "Jak" jest Twoją sprawą — za to Ci płacą.
2. **Język technologii zamiast języka skutków.** Każde zdanie do biznesu przetestuj pytaniem: *czy odbiorca może na tej podstawie podjąć decyzję?* "Mamy dług techniczny w module rezerwacji" — nie może. "Każda zmiana w module rezerwacji trwa 3× dłużej niż rok temu i każde wdrożenie to ryzyko awarii w godzinach szczytu" — może.
3. **Pokazywanie niepewności w złym miejscu.** Inżynierska uczciwość ("to może nie zadziałać, są niewiadome…") podana bez struktury brzmi jak brak kompetencji. Niepewność komunikuj jako **zarządzane ryzyko z planem**: "główne ryzyko to X; mitygujemy je przez Y; punkt kontrolny po 2 tygodniach".
4. **Odpowiadanie na pytanie, które nie padło.** Pytanie zarządu "czy zdążymy przed sezonem?" to pytanie o datę i pewność, nie zaproszenie do wykładu o CI/CD. Odpowiedz na pytanie, potem zamilcz — szczegóły na żądanie.
5. **Brak opcji.** Przychodzenie z jednym rozwiązaniem ("musimy zrobić X") odbiera decydentowi decyzję — a ludzie, którym odebrano decyzję, mówią "nie". Daj 2–3 opcje z rekomendacją: "opcja A: 6 tygodni, znika klasa błędów; opcja B: 1 tydzień łatka, ryzyko wraca za pół roku; rekomenduję A, oto dlaczego".

### Jak mówić o długu technicznym językiem ryzyka biznesowego

**Dług techniczny** (technical debt) to metafora: decyzje techniczne na skróty, które przyspieszają dziś kosztem spowolnienia jutro — jak kredyt: bierzesz czas teraz, spłacasz z odsetkami później. Problem: dla biznesu "dług techniczny" to abstrakcja, a prośba o "czas na refaktoring" brzmi jak "chcemy sprzątać zamiast dowozić".

Tłumaczenie działa, gdy zamienisz dług na jedną z czterech walut biznesowych:

| Waluta | Pytanie tłumaczące | Przykład komunikatu |
|---|---|---|
| **Pieniądze** | Ile to kosztuje miesięcznie? | "Obsługa ręcznych korekt rezerwacji to ~2 dni pracy zespołu miesięcznie" |
| **Czas / velocity** | O ile wolniej dowozimy? | "Feature, który rok temu zajmował tydzień, dziś zajmuje trzy — i ten trend się pogłębia" |
| **Ryzyko** | Co i z jakim prawdopodobieństwem może się stać? | "Przy obecnym ruchu raz na kwartał ryzykujemy awarię w dniu o najwyższej sprzedaży" |
| **Opcje** | Czego przez to NIE możemy zrobić? | "Nie możemy wejść na rynek X, bo system nie obsłuży drugiej strefy czasowej bez przebudowy" |

Zasada: **liczba albo scenariusz, nigdy przymiotnik**. "Kod jest w złym stanie" — przymiotnik, zero decyzji. "Każde wdrożenie wymaga 2h okna serwisowego w nocy i raz na 5 razy coś się wywala" — scenariusz, decyzja możliwa. Jeśli nie masz liczb — i często nie masz dokładnych — podaj rząd wielkości i powiedz, że to szacunek. Szacunek z uzasadnieniem buduje zaufanie; pseudoprecyzja je niszczy.

### Trzy wersje w praktyce — szkielety

**Wersja dla zarządu (30–60 sekund, zero technikaliów):**
1. Rekomendacja + koszt w czasie/pieniądzach.
2. Problem w języku ryzyka/pieniędzy (jedna liczba lub jeden scenariusz).
3. Co się zmieni po (mierzalnie) + główne ryzyko z mitygacją.
4. Czego potrzebujesz (decyzja, budżet, priorytet).

**Wersja dla PM-a (2–5 minut, zakres i trade-offy):**
1. Co się zmienia w produkcie i roadmapie (co wchodzi, co wypada, co się przesuwa).
2. Trade-off wprost: "robiąc X, nie robimy Y do daty Z" — PM żyje z zarządzania tym napięciem, ukrywanie go przed nim to sabotaż.
3. Kamienie milowe i punkty kontrolne — kiedy PM zobaczy postęp i kiedy może zmienić zdanie.
4. Wpływ na użytkownika: co zauważy, co przestanie się psuć.

**Wersja dla zespołu (15–45 minut, pełna głębia):**
1. Kontekst i problem — łącznie z presją biznesową, którą często przed zespołem się (błędnie) ukrywa. Zespół, który zna "po co", podejmuje lepsze mikrodecyzje.
2. Rozważane opcje i dlaczego odrzucone — to najważniejsza część; decyzja bez pokazania alternatyw wygląda na widzimisię.
3. Wybrane rozwiązanie technicznie: diagramy, kontrakty, wpływ na istniejący kod.
4. Otwarte pytania i miejsce na sprzeciw — wersja dla zespołu jako jedyna z trzech jest **zaproszeniem do dyskusji**, nie komunikatem (więcej w lekcji 02 o design review).

### Kiedy tego NIE stosować (trade-offy)

- **Piramida nie nadaje się do komunikowania złych wiadomości wymagających kontekstu zaufania.** "Rekomenduję wyłączenie projektu, w który wierzysz" jako pierwsze zdanie zamyka rozmowę. Przy trudnych emocjonalnie tematach najpierw krótki kontekst wspólnego celu, potem wniosek.
- **Trzy wersje to nie trzy prawdy.** Jeśli wersja dla zarządu przemilcza ryzyko, które znasz, to nie komunikacja, tylko manipulacja — i wróci do Ciebie przy pierwszej awarii. Selekcjonujesz szczegóły, nigdy fakty zmieniające decyzję.
- **Nie każda decyzja zasługuje na trzy wersje.** Wybór biblioteki do logowania nie potrzebuje prezentacji dla zarządu. Pełny rytuał stosuj do decyzji drogich lub trudno odwracalnych (one-way doors z modułu 1).

## Praktyka

- [ ] Wybierz **jedną prawdziwą decyzję architektoniczną z własnej kariery** (Perfect Gym lub Aerotunel; dobry kandydat: przebudowa integracji, wprowadzenie kolejek, decyzja po awarii produkcyjnej). Zapisz w 2 zdaniach, czego dotyczyła i co było stawką.
- [ ] Napisz **wersję dla zarządu**: maksymalnie 1 akapit (5–7 zdań), zero terminów technicznych, struktura piramidy. Test: czy osoba nietechniczna może po przeczytaniu podjąć decyzję "tak/nie"?
- [ ] Napisz **wersję dla PM-a**: 2–4 akapity, z jawnym trade-offem ("robiąc X, nie robimy Y") i punktami kontrolnymi w czasie.
- [ ] Napisz **wersję dla zespołu**: do 1 strony, z sekcją "rozważane opcje i dlaczego odrzucone" oraz co najmniej jednym otwartym pytaniem do dyskusji.
- [ ] Przeczytaj wszystkie trzy na głos. Po każdej odpowiedz pisemnie: *na jakie pytanie odbiorcy ta wersja odpowiada w pierwszych dwóch zdaniach?* Jeśli nie umiesz wskazać — przepisz początek.
- [ ] (Opcjonalnie, polecane) Daj wersję "dla zarządu" do przeczytania komuś nietechnicznemu i zapytaj: "co byś zdecydował i czego Ci brakuje?".

## Artefakt

**Jedna decyzja architektoniczna spisana w trzech wersjach** (zarząd / PM / zespół) w pliku `trzy-wersje.md` w folderze tej lekcji. Bonus: wersja dla zarządu po lekkiej redakcji to gotowy post na LinkedIn typu "jak tłumaczyć decyzje techniczne biznesowi — na przykładzie z mojej kariery".

## Definition of Done

- [ ] Plik `trzy-wersje.md` istnieje i zawiera wszystkie trzy wersje tej samej decyzji.
- [ ] Wersja dla zarządu nie zawiera ani jednego terminu technicznego i zaczyna się od rekomendacji/wniosku (piramida).
- [ ] Wersja dla PM-a zawiera co najmniej jeden jawnie nazwany trade-off zakresu lub terminu.
- [ ] Wersja dla zespołu zawiera sekcję odrzuconych opcji z uzasadnieniem.
- [ ] Dług techniczny lub ryzyko w wersji biznesowej jest wyrażone liczbą albo scenariuszem — nie przymiotnikiem.
- [ ] Potrafisz powiedzieć z pamięci, na jakie pytanie w głowie odbiorcy odpowiada każda z trzech wersji.

## Materiały

1. Barbara Minto, *The Pyramid Principle* — źródło techniki "wniosek najpierw"; wystarczy część I.
2. [Technical debt — artykuł Martina Fowlera](https://martinfowler.com/bliki/TechnicalDebt.html) — krótkie, kanoniczne ujęcie metafory długu, przydatne jako wspólny język z biznesem.
