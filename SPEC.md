# SPEC.md - specyfikacja techniczna v0.1.0-alpha

## Architektura skilla

```
input/
  umowy/                    - folder PDF/DOCX wskazany przez uzytkownika
  schemat.yaml              - definicja kolumn (opcjonalne, default schemat NDA)

skill workflow:
  [1] preprocessing PDF -> tekst (markitdown / opendataloader-pdf / Docling)
  [2] pseudonimizacja PII -> tekst z placeholderami + mapowanie lokalne
  [3] per umowa per kolumna: prompt LLM (provider z config)
  [4] mechaniczna walidacja cytatu (substring match)
  [5] depseudonimizacja outputu lokalnie
  [6] generacja .docx z naglowkiem kancelarii

output/
  RAPORT-contract-review-YYYY-MM-DD.docx
```

## Format `schemat.yaml`

```yaml
naglowek_kancelarii:
  nazwa: "Kancelaria Adwokacka Jan Kowalski"
  adres: "ul. Pilna 1, 00-000 Warszawa"
  logo: "~/.config/contract-review-pl/logo.png"  # opcjonalne

llm_provider: "ollama"  # ollama / claude / gemini / gpt
llm_model: "llama3.3:70b"  # nazwa modelu providera

pseudonimizacja:
  imiona_nazwiska: true
  pesel: true
  nip: true
  adresy: true
  emaile: true
  numery_rachunku: true
  nazwy_firm: false  # opcjonalne, --anonymize-companies

kolumny:
  - id: "data_podpisania"
    prompt: "Jaka jest data podpisania umowy? Format YYYY-MM-DD."
    typ: "data"
    cytat_required: true

  - id: "strona_A"
    prompt: "Kim jest pierwsza strona umowy (pelna nazwa firmy + NIP jezeli widoczny)?"
    typ: "tekst"
    cytat_required: true

  - id: "strona_B"
    prompt: "Kim jest druga strona umowy (pelna nazwa firmy + NIP jezeli widoczny)?"
    typ: "tekst"
    cytat_required: true

  - id: "czas_trwania_NDA"
    prompt: "Ile lat trwa zobowiazanie do zachowania poufnosci? Liczbowo (np. 5)."
    typ: "liczba_calkowita"
    cytat_required: true
    red_flag_jezeli: "> 5"

  - id: "wylaczenia"
    prompt: "Jakie sa wylaczenia z obowiazku poufnosci (publicznie znane / od osoby trzeciej / nakaz sadowy / inne)? Lista."
    typ: "tekst"
    cytat_required: true
    red_flag_jezeli: "brak"

  - id: "prawo_wlasciwe"
    prompt: "Jakie prawo jest wlasciwe dla tej umowy? (PL / EN / inne)"
    typ: "tekst"
    cytat_required: true

  - id: "sad_wlasciwy"
    prompt: "Ktory sad jest wlasciwy dla sporow z tej umowy?"
    typ: "tekst"
    cytat_required: true
```

Default schemat NDA dostarczony w `examples/portfel-nda-przyklad/schemat.yaml`.

## Pseudonimizacja PII

**Algorytm** (deterministyczny, regex + checksum):

1. **PESEL** - regex `\d{11}` + walidacja checksum algorytm wag 1-3-7-9-1-3-7-9-1-3 modulo 10
2. **NIP** - regex `\d{10}` lub `\d{3}-\d{3}-\d{2}-\d{2}` + walidacja checksum NIP (wagi 6,5,7,2,3,4,5,6,7 modulo 11)
3. **REGON** - regex `\d{9}` lub `\d{14}` + walidacja checksum REGON (wagi)
4. **Numer rachunku** - regex `\d{26}` lub `PL\d{26}` + walidacja IBAN checksum
5. **Email** - regex standardowy email
6. **Adres** - heurystyka: linia z `ul. / al. / pl.` + numer + `\d{2}-\d{3}` (kod pocztowy) + nazwa miasta
7. **Imie+nazwisko** - gazetteer polskich imion + heurystyka kapitalizacji (PRZED nazwami firm, gazetteer KRS)
8. **Nazwa firmy** (opcjonalnie) - gazetteer KRS top-N firm + heurystyka "sp. z o.o. / S.A. / sp. j."

Placeholdery:
- `[OSOBA_1]`, `[OSOBA_2]` - imie nazwisko (per umowa unikalne)
- `[PESEL_1]` - PESEL
- `[NIP_1]` - NIP
- `[REGON_1]` - REGON
- `[RACHUNEK_1]` - numer rachunku bankowego
- `[EMAIL_1]` - email
- `[ADRES_1]` - adres
- `[FIRMA_1]` (opcjonalnie) - nazwa firmy

Mapowanie placeholder <-> wartosc trzymane **wylacznie lokalnie** (Python dict in-memory, gitignore jezeli persisted).

Reuse: ten sam algorytm co Patron `backend/src/lib/pl-entities/` (zachowanie konsystencji w ekosystemie MateMatic).

## Mechaniczna walidacja cytatu

