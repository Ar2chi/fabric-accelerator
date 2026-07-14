---
title: Data Factory Pipelines
tags: [fabric-accelerator, data-factory, pipelines]
---

# Reużywalne pipeline'y Data Factory

Powiązane: [[04-ELT-Data-Model]] · [[06-Spark-Notebooks]] · [[08-L1-vs-L2-Architecture]] · [[09-Capacity-and-Cost-Considerations]]

Filozofia projektowa: **generyczne pipeline'y** dla różnych źródeł danych, reużywalne i skalowalne
przez metadane z ELT Framework.

## Ingest: źródło → bronze

### [a] SQL Server (`workspace/pipeline/reusable/azure-sql/`)

Dwa pipeline'y współpracujące ze sobą:

- **`Master ELT ASQL`** — pipeline rodzic, orkiestruje ingest dla źródła SQL Server, wywołując
  pipeline potomny dla każdej tabeli/widoku/zapytania.
  Parametry: `SourceSystemName`, `StreamName` (opcjonalny), `DelayTransformation`,
  `BronzeObjectID/WorkspaceID`, `SilverObjectID/WorkspaceID`, `GoldObjectID/WorkspaceID/DWEndpoint`.
- **`Ingest ASQL Table`** — pipeline dziecko, realny ingest pojedynczej tabeli SQL Server do OneLake
  (Copy Activity, `AzureSqlSource` → `ParquetSink` na Lakehouse `Files`).

Obsługuje pełne i przyrostowe ładowania (na bazie metadanych `WatermarkColName`, `LastDeltaDate/Number`).

### [b] File Ingestion (planowane)
Generyczny pipeline do ingestu plików z OneLake / dowolnego storage HDFS-compatible.
*Feature planowany na przyszłe wydanie — jeszcze niezaimplementowany.*

### [c] REST API Ingestion (planowane)
Generyczny pipeline do ingestu z REST API, dane lądują jako JSON w bronze.
*Feature planowany na przyszłe wydanie — jeszcze niezaimplementowany.*

### MirrorDB (`workspace/pipeline/reusable/mirrordb/`)
- **`Master ELT MirrorDB`** — orkiestracja dla źródeł typu mirrored database.
- **`Ingest MirrorDB Table`** — ingest pojedynczej tabeli z mirrored DB (np. `WideWorldImporters-mirror`).

## Transformacja: bronze → silver → gold

### [a] Level 1 Transform — bronze → silver (`workspace/pipeline/reusable/level1-transform/`)

Reużywalne pipeline'y biorące dane z bronze, czyszczące/wzbogacające/kuratorujące, lądujące w silver
**z zachowaniem granularności**. Tylko jedna instancja tych pipeline'ów potrzebna w całej platformie —
notebooki Spark (podłączone/odłączone przez metadane) wykonują właściwe transformacje.

- **`Master Level1 Transform`** — rodzic; `ForEach` (domyślnie `isSequential: false, batchCount: 4`)
  wywołuje pipeline dziecko dla każdego pliku/tabeli/zapytania w bronze.
- **`Level1 Transform`** — dziecko; aktywność `TridentNotebook` z `notebookId = @pipeline().parameters.ComputeName`
  (dynamiczny wybór notebooka z metadanych `L1TransformDefinition.ComputeName`).
  Otoczona aktywnościami `SqlServerStoredProcedure` (`UpdateTransformInstance_L1`) do zapisu statusu
  RUNNING/SUCCESS/FAILURE w control DB.

> [!warning] Koszt capacity
> `batchCount: 4` bez włączonego High Concurrency Mode oznacza **4 osobne sesje Spark jednocześnie**.
> Patrz [[09-Capacity-and-Cost-Considerations]].

### [b] Level 2 Transform — silver → gold (`workspace/pipeline/reusable/level2-transform/`)

Stosują reguły biznesowe do danych z silver (czasem też z bronze), lądując w gold. Granularność
danych **zwykle się zmienia** (agregacja, konsolidacja, snapshoty, schemat gwiazdy).

- **`Master Level2 Transform`** — rodzic; orkiestruje wywołania dla każdego pliku/tabeli/zapytania.
- **`Level2 Transform`** — dziecko; aktywność `SqlServerStoredProcedure` z
  `storedProcedureName = @pipeline().parameters.ComputeName` — **to zawsze T-SQL, nie Spark**
  (patrz [[08-L1-vs-L2-Architecture]] po szczegóły i przykład).

## Pipeline użytkowy

### `wwi-elt-framework` (`workspace/pipeline/elt-framework/`)
Pipeline referencyjny do wypełniania metadanych ELT Framework — w tym przypadku dla wszystkich
tabel Wide World Importers (WWI) SQL database. Parametr: `NotebookID_L1Transform-Generic-Fabric`
(Fabric object ID notebooka generycznego L1).

## Podsumowanie: dwuwarstwowy wzorzec (Master + Child)

Każdy etap (Ingest, L1, L2) używa tego samego wzorca: **pipeline rodzic** czyta metadane
(`Lookup` na stored procedure) i pętlą `ForEach` wywołuje **pipeline dziecko** per encja/tabela,
przekazując parametry z rekordu metadanych. Dzięki temu dodanie nowej tabeli = nowy wiersz
w control DB, zero zmian w kodzie pipeline'u.
