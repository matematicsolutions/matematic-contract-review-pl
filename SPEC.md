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
