---
title: Mapowanie kolumn i customizacja per tabela
tags: [fabric-accelerator, analysis, column-mapping]
---

# Mapowanie kolumn i customizacja transformacji per tabela

> Ten dokument to zapis wniosków z analizy kodu przeprowadzonej w rozmowie z Copilotem (2026-07-13),
> odpowiadający na pytanie: *"skoro L1Transform-Generic-Fabric jest identyczny dla każdej tabeli,
> to jak framework obsługuje różne nazwy kolumn i różne wymagania per tabela?"*

Powiązane: [[04-ELT-Data-Model]] · [[06-Spark-Notebooks]] · [[08-L1-vs-L2-Architecture]]

## Pytanie wyjściowe

`L1Transform-Generic-Fabric.Notebook` wykonuje identyczny zestaw operacji dla każdej tabeli
(`trim`, `replaceNull`, `deDuplicate`, `upsert/insert`). Skoro każda tabela źródłowa ma inne nazwy
kolumn — gdzie następuje mapowanie/zmiana nazw?

## Mechanizm 1: podmiana całego notebooka (`ComputeName`)

Jeśli tabela wymaga **innej logiki transformacji** (nie tylko zmiany nazw), `L1TransformDefinition.ComputeName`
i `ComputePath` wskazują inny notebook niż domyślny generyczny. Pipeline `Level1 Transform.DataPipeline`
uruchamia notebook dynamicznie:
```
"notebookId": { "value": "@pipeline().parameters.ComputeName", "type": "Expression" }
```
→ dla większości tabel `ComputeName` = `L1Transform-Generic-Fabric`; dla wyjątków (SCD2, flatten JSON,
tłumaczenie kolumn, konwersje Julian date) tworzy się osobny notebook.

## Mechanizm 2: mapowanie/zmiana nazw kolumn — na etapie INGESTU, nie L1

Zmiana nazw kolumn ma **osobny kanał metadanowy**, realizowany na etapie ingestu (bronze), zanim dane
w ogóle trafią do notebooka L1:

1. **Tabela `ELT.ColumnMapping`** — per tabela/strumień wiersze `SourceName → TargetName` (+ `TargetOrdinalPosition`).
2. **Funkcje SQL** (`ELT.Functions`), generujące gotowe artefakty z tej tabeli:
   - `uf_GetColumnMapping` — buduje JSON w formacie ADF `TabularTranslator`
     (`{"source":{"name":"..."},"sink":{"name":"..."}}`).
   - `uf_GetIngestionColumnList` — buduje listę `SourceName as TargetName` do wstrzyknięcia
     w zapytanie SQL pobierające dane ze źródła.
   - `uf_GetTabularTranslatorMappingJson` — opakowuje mapping w pełny JSON translatora ADF.
3. **Kolumna `ELT.IngestDefinition.DataMapping`** (JSON) przechowuje wynik tych funkcji per tabela —
   **jest realnie parametrem pipeline'u ingestu** (`Ingest ASQL/MirrorDB/OneLake Lakehouse Table`),
   podłączonym do `translator` aktywności Copy.

## ⚠️ Ważne zastrzeżenie (zweryfikowane w kodzie)

Sprawdzenie `grep` po całym repo pokazuje, że **`uf_GetColumnMapping`, `uf_GetIngestionColumnList`,
`uf_GetTabularTranslatorMappingJson` nigdzie nie są automatycznie wywoływane** — ani przez pipeline,
ani przez stored procedurę. Potwierdza to również oficjalne wiki `elt-framework`
([05-Helper-Procs](https://github.com/bennyaustin/elt-framework/wiki/05-Helper-Procs)), które **nie
wymienia** tych trzech funkcji wśród udokumentowanych, "wspieranych" procedur pomocniczych.

**Wniosek praktyczny:** to są funkcje-narzędzia do **ręcznego** wygenerowania JSON-a mappingu
(np. w SSMS / Query Editor) przy onboardingu nowego źródła. Trzeba:
1. Wypełnić `ColumnMapping` dla danej tabeli.
2. Ręcznie wywołać `uf_GetColumnMapping` / `uf_GetTabularTranslatorMappingJson` i zapisać wynik
   w `IngestDefinition.DataMapping`.

Nie ma zautomatyzowanego kroku "wypełnij `ColumnMapping` → system sam zbuduje `DataMapping`".
Dodatkowo w przykładowym pipeline `Ingest ASQL Table` aktywność Copy ma w `translator` tylko
`typeConversion`, bez wpiętego `mappings` z parametru `DataMapping` — czyli samo podłączenie
mappingu do właściwości `translator.mappings` też bywa czymś, co trzeba dorobić samodzielnie
w niektórych reużywalnych pipeline'ach.

## Podsumowanie koncepcyjne

| Rodzaj zmiany | Mechanizm | Poziom |
|---|---|---|
| Zmiana nazw kolumn | `ColumnMapping` + `IngestDefinition.DataMapping` (JSON translator ADF) | Ingest (bronze) |
| Inna logika transformacji (SCD2, flatten, itd.) | Osobny notebook + `L1TransformDefinition.ComputeName` | L1 (bronze→silver) |
| Parametryzacja bez nowego notebooka | `L1TransformDefinition.CustomParameters` (JSON) — dziś nieużywany w generycznym notebooku, ale dostępny jako hak rozszerzeń | L1 |

Dlatego generyczny notebook L1 może wyglądać identycznie dla każdej tabeli: zakłada, że nazwy kolumn
są już ujednolicone na etapie ingestu, i operuje na całym DataFrame w sposób niezależny od konkretnych
nazw kolumn.
