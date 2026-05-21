# CHANGELOG

Format zgodny z [Keep a Changelog](https://keepachangelog.com/pl/1.1.0/), wersjonowanie [SEMVER](https://semver.org/lang/pl/).

## [0.1.1-alpha] - 2026-05-21

### Dodane (cherry-pick 3 patternow operacyjnych z gregmos/PII-Shield MIT)

- **Pattern 1: `pseudonim_audit.log` "proves no PII leaves"** - osobny plain-text log file (`~/.config/contract-review-pl/pseudonim_audit.log`) czytelny dla Inspektora ochrony danych. Per linia: timestamp, event, doc_id, source_hash sha256, entity counts per typ, bytes_in/out. Linia `llm-call-out` z `PII_count=0` jest **dowodem** ze prompt jest czysty (jezeli > 0 skill ZATRZYMUJE wywolanie). Inspektor moze otworzyc log w 5 min i zweryfikowac: (1) liczba pseudonim-applied == liczba llm-call-out, (2) wszystkie llm-call-out maja PII_count=0, (3) mapping-cleanup dziala. Implementacja: `helpers/audit-logger.py`.

- **Pattern 2: `session_id` w docx custom properties** - kazdy generowany `.docx` ma w custom properties Word: `MateMaticContractReviewSessionId` (ULID), `MateMaticContractReviewToolVersion`, `MateMaticContractReviewTimestamp`. Tygodnie pozniej prawnik moze `contract-review-pl --reopen-session ~/Desktop/RAPORT-...docx` - skill rozpoznaje sesje, oferuje historie / re-deanonymize / powtorny raport. Implementacja: `helpers/docx-session-tagging.py`.

- **Pattern 3: TTL mapping cleanup** (default 7 dni, configurable w `~/.config/contract-review-pl/policy.yaml`) - mapping placeholder ↔ wartosc zywa wygasa po N dniach. Skill przy kazdym uruchomieniu sprawdza expired sessions i usuwa. RODO art. 5 ust. 1 lit. e (ograniczenie przechowywania). Implementacja: `helpers/mapping-cleanup.py`.

### Pochodzenie patternow 1-3

Cherry-pick z [gregmos/PII-Shield](https://github.com/gregmos/PII-Shield) (MIT, snapshot v2.0.2 2026-05-21, autor Grigorii Moskalev - Microsoft Presidio team, 92 gwiazdek). NIE forkujemy kodu - implementacja napisana od zera pod Python helpers + polskie nazewnictwo + integracja z 4 zasadami konstytucyjnymi v1.0.0.

### Czego NIE bierzemy z PII-Shield

- GLiNER zero-shot NER + ONNX Runtime (>100 MB modeli) - lamie nasz model "regex + checksum + gazetteer" deterministyczny
- MCP server architecture - zostajemy skillem Claude Code (skill, nie konektor)
- 33 entity types US/UK/DE/FR/IT/ES/CY - mamy juz polskie PII first (PESEL/NIP/REGON/IBAN PL/adresy)
- AES-GCM session archive z scrypt - planowane v0.2.0 (osobny refactor sessions architecture)

### Roadmap v0.2.0+

- AES-GCM session archive z scrypt-derived key (pattern 4 z PII-Shield, pominiety w v0.1.1)
- Default schematy dla M&A / umowy dostawcze / umowy powierzenia art. 28 RODO
- Eksport `.csv` raw dataset
- Composability code z lpm-pl

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
