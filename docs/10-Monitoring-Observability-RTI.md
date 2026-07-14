---
title: Monitoring, Obserwowalność i Real-Time Intelligence
tags: [fabric-accelerator, rti, observability, monitoring]
---

# Monitoring, Alerty i Real-Time Intelligence (RTI)

Powiązane: [[02-Architecture]] · [[00-Home]]

Fabric Accelerator wykorzystuje **Fabric Events** do monitorowania i alertowania w czasie
rzeczywistym, zwiększając observability platformy danych: aktywność w OneLake i workspace'ach
Fabric, wykonanie pipeline'ów i notebooków Spark.

## Jak to działa (przepływ danych)

Lokalizacja: `workspace/rti/`

1. **Eventstream `es_fabricEvents`** łączy się ze źródłami eventów w platformie danych.
2. Podłączone źródła danych:

   | Źródło | Zakres |
   |---|---|
   | Fabric Workspace Item Events | Domyślny workspace akceleratora |
   | Fabric Job Events | `Master ETL ASQL` pipeline, `Optimize DeltaLake Tables` notebook |
   | Fabric OneLake Events | `lh_bronze`, `lh_silver` (Lakehouse), `dw_gold` (Warehouse) |

3. Eventstream filtruje dane wg schematu (workspace/job/OneLake) i ląduje w **KQL Database
   `kdb_fabricEvents`** (Eventhouse `eh_fabricAccelerator`) — to warstwa **bronze**.
   Dedykowane tabele Kusto: `workspaceEvent`, `jobEvent`, `storageEvent`.
4. **Update Policies** wyciągają dane z kolumn JSON i deduplikują w czasie rzeczywistym:
   - `workspaceEvent` → `expandWorkspaceEvents()` → `workspaceEventsExpanded` (**silver**)
   - `jobEvent` → `expandJobEvents()` → `jobEventsExpanded` (**silver**)
   - `storageEvent` → `expandStorageEvents()` → `storageEventsExpanded` (**silver**)
5. **Materialized Views** dają dzienne snapshoty — warstwa **gold**:
   `dailyAggWorkspaceEvents`, `dailyAggJobEvents`, `dailyAggStorageEvents`.
6. **Real-Time Dashboard `fabricEventsDashboard`** buduje się na widokach gold, każdy kafelek to
   osobne zapytanie KQL.
7. **Data Activator Alerts** wyzwalane z kafelków dashboardu.

## Co jest monitorowane

- **Workspace Events** — najczęściej używane workspace'y, typy elementów, elementy, użytkownicy, akcje.
- **Job Events** — najczęściej uruchamiane pipeline'y i notebooki Spark: czas trwania, status,
  typ triggera, typ jobu, harmonogram.
- **OneLake Events** — najczęściej używane akcje OneLake przez użytkowników.
- **Job Anomalies** — anomalie wykonania pipeline'ów/jobów Spark.
- **OneLake Anomalies** — nietypowe wzorce użycia OneLake.

## Co jest notyfikowane (alerty)

- Alerty dla jobów wykazujących trend regresji względem ostatnich 60 dni.
- Alerty dla anomalii wykonania jobów.
- Alerty dla anomalii użycia OneLake.
- Alerty dla nowych użytkowników.

> Konfiguracja Data Activator (kanały alertów, grupy powiadomień) **nie jest częścią akceleratora** —
> zależy od indywidualnych potrzeb wdrożenia.

## Elementy RTI (spis)

| Element | Ścieżka w repo | Rola |
|---|---|---|
| Eventstream `es_fabricEvents` | `workspace/rti/eventstream/es_fabricEvents.Eventstream` | Źródło→routing eventów |
| Eventhouse `eh_fabricAccelerator` | `workspace/rti/eventhouse/eh_fabricAccelerator.Eventhouse` | Kontener KQL DB |
| KQL Database `kdb_fabricEvents` | `workspace/rti/eventhouse/kdb_fabricEvents.KQLDatabase` | Bronze/silver/gold (Kusto) |
| Real-Time Dashboard `fabricEventsDashboard` | `workspace/rti/dashboard/fabricEventsDashboard.KQLDashboard` | Wizualizacja + źródło alertów |

## ELT Insights (Analytics)

ELT Framework korzysta z Fabric SQL Database jako repozytorium metadanych. Planowane (upcoming
release wg wiki): raportowanie w czasie rzeczywistym przez Direct Lake semantic models, z Power BI
+ Copilot jako warstwą self-service.
