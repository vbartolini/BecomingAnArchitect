# Lekcja 03 — Dziennik decyzji: trening formatu ADR na sobie

## Cel lekcji

Po tej lekcji będziesz miał w repo plik `notes/decisions-log.md` — dziennik własnych decyzji prowadzony w uproszczonym formacie ADR (kontekst → decyzja → konsekwencje) — oraz nawyk dopisywania wpisu za każdym razem, gdy podejmujesz decyzję, która ma konsekwencje na dłużej niż tydzień.

## Dlaczego to ważne

**ADR (Architecture Decision Record)** to krótki dokument zapisujący jedną decyzję architektoniczną: co postanowiono, w jakich okolicznościach i jakim kosztem. To jedno z najtańszych i najbardziej niedocenianych narzędzi architekta — i jeden z tematów, który na rozmowach rekrutacyjnych na role architektoniczne pada wprost ("jak dokumentujesz decyzje?", "opowiedz o decyzji, której żałujesz — skąd wiesz, dlaczego ją podjęto?"). Pełny format ADR poznasz w module 1. Ta lekcja robi coś sprytniejszego: zanim zaczniesz pisać ADR-y "na poważnie", **wytrenujesz sam ruch myślowy na własnych, codziennych decyzjach** — tam, gdzie kontekst znasz doskonale, a stawka błędu jest niska.

Dlaczego trening na sobie działa: trudność ADR-ów nie leży w formacie (trzy nagłówki), tylko w dyscyplinie myślenia — oddzieleniu kontekstu od decyzji i uczciwym nazwaniu konsekwencji, także negatywnych. Ten mięsień ćwiczy się częstotliwością, a decyzji osobistych podejmujesz kilka tygodniowo: pora bloków nauki, publiczność repo, "DayChunks bez backendu". Bonus: dziennik decyzji to gotowy surowiec na posty LinkedIn — wpis "dlaczego DayChunks nie ma backendu" to w zasadzie szkic posta o trade-offach.

## Teoria

### Problem: decyzje znikają, zostają tylko skutki

Znasz to z 20 lat na produkcji: w systemie jest dziwna konstrukcja — kolejka tam, gdzie "wystarczyłby" zwykły call, tabela zduplikowana, integracja przez pliki. Pytasz "dlaczego tak?", a odpowiedź brzmi: "nie wiadomo, tak było, jak przyszedłem". Decyzja została podjęta — być może bardzo dobra, w realiach, których już nikt nie pamięta — ale **kontekst wyparował**. Zostaje kod, który wygląda na błąd, i zespół, który boi się go ruszyć albo, gorzej, "naprawia" go z powrotem na rozwiązanie, które kiedyś świadomie odrzucono.

Naiwne rozwiązanie nr 1: "pamiętam swoje decyzje". Nie pamiętasz — pamiętasz **obecne uzasadnienie**, dorobione wstecz. Pamięć ludzka rekonstruuje, nie odtwarza; po pół roku Twoje "dlaczego" będzie wersją zracjonalizowaną.

Naiwne rozwiązanie nr 2: gruba dokumentacja architektury — dokument Worda na 40 stron, aktualizowany "przy okazji". Dlaczego się psuje: koszt aktualizacji jest tak wysoki, że dokument umiera po drugiej zmianie i zaczyna kłamać. Dokumentacja, która kłamie, jest gorsza niż żadna.

Właściwy wzorzec: **zapis przyrostowy, per decyzja, w momencie podejmowania**. Zamiast jednego wielkiego dokumentu opisującego stan — strumień małych, niemutowalnych wpisów opisujących **zmiany**. Jeśli to brzmi jak event sourcing (wzorzec, w którym stan systemu odtwarza się ze strumienia zdarzeń, a nie z migawki) — to dlatego, że to jest dokładnie ta sama idea, zastosowana do wiedzy zamiast danych. ADR-ów się nie edytuje wstecz; jeśli decyzja się zmienia, dopisujesz nowy wpis, który unieważnia stary. Historia myślenia zostaje nietknięta.

### ADR-lite: kontekst → decyzja → konsekwencje

Pełny ADR (poznasz go w module 1) ma zwykle pola: tytuł, status, kontekst, rozważane opcje, decyzja, konsekwencje. Wersja "lite" do dziennika osobistego tnie to do trzech sekcji — minimum, poniżej którego zapis przestaje mieć wartość:

1. **Kontekst** — sytuacja i siły, które na Ciebie działają, opisane tak, jakby czytał to ktoś obcy (albo Ty za rok). Co jest problemem? Jakie są ograniczenia (czas, energia, pieniądze, rodzina, praca)? Kluczowy test: kontekst ma być prawdziwy **niezależnie od decyzji** — to opis świata, nie uzasadnienie wyboru.
2. **Decyzja** — jedno-dwa zdania w trybie oznajmującym: "Robię X". Nie "chyba spróbuję", nie "rozważam" — dziennik zapisuje decyzje podjęte. Jeśli odrzuciłeś jakąś oczywistą alternatywę, wspomnij ją tutaj jednym zdaniem ("zamiast Y, bo...").
3. **Konsekwencje** — co staje się prawdą po tej decyzji, **w obie strony**. Co zyskujesz, ale też: co Cię to kosztuje, co staje się trudniejsze, co świadomie odpuszczasz. To najważniejsza i najtrudniejsza sekcja — i dokładnie ten mięsień, który odróżnia architekta od entuzjasty. Decyzja bez wad nie istnieje; jeśli nie widzisz minusów, nie przemyślałeś decyzji, tylko ją zaklepałeś.

Analogia: ADR-lite to **paragon za decyzję**. Paragon nie ocenia, czy zakup był mądry — zapisuje co, kiedy i za ile. Po pół roku, patrząc na skutki, możesz uczciwie ocenić: "kupiłem X za Y w warunkach Z — czy dziś, znając skutki, zapłaciłbym znowu?". Bez paragonu zostaje tylko "jakoś tak wyszło".

### Szablon wpisu

Wpisy dopisuj na górze pliku (najnowsze pierwsze), numeruj kolejno:

```markdown
## D-NNN: Krótki tytuł decyzji (RRRR-MM-DD)

**Kontekst:** 2–5 zdań. Sytuacja, ograniczenia, siły. Opis świata,
nie uzasadnienie wyboru.

**Decyzja:** 1–2 zdania w trybie oznajmującym. Robię X (zamiast Y, bo ...).

**Konsekwencje:**
- (+) co zyskuję
- (+) ...
- (–) co mnie to kosztuje / co staje się trudniejsze
- (–) ...
- (?) czego nie wiem — co zweryfikuje czas (opcjonalnie)
```

Zasady prowadzenia:

- **Wpis powstaje w momencie decyzji** (lub maksymalnie do najbliższego przeglądu tygodnia) — nie wstecz po miesiącu, bo wtedy zapisujesz racjonalizację, nie decyzję.
- **Wpisów nie edytujesz.** Zmiana zdania = nowy wpis z adnotacją "unieważnia D-NNN". Dzięki temu widzisz, jak ewoluuje Twoje myślenie — to jest cała wartość dziennika.
- **Próg wejścia: decyzja ma konsekwencje dłuższe niż tydzień.** "Co zjem na obiad" — nie. "Repo publiczne czy prywatne", "uczę się rano czy wieczorem", "DayChunks bez backendu" — tak.
- Cały wpis ma zajmować **5–10 minut**. Jeśli zajmuje 30, piszesz esej, nie rekord.

### Przykładowe wpisy

```markdown
## D-002: Repo nauki publiczne od początku (2026-06-12)

**Kontekst:** Zakładam repo nauki Architect Track (plan, notatki, kod
ćwiczeń). Buduję równolegle widoczność na LinkedIn; repo może być
sygnałem na rynku pracy. Obawa: notatki pisane "na szybko" i błędy
w ćwiczeniach będą widoczne publicznie. Alternatywa: zacząć prywatnie
i upublicznić po module 1, gdy repo nabierze kształtu.

**Decyzja:** Repo jest publiczne od pierwszego commita (zamiast startu
prywatnego, bo "upublicznię później" to decyzja, którą łatwo odkładać
w nieskończoność).

**Konsekwencje:**
- (+) historia commitów buduje wiarygodność od dnia 1 — sygnału nie da
  się podrobić wstecz
- (+) świadomość publiczności podnosi jakość notatek (piszę pełnymi
  zdaniami, definiuję pojęcia) — a to surowiec na posty
- (+) link do repo mogę od razu dać w Featured na LinkedIn
- (–) muszę pilnować twardej zasady: zero treści z pracy zawodowej,
  przykłady z Perfect Gym/Aerotunel tylko zanonimizowane i uogólnione
- (–) presja "jak to wygląda" może spowalniać commitowanie szkiców;
  mitygacja: zasada "commit po każdym bloku" jest ważniejsza niż uroda
- (?) czy ktokolwiek faktycznie tam zajrzy przed etapem rozmów — zweryfikuję
  po 3 miesiącach
```

```markdown
## D-001: DayChunks bez backendu (2026-06-10)

**Kontekst:** Buduję DayChunks po godzinach, w pojedynkę, równolegle
z planem nauki na architekta. Każda godzina na DayChunks konkuruje
z godziną nauki. Backend (API + baza + konta użytkowników) dałby
synchronizację między urządzeniami, ale oznacza hosting, koszty,
utrzymanie, bezpieczeństwo danych użytkowników i RODO. Dane aplikacji
(plan dnia) są małe i naturalnie lokalne.

**Decyzja:** DayChunks działa wyłącznie lokalnie, bez backendu i bez
kont użytkowników (zamiast architektury klient–serwer, bo na tym etapie
płaciłbym pełny koszt utrzymania serwera za funkcję — synchronizację —
której nikt jeszcze nie potrzebuje).

**Konsekwencje:**
- (+) zero kosztów stałych i zero utrzymania produkcji — projekt
  przeżyje miesiące małej aktywności
- (+) brak danych użytkowników po mojej stronie = brak tematu RODO,
  wycieków, backupów cudzych danych
- (+) krótszy time-to-market; czas idzie w produkt, nie w infrastrukturę
- (–) brak synchronizacji między urządzeniami — dla części użytkowników
  to może być deal-breaker
- (–) jeśli kiedyś dodam backend, migracja danych lokalnych użytkowników
  będzie trudniejsza niż start z backendem od zera
- (?) czy brak synchronizacji realnie blokuje adopcję — zweryfikują
  pierwsi użytkownicy
```

