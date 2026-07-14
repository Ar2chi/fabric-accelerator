---
title: Praktyczny spacer po akceleratorze
tags: [fabric-accelerator, tutorial, walkthrough]
---

# Eksploracja akceleratora — praktyczny spacer

Powiązane: [[01-Setup-and-Deployment]] · [[03-ELT-Framework-Overview]] · [[04-ELT-Data-Model]] · [[05-Data-Factory-Pipelines]]

Rekomendowana kolejność kroków po zakończeniu wdrożenia (patrz [[01-Setup-and-Deployment]]).

## 1. Wygeneruj metadane ELT Framework

Uruchom pipeline **`wwi-elt-framework`** (Data Factory) — wypełnia metadane do ekstrakcji danych
z Wide World Importers (WWI).

## 2. Zapytania SQL do zrozumienia metadanych

Uruchamiaj po kolei w control DB:

```sql
DECLARE @SourceSystem VARCHAR(50), @StreamName VARCHAR(100)
SET @SourceSystem = 'WWI'
SET @StreamName = NULL

-- Ingest Definition
SELECT * FROM ELT.IngestDefinition
WHERE SourceSystemName = @SourceSystem AND StreamName LIKE ISNULL(@StreamName,'%')

-- Kolejny ingest (co pipeline faktycznie wykona)
EXEC ELT.GetIngestDefinition @SourceSystem, @StreamName, 50

-- Historia instancji ingestu
SELECT * FROM ELT.IngestInstance
WHERE IngestID IN (SELECT IngestID FROM ELT.IngestDefinition
                   WHERE SourceSystemName = @SourceSystem AND StreamName LIKE ISNULL(@StreamName,'%'))
ORDER BY IngestInstanceID DESC

-- L1 Transform Definition
SELECT * FROM ELT.L1TransformDefinition
WHERE IngestID IN (SELECT IngestID FROM ELT.IngestDefinition
                   WHERE SourceSystemName = @SourceSystem AND StreamName LIKE ISNULL(@StreamName,'%'))

-- Kolejne instancje L1
EXEC ELT.GetTransformInstance_L1 @SourceSystem, @StreamName

-- Historia L1
SELECT * FROM ELT.L1TransformInstance
WHERE IngestID IN (SELECT IngestID FROM ELT.IngestDefinition
                   WHERE SourceSystemName = @SourceSystem AND StreamName LIKE ISNULL(@StreamName,'%'))
ORDER BY L1TransformInstanceID

-- L2 Transform Definition
SELECT * FROM ELT.L2TransformDefinition
WHERE IngestID IN (SELECT IngestID FROM ELT.IngestDefinition
                   WHERE SourceSystemName = @SourceSystem AND StreamName LIKE ISNULL(@StreamName,'%'))

-- Kolejne instancje L2
EXEC ELT.GetTransformInstance_L2 @SourceSystem, @StreamName

-- Historia L2
SELECT * FROM ELT.L2TransformInstance
WHERE IngestID IN (SELECT IngestID FROM ELT.IngestDefinition
                   WHERE SourceSystemName = @SourceSystem AND StreamName LIKE ISNULL(@StreamName,'%'))
ORDER BY L2TransformInstanceID
```

## 3. Zrozum koncepcje ELT Framework

Przeczytaj [[03-ELT-Framework-Overview]] (koncepcje Definition/Instance, Ingest/L1/L2) oraz
[[04-ELT-Data-Model]] (pełny model danych).

## 4. Uruchom `Master ELT ASQL Pipeline`

Parametry do ustawienia (do testu end-to-end na jednej tabeli, np. `StreamName = 'Colors'`):

| Parametr | Opis | Przykład |
|---|---|---|
| `SourceSystemName` | Metadane źródła danych | `WWI` |
| `StreamName` | Metadane nazwy tabeli/encji | zostaw puste dla wszystkich, lub np. `Colors` do testu jednej tabeli |
| `DelayTransformation` | Czy L1 wykonuje się jako część ingestu | `0` |
| `BronzeObjectID` / `BronzeWorkspaceID` | ID Lakehouse `lh_bronze` / workspace | z Fabric portal |
| `SilverObjectID` / `SilverWorkspaceID` | ID Lakehouse `lh_silver` / workspace | z Fabric portal |
| `GoldObjectID` / `GoldWorkspaceID` / `GoldDWEndpoint` | ID Warehouse `dw_gold` / workspace / endpoint SQL | z Fabric portal |

Z tymi parametrami pipeline: pobierze metadane z control DB → wgra dane z WWI do `lh_bronze` →
wykona Level 1 transformację do `lh_silver` (notebooki Spark) → utworzy zagregowany snapshot jako
Level 2 transformację w `dw_gold` (stored procedures).

## 5. Obserwuj wykonanie

Śledź status w Monitor (Data Factory) oraz w tabelach `*Instance` (control DB) — statusy
`Running → Success/Failure`, liczniki insert/update/delete, GUID-y `ADFPipelineRunID` do lineage.

## Wideo

Przegląd akceleratora: https://youtu.be/xuCPuezUm7E
