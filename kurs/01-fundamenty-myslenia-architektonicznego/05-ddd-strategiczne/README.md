# Lekcja 05 — DDD strategiczne: bounded contexts i context mapping

## Cel lekcji

Po tej lekcji będziesz umiał dzielić duży system na **bounded contexts** (konteksty ograniczone), budować w każdym z nich **ubiquitous language** (język wszechobecny) i rysować **context mapy** z nazwanymi typami relacji (partnership, customer–supplier, conformist, anti-corruption layer). Będziesz też wiedział, czym DDD strategiczne różni się od taktycznego — i dlaczego taktyczne w tym kursie świadomie pomijamy.

## Dlaczego to ważne

Najdroższe błędy architektoniczne to błędy **granic**: moduły pocięte w złych miejscach, jedna "uniwersalna" encja `Customer` z 70 kolumnami obsługująca pół firmy, zmiana w jednym miejscu psująca trzy inne. Granic nie naprawia się refactoringiem w sprincie — to one-way doors (lekcja 02). DDD strategiczne to najlepsze dostępne narzędzie do **znajdowania granic tam, gdzie są naprawdę** — czyli w języku biznesu, nie w tabelach bazy.

Na rozmowach: "bounded context" to słownik-klucz przy każdej dyskusji o mikroserwisach i modularnym monolicie (lekcja 06 stoi na tej lekcji). Pytanie "jak podzieliłbyś ten system na serwisy?" jest w gruncie rzeczy pytaniem o konteksty. A ty masz przewagę: po latach w Perfect Gym widziałeś organizm, w którym "członek" znaczył co innego dla recepcji, księgowości i bramki — zostało to tylko nazwać.

## Teoria

### Czym jest DDD i czemu bierzemy tylko połowę

**Domain-Driven Design** (DDD, projektowanie sterowane domeną) — podejście opisane przez Erica Evansa (2003): oprogramowanie powinno być modelowane wokół **domeny biznesowej** (tego, czym firma naprawdę się zajmuje) i w ścisłej współpracy z ekspertami domenowymi. DDD ma dwie warstwy:

- **DDD strategiczne** — *makro*: jak podzielić duży problem na konteksty, jak nazywać rzeczy, jak konteksty mają ze sobą rozmawiać. Działa na poziomie systemów, zespołów i map — to narzędziownik **architekta**.
- **DDD taktyczne** — *mikro*: wzorce wewnątrz pojedynczego kontekstu — entity, value object, aggregate, repository, domain event… Działa na poziomie klas — to narzędziownik **developera projektującego kod bogatej domeny**.

Dlaczego pomijamy taktyczne: po pierwsze, ten moduł uczy myślenia architektonicznego, a 90% architektonicznej wartości DDD siedzi w strategicznym — złe granice kontekstów zabiją system z najpiękniejszymi agregatami w środku; odwrotnie to nie działa. Po drugie, taktyczne DDD jest najczęściej kargo-kultowane: zespoły zaczynają od repository i value objects ("bo tak się robi DDD"), nie tknąwszy granic ani języka — i dostają drogi kod bez żadnej z obietnic DDD. Uczymy się w kolejności ważności. (Taktyczne wzorce zahaczysz przy DayChunks, jeśli domena ci urośnie — ale to decyzja na poziomie kodu, nie architektury.)

### Ubiquitous language — język wszechobecny

**Problem:** biznes mówi "karnet", analityk pisze "subskrypcja", developer koduje `MembershipContract`, w bazie jest tabela `Agreements`, a w UI "Twoje pakiety". Każda rozmowa o wymaganiu zawiera rundę tłumaczenia, a każde tłumaczenie gubi i przekłamuje (jak w głuchym telefonie). Bugi rodzą się dokładnie w tych szczelinach: "anulowanie karnetu" znaczyło dla biznesu "od następnego okresu rozliczeniowego", dla developera "natychmiast".

**Naiwne rozwiązanie:** słownik pojęć w Confluence. Umiera w tydzień, bo nikt nie aktualizuje dokumentu odklejonego od pracy.

