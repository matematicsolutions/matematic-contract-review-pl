# AGENTS.md - matematic-contract-review-pl

Plik standardu [agents.md](https://agents.md) (Linux Foundation / Agentic AI Foundation) - kanoniczne instrukcje dla agentow AI pracujacych z tym repozytorium. Czytany natywnie przez Cursor, Codex (OpenAI), Jules (Google), Devin / Windsurf, Aider, Amp, Factory, GitHub Copilot.

## Cel projektu

`matematic-contract-review-pl` to otwarty **skill Claude Code** do **bulk audit umow** w polskiej kancelarii. Kopiujesz folder PDF/DOCX, definiujesz schemat kolumn, skill zwraca `.docx` z tabela (wiersz = umowa, kolumna = pole) + czerwone flagi + cytaty zrodlowe.

Inspiracja cherry-pick: [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review) (MIT). Tresc skilla **napisana od zera** pod polskie realia (RODO, PoA art. 6, URP art. 3, AI Act).

**To nie jest [Patron](https://github.com/matematicsolutions/patron)**. Patron = produkcyjny hostowany agent z Postgres + audit trail + UI. `contract-review-pl` = lekki skill CLI w Claude Code. Patron Contract Review Module ([ADR-0010](https://github.com/matematicsolutions/patron/blob/main/governance/adr/0010-contract-review-module-tabular.md)) to **wersja produkcyjna** tego samego use case.

**To nie jest [lpm-pl](https://github.com/matematicsolutions/lpm-pl)**. lpm-pl = portfolio management spraw (status / scope / RAID). contract-review-pl = bulk extraction umow. Ortogonalne tematy, ale composable.

## Kontekst MateMatic (TWARDE OGRANICZENIA)

Repo prowadzi [MateMatic Solutions](https://matematicsolutions.com). Skill dotyczy:

- **Tajemnica zawodowa** (PoA art. 6, URP art. 3) - skill **NIE wysyla** niezanonimizowanych imion / PESEL / adresow do LLM. Pseudonimizacja PII PRZED kazdym wywolaniem LLM ([CONSTITUTION.md](./CONSTITUTION.md) Zasada 1).
- **RODO art. 5/25/30/32** - minimalizacja, privacy by design, mapowanie placeholderow trzymane lokalnie, gitignore.
- **Multi-provider** ([CONSTITUTION.md](./CONSTITUTION.md) Zasada 2) - Ollama / Claude / Gemini / GPT wymiennie. Default Ollama lokalny dla RODO-safe maximum. Klucze providerow w `~/.config/contract-review-pl/providers.yaml` (gitignore), nigdy w kodzie.
- **Mechaniczna walidacja cytatu** ([CONSTITUTION.md](./CONSTITUTION.md) Zasada 3) - kazda komorka tabeli ma cytat zwerifikowany w tekscie zrodlowym (substring match). Brak cytatu = komorka `null` + `confidence: failed`. Halucynacja niemozliwa na warstwie struktury.

## Struktura repo

```
skills/
  contract-review-pl/
    SKILL.md             - implementacja skill (frontmatter + tresc)
    helpers/             - skrypty Python (pseudonim, walidacja, .docx gen)
examples/
  portfel-nda-przyklad/  - 3 zanonimizowane NDA + schemat.yaml + oczekiwany output
docs/
  preprocessing-decyzja.md - drabinka decyzyjna preprocesora PDF
CONSTITUTION.md          - 4 zasady konstytucyjne v1.0.0
SPEC.md                  - specyfikacja techniczna v0.1.0-alpha
CHANGELOG.md             - historia wersji
README.md                - opis dla ludzi
LICENSE                  - Apache 2.0
AGENTS.md                - ten plik
CLAUDE.md                - wskaznik z @AGENTS.md import dla Claude Code
```

## Build i test

Repo nie ma kompilacji - to skill Markdown + helpers Python.

"Test" = przepuszczenie skilla przez **zanonimizowany portfel testowy** w [examples/portfel-nda-przyklad/](./examples/portfel-nda-przyklad/) i porownanie outputu z `expected/RAPORT.docx`.

Test krytyczny **pseudonimizacji**:
- Mock LLM provider (zapis promptu do pliku zamiast wysyl)
- Assert: zapisany prompt **NIE zawiera** PESEL / imienia / nazwiska z testowej umowy
- Assert: zapisany prompt **zawiera** placeholdery `[OSOBA_1]` itp.

Instalacja lokalna (Claude Code):

```bash
cd ~/.claude/skills/
git clone https://github.com/matematicsolutions/matematic-contract-review-pl
ln -s matematic-contract-review-pl/skills/contract-review-pl contract-review-pl
```

Na Windows zamiast symlinka - kopia folderu `skills/contract-review-pl/` do `~/.claude/skills/`.

## Zasady pisania skilla

- **Polski jezyk** - skill rozmawia z prawnikiem po polsku, frontmatter description triggeruje po polskich frazach ("audyt umow", "tabular review", "bulk audit NDA", "przejrzyj portfel kontraktow").
- **Judgment calls embedded** - skill rozumie kiedy "wartosc do uzgodnienia" znaczy "luka", kiedy klauzula jest niejednoznaczna, kiedy brakuje cytatu zrodlowego.
- **RAG framework** - kazda umowa dostaje status Czerwony / Bursztynowy / Zielony per zdefiniowane czerwone flagi.
- **Output `.docx` z naglowkiem kancelarii** - gotowy do wyslania do partnera, nie do przepisywania.
- **Bez polskich znakow w commit messages** (konwencja organizacji).
- **Wewnetrzny review tresci (2 rundy)** przed kazdym commitem zmieniajacym tresc SKILL.md / README.md / CONSTITUTION.md.

## Czego NIE robic (twarde reguly)

- **NIE wysylaj niezanonimizowanych danych osobowych do LLM** - bezwzglednie. Zasada konstytucyjna 1.
- **NIE faworyzuj zadnego providera** w defaultach (oprocz Ollama dla RODO-safe). Wszystkie 4 maja byc rownouprawnione. Zasada konstytucyjna 2.
- **NIE akceptuj odpowiedzi LLM bez cytatu** - jezeli LLM zwroci wartosc bez cytatu lub z cytatem ktorego nie ma w tekscie, komorka `null` + `confidence: failed`. Zasada konstytucyjna 3.
- **NIE nazywaj firm w summary outputu** - "Strona A", "Dostawca", "Klient". Zasada konstytucyjna 4.
- **NIE commituj kluczy providerow** ani prawdziwych umow klientow w `examples/`.
- **NIE buduj web UI** - skill jest CLI-only przez Claude Code. Web UI to Patron Contract Review Module (ADR-0010), nie ten skill.

## Zrodla prawdy (kolejnosc czytania)

1. [README.md](./README.md) - opis dla ludzi
2. [CONSTITUTION.md](./CONSTITUTION.md) - 4 zasady konstytucyjne v1.0.0
3. [SPEC.md](./SPEC.md) - specyfikacja techniczna v0.1.0-alpha
4. [skills/contract-review-pl/SKILL.md](./skills/contract-review-pl/SKILL.md) - implementacja
5. [examples/portfel-nda-przyklad/](./examples/portfel-nda-przyklad/) - prawdziwe pelne przejscie
6. [CHANGELOG.md](./CHANGELOG.md) - historia wersji

## Kompatybilnosc agentow

Ten plik (AGENTS.md) jest standardem [agents.md](https://agents.md) (Linux Foundation). Skill `contract-review-pl` pisany pod Claude Code, ale pattern (pseudonimizacja PRZED LLM, mechaniczna walidacja cytatu, `.docx` output) jest **agent-agnostic** - mozesz przepisac SKILL.md pod Cursor / Codex / Devin adaptujac tylko frontmatter.

Dla Claude Code dodatkowo istnieje plik [CLAUDE.md](./CLAUDE.md) importujacy ten dokument przez `@AGENTS.md`.

## Licencja i atrybucja

- **Apache 2.0** - patrz [LICENSE](./LICENSE). Mozesz wziac, modyfikowac, sprzedawac wdrozenie. Wymagamy atrybucji.
- Pattern UX: cherry-pick z [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review) (MIT, snapshot 2026-05-21).
- Tresc skilla: napisana od zera pod polskie realia.

Cytowanie: *MateMatic Solutions (2026), matematic-contract-review-pl - tabular review umow dla polskiej kancelarii, https://github.com/matematicsolutions/matematic-contract-review-pl, Apache 2.0.*
