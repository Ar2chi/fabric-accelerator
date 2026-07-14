---
title: ELT Framework — koncepcje
tags: [fabric-accelerator, elt-framework, metadata]
---

# ELT Framework — przegląd i koncepcje

Powiązane: [[00-Home]] · [[02-Architecture]] · [[04-ELT-Data-Model]] · [[05-Data-Factory-Pipelines]]

Repo źródłowe frameworka: https://github.com/bennyaustin/elt-framework

## Czym jest ELT Framework

Metadata-driven narzędzie orkiestracji ingestu i transformacji dla nowoczesnych platform danych.
Wspiera batch ingestion, testowane z Microsoft Fabric, Azure Databricks i Azure Synapse.
Metadane przechowywane w kompatybilnej z ANSI **control database** (w tym repo: Fabric SQL Database).

## Kluczowe cechy

- **Konfigurowalny i rozszerzalny** — łatwa adaptacja do specyficznych potrzeb.
- **Agnostyczny względem źródła danych** — bazy, Delta Lake, REST API, pliki płaskie, JSON, XML —
  **bez przechowywania connection stringów w metadanych**.
- **Delta i Full Loads** — wsparcie dla ładowań przyrostowych i pełnych.
- **Re-run i Retry** — automatyczna obsługa błędów bez interwencji ręcznej.
- **Wbudowany audyt** — śledzenie przetwarzania danych.
- **Rozszerzony audyt** — integracja z Azure PaaS (Diagnostic Logging).
- **Eliminacja ręcznego patchowania danych**.
- **Wsparcie dla lineage danych**.
- **Transformacje Level 1 i Level 2** — wsparcie dla relacji 1:N i N:N.
- **Zarządzanie on-demand** — włączanie/wyłączanie pipeline'ów i transformacji.

## Czym ELT Framework NIE jest

- To NIE jest silnik reguł biznesowych (rules engine).
- To NIE jest generator kodu.
- NIE wspiera źródeł streamingowych.

## Koncepcje: Definition vs Instance

- **Definition** — jednorazowa konfiguracja metadanowa (`IngestDefinition`, `L1TransformDefinition`,
  `L2TransformDefinition`) — definiuje JAK ma przebiegać ingest/transformacja.
- **Instance** — każde wykonanie definicji generuje rekord instancji (`IngestInstance`,
  `L1TransformInstance`, `L2TransformInstance`) — używany do audytu, lineage, re-runów.

## Source System i Stream

- **Source System** — dowolne źródło danych wejściowych (ERP, system floty, Historian, system rejestracji itd.)
- **Stream** — encja w ramach systemu źródłowego, np. tabela/widok, endpoint REST API, plik płaski.

## Poziomy transformacji

### Ingest Definition
Konfiguruje ładowanie danych ze źródeł do strefy Raw/Landing (Data Lake). Dane lądują "as-is",
zachowując oryginalną granularność (zwykle parquet dla baz danych, natywny format dla API/plików).
Foldery partycjonowane wg Source/Entity/Year/Month/Day.

### L1 Transform Definition
Po zaingestowaniu, dane z Raw zone są transformowane (Spark) i lądują w Trusted/Structured zone.
Typowe wzbogacenia: deduplikacja, trim, upsert/merge, konwersja stref czasowych, standaryzacja
formatów dat (np. Julian date), spłaszczanie JSON/XML, dodawanie nagłówków, tłumaczenie nazw kolumn
(np. z niemieckiego SAP), usuwanie kolumn systemowych, wzorce SCD.

> Zobacz [[08-L1-vs-L2-Architecture]] i [[07-Column-Mapping-and-Customization]] dla pogłębionej analizy
> tego, jak to wygląda faktycznie w kodzie tego repo.

### L2 Transform Definition
Drugi poziom transformacji — zmienia się granularność danych przez zastosowanie konkretnych reguł
biznesowych. Źródłem może być Raw, Trusted (Curated1) lub tabela DW. Typowe transformacje:
agregacja, pivot/unpivot, redakcja, konsolidacja, mash-up danych z różnych systemów, snapshoty,
post-processing, fact/dim.

## Deploy controlDB

1. Zforkuj https://github.com/bennyaustin/elt-framework
2. Uruchom GitHub Action **ControlDB Deployment** (deployuje obiekty control DB).
3. Wymagane repository secrets: `CLIENT_ID`, `CLIENT_SECRET`, `SUBSCRIPTION_ID`, `TENANT_ID`,
   `CONTROLDB_CONNECTIONSTRING` (format Service Principal, np.:
   `Server=tcp:<Fabric SQL Server>.database.fabric.microsoft.com,1433;Initial Catalog=<controlDB Name>;...;Authentication=Active Directory Service Principal;...`)
4. Uruchom workflow ręcznie (`Run Workflow`).

## Zastosowania (Implementation References)
- Microsoft Fabric data platform (ten akcelerator)
- Azure Databricks data platform
- Azure Synapse data platform