**Właściwy wzorzec — ubiquitous language:** w obrębie kontekstu **wszyscy** (biznes, developerzy, testerzy) używają **jednego, uzgodnionego języka — i ten język jest wprost w kodzie**. Jeśli biznes mówi "wejście" (na siłownię), to w kodzie jest `CheckIn`, nie `EntryEventRecord`. Jeśli na spotkaniu pada słowo, którego nie ma w kodzie — albo kod, albo rozmowa kłamie i trzeba to natychmiast wyjaśnić. Język jest żywy: odkrycie nowego pojęcia biznesowego = refactoring nazw.

Trade-off: to kosztuje dyscyplinę i czas rozmów z biznesem. Zwraca się w domenach złożonych biznesowo; w technicznym CRUD-zie bez logiki — niekoniecznie (o tym w "kiedy NIE" poniżej).

### Bounded context — kontekst ograniczony

Kluczowa obserwacja Evansa, która odróżnia DDD od naiwnego modelowania: **nie istnieje jeden spójny model całej firmy**. Te same słowa znaczą różne rzeczy w różnych częściach organizacji — i to nie jest błąd do naprawienia, tylko właściwość rzeczywistości do zaakceptowania.

Kanoniczny przykład: **"klient" w sprzedaży vs w windykacji**. Dla sprzedaży klient to szansa: lead, potrzeby, historia kontaktów, scoring potencjału — może jeszcze nic nie kupił. Dla windykacji klient to ryzyko: zaległe faktury, harmonogram spłat, etap postępowania — z definicji ktoś, kto już kupił i nie zapłacił. Próba zbudowania JEDNEJ klasy `Klient` obsługującej oba światy daje potwora: 70 pól, z których każdy moduł używa siedmiu, walidacje wzajemnie sprzeczne ("klient musi mieć NIP" — windykacja tak, lead nie), i sprzężenie, przez które zmiana dla windykacji psuje sprzedaż.

**Bounded context** to **granica, wewnątrz której dany model i język są spójne i jednoznaczne**. Wewnątrz kontekstu "Sprzedaż" słowo "klient" ma dokładnie jedno znaczenie. W kontekście "Windykacja" — inne, i to jest **dobrze**: dwa konteksty, dwa modele `Klient`, każdy mały i precyzyjny, powiązane co najwyżej wspólnym identyfikatorem. Analogia: słowo "zamek" w kontekście budownictwa, krawiectwa i historii — nikt nie próbuje stworzyć jednej definicji zamka; kontekst rozmowy rozstrzyga znaczenie.

Przykład z twojego świata — **"członek" (member) w systemie dla siłowni**:
- kontekst **Sprzedaż/Umowy**: członek = strona umowy — karnet, okres, cennik, zgody marketingowe;
- kontekst **Kontrola dostępu**: członek = uprawnienie do wejścia — aktywny/nieaktywny, strefy, godziny; bramka nie potrzebuje wiedzieć o cenniku, potrzebuje odpowiedzi w 200 ms (pamiętasz wymaganie dostępności z lekcji 01);
- kontekst **Rozliczenia**: członek = płatnik — metoda płatności, zaległości, faktury;
- kontekst **Treningi/Grafik**: członek = uczestnik zajęć — rezerwacje, obecności, ograniczenia (np. limit wejść).

Cztery znaczenia, cztery modele. Systemy, które tego nie rozdzieliły, mają jedną tabelę `Members` z dziesiątkami kolumn i każdy zespół boi się ją tknąć — brzmi znajomo?

Jak **znajdować** granice (heurystyki): (1) słuchaj języka — gdy to samo słowo zaczyna znaczyć co innego albo eksperci z dwóch działów poprawiają się nawzajem, właśnie przekroczyłeś granicę; (2) patrz na strukturę organizacji — granice działów często (nie zawsze!) pokrywają się z granicami kontekstów (prawo Conwaya: architektura odzwierciedla strukturę komunikacji organizacji); (3) różne tempo zmian i różne atrybuty jakościowe (bramka: dostępność i latencja; rozliczenia: spójność i audytowalność) to silny sygnał osobnych kontekstów. Warsztat grupowy do tego (event storming) poznasz w module 6 — na razie wystarczą heurystyki i własna pamięć domeny.

### Context mapping — mapa kontekstów i typy relacji

Konteksty muszą ze sobą rozmawiać (windykacja musi wiedzieć, co sprzedano). **Context map** to diagram: konteksty jako bąble + relacje między nimi, przy czym relacje opisują nie tylko przepływ danych, ale **układ sił**: kto się do kogo dostosowuje. To wymiar polityczno-organizacyjny, którego nie ma na C4 — i dlatego context map uzupełnia C4, a nie dubluje.