Po kazdej odpowiedzi LLM:

```python
def cytat_obecny(cytat: str, tekst_zrodlowy: str) -> bool:
    """
    Sprawdza czy cytat istnieje w tekscie zrodlowym.
    Tolerancja: whitespace (\\s+), case-insensitive.
    """
    # normalizacja whitespace
    cytat_norm = re.sub(r'\s+', ' ', cytat.strip()).lower()
    tekst_norm = re.sub(r'\s+', ' ', tekst_zrodlowy).lower()
    return cytat_norm in tekst_norm
```

Jezeli `False` -> komorka `null`, `confidence: failed`, wpis do "Luki".

Cytat jest **niedepseudonimizowany** (z placeholderami) - sprawdza sie obecnosc w **zanonimizowanym tekscie**. To gwarantuje ze placeholdery sa konsystentne.

## Output `.docx`

Struktura (uzywa Python `python-docx` lub LLM-generated markdown -> pandoc):

```
[NAGLOWEK KANCELARII - logo + nazwa + adres]

# RAPORT TABULAR REVIEW UMOW
**Data**: YYYY-MM-DD
**Liczba umow**: NN
**LLM provider uzyty**: ollama (llama3.3:70b)
**Konfiguracja**: schemat.yaml v.X

## Tabela glowna

[tabela NN x M, kolorowanie RAG, czerwone flagi pogrubione]

## Czerwone flagi (NN)

- Umowa 3 (NDA z Strona A) - czas trwania 7 lat (red_flag_jezeli > 5)
- Umowa 7 (NDA z Strona B) - brak wylaczen (red_flag_jezeli "brak")
- ...

## Luki (komorki failed) (NN)

- Umowa 5, kolumna "prawo_wlasciwe" - LLM zwrocil "Polska" ale cytat
  "Niniejsza umowa podlega prawu polskiemu" NIE zostal znaleziony w
  tekscie. Mozliwe: model halucynowal, lub fraza jest w innym brzmieniu.
  ACTION: prawnik manualnie sprawdz strone N.
- ...

## Cytaty zrodlowe (per komorka)

### Umowa 1
- data_podpisania: "z dnia 15 stycznia 2026 r." (str. 1)
- strona_A: "Strona A: ABC sp. z o.o. ..." (str. 1)
- ...

## Metadata raportu

- Skill: matematic-contract-review-pl v0.1.0-alpha
- Hash konfiguracji: sha256(schemat.yaml) = ...
- Liczba wywolan LLM: 72 (12 umow x 6 kolumn)
- Sredni czas / komorke: 1.2s
- Liczba walidacji cytatow: 72 (passed: 68, failed: 4 = 5.5%)
```

## Provider config

```
~/.config/contract-review-pl/providers.yaml      # gitignore
```

Format:

```yaml
ollama:
  base_url: "http://localhost:11434"
  default_model: "llama3.3:70b"

claude:
  api_key: "sk-ant-..."
  default_model: "claude-sonnet-4-6-20250929"

gemini:
  api_key: "..."
  default_model: "gemini-2.5-pro"

gpt:
  api_key: "..."
  default_model: "gpt-4o"
```

Klucze providerow **NIGDY** w kodzie skilla. **NIGDY** w outputach. **NIGDY** w commit.

## Wymagania techniczne

- **Claude Code** zainstalowany
- **Python 3.10+** dla pseudonimizacji + walidacji cytatu (skill uruchamia subprocess)
- **Preprocessor PDF** - co najmniej jeden z: `markitdown`, `opendataloader-pdf`, `docling`
- **Pandoc** (opcjonalnie) dla generacji `.docx` z markdown
- **Ollama** (opcjonalnie) dla provider RODO-safe lokalnego

## Granice MVP v0.1.0-alpha

**JEST**:
- 1 skill: contract-review-pl
- Default schemat NDA
- Pseudonimizacja PII (regex + checksum)
- Mechaniczna walidacja cytatu
- Output `.docx` z tabela + czerwone flagi + luki + cytaty
- Provider config (4 providerow)

**BRAK** (planowane v0.2.0+):
- Web UI (skill jest CLI-only przez Claude Code)
- Persistence projektu (kazde uruchomienie skilla = nowy raport, brak historii)
- Multi-skill composability z lpm-pl
- Default schematy dla M&A / umowy dostawcze / umowy powierzenia art. 28 RODO
- Eksport `.csv` raw dataset (tylko `.docx` w v0.1.0)
- Audit bundle AI Act art. 12 (planowane v0.3.0)
- Walidacja klauzul wzorcowych (np. "czy klauzula limitacji odpowiedzialnosci jest zgodna z art. 473 KC?")

**NIGDY** (poza scope - to robi Patron):
- Hostowany web app
- Multi-tenant z RLS
- Postgres persistence
- Real-time chat over dataset

## Operational patterns v0.1.1 (cherry-pick z gregmos/PII-Shield MIT)

