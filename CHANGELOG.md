# CHANGELOG

Format zgodny z [Keep a Changelog](https://keepachangelog.com/pl/1.1.0/), wersjonowanie [SEMVER](https://semver.org/lang/pl/).

## [0.1.0-alpha] - 2026-05-21

### Dodane

- Pierwsza wersja MVP skilla `contract-review-pl` (tabular review umow).
- 4 zasady konstytucyjne ([CONSTITUTION.md](CONSTITUTION.md) v1.0.0):
  - RODO-safe by default (pseudonimizacja PII PRZED LLM)
  - Multi-provider LLM (Ollama / Claude / Gemini / GPT, decyzja kancelarii)
  - Cytat fizycznie obecny w tekscie zrodlowym (mechaniczna walidacja)
  - Bez nazywania firm w outputach summary
- Specyfikacja techniczna ([SPEC.md](SPEC.md)) - architektura, format schemat.yaml, algorytm pseudonimizacji, walidacja cytatu, struktura outputu .docx.
- Default schemat NDA w `examples/portfel-nda-przyklad/schemat.yaml`.
- Plik `AGENTS.md` (standard [agents.md](https://agents.md), Linux Foundation / Agentic AI Foundation) + `CLAUDE.md` (wskaznik z `@AGENTS.md` import).

### Pochodzenie

- Pattern UX (tabular review umow + dynamic schema columns + per-cell citation back-jump): inspiracja [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review) (MIT, autor Jamie Tso, snapshot 2026-05-21, commit a2d01ee).
- Tresc skilla (workflow, pseudonimizacja, walidacja cytatu, multi-provider, judgment calls, RAG triage, output .docx z naglowkiem kancelarii, polski jezyk, polskie szablony klauzul): **napisana od zera** pod polskie realia. NIE jest fork ani tlumaczenie.

### Cherry-pick a co nie

- **Bierzemy**: pattern UX, integracja z lokalnym preprocesorem PDF, idea per-cell citation.
- **NIE bierzemy** (anti-pattern Tabular_Review): Gemini-only (multi-provider), API key w frontendzie (provider config lokalny), brak pseudonimizacji (zasada konstytucyjna 1), zero testow (test e2e na zanonimizowanym portfelu wymagany przed merge).

### Status

`v0.1.0-alpha` - **przed walidacja produkcyjna**. NIE uzywaj na zywych aktach klienta zanim nie sprawdzisz na zanonimizowanym portfelu testowym ([examples/portfel-nda-przyklad/](examples/portfel-nda-przyklad/)).

### Planowane v0.2.0+

- Default schematy dla M&A / umowy dostawcze / umowy powierzenia art. 28 RODO
- Eksport `.csv` raw dataset
- Composability z [lpm-pl](https://github.com/matematicsolutions/lpm-pl) risk-and-issues-manager-pl

### Planowane v0.3.0+

- Audit bundle AI Act art. 12 jako artefakt zgodnosci dla projektow contract review
- Walidacja klauzul wzorcowych (np. "czy klauzula limitacji odpowiedzialnosci jest zgodna z art. 473 KC?")
