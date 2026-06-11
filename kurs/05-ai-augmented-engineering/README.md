# Moduł 5 — AI-augmented engineering

**Czas trwania:** 3–4 tygodnie efektywnej pracy, ale rozłożone **częściowo równolegle z modułami 2–4** (szczegóły niżej).
**Pozycja w kursie:** zaczynasz go razem z modułem 2 i prowadzisz jako drugi tor. Tylko lekcja 04 wymaga osobnego, skupionego czasu.

## Cel modułu

Ten moduł różni się od pozostałych: nie uczysz się tu rzeczy nowej, tylko **dokumentujesz i pogłębiasz coś, co już robisz codziennie** — pracę z Claude Code przy DayChunks. Różnica między "używam AI" a "mam opisany, powtarzalny workflow pracy z agentem i wiem, gdzie agent systematycznie błądzi" jest dokładnie taka, jak różnica między "robiłem kolejki" a "umiem wyjaśnić idempotency i outbox" z modułu 2. Pierwsze ma dziś każdy. Drugie — prawie nikt z 20-letnim stażem.

To Twój wyróżnik rynkowy. Na rozmowie na Solution Architecta w 2026 pytanie "jak używacie AI w developmencie?" pada niemal zawsze — i większość kandydatów odpowiada ogólnikami. Ty po tym module odpowiesz konkretami: specyfikacje zamiast promptów, checklist review pracy agenta, log eksperymentów z liczbami, jeden feature AI zbudowany end-to-end i rozumienie, kiedy RAG a kiedy długi kontekst. Dodatkowo moduł działa jak dźwignia dla reszty kursu: lepszy workflow z agentem przyspiesza moduły 2–4 i Capstone.

## Lekcje

| Lekcja | Temat | Artefakt |
|---|---|---|
| [01](01-spec-driven-development/) | Spec-driven development — specyfikacja zamiast promptu | szablon spec + CLAUDE.md dla repo messagingowego + porównanie spec vs ad-hoc |
| [02](02-code-review-pracy-agenta/) | Code review pracy agenta — gdzie AI systematycznie błądzi | checklist review + log eksperymentów (szablon + pierwsze wpisy) |
| [03](03-ai-w-cyklu-architekta/) | AI w cyklu architekta: ADR-y, diagramy, trade-offy, legacy | workflow "AI w pracy architektonicznej" + ADR i diagram wytworzone tym workflow |
| [04](04-funkcje-ai-w-produkcie/) | Funkcje AI w produkcie: LLM z .NET, RAG, koszty, ewaluacja | działający mały feature AI end-to-end + notatka architektoniczna |

## Jak prowadzić ten moduł równolegle z modułami 2–4

Moduł 5 **nie dostaje własnych bloków deep work kosztem modułów 2–4**. Lekcje 01–03 wykonujesz *na materiale* tamtych modułów — to nie jest dodatkowa praca, tylko inna jakość tej samej pracy. Konkretny plan:

| Kiedy | Co z modułu 5 | Na czym ćwiczysz |
|---|---|---|
| Moduł 2, tygodnie 1–2 | **Lekcja 01** (spec-driven development) | zadania w repo messaging-patterns — od pierwszego dnia deleguj je agentowi przez spec, nie przez czat |
| Moduł 2, tygodnie 3–5 | **Lekcja 02** (code review pracy agenta) | kod, który agent pisze do repo messagingowego; tu zaczynasz log eksperymentów |
| Moduł 3 (dowolny tydzień) | **Lekcja 03** (AI w cyklu architekta) | ADR-y i diagramy, które i tak tworzysz w module 3; przegląd własnego legacy |
| Po module 4 (lub w przerwie między 3 a 4) | **Lekcja 04** (funkcje AI w produkcie) | **osobne 1,5–2 tygodnie skupionej pracy** — to jedyna lekcja z nowym materiałem teoretycznym i własnym projektem |

Mechanika dnia: gdy w module 2 siadasz do implementacji outboxa, najpierw poświęcasz 15 minut na spec (lekcja 01), potem agent pracuje, potem robisz review wg checklisty (lekcja 02) i dopisujesz wpis do logu. Koszt: ~20–30 minut dziennie ponad to, co i tak byś robił. Zysk: po 4 tygodniach masz udokumentowany workflow z dowodami, a nie deklarację.

Lekcja 04 jest inna — wymaga nauki nowych pojęć (embeddings, RAG, ewaluacja) i zbudowania czegoś od zera. Zaplanuj na nią pełne bloki deep work, tak jak na zwykłą lekcję kursu.

## Artefakty modułu

- [ ] **Udokumentowany workflow pracy z agentem** — jeden plik (np. `ai-workflow.md` w repo nauki) spinający: szablon spec, checklist review, zasady z lekcji 03. To jest rzecz, którą pokazujesz/opowiadasz na rozmowie.
- [ ] **Log eksperymentów** (lekcja 02) — prowadzony ciągle, minimum 10 wpisów do końca modułu. To surowiec na posty.
- [ ] **Jeden mały feature AI end-to-end** (lekcja 04) — działający kod w repo.
- [ ] **Seria postów "AI-augmented engineering — honest takes"** — 1–2 posty/miesiąc, **długoterminowo** (seria nie kończy się z modułem). Pierwsze tematy wychodzą wprost z logu: "Czego agent nie zrobi dobrze, choćbyś prosił", "Spec zamiast promptu: jeden tydzień, dwa podejścia, liczby", "Zbudowałem feature AI w .NET — co mnie zaskoczyło w kosztach".

## Definition of Done modułu

- [ ] Masz spisany, powtarzalny workflow pracy z agentem (spec → delegacja → review → log) i **stosujesz go faktycznie** w pracy nad modułami 2–4, nie tylko opisałeś.
- [ ] Log eksperymentów ma ≥10 wpisów, w tym przykłady porażek agenta z Twoją diagnozą przyczyny.
- [ ] Umiesz na rozmowie odpowiedzieć konkretami na pytania: "jak pracujesz z agentem?", "gdzie AI błądzi i jak to łapiesz?", "jak byś zaprojektował feature oparty o LLM i co z kosztami/niedeterminizmem?".
- [ ] Feature AI z lekcji 04 działa i ma notatkę architektoniczną (model, koszty, obsługa niedeterminizmu, granice bezpieczeństwa).
- [ ] Opublikowane minimum 2 posty z serii "honest takes" i seria ma zaplanowaną kontynuację.

## Materiały bazowe modułu (max 2 źródła)

1. **[Anthropic — dokumentacja Claude Code i przewodniki engineering](https://docs.anthropic.com/)** — w szczególności materiały o agentic coding, plikach CLAUDE.md i budowaniu na API. Źródło pierwotne dla lekcji 01–02 i części 04.
2. **[Microsoft Learn — dokumentacja Microsoft.Extensions.AI i AI dla .NET](https://learn.microsoft.com/dotnet/ai/)** — kanoniczne źródło dla lekcji 04 (wywołania LLM z .NET, structured outputs, function calling, embeddings). Sprawdzaj bieżący stan — ekosystem zmienia się co kwartał.

Po zaliczeniu: odhacz Moduł 5 w `architect-track-plan.md`, zaktualizuj `progress.md`. Seria postów i log eksperymentów **biegną dalej** — to nawyki, nie zadania.