3 patternu architektoniczne dodane w v0.1.1 (2026-05-21) - cherry-pick z
[gregmos/PII-Shield](https://github.com/gregmos/PII-Shield) (MIT, autor
Grigorii Moskalev - Microsoft Presidio team, snapshot v2.0.2). Patterny
operacyjne, NIE zmieniaja 4 zasad konstytucyjnych:

### Pattern 1: `pseudonim_audit.log` "proves no PII leaves"

Osobny plain-text log file `~/.config/contract-review-pl/pseudonim_audit.log`
**czytelny dla Inspektora ochrony danych** (bez wymogu odszyfrowania
hash-chain). Per linia:

```
2026-05-21T18:42:13Z | pseudonim-applied | doc_id=01HXY... | source_hash=sha256:abc123... | entities={OSOBA:3,PESEL:1,NIP:2,ADRES:1} | bytes_in=12450 | bytes_out=12180
2026-05-21T18:42:14Z | llm-call-out | provider=ollama | model=llama3.3:70b | prompt_chars=8200 | placeholders=12 | PII_count=0
2026-05-21T18:42:18Z | llm-call-in | provider=ollama | response_chars=240 | cytat_present=true
2026-05-21T18:42:18Z | mapping-stored | doc_id=01HXY... | expires_at=2026-05-28T18:42:18Z | bytes=2340
2026-05-21T18:45:01Z | docx-generated | output=/folder/RAPORT-...docx | session_id=01HXY... | depseudonim_count=12
2026-05-21T18:45:01Z | mapping-cleanup | TTL=7days | removed_sessions=0
```

Linia `llm-call-out` z `PII_count=0` jest **dowodem** ze pseudonimizacja zadziala (bo `PII_count` zlicza wszystkie znalezione PESEL/NIP/imiona w **promptcie wyslanym do LLM** - jezeli > 0, skill **ZATRZYMUJE** wywolanie i raportuje bug).

Linie sa **rownolegle do** istniejacego workflow, nie zastepuja walidacji pseudonimizacji (Zasada 1 nadal bezwzgledna).

**Implementacja**: `skills/contract-review-pl/helpers/audit-logger.py` (nowy plik v0.1.1).

### Pattern 2: `session_id` w docx custom properties

Kazdy `.docx` generowany przez skill ma `session_id` (UUID) embed w **custom properties** Word - tygodnie pozniej skill rozpoznaje sesje i moze:

- Pokazac uzytkownikowi historie generacji (data, provider, hash konfiguracji)
- Powtorzyc raport z innym providerem (jezeli mapping wciaz w TTL)
- Re-deanonymize fragmenty (jezeli prawnik chce wrocic do PII po zlozeniu raportu)

Workflow:

1. Skill generuje `session_id = ULID()`
2. Mapping zapisany do `~/.config/contract-review-pl/sessions/{session_id}.json` (gitignore, AES-256-GCM szyfrowanie scrypt-derived key z hasla uzytkownika)
3. `.docx` ma custom properties: `MateMaticContractReviewSessionId = {session_id}`, `MateMaticContractReviewToolVersion = v0.1.1`, `MateMaticContractReviewTimestamp = ISO8601`
4. Pozniej: `contract-review-pl --reopen-session ~/Desktop/RAPORT-....docx` czyta custom properties, ladowuje sesje, oferuje akcje

**Implementacja**: `skills/contract-review-pl/helpers/docx-session-tagging.py` + rozszerzenie `helpers/generuj_docx.py` (nowe v0.1.1).

### Pattern 3: TTL mapping cleanup (default 7 dni, configurable)

Mapping placeholder ↔ wartosc zywa **wygasa** po N dniach (default 7, configurable w `~/.config/contract-review-pl/policy.yaml`):

```yaml
# ~/.config/contract-review-pl/policy.yaml
pseudonim_mapping_ttl_days: 7  # default; mozesz ustawic 180 dla long-running M&A
auto_cleanup_on_invocation: true  # przy kazdym uruchomieniu skilla, sprawdz expired sessions
```

Skill przy **kazdym uruchomieniu** sprawdza `~/.config/contract-review-pl/sessions/`, usuwa pliki sesji ktore `expires_at < now`. Wpis do `pseudonim_audit.log`: `mapping-cleanup | TTL=7days | removed_sessions=3 | reclaimed_bytes=12480`.

**Wartosc**: RODO art. 5 ust. 1 lit. e (ograniczenie przechowywania) - mapping placeholder ↔ wartosc zywa to PII derivative, retencja minimalna.

**Implementacja**: `skills/contract-review-pl/helpers/mapping-cleanup.py` (nowy plik v0.1.1).

## Atrybucja patternow 1-3

Patterny operacyjne 1-3 sa cherry-pick z [gregmos/PII-Shield](https://github.com/gregmos/PII-Shield) (MIT, snapshot 2026-05-21, autor Grigorii Moskalev). NIE forkujemy kodu - implementacja w naszym skillu napisana od zera pod Python helpers + polskie nazewnictwo + integracja z 4 zasadami konstytucyjnymi v1.0.0.