Zwróć uwagę na wpis D-001: to jest dokładnie ta decyzja, dla której w module 1 napiszesz pełny, formalny ADR. Będziesz mógł porównać oba zapisy i zobaczyć, co dodaje pełny format (status, rozważane opcje), a co było już w wersji lite.

### Kiedy tego NIE robić (trade-offy)

- **Nie zapisuj wszystkiego.** Dziennik z 40 wpisami miesięcznie umrze w trzy tygodnie, jak każdy przeciążony proces. Próg "konsekwencje > tydzień" jest po to, żeby pisać 1–3 wpisy tygodniowo, nie dziennie.
- **Nie używaj dziennika jako narzędzia podejmowania decyzji.** To rejestr, nie framework decyzyjny — najpierw decydujesz (głową), potem zapisujesz. Jeśli pisanie wpisu zmienia Twoją decyzję, świetnie, to bonus — ale celem jest zapis.
- **Nie poleruj.** Wpis brzydki, ale istniejący, bije wpis idealny, ale nienapisany. To dziennik roboczy, nie publikacja — publikacją stanie się dopiero post, który z niego wytniesz.
- Pytanie kontrolne: **kiedy ADR-lite nie wystarcza?** Gdy decyzja dotyczy wielu osób lub wymaga porównania kilku opcji z wagami — wtedy potrzebny jest pełny ADR z sekcją rozważanych opcji (moduł 1) albo RFC. Dziennik osobisty to świadomie odchudzona wersja na decyzje jednoosobowe.

## Praktyka

- [ ] Utwórz plik `notes/decisions-log.md` z krótkim nagłówkiem (czym jest ten plik, zasady: wpisy niemutowalne, najnowsze na górze, próg "konsekwencje > tydzień") i szablonem wpisu z tej lekcji.
- [ ] Napisz wpis D-001 o realnej, już podjętej decyzji dotyczącej DayChunks (np. własną wersję "DayChunks bez backendu" — przykład z lekcji potraktuj jako wzór formy, treść napisz po swojemu, zgodnie z faktycznym kontekstem).
- [ ] Napisz wpis D-002 o decyzji z tego modułu: publiczność repo (z lekcji 02) **albo** Twój budżet błędów i pora bloków nauki (z lekcji 01).
- [ ] Test obcego czytelnika: przeczytaj oba wpisy udając, że nie znasz autora. Czy kontekst opisuje świat, a nie broni decyzji? Czy w konsekwencjach jest co najmniej jeden uczciwy minus? Popraw, jeśli nie.
- [ ] Dodaj do formatu przeglądu tygodnia (lekcja 01) jedno pytanie: "czy w tym tygodniu podjąłem decyzję wartą wpisu do decisions-log?" — to mechanizm, który utrzyma dziennik przy życiu.
- [ ] Zacommituj plik zgodnie z konwencją z lekcji 02 (np. `notes(m0): dziennik decyzji + D-001, D-002`).

## Artefakt

**`notes/decisions-log.md`** — z nagłówkiem objaśniającym, szablonem wpisu i co najmniej dwoma prawdziwymi wpisami w formacie kontekst → decyzja → konsekwencje. To jest artefakt wymagany przez Definition of Done całego modułu 0.

## Definition of Done

- [ ] Plik `notes/decisions-log.md` istnieje w repo i jest zacommitowany.
- [ ] Zawiera minimum 2 wpisy o realnych, podjętych decyzjach (nie wymyślonych na potrzeby ćwiczenia).
- [ ] Każdy wpis ma wszystkie trzy sekcje, a w konsekwencjach co najmniej jeden szczery minus.
- [ ] Kontekst każdego wpisu przechodzi test obcego czytelnika (zrozumiały bez wiedzy z Twojej głowy).
- [ ] Pytanie o nowe decyzje jest dodane do formatu cotygodniowego przeglądu.

## Materiały

1. Michael Nygard, [*Documenting Architecture Decisions*](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) (2011) — tekst źródłowy, który wprowadził format ADR; krótki, przeczytaj w całości przed modułem 1.
2. [adr.github.io](https://adr.github.io/) — przegląd wariantów formatu ADR (w tym MADR); na razie tylko do orientacji, że "lite" to świadomy wybór z szerszego spektrum.
