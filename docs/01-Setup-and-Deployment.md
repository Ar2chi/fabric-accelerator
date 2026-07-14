---
title: Set-up i wdrożenie
tags: [fabric-accelerator, deployment, ci-cd]
---

# Set-up i wdrożenie Fabric Accelerator

Powiązane: [[00-Home]] · [[02-Architecture]] · [[03-ELT-Framework-Overview]]

## 1. Wymagania wstępne

1. **Fork repozytorium** — https://github.com/bennyaustin/fabric-accelerator
2. **Service Principal (SPN)** — utworzony lub istniejący, z rolami **Contributor** i **User Access Administrator**
   na poziomie subskrypcji. Zanotuj: Application (client) ID, Object ID, secret.
3. **Połączenie z Wide World Importers (WWI)** — przykładowe źródło danych:
   - Zainstaluj WWI jako bazę Azure SQL (wystarczy S0 DTU, Standard tier, 50GB).

   > [!info] Skąd pobrać przykładową bazę WWI
   > Oficjalne przykładowe bazy Microsoft (WideWorldImporters, WideWorldImportersDW) do pobrania stąd:
   > https://www.mssqltips.com/sqlservertip/4394/download-and-install-sql-server-2016-sample-databases-wideworldimporters-and-wideworldimportersdw/

   > [!warning] Azure SQL Database (nie Managed Instance) NIE wspiera zwykłego `.bak`
   > Standardowy plik kopii zapasowej (`.bak`, `RESTORE DATABASE ... FROM DISK`) **nie zadziała** na
   > pojedynczej bazie Azure SQL Database — `.bak`/`RESTORE` działa tylko na pełnym SQL Server lub
   > Azure SQL Managed Instance. Żeby wgrać WWI na Azure SQL Database, potrzebny jest plik **`.bacpac`**
   > (Data-tier Application), nie `.bak`:
   > 1. Jeśli masz plik `.bak` (jak z linku powyżej) — przywróć go najpierw na lokalnej instancji
   >    SQL Server (`RESTORE DATABASE WideWorldImporters FROM DISK = '...'`) albo na Azure SQL Managed Instance.
   > 2. Z tej przywróconej bazy wyeksportuj **`.bacpac`**: w SSMS kliknij prawym na bazę →
   >    **Tasks → Export Data-tier Application...**
   > 3. Zaimportuj otrzymany `.bacpac` na docelową bazę Azure SQL Database: w SSMS kliknij prawym na
   >    serwer → **Tasks → Import Data-tier Application...** (albo `SqlPackage.exe /Action:Import`).
   >
   > Bez tego kroku (próba bezpośredniego `RESTORE` z `.bak` na Azure SQL Database) import się nie powiedzie.

   - Nadaj SPN uprawnienia `db_owner`:
     ```sql
     CREATE USER [<service_principal>] FROM EXTERNAL PROVIDER
     GO
     EXEC sp_addrolemember 'db_owner', [<service_principal>]
     GO
     ```
   - Utwórz połączenie do WWI z poziomu Fabric portal (autoryzacja Service Principal), zanotuj **Connection ID**.
4. **Ustawienia tenant Fabric** (jako administrator Fabric):
   - Microsoft Fabric: użytkownicy mogą tworzyć elementy Fabric
   - Workspace settings: tworzenie workspace'ów
   - Developer settings: SPN może tworzyć workspace'y, połączenia, deployment pipelines
   - Developer settings: SPN może wywoływać publiczne API Fabric
   - Admin API settings: SPN ma dostęp do read-only i update Admin API

## 2. Repository Secrets (GitHub)

W forkowanym repozytorium ustaw sekrety:

| Sekret | Opis |
|---|---|
| `ACTION_SPN_CLIENTID` | Client/Application ID SPN |
| `ACTION_SPN_SECRET` | Sekret SPN |
| `AZURE_RG` | Resource Group na Fabric capacity |
| `SUBSCRIPTION_ID` | ID subskrypcji Azure |
| `TENANT_ID` | ID tenanta Entra |

## 3. Konfiguracja workflow `deploy-fabric-accelerator.yml`

Kluczowe zmienne środowiskowe (`env:`) do edycji przed uruchomieniem:

```yaml
FABRIC_CAPACITY_NAME: <nazwa capacity>
FABRIC_CAPACITY_LOCATION: <region>
FABRIC_CAPACITY_SKU: F2          # zalecane min. F4
FABRIC_CAPACITY_ADMIN_ID: <Object ID SPN>
RESOURCE_GROUP: <resource group capacity>
WORKSPACE_NAME: <nazwa workspace>
WORKSPACE_ADMIN_ID: <Object ID użytkownika/grupy admina>
WORKSPACE_ADMIN_TYPE: user   # lub group
FABRIC_SQL_DB_NAME: controldb-fabric-accelerator
BRONZE_LH: bronze_fabric_accelerator
SILVER_LH: silver_fabric_accelerator
GOLD_DW: gold_fabric_accelerator
WIDE_WORLD_IMPORTERS_CONNECTION_ID: <Connection ID z kroku 3>
```

Uruchom workflow **Deploy Fabric Accelerator** z zakładki *Actions* (`workflow_dispatch`).
Czas trwania: ok. 15–20 minut.

### Co robi ten workflow (job `deploy-fabric-accelerator`)

1. Instaluje `ms-fabric-cli`, loguje się jako SPN.
2. Tworzy Fabric Capacity (jeśli nie istnieje) i workspace, nadaje ACL adminowi.
3. Włącza Managed Identity workspace'u i nadaje jej rolę Contributor.
4. Usuwa istniejący Eventstream (tymczasowy workaround na redeployach).
5. Tworzy strukturę folderów (`elt-framework`, `lakehouse`, `warehouse`, `mirror`, `notebook/reusable/...`,
   `pipeline/reusable/...`, `rti/...`).
6. Tworzy Fabric SQL Database (control DB) i połączenie do niej.
7. Deployuje **mirrored database** WideWorldImporters-mirror (polling statusu mirroringu, potem `startMirroring`).
8. Tworzy medalionowe warstwy: `lh_bronze`, `lh_silver` (Lakehouse), `dw_gold` (Warehouse).
9. **Podmienia stare ID (`OLD_*`) na nowe (`NEW_*`)** we wszystkich plikach `workspace/` i `ipynb/` —
   to kluczowy mechanizm re-baselinowania artefaktów Fabric (pipeline'y, notebooki, połączenia) między tenantami.
10. Importuje notebooki (`fab import ... --format .ipynb`, z retry na niestabilne odpowiedzi API).
11. Deployuje wszystkie pipeline'y (`wwi-elt-framework`, `Level1/2 Transform`, `Master Level1/2 Transform`,
    `Ingest ASQL/MirrorDB Table`, `Master ELT ASQL/MirrorDB`) przez `fab api POST .../dataPipelines` lub
    `updateDefinition` jeśli już istnieją — z obsługą operacji asynchronicznych (`x-ms-operation-id`, polling).
12. Tworzy Eventhouse, KQL Database, Real-Time Dashboard, Eventstream (RTI).

### Drugi job: `update-gold-dw`

Buduje i publikuje **dacpac** dla `dw_gold.Warehouse` (SSDT/MSBuild), używając `Azure/sql-action`
do publikacji na Gold Data Warehouse (autoryzacja SPN, connection string z Fabric CLI).

## 4. Po wdrożeniu — WAŻNE dla kosztów

> [!tip] Włącz High Concurrency Mode
> Zaraz po zakończeniu deploymentu **włącz High Concurrency Mode** dla notebooków i dla pipeline'ów
> uruchamiających wiele notebooków — patrz [[09-Capacity-and-Cost-Considerations]].
> Bez tego każda równoległa aktywność Notebook w `ForEach` (domyślnie `batchCount=4` w
> `Master Level1 Transform.DataPipeline`) uruchamia **osobną sesję Spark**, co znacząco podnosi koszt capacity.

## 5. Deploy controlDB (obiekty ELT Framework)

Zobacz [[03-ELT-Framework-Overview#deploy-controldb]] — osobny GitHub Action w repo
`elt-framework` deployuje obiekty bazy danych (tabele, procedury, funkcje) do control DB.

## 6. Pierwsze kroki po wdrożeniu

Patrz [[11-Exploring-the-Accelerator]] — uruchomienie `wwi-elt-framework` (generowanie metadanych),
zapytania SQL do zrozumienia metadanych, uruchomienie `Master ELT ASQL`.
