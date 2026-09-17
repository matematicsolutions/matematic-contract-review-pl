# matematic-contract-review-pl - tabular review umow dla polskiej kancelarii

Otwarty skill **Claude Code** do **bulk audit umow** - kopiujesz folder PDF/DOCX, definiujesz interesujace pola (data podpisania, strona, kwota, klauzule limitacji odpowiedzialnosci, prawo wlasciwe, sad wlasciwy), skill zwraca **`.docx` z tabela** plus **lista luk i czerwonych flag** per umowa.

**Use case**: due diligence M&A (47 umow w 2h zamiast 2 tygodni juniora), audyt portfela kontraktow dostawczych, weryfikacja umow powierzenia danych (compliance art. 28 RODO), przeglad NDA przed podpisaniem przez kancelarie.

**To nie jest [Patron](https://github.com/matematicsolutions/patron).** Patron pracuje na pojedynczej sprawie (research / drafting / RAG po aktach). **contract-review-pl** pracuje na **macierzy umow** - kazdy wiersz to jedna umowa, kazda kolumna to pole do wyciagniecia. Dwa rozne narzedzia dla dwoch roznych scenariuszy.

**To nie jest [lpm-pl](https://github.com/matematicsolutions/lpm-pl).** lpm-pl pracuje nad portfelem **spraw** (status / scope / ryzyka). contract-review-pl pracuje nad portfelem **dokumentow** (parametry / cytaty / luki). Skille sa ortogonalne, ale **composable** - jezeli contract-review zwroci czerwone flagi w portfelu, lpm-pl risk-and-issues-manager moze je przejac jako nowe ryzyka sprawy.

## Dla kogo

- **Prawnik transakcyjny** (M&A) - bulk audit dokumentow due diligence
- **In-house counsel** (corporate) - portfel kontraktow dostawczych, ranking ryzyka
- **Inspektor ochrony danych** (kancelaria) - audyt umow powierzenia art. 28 RODO
- **Office Manager / wspolnik zarzadzajacy** - przeglad umow szkoleniowych / partnerskich / outsourcingu kancelarii

## Co jest w MVP (v0.1.0-alpha)

Jeden skill pilotazowy:

**contract-review-pl** - kopiujesz folder `umowy/` (PDF/DOCX), definiujesz `schemat.yaml` (lista kolumn z promptami natural-language), skill:

1. **Preprocessing PDF -> tekst** przez warstwe lokalna (markitdown / opendataloader-pdf / Docling - wedlug drabinki [decyzyjnej](./docs/preprocessing-decyzja.md))
2. **Pseudonimizacja PII PRZED LLM** - imiona / PESEL / NIP / adresy zamienione na placeholdery (`[OSOBA_1]`, `[FIRMA_2]`) zanim cokolwiek pojdzie do LLM
3. **Per umowa per kolumna extraction** - LLM (Claude / Ollama / Gemini wedlug konfiguracji uzytkownika, NIE hardcoded) wyciaga wartosc + **fizyczny cytat** z tekstu
4. **Mechaniczna walidacja cytatu** - skill sprawdza ze cytat **rzeczywiscie istnieje** w tekscie zrodlowym (substring match z tolerancja whitespace). Jezeli nie istnieje -> komorka `null` + `confidence: failed`
5. **Output `.docx`** z naglowkiem kancelarii: tabela glowna (wiersz=umowa, kolumna=pole), kolorowanie RAG (Czerwony/Bursztynowy/Zielony per umowa), sekcja "Luki" (umowy gdzie cos sie nie wyciagnelo), sekcja "Cytaty zrodlowe" (per komorka z linkiem do oryginalu)

Pelny przyklad input/output: [examples/portfel-nda-przyklad/](examples/portfel-nda-przyklad/).

## Filozofia (4 zasady, patrz [CONSTITUTION.md](CONSTITUTION.md))

1. **Pseudonimizacja domyslnie** - PII maskowane PRZED kazdym wywolaniem LLM. Skill nie wysyla niezanonimizowanych imion / PESEL / adresow do chmury.
2. **Multi-provider LLM** - skill rozmawia z Claude / Ollama / Gemini wymiennie (decyzja kancelarii, nie skilla). Default Ollama lokalny (zero transferu do US).
3. **Cytat fizycznie obecny w tekscie** - kazda komorka tabeli ma cytat ktory mechanicznie zwerifikowano w tekscie zrodlowym. Brak cytatu = brak komorki. Halucynacja niemozliwa na warstwie struktury (model moze zle zinterpretowac, ale nie wymysli klauzuli ktorej fizycznie nie ma).
4. **Bez nazywania firm w outputach** - skill mowi "Strona A", "Strona B", "Dostawca", "Klient" w summary; pelne nazwy tylko w tabeli (gdzie sa wprost wyciagniete z umowy).

## Instalacja (Claude Code)

```bash
cd ~/.claude/skills/
git clone https://github.com/matematicsolutions/matematic-contract-review-pl
ln -s matematic-contract-review-pl/skills/contract-review-pl contract-review-pl
```

Na Windows zamiast symlinka - kopia folderu `skills/contract-review-pl/` do `~/.claude/skills/`.

## Uzycie

```
W Claude Code:

Wieslaw: contract-review-pl - mam folder ~/Desktop/umowy-NDA/ z 12 NDA do
podpisania w przyszlym tygodniu. Wyciagnij: data, strona zobowiazana,
czas trwania zobowiazania poufnosci, wylaczenia (publicznie znane / od
osoby trzeciej / nakaz sadowy), prawo wlasciwe, sad wlasciwy. Zaznacz
NDA z czasem dluzszym niz 5 lat lub bez wylaczen.

Skill:
1. Wczytuje 12 plikow z ~/Desktop/umowy-NDA/
2. Preprocessing per umowa (markitdown / Docling)
3. Pseudonimizacja PII -> placeholdery
4. Extraction per komorka (6 kolumn x 12 umow = 72 wywolan LLM)
5. Walidacja cytatow mechaniczna
6. Output: ~/Desktop/umowy-NDA/RAPORT-contract-review-YYYY-MM-DD.docx
   - tabela 12x6 z kolorowaniem RAG
   - sekcja "Luki" (komorki failed)
   - sekcja "Czerwone flagi" (czas > 5 lat, brak wylaczen)
   - sekcja "Cytaty zrodlowe" (per komorka)
```

## Licencja

**Apache 2.0** - patrz [LICENSE](LICENSE). Mozesz wziac, modyfikowac, sprzedawac wdrozenie. Wymagamy zachowania atrybucji.

## Pochodzenie i atrybucja

Pattern UX (tabular review umow + dynamic schema columns + per-cell citation back-jump): inspiracja [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review) (MIT, autor Jamie Tso, snapshot 2026-05-21).

Tresc skilla (workflow, pseudonimizacja przed LLM, mechaniczna walidacja cytatu, multi-provider LLM, judgment calls embedded, RAG triage, output `.docx` z naglowkiem kancelarii, polski jezyk i polskie szablony klauzul): **napisana od zera** pod polskie realia kancelarii i wymogi RODO + tajemnica zawodowa (PoA art. 6, URP art. 3). NIE jest to fork ani tlumaczenie 1:1.

Brand i utrzymanie: [MateMatic Solutions](https://matematicsolutions.com).

## Status

**v0.1.0-alpha** - jeden skill MVP. Przed walidacja na zywym portfelu. Nie uzywaj produkcyjnie zanim nie sprawdzisz na **zanonimizowanym portfelu testowym** (przyklad w [examples/](examples/)).

## Powiazane

- [matematicsolutions/patron](https://github.com/matematicsolutions/patron) - jezeli potrzebujesz Contract Review jako **modul produkcyjny** w samym hostowanym Patronie (z Postgres + audit trail + UI), patrz [ADR-0010](https://github.com/matematicsolutions/patron/blob/main/governance/adr/0010-contract-review-module-tabular.md). Ten skill (contract-review-pl) jest **lekka alternatywa** w Claude Code, bez konfiguracji infrastruktury.
- [matematicsolutions/lpm-pl](https://github.com/matematicsolutions/lpm-pl) - composable: jezeli skill zwroci czerwone flagi, mozesz je przejac do risk-and-issues-manager-pl jako nowe ryzyka sprawy.
- [matematicsolutions/matematic-readiness](https://github.com/matematicsolutions/matematic-readiness) - audyt gotowosci kancelarii do AI (czy mozesz w ogole uzywac LLM do umow klienta? CONSTITUTION.md skilla mowi "decyzja kancelarii, nie skilla").
