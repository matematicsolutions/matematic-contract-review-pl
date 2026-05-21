---
name: contract-review-pl
description: Tabular review umow dla polskiej kancelarii - bulk audit PDF/DOCX z folderu, ekstrakcja kluczowych pol (data, strony, klauzule, prawo wlasciwe) do tabeli .docx z czerwonymi flagami i cytatami zrodlowymi. RODO-safe (pseudonimizacja PII PRZED LLM), multi-provider (Ollama default / Claude / Gemini / GPT), mechaniczna walidacja cytatu. Uzywaj gdy uzytkownik mowi "audyt umow", "tabular review", "bulk audit NDA", "przejrzyj portfel kontraktow", "wyciagnij parametry z 12 umow", "due diligence kontraktow", "audyt umow powierzenia RODO".
---

# contract-review-pl

Skill do **bulk audit umow** w polskiej kancelarii. Pracuje na **folderze umow** (PDF/DOCX), zwraca **`.docx` z tabela** + czerwone flagi + cytaty zrodlowe.

## Kiedy uzyc

Triggeruj gdy uzytkownik mowi:
- "audyt umow", "tabular review", "bulk audit"
- "przejrzyj portfel [NDA / kontraktow / umow dostawczych]"
- "wyciagnij parametry z [N] umow"
- "due diligence kontraktow"
- "audyt umow powierzenia [art. 28 RODO]"
- "weryfikacja portfela kontraktow dostawczych"
- "ranking ryzyka umow"

