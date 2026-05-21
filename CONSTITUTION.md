# CONSTITUTION.md - matematic-contract-review-pl

**Wersja**: v1.0.0
**Data**: 2026-05-21
**Status**: Obowiazujaca

Cztery zasady konstytucyjne pod ktorymi piszemy skill `contract-review-pl`. Modyfikacja tych zasad wymaga bump wersji v1.x.0 i akceptacji w PR.

## Zasada 1: RODO-safe by default

Skill **NIE wysyla niezanonimizowanych danych osobowych do LLM**. Pseudonimizacja PII PRZED kazdym wywolaniem LLM jest **bezwzgledna**.

Co znaczy "PRZED kazdym wywolaniem":
- Imie / nazwisko / PESEL / NIP / adres / numer rachunku / email osoby fizycznej zamieniane na placeholdery `[OSOBA_1]`, `[PESEL_1]`, `[ADRES_1]` itp.
- Mapowanie placeholder <-> wartosc zywa **NIGDY** nie idzie do LLM. Trzymane w pamieci skill, uzywane tylko do depseudonimizacji outputu lokalnie.
- Nazwy firm (KRS) **nie sa PII** ale moga byc poufne (klient kancelarii). Skill **opcjonalnie** pseudonimizuje nazwy firm (`[FIRMA_1]`) - decyzja uzytkownika przez flage `--anonymize-companies`.

Konsekwencja:
- Skill **dziala wolniej** (dodatkowa warstwa preprocessingu) - akceptowalny koszt.
- Skill **wymaga uzycia LLM ktory rozumie pseudonimizowany tekst** - wszystkie nowoczesne LLM to potrafia, nie jest problem.

## Zasada 2: Multi-provider LLM (vendor-neutrality)

Skill **rozmawia z dowolnym LLM** wedlug konfiguracji uzytkownika:

- **Ollama lokalny** (default, RODO-safe maximum) - llama 3.3 / qwen 2.5 / mistral / inne
- **Claude** (Anthropic) - sonnet / opus
- **Gemini** (Google) - flash / pro
- **GPT** (OpenAI) - 4o / o1

Wybor LLM = **decyzja kancelarii**, nie skilla. Skill nie faworyzuje zadnego providera w defaultach (oprocz Ollama dla RODO-safe).

Cherry-pick lekcja z [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review): tamten skill **hardcoduje** Gemini i bundluje API key do frontendu (anti-pattern security). My **nie powtarzamy** - klucze providerow w `~/.config/contract-review-pl/providers.yaml` (gitignore), nigdy w kodzie skilla, nigdy w outputach.

## Zasada 3: Cytat fizycznie obecny w tekscie zrodlowym

Kazda komorka tabeli **musi miec cytat** (fragment tekstu umowy zrodlowej) **mechanicznie zwerifikowany** (substring match z tolerancja whitespace).

Workflow:
1. LLM zwraca: wartosc komorki + cytat (np. `wartosc: "5 lat"`, `cytat: "Czas trwania zobowiazania poufnosci wynosi 5 (pieciu) lat"`)
2. Skill **mechanicznie szuka cytatu** w tekscie umowy zrodlowej (case-insensitive, whitespace-tolerant)
3. **Jezeli cytat istnieje**: komorka `confidence: high`, cytat zapisany w sekcji "Cytaty zrodlowe" outputu
4. **Jezeli cytat NIE istnieje** (halucynacja LLM): komorka `null` + `confidence: failed`, wpis do sekcji "Luki" outputu

Konsekwencja:
- **Halucynacja niemozliwa na warstwie struktury** - model moze zle zinterpretowac fragment, ale nie wymysli klauzuli ktorej fizycznie nie ma.
- Skill **swiadomie zwraca puste komorki** zamiast falszywych - lepiej "luka" niz "false confident".
- Prawnik dostaje **zweryfikowany dataset**, nie "AI-summarized" probability.

## Zasada 4: Bez nazywania firm w outputach summary

Skill **nie nazywa firm w sekcji summary i czerwonych flagach**. Mowi "Strona A", "Strona B", "Dostawca", "Klient", "Wykonawca", "Zamawiajacy".

Konsekwencja:
- Output **mozna przekazac wewnatrz kancelarii** (junior do partnera) bez wycieku informacji ktora umowa od ktorego klienta.
- Pelne nazwy stron sa w **tabeli glownej** (kolumna "Strona A" / "Strona B") gdzie sa wprost wyciagniete z umow - bo to dokumenty, ktore prawnik czyta i tak.
- Cytaty zrodlowe (sekcja na koncu raportu) sa **pelne** (z nazwami) - to surowe dane do weryfikacji, nie summary.

## Bramki commit (przed merge do main)

1. **Marko-pl 2x runda** na SKILL.md + README.md + CONSTITUTION.md (zarzuty -> poprawki -> "ok")
2. **Test na zanonimizowanym portfelu** w `examples/` - skill produkuje sensowny `.docx`
3. **Walidacja PSEUDONIMIZACJA** - test e2e ze niezanonimizowany PESEL / imie **nigdy** nie idzie do LLM (mock LLM + assert na zawartosci promptu)
4. **Bramka jakosci output** - tabela ma kolory RAG, sekcja "Luki" istnieje, sekcja "Cytaty zrodlowe" istnieje, naglowek kancelarii konfigurowalny

## Ewolucja

Konstytucja jest **wersjonowana SEMVER**:
- **MAJOR** (v1.0.0 -> v2.0.0): zmiana podstawowa (np. rezygnacja z multi-provider).
- **MINOR** (v1.0.0 -> v1.1.0): dodanie nowej zasady (np. zasada 5).
- **PATCH** (v1.0.0 -> v1.0.1): doprecyzowanie istniejacej zasady bez zmiany merytorycznej.

Kazda zmiana wymaga PR + akceptacji + bump CHANGELOG.md.

## Powiazane

- [README.md](README.md) - opis dla ludzi
- [SPEC.md](SPEC.md) - specyfikacja techniczna v0.1.0-alpha
- [skills/contract-review-pl/SKILL.md](skills/contract-review-pl/SKILL.md) - implementacja zasad
- [Patron Konstytucja](https://github.com/matematicsolutions/patron/blob/main/governance/CONSTITUTION.md) - vendor-neutrality Art. 4 (zrodlo Zasady 2)
- [AGENTS.md](AGENTS.md) - instrukcje dla agentow AI pracujacych z tym repo
