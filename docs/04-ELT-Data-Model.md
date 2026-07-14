---
title: Model danych ELT (control DB)
tags: [fabric-accelerator, elt-framework, sql, metadata]
---

# Model danych ELT Framework (control DB)

Powiązane: [[03-ELT-Framework-Overview]] · [[05-Data-Factory-Pipelines]] · [[07-Column-Mapping-and-Customization]]

Lokalizacja w repo: `workspace/elt-framework/controlDB.SQLDatabase/ELT/Tables/`

## Przegląd tabel

| Tabela | Rola |
|---|---|
| `IngestDefinition` | Definicja: jak ładować dane źródłowe do Raw/bronze |
| `IngestInstance` | Audyt/log każdego uruchomienia ingestu |
| `L1TransformDefinition` | Definicja: transformacja bronze → silver (Spark) |
| `L1TransformInstance` | Audyt/log każdego uruchomienia L1 |
| `L2TransformDefinition` | Definicja: transformacja silver → gold (T-SQL SP) |
| `L2TransformInstance` | Audyt/log każdego uruchomienia L2 |
| `ColumnMapping` | Mapowanie nazw kolumn źródło→cel (per Ingest/L1/L2) |

`U` = wypełniane przez użytkownika (initialization scripts), `S` = generowane systemowo.

## 1. IngestDefinition

Kluczowe kolumny:

| Kolumna | Opis |
|---|---|
| `IngestID` | PK, generowany automatycznie |
| `SourceSystemName` / `StreamName` | Identyfikacja źródła i encji (np. `WWI` / tabela) |
| `Backend` | Typ repozytorium źródła (SQL on-prem, Oracle, HTTP API, Drop File...) |
| `EntityName` | Bazowa tabela/widok/endpoint |
| `WatermarkColName` | Kolumna delta (timestamp lub running number); `NULL` = pełny ładunek za każdym razem |
| `LastDeltaDate` / `LastDeltaNumber` | Ostatnia wartość watermarku |
| `MaxIntervalMinutes` / `MaxIntervalNumber` | Maks. zakres ekstrakcji na jedno uruchomienie |
| `DataMapping` | **JSON** (ADF `TabularTranslator`) do renamingu kolumn źródłowych lub nadawania nagłówków plikom bez nagłówka — `CHECK (isjson(...)=1)` |
| `SourceStructure` | JSON opisujący dokładnie jakie kolumny/typy pobrać z ADF dataset |
| `DestinationRawFileSystem/Folder/File` | Docelowa lokalizacja w bronze (wspiera tokeny `YYYY/MM/DD/HH/MI/SS`) |
| `RunSequence` | Kolejność uruchamiania pipeline'ów w ramach `SourceSystemName` |
| `MaxRetries`, `ActiveFlag` | Retry i włącz/wyłącz |
| `L1TransformationReqdFlag`, `L2TransformationReqdFlag` | Czy wymagana transformacja L1/L2 |
| `DelayL1/L2TransformationFlag` | Czy transformacja odroczona do osobnego harmonogramu |

> [!note] `DataMapping` — patrz [[07-Column-Mapping-and-Customization]] po pełny mechanizm (w tym uwaga,
> że generujące go funkcje SQL w tym repo nie są nigdzie automatycznie wywoływane).

## 2. IngestInstance

Log wykonania ingestu: `IngestInstanceID`, `DataFromTimestamp/Number`, `DataToTimestamp/Number`,
`SourceCount`, `IngestCount`, `IngestStartTimestamp/EndTimestamp`,
`IngestStatus` (`Running|Success|ReRunSuccess|Failure|ReRunFailure`), `RetryCount`, `ReloadFlag`,
`ADFIngestPipelineRunID`.

## 3. L1TransformDefinition

W tym repo dokładna definicja tabeli (`workspace/elt-framework/controlDB.SQLDatabase/ELT/Tables/L1TransformDefinition.sql`):

