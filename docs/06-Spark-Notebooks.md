---
title: Spark Notebooks (reużywalne)
tags: [fabric-accelerator, pyspark, notebooks]
---

# Reużywalne notebooki Spark

Powiązane: [[05-Data-Factory-Pipelines]] · [[07-Column-Mapping-and-Customization]] · [[08-L1-vs-L2-Architecture]] · [[09-Capacity-and-Cost-Considerations]]

Lokalizacja źródeł: `ipynb/*.Notebook/notebook-content.ipynb`,
docelowo w workspace: `workspace/notebook/reusable/...`

Filozofia: pisać reużywalny kod dla powtarzalnych wzorców danych (upserty, SCD Type 2,
zamiana pustych wartości na domyślne, konwersje stref czasowych itd.) — notebooki wywoływane
z pipeline'ów Data Factory z minimalną konfiguracją.

## CommonTransforms

`workspace/notebook/reusable/common-pyspark/CommonTransforms.Notebook`

Klasa Python (PySpark) rozszerzająca operacje czyszczenia/wzbogacania na poziomie **całego DataFrame**
(nie kolumna po kolumnie). Funkcje:

| Funkcja | Opis |
|---|---|
| `trim` | Usuwa spacje wiodące/końcowe ze wszystkich kolumn string |
| `replaceNull` | Zamienia `NULL` na wartość domyślną (numeryczna/string/data/timestamp/bool/dict per kolumna) |
| `deDuplicate` | Usuwa duplikaty, opcjonalnie wg podzbioru kolumn kluczowych |
| `utc_to_local` / `local_to_utc` / `changeTimezone` | Konwersje stref czasowych timestampów |
| `dropSysColumns` | Usuwa kolumny systemowe/niebiznesowe |
| `addChecksumCol` | Tworzy kolumnę checksum na bazie wszystkich kolumn |
| `julian_to_calendar` | Konwertuje datę Julian (5/7-cyfrową) na datę kalendarzową |
| `addLitCols` | Dodaje kolumny z wartościami literalnymi (np. kolumny audytowe) |

## DeltaLakeFunctions

`workspace/notebook/reusable/delta-lake/DeltaLakeFunctions.Notebook`

Funkcje do operacji na Lakehouse:

| Funkcja | Opis |
|---|---|
| `getAbfsPath` | Ścieżka ABFS warstwy medalionowej OneLake |
| `readFile` | Czyta plik z OneLake jako DataFrame |
| `readMedallionLHTable` | Czyta tabelę Lakehouse z warstw medalionowych (z filtrowaniem/wyborem kolumn) |
| `readLHTable` | Czyta dowolną tabelę Lakehouse spoza warstw medalionowych (np. mirrored DB) |
| `insertDelta` | Insert (append/overwrite); tworzy tabelę jeśli nie istnieje |
| `upsertDelta` | Insert/update (MERGE) na bazie `LookupColumns` i `WatermarkColName` |
| `getHighWaterMark` | Pobiera wysoki znak wodny z tabeli Lakehouse dla zakresu dat |
| `optimizeDelta` | Kompaktuje małe pliki, usuwa nieużywane wersje poza retencją |

## entra-functions

`workspace/notebook/reusable/entra/entra-functions.Notebook`

- `getBearerToken` — zwraca bearer token dla uwierzytelniania Service Principal Entra.

## Optimize Delta Lake Tables

`workspace/notebook/reusable/delta-lake/Optimize Delta Lake Tables.Notebook`. Iteruje przez wszystkie
tabele w Lakehouse i uruchamia `OPTIMIZE` + `VACUUM` (przy użyciu `optimizeDelta()` z `DeltaLakeFunctions`).
Zalecane uruchamianie co tydzień (zdrowie Lakehouse: małe pliki, stare wersje).

## L1Transform-Generic-Fabric

`workspace/notebook/reusable/level1-transform/L1Transform-Generic-Fabric.Notebook`

Przykładowa, **w pełni generyczna** transformacja Level 1:

1. Wywołuje `CommonTransforms`: `deDuplicate()`, `trim()`, `replaceNull()` (dla liczb/stringów/dat).
2. Wywołuje `DeltaLakeFunctions`: `upsertDelta()` (jeśli `OutputDWTableWriteMode='append'` i są
   `LookupColumns`) albo `insertDelta()` (overwrite).
3. Zwraca liczbę wierszy (insert/update/delete) do wołającego pipeline'u przez `notebookutils.notebook.exit()`.

Parametry wejściowe notebooka (komórka parametrów) odpowiadają 1:1 kolumnom `L1TransformDefinition` /
`L1TransformInstance`: `L1TransformInstanceID`, `L1TransformID`, `IngestID`, `CustomParameters`,
`InputRawFileSystem/Folder/File` lub `InputRawTable`, `OutputL1Curate...`, `OutputDWTable`,
`LookupColumns`, `WatermarkColName`, `DataFromTimestamp/ToTimestamp`.

> [!info] Dlaczego ten notebook jest identyczny dla każdej tabeli
> Operuje na całym DataFrame (nie po konkretnych nazwach kolumn) — `trim()`, `replaceNull()`,
> `deDuplicate()` działają uniwersalnie niezależnie od schematu. Zakłada, że nazwy kolumn są już
> ujednolicone na etapie ingestu. Zobacz [[07-Column-Mapping-and-Customization]] i [[08-L1-vs-L2-Architecture]].

## EnvSettings

`workspace/notebook/reusable/EnvSettings.Notebook`

Ustawia zmienne środowiskowe (prod/non-prod) używane przez inne notebooki — konfigurowane raz per
środowisko: `bronzeWorkspaceId`, `bronzeLakehouseName`, `bronzeDatawarehouseName`, analogicznie dla
silver i gold.

## Testy jednostkowe

- `Unit Test CommonTransforms.Notebook`
- `UnitTest_DeltaLakeFunctions.Notebook`
