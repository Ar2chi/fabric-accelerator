---
title: L1 vs L2 — architektura wykonawcza
tags: [fabric-accelerator, analysis, architecture]
---

# L1 vs L2 Transform — dlaczego to dwa różne światy wykonawcze

> Zapis wniosków z analizy kodu przeprowadzonej w rozmowie z Copilotem (2026-07-13).

Powiązane: [[04-ELT-Data-Model]] · [[05-Data-Factory-Pipelines]] · [[06-Spark-Notebooks]] · [[07-Column-Mapping-and-Customization]]

## Teza wyjściowa

L1 ma jeden generyczny notebook Spark używany dla (prawie) każdej tabeli. Czy L2 działa podobnie?
**Nie** — L2 ma zupełnie inny mechanizm wykonawczy.

## L1 Transform: pipeline → Notebook Spark

`Level1 Transform.DataPipeline`:
1. `SqlServerStoredProcedure` → `UpdateTransformInstance_L1` (status RUNNING)
2. **`TridentNotebook`** aktywność, `notebookId = @pipeline().parameters.ComputeName`
   (dynamiczny wybór notebooka Spark z metadanych)
3. `SqlServerStoredProcedure` → `UpdateTransformInstance_L1` (status SUCCESS/FAILURE)

`ComputeName` w `L1TransformDefinition` → **nazwa/ID notebooka Fabric**. Domyślnie
`L1Transform-Generic-Fabric` dla większości tabel.

## L2 Transform: pipeline → Stored Procedure T-SQL

`Level2 Transform.DataPipeline`:
1. `SqlServerStoredProcedure` → `InsertTransformInstance_L2` (status RUNNING)
2. **`EXEC L2Transform`** — aktywność typu **`SqlServerStoredProcedure`**,
   `storedProcedureName = @pipeline().parameters.ComputeName`, wykonywana bezpośrednio na
   warehouse `dw_gold` (linkedService typu `DataWarehouse`)
3. `SqlServerStoredProcedure` → `UpdateTransformInstance_L2` (status SUCCESS/FAILURE)

`ComputeName` w `L2TransformDefinition` → **nazwa procedury T-SQL** w schemacie `gold`
(`workspace/warehouse/dw_gold.Warehouse/gold/StoredProcedures/`). **Nie ma tu Sparka w ogóle.**

### Przykład: `create_orders_monthly_snapshot`

```sql
CREATE PROC [gold].[create_orders_monthly_snapshot]
    (@L2TransformInstanceID INT NULL, @L2TransformID INT NULL, ... )
AS
BEGIN
    IF EXISTS (SELECT 1 FROM INFORMATION_SCHEMA.TABLES
               WHERE TABLE_SCHEMA='gold' AND TABLE_NAME='orders_monthly_snapshot')
        DROP TABLE gold.orders_monthly_snapshot

    CREATE TABLE gold.orders_monthly_snapshot AS
    SELECT s.SalespersonPersonID, sp.FullName,
           EOMONTH(s.OrderDate) AS OrderMonth,
           COUNT(1) AS OrderCount
    FROM lh_silver.dbo.silver_sales_orders AS s
    LEFT JOIN lh_silver.dbo.silver_application_people AS sp
        ON sp.PersonID = s.SalespersonPersonID AND sp.IsSalesperson = 1
    GROUP BY EOMONTH(s.OrderDate), s.SalespersonPersonID, sp.FullName
END
```

To pełny, ręcznie napisany biznesowy SQL (agregacja, JOIN, snapshot) — **nie ma generycznego
szablonu L2**, w przeciwieństwie do L1.

## Tabela porównawcza

| | **L1 (bronze → silver)** | **L2 (silver → gold)** |
|---|---|---|
| Aktywność pipeline'u | `TridentNotebook` | `SqlServerStoredProcedure` |
| `ComputeName` wskazuje na | Notebook Spark | Stored procedure T-SQL |
| Silnik wykonawczy | PySpark (sesja Spark) | Silnik SQL Warehouse |
| Generyczny szablon? | Tak — `L1Transform-Generic-Fabric` pasuje do większości tabel | Nie — każda tabela wynikowa ma dedykowaną procedurę |
| Typowa logika | dedup, trim, null-handling, upsert (zachowana granularność) | agregacja, JOIN, snapshot, star schema (zmieniona granularność) |
| Customizacja per tabela | Podmiana notebooka przez `ComputeName`, ewent. `CustomParameters` | Zawsze własna procedura — nie ma "generic L2" |

## Dlaczego L2 nie ma generycznego odpowiednika

Z natury L2 wykonuje transformacje specyficzne biznesowo (agregacje, pivoty, fact/dim) — zgodnie
z definicją w wiki ELT Framework (`L2 Transform Definition`). Nie da się tego uogólnić do jednego
szablonu tak, jak `trim`/`replaceNull`/`deDuplicate` dają się uogólnić dla L1.

Metadane `L2TransformDefinition` konfigurują wyłącznie **parametry orkiestracji** (skąd czytać przez
`InputType`, watermarki, zakresy dat, retry, `RunSequence`, tryb zapisu) — nie samą logikę biznesową.
`CustomParameters` (JSON) jest przekazywany do procedury, ale to zależy od implementacji, czy
konkretna procedura go użyje (np. przez `OPENJSON(@CustomParameters)`).