| Kolumna | Opis |
|---|---|
| `L1TransformID` | PK |
| `IngestID` | FK do `IngestDefinition` |
| `ComputePath` / `ComputeName` | **Ścieżka i nazwa notebooka Spark, który ma zostać uruchomiony** — steruje tym pipeline `Level1 Transform.DataPipeline` (`notebookId = @pipeline().parameters.ComputeName`) |
| `CustomParameters` | JSON — hak rozszerzeń przekazywany do notebooka, w domyślnym `L1Transform-Generic-Fabric` **nieużywany** |
| `InputRawFileSystem/Folder/File` lub `InputRawTable` | Źródło danych (plik lub tabela mirror DB) — dokładnie jedno z dwóch (constraint) |
| `OutputL1CurateFileSystem/Folder/File` | Cel w silver |
| `OutputL1CuratedFileWriteMode` | `append│overwrite│ignore│error│errorifexists` |
| `OutputDWStagingTable`, `LookupColumns` | Wsparcie dla MERGE/upsert |
| `OutputDWTable`, `OutputDWTableWriteMode` | Tabela docelowa w silver Lakehouse |
| `WatermarkColName` | Kolumna delty (dla źródeł typu mirror DB) |
| `MaxRetries`, `ActiveFlag` | Retry / aktywność |

## 4. L1TransformInstance

Log wykonania L1: `IngestCount`, `L1TransformInsertCount/UpdateCount/DeleteCount`,
`L1TransformStartTimestamp/EndTimestamp`, `L1TransformStatus`, `RetryCount`,
`IngestADFPipelineRunID`, `L1TransformADFPipelineRunID`, `DataFromTimestamp/ToTimestamp`.

## 5. L2TransformDefinition

| Kolumna | Opis |
|---|---|
| `L2TransformID` | PK |
| `IngestID`, `L1TransformID` | Powiązania z wcześniejszymi etapami |
| `ComputePath` / `ComputeName` | **Nazwa stored procedury T-SQL** w gold Warehouse (nie notebook!) — pipeline `Level2 Transform.DataPipeline` woła ją aktywnością `SqlServerStoredProcedure` |
| `InputType` | `Raw│Curated│Datawarehouse` — skąd czytać dane wejściowe |
| `RunSequence` | Kolejność transformacji L2 względem siebie |
| `OutputDWTable`, `OutputDWTableWriteMode` | Cel w gold |

> [!info] Kluczowa różnica względem L1
> `ComputeName` w L1 = notebook Spark. `ComputeName` w L2 = stored procedure T-SQL.
> Pełne omówienie: [[08-L1-vs-L2-Architecture]].

## 6. L2TransformInstance

Analogicznie do L1: `L2TransformInsertCount/UpdateCount/DeleteCount`, `L2TransformStatus`,
`L2TransformADFPipelineRunID`, `DataFromTimestamp/ToTimestamp`, `DataFromNumber/ToNumber`.

## 7. ColumnMapping

`workspace/elt-framework/controlDB.SQLDatabase/ELT/Tables/ColumnMapping.sql`

| Kolumna | Opis |
|---|---|
| `MappingID` | PK |
| `IngestID` / `L1TransformID` / `L2TransformID` | Do którego etapu odnosi się mapowanie (dokładnie jeden z nich) |
| `SourceName`, `TargetName` | Nazwa kolumny źródłowej i docelowej |
| `TargetOrdinalPosition` | Kolejność kolumny docelowej |
| `ActiveFlag` | Włącz/wyłącz mapowanie |

Powiązane funkcje SQL (`ELT.Functions`): `uf_GetColumnMapping`, `uf_GetIngestionColumnList`,
`uf_GetTabularTranslatorMappingJson` — generują JSON/SQL na podstawie tej tabeli.
**Nie są automatycznie wywoływane przez żaden pipeline w tym repo** — patrz [[07-Column-Mapping-and-Customization]].

## Helper Procs (stored procedures)

| Procedura | Rola |
|---|---|
| `GetIngestDefinition` | Zwraca parametry ingestu + generuje SQL (delta/full) dla ADF Lookup |
| `GetIngestDefinition_FileDrop` | Wariant dla pipeline'ów File Drop |
| `InsertIngestInstance` | Tworzy rekord IngestInstance na starcie |
| `UpdateIngestDefinition` / `UpdateIngestInstance` | Aktualizacja statusu po zakończeniu |
| `GetTransformDefinition_L1/L2`, `GetTransformInstance_L1/L2` | Analogiczne dla L1/L2 |
| `InsertTransformInstance_L1/L2`, `UpdateTransformDefinition_L2`, `UpdateTransformInstance_L1/L2` | Audyt/statusy L1/L2 |

## Konfiguracja

Skrypt `Script.PostDeployment.sql` (wg wiki elt-framework) służy do konfiguracji nowego źródła danych —
trzeba go zaktualizować przy onboardingu każdego nowego systemu źródłowego.
