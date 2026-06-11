# Moduł 4 — Nowoczesny stack .NET

**Czas trwania:** 2–3 tygodnie (moduł celowo lekki)
**Pozycja w kursie:** po module 3 (Azure), częściowo może iść równolegle z modułem 5. Korzysta z repo messaging-patterns z modułu 2 — to na nim wykonasz główne ćwiczenie refaktoryzacyjne.

## Cel modułu

Domknąć luki względem aktualnego ekosystemu .NET — **bez kursowania od zera**, bo fundament masz mocniejszy niż większość rynku. Po 20+ latach w .NET nie potrzebujesz lekcji o LINQ ani async/await. Potrzebujesz mapy tego, co zmieniło się w ostatnich latach: jak wygląda idiomatyczny projekt w 2026 (minimal APIs, vertical slices, nullable reference types włączone na serio), jak testuje się dziś aplikacje integracyjne (Testcontainers zamiast mocków wszystkiego) i które nowości mają znaczenie architektoniczne, a które są tylko cukrem składniowym.

Ten moduł to nie nauka nowego — to **aktualizacja słownika i nawyków**. Różnica jest praktyczna: na rozmowie rekrutacyjnej kandydat z 20-letnim stażem, którego przykładowy kod wygląda jak z 2014 roku, wysyła sygnał "zatrzymał się". Ten sam kandydat z kodem, który wygląda jak współczesny .NET, wysyła sygnał "20 lat doświadczenia + na bieżąco" — a to jest dokładnie profil, za który płaci się najwięcej.

## Lekcje (przerabiaj po kolei)

| Lekcja | Temat | Artefakt |
|---|---|---|
| [01](01-stan-ekosystemu-dotnet/) | Stan ekosystemu .NET — mapa dla wracającego z dłuższej podróży | notatka "stan ekosystemu na dziś" + szkic posta |
| [02](02-modular-monolith-i-vertical-slices/) | Modular monolith i vertical slice architecture w praktyce | repo messaging-patterns zrefaktoryzowane do vertical slices |
| [03](03-nowoczesne-testy/) | Nowoczesne testy: Testcontainers, testy architektury, contract testing | testy integracyjne + test architektury w repo |
| [04](04-wydajnosc/) | Wydajność okiem architekta: profilowanie, EF Core, source generators | benchmark + notatka o regułach perf |

## Rozkład na tygodnie

Rytm z modułu 0: 4 bloki deep work po 2h tygodniowo.

| Tydzień | Lekcje | Akcent |
|---|---|---|
| 1 | 01 + start 02 | Mapa ekosystemu (1 blok wystarczy na teorię, 1 na weryfikację stanu i notatkę), potem start refaktoryzacji do vertical slices. |
| 2 | 02 + 03 | Dokończenie refaktoryzacji, testy z Testcontainers i test architektury. To serce modułu — większość czasu przy klawiaturze, nie przy lekturze. |
| 3 | 04 + domknięcie | Wydajność (przegląd + jeden benchmark), napisanie i publikacja posta modułowego, przegląd Definition of Done. |

Jeśli idziesz szybko, moduł da się zamknąć w 2 tygodnie — wtedy lekcję 04 potraktuj jako 1–2 bloki przeglądowe. Nie odwrotnie: nie poświęcaj refaktoryzacji (lekcja 02–03) na rzecz pogłębiania perf — wartość tego modułu powstaje w kodzie, który po nim zostaje.

## Artefakty modułu

- [ ] **Post: "Co realnie zmieniło się w .NET przez ostatnie lata — okiem kogoś, kto pamięta WebForms"** — materiał zbierasz przez cały moduł (lekcja 01 daje strukturę, lekcje 02–04 dają konkrety z własnego kodu). Publikacja w tygodniu 3. To post z dużym potencjałem zasięgu: łączy nostalgię (ViewState, `<asp:GridView>`) z konkretną, aktualną wiedzą.
- [ ] **Repo messaging-patterns w wersji "modern .NET"** — vertical slices, testy z Testcontainers, test architektury. Wzmacnia repo z modułu 2 jako kandydata do Featured i bazę pod Capstone.
- [ ] Mniejsze artefakty per lekcja: notatka o stanie ekosystemu, benchmark, reguły perf — szczegóły w lekcjach.

## Definition of Done modułu

Moduł 4 jest zaliczony, gdy **wszystkie** poniższe są prawdą:

- [ ] **Test główny:** Twój kod ćwiczeniowy wygląda jak współczesny, idiomatyczny .NET — ktoś przeglądający repo messaging-patterns nie pozna po kodzie, że autor zaczynał od WebForms. Konkretnie: minimal APIs lub świadomie wybrane kontrolery, nullable reference types włączone i respektowane, records tam gdzie pasują, struktura per feature, testy na prawdziwych zależnościach w kontenerach.
- [ ] Umiesz w 5 minut opowiedzieć, czym różni się vertical slice architecture od clean architecture i **kiedy wybrałbyś każdą z nich** — z trade-offami, nie z wyznaniem wiary.
- [ ] Repo ma zielony test architektury, który pilnuje granic modułów, i co najmniej jeden test integracyjny na prawdziwej infrastrukturze (Testcontainers).
- [ ] Wiesz, jakim narzędziem zaczniesz diagnozę problemu wydajnościowego w .NET (i dlaczego nie od "optymalizacji na czuja").
- [ ] Post modułowy jest opublikowany.
- [ ] `progress.md` ma wpisy za każdy tydzień modułu.

Po zaliczeniu: odhacz Moduł 4 w `architect-track-plan.md` i przejdź do modułu 5 (jeśli jeszcze nie biegnie równolegle) lub 6.

## Materiał bazowy modułu (max 2 źródła)

1. **Oficjalna dokumentacja Microsoft Learn** ([learn.microsoft.com/dotnet](https://learn.microsoft.com/dotnet)) — w tym module to celowo źródło numer jeden: weryfikujesz aktualny stan platformy, więc czytasz źródło pierwotne, nie blogi sprzed trzech lat. Szczególnie sekcje "What's new in .NET / C#" i dokumentacja .NET Aspire.
2. **Andrew Lock, "ASP.NET Core in Action"** (Manning, najnowsze wydanie) + jego blog [andrewlock.net](https://andrewlock.net) — najlepsze dostępne wyjaśnienia "dlaczego to tak działa" dla współczesnego ASP.NET Core. Sięgaj punktowo, według tematów lekcji.
