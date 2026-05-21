# Drabinka decyzyjna preprocesora PDF

Skill `contract-review-pl` uzywa **jednego z trzech** preprocesorow PDF -> tekst markdown. Wybor wedlug typu umowy:

## 1. markitdown (default dla prostych umow tekstowych)

**Kiedy**: NDA standardowe, listy intencyjne, oferty handlowe, korespondencja umowna - dokumenty z czytelnym one-column flow tekstu.

**Plusy**: szybki, prosty, MIT, Microsoft maintained, dobre dla 80% prostych umow.

**Minusy**: gorzej radzi sobie z tabelami klauzul i multi-column layouts (czesto rozsypuje reading order).

**Install**: `pip install markitdown`

## 2. opendataloader-pdf (dla umow z tabelami klauzul)

**Kiedy**: umowy ramowe z **zalacznikami tabelarycznymi** (cennik, harmonogram, SLA matrix), umowy dostawcze z lista produktow/uslug, umowy IT outsourcing z RACI matrix.

**Plusy**: reading order + struktura tabel - krytyczne dla audytow LegalTech. Lepsza dokladnosc na klauzulach numerowanych (zachowuje hierarchie 1.1.1).

**Minusy**: wolniejszy, wymaga wiecej zasobow CPU.

**Install**: [opendataloader-pdf GitHub](https://github.com/opendataloader-project/opendataloader-pdf) (sprawdz aktualne instrukcje).

## 3. docling (dla umow z grafika / pieczeciami / zlozonym layoutem)

**Kiedy**: skanowane umowy z pieczeciami i podpisami, umowy z embedded charts/diagramami (np. mapy granic dla nieruchomosci), umowy multimedialne (M&A SPA z zalacznikami graficznymi).

**Plusy**: najlepsza jakosc reading order na zlozonych dokumentach, IBM open source MIT, akceleracja GPU (CUDA / MPS Apple).

**Minusy**: najwolniejszy, wymaga ~2 GB modeli (download przy pierwszym uruchomieniu), Python heavy dependency.

**Install**: `pip install docling`

Cherry-pick z [jamietso/Tabular_Review](https://github.com/jamietso/Tabular_Review): tamten skill uzywa **wylacznie Docling** (89-liniowy FastAPI wrapper). W naszym skillu Docling jest **opcjonalna 3. opcja** - nie chcemy wymuszac ciezkiej zaleznosci na uzytkownikach ze stara umowa NDA do audytu.

## 4. Skany papierowe bez warstwy tekstowej (poza zakresem v0.1.0)

**Kiedy**: kserowki, faxy, papier sadowy bez OCR.

**Planowane v0.2.0+**: integracja z [Chandra OCR](https://github.com/AAU-CSE/chandra-ocr) (lokalna inference, polski 85.3%, RODO-safe).

**Workaround MVP**: prawnik **manualnie OCR-uje** skan (Adobe Acrobat / inny narzedzie z OCR), zapisuje jako PDF z warstwa tekstowa, podaje do contract-review-pl jako zwykla umowe.

## Drabinka decyzyjna w skrocie

```
PDF z czystym tekstem (born-digital) -> markitdown (default, najszybszy)
PDF z tabelami klauzul / numeracja 1.1.1 -> opendataloader-pdf
PDF z grafika / pieczeciami / zlozonym layoutem -> docling
Skan bez OCR -> manual OCR -> markitdown (lub czekaj na v0.2.0)
```

## Konfiguracja w `schemat.yaml`

Skill **auto-wykrywa** dostepne preprocesory (PATH check) i wybiera pierwszy wedlug drabinki. Jezeli chcesz **wymusic** konkretny:

```yaml
# w schemat.yaml
preprocessor: "docling"  # opcjonalne, domyslnie auto-wybor
```

## Powiazane

- [MEMORY operacyjna MateMatic - drabinka PDF eskalacji](w `~/.claude/CLAUDE.md` sekcja "Reading PDFs")
- [matematic-readiness audyt RODO / preprocessing strategia](https://github.com/matematicsolutions/matematic-readiness)
- [SPEC.md - sekcja preprocessing](../SPEC.md)