Najważniejsze typy relacji (notacja: **U**pstream — dostawca modelu, wpływa; **D**ownstream — odbiorca, podlega wpływowi):

- **Partnership (partnerstwo)** — dwa zespoły/konteksty zależą od siebie wzajemnie i planują zmiany razem; nikt nie dyktuje. Drogie w koordynacji, sensowne gdy sukces jest naprawdę wspólny.
- **Customer–Supplier (klient–dostawca)** — upstream dostarcza, ale downstream jest "klientem", którego potrzeby trafiają do planu dostawcy; jest negocjacja. Np. zespół Grafiku (D) zgłasza potrzeby zespołowi Umów (U) i ten je realizuje.
- **Conformist (konformista)** — downstream przyjmuje model upstreama **bez negocjacji i bez tłumaczenia**, bo nie ma siły przebicia (upstream to zewnętrzny gigant albo zespół, który "nie ma czasu"). Tanio i szybko, ale obcy model rozlewa się po twoim kodzie i każda zmiana upstreama cię łamie. Świadomy konformizm bywa OK (np. wobec API VAT-owego ministerstwa); nieświadomy to dług.
- **Anti-Corruption Layer (ACL, warstwa antykorupcyjna)** — downstream buduje **warstwę tłumaczącą** obcy model na własny: na granicy siedzi translator (adapter), a wewnątrz kontekstu obcych pojęć nie ma. Analogia: ambasada z tłumaczem — delegacje rozmawiają, ale prawo wewnątrz kraju pozostaje twoje. Klasyczne miejsce: integracja z systemem legacy lub zewnętrznym vendorem. Znasz ten wzorzec z produkcji, nawet jeśli nazywał się "klasa mapująca payload partnera na nasze DTO" — różnica w tym, że ACL robi się świadomie i pilnuje, by przeciek modelu był zerowy. Trade-off: dodatkowy kod i mapowania (nuda, utrzymanie) w zamian za niezależność modelu i jedną klasę uderzeniową, gdy vendor zmieni API.
- Pozostałe, do rozpoznawania: **Shared Kernel** (wspólny kawałek modelu/kodu dwóch kontekstów — wymaga koordynacji każdej zmiany, używać oszczędnie), **Open Host Service / Published Language** (upstream wystawia publiczne, udokumentowane API ze stabilnym językiem — tak Perfect Gym wystawiał API integratorom), **Separate Ways** (świadomy brak integracji — czasem najtańsza relacja to żadna).

### Przykład: mapa kontekstów systemu dla siłowni

```
                       ┌──────────────────┐
                       │  Umowy/Sprzedaż  │  „członek = strona umowy"
                       └──┬────────────┬──┘
                  U       │            │       U
        ┌─────────────────┘            └──────────────────┐
        │ publikuje zdarzenia:                            │ Customer–Supplier
        │ UmowaAktywowana / Zawieszona…                   │ (Rozliczenia zgłasza
        ▼ D                                               ▼ D     potrzeby)
┌───────────────────┐                            ┌──────────────────┐
│ Kontrola dostępu  │                            │   Rozliczenia    │
│ „członek =        │                            │ „członek =       │
│  uprawnienie"     │                            │   płatnik"       │
│ (Conformist —     │                            └───┬──────────────┘
│  przyjmuje        │                          U     │  ACL po stronie
│  zdarzenia Umów   │                                │  Rozliczeń
│  wprost; prostota │                                ▼ D
│  > autonomia)     │                       [Bramka płatności]
└───────────────────┘                       (system zewnętrzny)
        ▲ D
        │ Open Host Service: publiczne API
        │ + ACL po naszej stronie
[Agregator fitness — system zewn.]          ┌──────────────────┐
                                            │  Grafik/Treningi  │ ←─ Customer–Supplier
                                            │ „członek =        │     z Umowami
                                            │   uczestnik"      │
                                            └──────────────────┘
```

Zwróć uwagę na decyzje (każda to trade-off do obrony): Kontrola dostępu jest conformistą wobec zdarzeń Umów (prostota i latencja > autonomia modelu — bramka i tak potrzebuje ułamka danych), ale wobec zewnętrznego agregatora stawiamy ACL (vendor zmienia API, nie chcemy, by jego pojęcia zalały nasz model). Rozliczenia mają ACL do bramki płatności z tego samego powodu.

