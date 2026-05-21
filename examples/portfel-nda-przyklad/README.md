# Portfel NDA - przyklad testowy

3 zanonimizowane NDA + `schemat.yaml` + oczekiwany output. Uzywaj do **walidacji skilla** przed produkcyjnym uruchomieniem na zywych aktach kancelarii.

## Zawartosc (planowane do v0.2.0, MVP placeholder)

```
schemat.yaml                       - default schemat NDA, gotowy do uzycia
umowa-1-zanonimizowana.pdf         - syntetyczna NDA #1 (czas 3 lata, prawo PL, sad warszawski) - oczekiwany RAG: ZIELONY
umowa-2-zanonimizowana.pdf         - syntetyczna NDA #2 (czas 7 lat, brak wylaczen) - oczekiwany RAG: CZERWONY (2 flagi)
umowa-3-zanonimizowana.pdf         - syntetyczna NDA #3 (klauzula bezterminowa, prawo USA) - oczekiwany RAG: CZERWONY (2 flagi)
expected/
  RAPORT-contract-review-EXPECTED.docx  - referencyjny output (oczekiwany ksztalt)
  RAPORT-pseudonim-validation.txt        - assert ze prompty do LLM nie zawieraja PII
```

## Status MVP v0.1.0-alpha

**Pliki PDF + expected output** sa **planowane do v0.2.0**. W MVP v0.1.0 mamy tylko `schemat.yaml` jako referencje formatu konfiguracji.

Pierwsze uruchomienie skilla po instalacji wymaga:
1. Twojego folderu z testowymi NDA (1-3 syntetyczne, zanonimizowane)
2. Dostosowanego `schemat.yaml` z naglowkiem Twojej kancelarii
3. Skonfigurowanego providera LLM w `~/.config/contract-review-pl/providers.yaml`

## Jak zbudowac wlasny portfel testowy

1. **Wygeneruj** 3-5 syntetycznych NDA (mozesz uzyc generatora online lub Claude z promptem "wygeneruj NDA NN-stronowa miedzy fikcyjnymi spolkami X i Y, z czasem trwania N lat, prawem PL, sadem warszawskim"). **Nie uzywaj realnych umow** - syntetyczne sa lepsze, bo testowalne (znasz oczekiwany output).
2. **Skopiuj** `schemat.yaml` z tego folderu, dostosuj `naglowek_kancelarii` i `llm_provider`.
3. **Uruchom skill**: `contract-review-pl --folder ./testowy-portfel/ --schemat ./schemat.yaml`
4. **Porownaj** output z oczekiwaniem (kazda Twoja syntetyczna NDA ma znany "oczekiwany RAG" i "oczekiwane czerwone flagi" - sprawdz).

## Walidacja krytyczna pseudonimizacji

**Przed produkcyjnym uzyciem** na zywych aktach, **uruchom test pseudonimizacji**:

```bash
contract-review-pl --folder ./testowy-portfel/ --schemat ./schemat.yaml --dry-run --mock-llm
```

Flagi:
- `--dry-run` = nie wywoluj LLM, zapisz prompty do pliku
- `--mock-llm` = uzywaj mock providera (deterministyczne odpowiedzi)

Sprawdz **kazdy zapisany prompt** w `/tmp/contract-review-pl/{umowa_id}/prompts/`:
- **NIE moze zawierac** PESEL, imienia, nazwiska, NIP, adresu, emaila z umow wejsciowych
- **MUSI zawierac** placeholdery `[OSOBA_1]`, `[PESEL_1]`, `[ADRES_1]`, `[NIP_1]`, `[EMAIL_1]`

Jezeli walidacja nie przejdzie - **NIE uzywaj skilla produkcyjnie**. Zglos jako bug w [Issues](https://github.com/matematicsolutions/matematic-contract-review-pl/issues).
