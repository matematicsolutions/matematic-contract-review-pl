# AGENTS.md - matematic-contract-review-pl

An [agents.md](https://agents.md) standard file (Linux Foundation / Agentic AI Foundation) - canonical instructions for AI agents working with this repository. Read natively by Cursor, Codex (OpenAI), Jules (Google), Devin / Windsurf, Aider, Amp, Factory, GitHub Copilot.

## Project goal

`matematic-contract-review-pl` is an open **Claude Code skill** for **bulk contract audit** in a Polish law firm. You copy in a folder of PDF/DOCX files, define a column schema, and the skill returns a `.docx` with a table (row = contract, column = field) plus red flags plus source citations.

Cherry-pick inspiration: [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review) (MIT). The skill content is **written from scratch** for Polish reality (RODO - the GDPR as implemented in Poland, PoA art. 6, URP art. 3, AI Act).

**This is not [Patron](https://github.com/matematicsolutions/patron)**. Patron = a production, hosted agent with Postgres + audit trail + UI. `contract-review-pl` = a lightweight CLI skill in Claude Code. The Patron Contract Review Module ([ADR-0010](https://github.com/matematicsolutions/patron/blob/main/governance/adr/0010-contract-review-module-tabular.md)) is the **production version** of the same use case.

**This is not [lpm-pl](https://github.com/matematicsolutions/lpm-pl)**. lpm-pl = matter portfolio management (status / scope / RAID). contract-review-pl = bulk contract extraction. Orthogonal topics, but composable.

## MateMatic context (HARD CONSTRAINTS)

The repo is run by [MateMatic Solutions](https://matematicsolutions.com). The skill concerns:

- **Professional secrecy** (PoA art. 6, URP art. 3) - the skill **does not send** un-anonymized names / PESEL (Polish national ID number) / addresses to an LLM. PII is pseudonymized BEFORE every LLM call ([CONSTITUTION.md](./CONSTITUTION.md) Principle 1).
- **RODO (GDPR) art. 5/25/30/32** - minimization, privacy by design, placeholder mapping kept locally, gitignored.
- **Multi-provider** ([CONSTITUTION.md](./CONSTITUTION.md) Principle 2) - Ollama / Claude / Gemini / GPT interchangeably. Default is local Ollama for maximum RODO-safe (GDPR-safe) posture. Provider keys live in `~/.config/contract-review-pl/providers.yaml` (gitignored), never in code.
- **Mechanical citation validation** ([CONSTITUTION.md](./CONSTITUTION.md) Principle 3) - every table cell has a citation verified against the source text (substring match). No citation = cell `null` + `confidence: failed`. Hallucination is impossible at the structural layer.

## Repo structure

```
skills/
  contract-review-pl/
    SKILL.md             - skill implementation (frontmatter + content)
    helpers/             - Python scripts (pseudonymization, validation, .docx gen)
examples/
  portfel-nda-przyklad/  - 3 anonymized NDAs + schemat.yaml + expected output
docs/
  preprocessing-decyzja.md - PDF preprocessor decision ladder
CONSTITUTION.md          - 4 constitutional principles v1.0.0
SPEC.md                  - technical specification v0.1.0-alpha
CHANGELOG.md             - version history
README.md                - human-facing description
LICENSE                  - Apache 2.0
AGENTS.md                - this file
CLAUDE.md                - pointer with @AGENTS.md import for Claude Code
```

## Build and test

The repo has no compilation step - it is a Markdown skill + Python helpers.

"Test" = running the skill against the **anonymized test portfolio** in [examples/portfel-nda-przyklad/](./examples/portfel-nda-przyklad/) and comparing the output with `expected/RAPORT.docx`.

Critical **pseudonymization** test:
- Mock LLM provider (write the prompt to a file instead of sending it)
- Assert: the written prompt **does not contain** any PESEL / first name / surname from the test contract
- Assert: the written prompt **does contain** placeholders `[OSOBA_1]` etc.

Local install (Claude Code):

```bash
cd ~/.claude/skills/
git clone https://github.com/matematicsolutions/matematic-contract-review-pl
ln -s matematic-contract-review-pl/skills/contract-review-pl contract-review-pl
```

On Windows, instead of a symlink - copy the `skills/contract-review-pl/` folder into `~/.claude/skills/`.

## Skill authoring rules

- **Polish language** - the skill talks to the lawyer in Polish, and the frontmatter description triggers on Polish phrases ("audyt umow", "tabular review", "bulk audit NDA", "przejrzyj portfel kontraktow").
- **Judgment calls embedded** - the skill understands when "value to be agreed" means "gap", when a clause is ambiguous, when a source citation is missing.
- **RAG framework** - each contract gets a Red / Amber / Green status per the defined red flags.
- **`.docx` output with a firm letterhead** - ready to send to a partner, not to be rewritten.
- **No Polish diacritics in commit messages** (organization convention).
- **Internal content review (2 rounds)** before every commit that changes the content of SKILL.md / README.md / CONSTITUTION.md.

## What NOT to do (hard rules)

- **Do NOT send un-anonymized personal data to an LLM** - absolutely. Constitutional principle 1.
- **Do NOT favor any provider** in defaults (except Ollama for RODO-safe/GDPR-safe). All 4 must be on equal footing. Constitutional principle 2.
- **Do NOT accept an LLM answer without a citation** - if the LLM returns a value with no citation, or with a citation absent from the text, the cell is `null` + `confidence: failed`. Constitutional principle 3.
- **Do NOT name companies in the output summary** - "Party A", "Supplier", "Client". Constitutional principle 4.
- **Do NOT commit provider keys** or real client contracts in `examples/`.
- **Do NOT build a web UI** - the skill is CLI-only through Claude Code. A web UI is the Patron Contract Review Module (ADR-0010), not this skill.

## Sources of truth (reading order)

1. [README.md](./README.md) - human-facing description
2. [CONSTITUTION.md](./CONSTITUTION.md) - 4 constitutional principles v1.0.0
3. [SPEC.md](./SPEC.md) - technical specification v0.1.0-alpha
4. [skills/contract-review-pl/SKILL.md](./skills/contract-review-pl/SKILL.md) - implementation
5. [examples/portfel-nda-przyklad/](./examples/portfel-nda-przyklad/) - a real full walkthrough
6. [CHANGELOG.md](./CHANGELOG.md) - version history

## Agent compatibility

This file (AGENTS.md) follows the [agents.md](https://agents.md) standard (Linux Foundation). The `contract-review-pl` skill is written for Claude Code, but the pattern (pseudonymization BEFORE the LLM, mechanical citation validation, `.docx` output) is **agent-agnostic** - you can rewrite SKILL.md for Cursor / Codex / Devin by adapting only the frontmatter.

For Claude Code there is additionally a [CLAUDE.md](./CLAUDE.md) file importing this document via `@AGENTS.md`.

## License and attribution

- **Apache 2.0** - see [LICENSE](./LICENSE). You may take, modify, and sell a deployment. We require attribution.
- UX pattern: cherry-picked from [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review) (MIT, snapshot 2026-05-21).
- Skill content: written from scratch for Polish reality.

Citation: *MateMatic Solutions (2026), matematic-contract-review-pl - tabular review of contracts for a Polish law firm, https://github.com/matematicsolutions/matematic-contract-review-pl, Apache 2.0.*
