---
title: Architektura (Hot Path / Cold Path)
tags: [fabric-accelerator, architecture]
---

# Architektura

Powiązane: [[00-Home]] · [[01-Setup-and-Deployment]] · [[03-ELT-Framework-Overview]] · [[10-Monitoring-Observability-RTI]]

Akcelerator korzysta z **architektury medalionowej** (bronze/silver/gold) w dwóch równoległych ścieżkach:

- **Hot Path** — dane czasu rzeczywistego / event-driven (Real-Time Intelligence)
- **Cold Path** — dane wsadowe (batch, ELT Framework)

Diagram źródłowy: `diagrams/Architecture.vsdx` w repo.

## Hot Path

Warstwy medalionowe oparte są o tabele i materialized views w **Kusto Query Language (KQL)**.

1. **Event Streams** pobierają dane z różnych źródeł, w tym eventy Fabric (Real-Time Hub).
2. Event Stream filtruje i kieruje dane do KQL Database — surowe dane tworzą **warstwę bronze**.
3. **Update policies** na tabelach Kusto transformują dane bronze → silver **automatycznie**
   (trigger przy zapisie nowych danych — brak potrzeby dodatkowej orkiestracji).
4. **Materialized views** agregują dane z tabel silver do warstwy **gold** (analityka).
5. **Real-Time Dashboards** serwują analitykę przez zapytania KQL.
6. **Data Activator Alerts** mogą być wyzwalane z Event Streams, dashboardów RTI i raportów Power BI.

Szczegóły implementacji w tym repo: [[10-Monitoring-Observability-RTI]].

## Cold Path

Warstwy medalionowe mogą być skonfigurowane jako pliki, Lakehouse lub Data Warehouse w OneLake.
W tym repo:
- **Bronze** = pliki (parquet) w Lakehouse `lh_bronze`
- **Silver** = tabele Lakehouse `lh_silver`
- **Gold** = tabele Data Warehouse `dw_gold`

Przepływ:

1. **Data Factory pipelines** pobierają dane ze źródeł chmurowych i on-premise (przez OPDG) do OneLake bronze.
2. Dane lądują w bronze jako pliki, w miarę możliwości w formacie parquet, **bez transformacji**.
3. **Spark notebooks** transformują dane z bronze → silver (Lakehouse tables): czyszczenie, spłaszczanie,
   standaryzacja — **przy zachowaniu granularności** źródła (1:1 lub 1:N).
4. **Stored procedures** w Data Warehouse stosują reguły biznesowe do danych z silver → gold:
   agregacje, snapshoty, scalanie wielu tabel, schemat gwiazdy — **granularność zwykle się zmienia** (N:1 lub 1:N).
5. Semantic models budowane na gold DW stanowią warstwę analityczną ("diamentową").
6. Całą orkiestracją steruje **ELT Framework** (control DB, metadata-driven) — patrz [[03-ELT-Framework-Overview]].
7. Power BI + Copilot jako warstwa self-service analytics.

> [!info] Kluczowa architektoniczna różnica L1 vs L2
> L1 (bronze→silver) to zawsze **Spark notebook** wywoływany dynamicznie po `ComputeName`.
> L2 (silver→gold) to zawsze **stored procedure T-SQL** w gold Warehouse, też wywoływana po `ComputeName`,
> ale przez aktywność `SqlServerStoredProcedure`, nie `TridentNotebook`.
> Pełna analiza: [[08-L1-vs-L2-Architecture]].
