# CONSTITUTION.md - matematic-contract-review-pl

**Version**: v1.0.0
**Date**: 2026-05-21
**Status**: In force

The four constitutional principles under which we write the `contract-review-pl` skill. Modifying these principles requires a v1.x.0 version bump and approval in a PR.

## Principle 1: GDPR-safe by default

The skill **does NOT send un-anonymized personal data to an LLM**. Pseudonymization of PII BEFORE every LLM call is **absolute**.

What "BEFORE every call" means:
- First name / surname / PESEL / NIP / address / bank account number / email of a natural person are replaced with placeholders `[OSOBA_1]`, `[PESEL_1]`, `[ADRES_1]` etc.
- The placeholder <-> live-value mapping **NEVER** goes to the LLM. It is held in the skill's memory, used only to de-pseudonymize the output locally.
- Company names (KRS - the National Court Register) **are not PII** but may be confidential (a law firm's client). The skill **optionally** pseudonymizes company names (`[FIRMA_1]`) - the user's decision via the `--anonymize-companies` flag.

Consequence:
- The skill **runs slower** (an extra preprocessing layer) - an acceptable cost.
- The skill **requires the use of an LLM that understands pseudonymized text** - all modern LLMs can do this, it is not a problem.

## Principle 2: Multi-provider LLM (vendor-neutrality)

The skill **talks to any LLM** according to the user's configuration:

- **Local Ollama** (default, maximum GDPR-safe) - llama 3.3 / qwen 2.5 / mistral / others
- **Claude** (Anthropic) - sonnet / opus
- **Gemini** (Google) - flash / pro
- **GPT** (OpenAI) - 4o / o1

The choice of LLM is the **law firm's decision**, not the skill's. The skill does not favor any provider in its defaults (except Ollama for GDPR-safe).

Cherry-picked lesson from [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review): that skill **hardcodes** Gemini and bundles the API key into the frontend (a security anti-pattern). We **do not repeat** this - provider keys live in `~/.config/contract-review-pl/providers.yaml` (gitignored), never in the skill's code, never in outputs.

## Principle 3: Citation physically present in the source text

Every table cell **must have a citation** (a fragment of the source contract text) **mechanically verified** (substring match with whitespace tolerance).

Workflow:
1. The LLM returns: cell value + citation (e.g. `wartosc: "5 lat"`, `cytat: "Czas trwania zobowiazania poufnosci wynosi 5 (pieciu) lat"`)
2. The skill **mechanically searches for the citation** in the source contract text (case-insensitive, whitespace-tolerant)
3. **If the citation exists**: cell `confidence: high`, the citation is recorded in the "Source citations" section of the output
4. **If the citation does NOT exist** (LLM hallucination): cell `null` + `confidence: failed`, an entry in the "Gaps" section of the output

Consequence:
- **Hallucination is impossible at the structural layer** - the model may misinterpret a fragment, but it will not invent a clause that physically does not exist.
- The skill **deliberately returns empty cells** instead of false ones - a "gap" is better than "false confident".
- The lawyer gets a **verified dataset**, not an "AI-summarized" probability.

## Principle 4: No naming of companies in summary outputs

The skill **does not name companies in the summary section and red flags**. It says "Party A", "Party B", "Supplier", "Client", "Contractor", "Ordering party".

Consequence:
- The output **can be passed around inside the law firm** (junior to partner) without leaking which contract came from which client.
- The full party names are in the **main table** (columns "Party A" / "Party B") where they are extracted directly from the contracts - because these are documents the lawyer reads anyway.
- The source citations (the section at the end of the report) are **complete** (with names) - this is raw data for verification, not a summary.

## Commit gates (before merge to main)

1. **Internal content review (2 rounds)** on SKILL.md + README.md + CONSTITUTION.md (charges -> fixes -> "ok")
2. **Test on the anonymized portfolio** in `examples/` - the skill produces a sensible `.docx`
3. **PSEUDONYMIZATION validation** - an e2e test that an un-anonymized PESEL / first name **never** goes to the LLM (mock LLM + assert on the prompt content)
4. **Output quality gate** - the table has RAG colors, a "Gaps" section exists, a "Source citations" section exists, the firm letterhead is configurable

## Evolution

The constitution is **versioned with SEMVER**:
- **MAJOR** (v1.0.0 -> v2.0.0): a fundamental change (e.g. dropping multi-provider).
- **MINOR** (v1.0.0 -> v1.1.0): adding a new principle (e.g. principle 5).
- **PATCH** (v1.0.0 -> v1.0.1): clarifying an existing principle without a substantive change.

Every change requires a PR + approval + a CHANGELOG.md bump.

## Related

- [README.md](README.md) - human-facing description
- [SPEC.md](SPEC.md) - technical specification v0.1.0-alpha
- [skills/contract-review-pl/SKILL.md](skills/contract-review-pl/SKILL.md) - implementation of the principles
- [Patron Constitution](https://github.com/matematicsolutions/patron/blob/main/governance/CONSTITUTION.md) - vendor-neutrality Art. 4 (source of Principle 2)
- [AGENTS.md](AGENTS.md) - instructions for AI agents working with this repo