### Kiedy NIE używać DDD strategicznego

- Domena jest trywialna (CRUD bez reguł biznesowych, formularz nad tabelą) — narzut warsztatów i rozdzielania modeli się nie zwróci. DDD płaci się złożonością modelowania za opanowanie złożoności **biznesowej**; gdzie jej nie ma, płacisz za nic.
- System jest mały i jednozespołowy o jednym spójnym języku — jeden kontekst to też legalna odpowiedź; nie tnij na siłę (DayChunks dziś to prawdopodobnie 1–2 konteksty, np. Planowanie + Statystyki, i tyle).
- Uwaga na odwrotny błąd: konteksty cięte po warstwach technicznych ("kontekst bazy danych", "kontekst API") albo po rzeczownikach z bazy — to nie są konteksty, to moduły techniczne w przebraniu.

## Praktyka

- [ ] Wypisz z pamięci 5+ pojęć z domeny siłowni, które znaczyły co innego w różnych częściach systemu/organizacji (kandydaci: członek, karnet, wejście, zajęcia, płatność, klub). Dla 2 z nich rozpisz różnice znaczeń per kontekst — to dowód istnienia granic.
- [ ] **Ćwiczenie główne:** narysuj mapę kontekstów dla systemu klasy "platforma dla sieci siłowni" (możesz wyjść od przykładu z lekcji, ale rozbuduj go o to, co pamiętasz: CRM/marketing? raportowanie centrali? aplikacja członkowska?). Dla **każdej** relacji nazwij typ (partnership / customer–supplier / conformist / ACL / OHS / separate ways) i w jednym zdaniu uzasadnij wybór.
- [ ] Dla jednej relacji z mapy, gdzie postawiłeś ACL, naszkicuj w C# interfejs tej warstwy: model zewnętrzny (DTO vendora) → translator → model własny kontekstu. Wystarczy 20–30 linii, chodzi o poczucie, gdzie żyje tłumaczenie.
- [ ] DayChunks: zdecyduj pisemnie, ile kontekstów ma dziś sens (uzasadnij, nawet jeśli odpowiedź brzmi "jeden") — i zapisz to jako ADR lub wpis w dzienniku decyzji.
- [ ] Pytanie kontrolne: dlaczego "jedna wspólna encja Customer dla całej firmy" to antywzorzec, skoro intuicyjnie wygląda na eliminację duplikacji? (Odpowiedz przez pojęcia coupling i niezależność zmian.)

## Artefakt

Mapa kontekstów systemu dla siłowni (diagram + legenda typów relacji + akapit uzasadnień) w repo nauki: `context-map-gym-platform.md` (+ grafika). Dodatkowo notatka o kontekstach DayChunks. Mapa kontekstów + C4 z lekcji 03 to razem komplet "jak myślę o strukturze systemu" — kandydat na drugi materiał Featured, jeśli wyjdzie czytelnie.

## Definition of Done

- [ ] Umiesz wyjaśnić różnicę DDD strategiczne vs taktyczne i uzasadnić, czemu architekt zaczyna od strategicznego.
- [ ] Umiesz zdefiniować bounded context i ubiquitous language prostym językiem, z przykładem "klient w sprzedaży vs windykacji" lub własnym.
- [ ] Na twojej mapie każda relacja ma nazwany typ i uzasadnienie; umiesz dla dowolnej z nich powiedzieć, co by się zmieniło, gdyby typ był inny (np. conformist zamiast ACL: co oszczędzamy, co ryzykujemy).
- [ ] Umiesz wskazać 2 sytuacje, w których DDD strategiczne to przerost formy.

## Materiały

1. **"Learning Domain-Driven Design"** (Vlad Khononov, O'Reilly) — część I (strategic design): najprzystępniejsze współczesne omówienie; rozdz. o context mapping czytaj z własną mapą obok. (Evans "Domain-Driven Design" to źródło pierwotne — sięgnij, gdy Khononov zostawi niedosyt; do tej lekcji nie jest konieczny.)
2. **Context Mapping — materiały DDD Crew** (github.com/ddd-crew/context-mapping) — darmowa, zwięzła ściąga wszystkich typów relacji z grafikami; idealna jako legenda do twojej mapy.