Nie triggeruj gdy:
- Uzytkownik pyta o **jedna konkretna umowe** (research / drafting / analiza) -> uzyj zwyklego flow Claude Code z `Read` tool albo Patron (jezeli dostepny).
- Uzytkownik pyta o **portfolio spraw** (nie umow) -> uzyj [`lpm-pl`](https://github.com/matematicsolutions/lpm-pl) status-report-drafter-pl / risk-and-issues-manager-pl.

## Pre-flight check (PRZED uruchomieniem)

1. **Folder umow podany?** Jezeli nie - zapytaj uzytkownika o sciezke (`~/Desktop/umowy-NDA/`, `~/projekty/Klient_X/due_diligence/`).
2. **Schemat kolumn podany?** Jezeli uzytkownik powiedzial tylko "audyt NDA" - uzyj **default schemat NDA** z `examples/portfel-nda-przyklad/schemat.yaml`. Jezeli scenariusz inny (M&A / umowy dostawcze / RODO art. 28) - poproc uzytkownika o `schemat.yaml` lub zaproponuj na podstawie kontekstu.
3. **Provider LLM skonfigurowany?** Sprawdz `~/.config/contract-review-pl/providers.yaml`. Jezeli pusty - poproc uzytkownika o wybor (Ollama lokalny RODO-safe / Claude / Gemini / GPT). Default **Ollama**.
4. **Pseudonimizacja wlaczona?** Zasada konstytucyjna 1 - **bezwzgledna**. Sprawdz czy `pseudonimizacja:` w `schemat.yaml` jest wlaczone dla `imiona_nazwiska`, `pesel`, `nip`, `adresy`, `emaile`. Jezeli nie - **NIE uruchamiaj** bez potwierdzenia uzytkownika (sygnalizuj ryzyko RODO).
5. **Preprocessor PDF dostepny?** Sprawdz w kolejnosci: `markitdown`, `opendataloader-pdf`, `docling`. Pierwszy znaleziony = uzyj. Jezeli zaden - poinformuj uzytkownika o instalacji.

## Workflow

### Faza 1: Preprocessing (per umowa)

Dla kazdego pliku w folderze umow:

1. **PDF/DOCX -> markdown** przez wybrany preprocessor:
   - **markitdown** dla prostych umow tekstowych
   - **opendataloader-pdf** dla umow ze zlozonymi tabelami klauzul (lepszy reading order)
   - **docling** dla skomplikowanych umow z grafika / pieczeciami
2. **Zapis** do `/tmp/contract-review-pl/{umowa_id}/tekst.md`
3. **Walidacja** - jezeli plik wynikowy < 100 znakow, sygnalizuj uzytkownikowi (uszkodzony PDF, scan bez OCR, problem z preprocessorem)

### Faza 2: Pseudonimizacja PII (per umowa) - BEZWZGLEDNA

1. **Regex + checksum** wykrywa: PESEL, NIP, REGON, IBAN, email, adres, imie+nazwisko (gazetteer polskich imion)
2. **Zamiana** na placeholdery: `[OSOBA_1]`, `[PESEL_1]`, `[NIP_1]`, `[ADRES_1]`, `[EMAIL_1]`, `[IBAN_1]`, `[REGON_1]`
3. **Mapowanie** placeholder <-> wartosc zywa zapis do `/tmp/contract-review-pl/{umowa_id}/mapping.json` (gitignore, usuwane po zakonczeniu sesji)
4. **Opcjonalnie nazwy firm** (jezeli flaga `--anonymize-companies`): `[FIRMA_1]`
5. **Walidacja** - przejrzyj `tekst_pseudonimowany.md` zeby upewnic ze nie ma residual PII (assert na regex po pseudonimizacji)

**Helper Python**: `helpers/pseudonimizuj.py` (uzywa tych samych regexow co Patron `pl-entities/`)

### Faza 3: Extraction (per umowa, per kolumna)

Dla kazdej kolumny w `schemat.yaml`:

1. **Prompt LLM** szablon:
   ```
   Jestes asystentem prawnym analizujacym umowe.
   Tekst umowy (zanonimizowany, placeholdery to dane osobowe):
   ---
   {tekst_pseudonimowany}
   ---
   Pytanie: {kolumna.prompt}
   Typ odpowiedzi: {kolumna.typ}

   Zwroc odpowiedz w formacie JSON:
   {
     "wartosc": "...",  // odpowiedz na pytanie
     "cytat": "...",    // DOKLADNY fragment tekstu (kopiuj-wklej) ktory uzasadnia odpowiedz
     "uwagi": "..."     // opcjonalne (np. "klauzula niejednoznaczna")
   }

   Jezeli nie ma odpowiedzi w tekscie - zwroc "wartosc": null + "cytat": null + "uwagi" opisujace dlaczego.

   Cytat MUSI byc fragmentem tekstu wprost - bez parafrazy, bez podsumowania.
   ```
2. **Wywolanie** providera LLM z `~/.config/contract-review-pl/providers.yaml`
3. **Parse JSON** odpowiedzi

### Faza 4: Mechaniczna walidacja cytatu (per komorka)

Dla kazdej odpowiedzi LLM:

1. **Sprawdz** czy `cytat` istnieje w `tekst_pseudonimowany.md` (substring match, case-insensitive, whitespace-tolerant)
2. **Jezeli TAK**: komorka `confidence: high`, cytat zapisany w sekcji "Cytaty zrodlowe"
3. **Jezeli NIE**: komorka `wartosc: null`, `confidence: failed`, wpis do sekcji "Luki" z uzasadnieniem "LLM zwrocil cytat '{cytat}' ktory nie istnieje w tekscie zrodlowym - mozliwa halucynacja"

**Helper Python**: `helpers/waliduj_cytat.py`

### Faza 5: Czerwone flagi (per komorka, per umowa)

Dla kazdej kolumny z `red_flag_jezeli` w schemacie:

1. **Sprawdz warunek** (np. `czas_trwania_NDA > 5`, `wylaczenia == "brak"`)
2. **Jezeli spelniony** -> czerwona flaga, wpis do sekcji "Czerwone flagi"
3. **Per umowa** - jezeli >= 1 czerwona flaga = status **Czerwony**, jezeli 0 ale >= 1 komorka `failed` = **Bursztynowy**, inaczej **Zielony**

### Faza 6: Depseudonimizacja outputu lokalnie

Przed generacja `.docx`:

1. **Zamien placeholdery** w wartosciach komorek i cytatach z powrotem na zywe dane (uzywajac `mapping.json` per umowa)
2. **Wyjatek - sekcja summary**: "Strona A", "Strona B", "Dostawca" zostaje (Zasada konstytucyjna 4 - bez nazywania firm w summary)

### Faza 7: Generacja `.docx`

Uzywa `helpers/generuj_docx.py` (python-docx) lub pandoc markdown -> docx z templatem:

1. **Naglowek kancelarii** (z `schemat.yaml` -> `naglowek_kancelarii`)
2. **Sekcja meta** (data, liczba umow, LLM provider, hash konfiguracji)
3. **Tabela glowna** (NN umow x M kolumn) z kolorowaniem RAG
4. **Czerwone flagi** (lista, per umowa, z odniesieniem do kolumny i wartosci)
5. **Luki** (komorki failed, z uzasadnieniem)
6. **Cytaty zrodlowe** (per komorka, z numerem strony PDF jezeli dostepne)
7. **Metadata raportu** (skill version, hash konfiguracji, liczba wywolan LLM, srednia / komorke, pass rate walidacji cytatow)

Output: `{folder_umow}/RAPORT-contract-review-{YYYY-MM-DD}.docx`

### Faza 8: Cleanup

1. **Usun** `/tmp/contract-review-pl/{umowa_id}/mapping.json` (PII!)
2. **Zachowaj** `/tmp/contract-review-pl/{umowa_id}/tekst_pseudonimowany.md` przez 24h dla debug (jezeli prawnik chce sprawdzic co dokladnie poszlo do LLM)
3. **Wpis audit log** lokalny w `~/.config/contract-review-pl/audit.log` (data, liczba umow, provider, brak PII)

## Judgment calls embedded

- **Wartosci "do uzgodnienia" / "TBD"** w umowie = traktuj jako luka (komorka `null`, wpis do "Luki"), nie zwracaj "do uzgodnienia" jako wartosc
- **Klauzule warunkowe** ("jezeli klient zaplaci w terminie, to wylaczenie odpowiedzialnosci wynosi X, inaczej Y") = zwroc obie wartosci z uwaga "warunkowa"
- **Sprzeczne klauzule** w tej samej umowie (np. w preamble "5 lat", w paragrafie X "10 lat") = czerwona flaga "wewnetrzna sprzecznosc", zwroc obie wartosci z cytatami
- **Brak cytatu w przyzwoitym tekscie** (LLM zwrocil wartosc bez cytatu) = traktuj jako halucynacja, komorka `failed`
- **Cytat jest parafraza zamiast fragmentem** (LLM nie skopiowal wprost) = traktuj jako halucynacja, komorka `failed`

## RAG triage (per umowa)

- **Czerwony**: >= 1 czerwona flaga (np. czas NDA > 5 lat, brak wylaczen, prawo USA dla polskiej kancelarii)
- **Bursztynowy**: 0 czerwonych flag ale >= 1 komorka `failed` (luka wymagajaca recznej weryfikacji prawnika)
- **Zielony**: 0 czerwonych flag, 0 failed - umowa wyglada standardowo

## Output - co prawnik dostaje

`{folder_umow}/RAPORT-contract-review-{YYYY-MM-DD}.docx` - gotowy do **wyslania do partnera**, nie do przepisywania:

- Tabela 12x6 z kolorowaniem RAG (5 zielonych, 4 bursztynowe, 3 czerwone)
- 7 czerwonych flag (z odniesieniem "Umowa 3 - czas NDA = 7 lat")
- 4 luki (z propozycja "ACTION: prawnik manualnie sprawdz strone N")
- 72 cytaty zrodlowe (pelne, z numerem strony jezeli dostepne)
- Metadata raportu (audit-friendly)

## Bezpieczenstwo

- **Nigdy** nie wysylaj niezanonimizowanego tekstu do LLM (assert na zawartosci promptu PRZED wywolaniem)
- **Nigdy** nie commituj `mapping.json` (gitignore)
- **Nigdy** nie commituj `~/.config/contract-review-pl/providers.yaml` (gitignore, klucze providerow)
- **Nigdy** nie commituj prawdziwych umow klientow do `examples/` (tylko zanonimizowane)
- Output `.docx` zawiera **zywe dane** (depseudonimizowane) - prawnik **musi** traktowac plik jak akta sprawy (szyfrowany dysk, kontrola dostepu, retencja zgodnie z polityka kancelarii)

## Powiazane

- [matematic-contract-review-pl/CONSTITUTION.md](../../CONSTITUTION.md) - 4 zasady konstytucyjne
- [matematic-contract-review-pl/SPEC.md](../../SPEC.md) - specyfikacja techniczna
- [matematic-contract-review-pl/examples/portfel-nda-przyklad/](../../examples/portfel-nda-przyklad/) - pelny przyklad
- [Patron ADR-0010 Contract Review Module](https://github.com/matematicsolutions/patron/blob/main/governance/adr/0010-contract-review-module-tabular.md) - wersja produkcyjna (Patron module)
- [lpm-pl risk-and-issues-manager-pl](https://github.com/matematicsolutions/lpm-pl) - composability: czerwone flagi z contract-review mozesz przejac do RAID rejestru sprawy
