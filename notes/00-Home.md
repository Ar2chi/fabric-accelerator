---
title: Fabric Accelerator - Dokumentacja
tags: [fabric-accelerator, moc]
---

# Fabric Accelerator — Dokumentacja projektu

Ten folder to lokalna baza wiedzy (Obsidian vault) dla repozytorium **fabric-accelerator**,
oparta na oficjalnym wiki projektu ([bennyaustin/fabric-accelerator](https://github.com/bennyaustin/fabric-accelerator/wiki)
i [bennyaustin/elt-framework](https://github.com/bennyaustin/elt-framework/wiki)) oraz na wnioskach
z analizy kodu tego konkretnego repozytorium.

## Mapa dokumentacji (MOC)

### Wprowadzenie i architektura
- [[01-Setup-and-Deployment]] — jak wdrożyć akcelerator (Service Principal, GitHub Actions, konfiguracja)
- [[02-Architecture]] — architektura medalionowa: Hot Path (RTI) i Cold Path (batch)

### Metadata-driven ELT Framework
- [[03-ELT-Framework-Overview]] — koncepcje: Definition/Instance, Source System, Stream
- [[04-ELT-Data-Model]] — pełny model danych control DB (IngestDefinition, L1/L2TransformDefinition, ColumnMapping...)
- [[05-Data-Factory-Pipelines]] — pipeline'y ingestu i transformacji (ASQL, MirrorDB, L1, L2)
- [[06-Spark-Notebooks]] — reużywalne notebooki (CommonTransforms, DeltaLakeFunctions, L1Transform-Generic-Fabric...)

### Głębsze analizy (z naszej rozmowy)
- [[07-Column-Mapping-and-Customization]] — jak realnie działa (i czego brakuje) w mapowaniu/zmianie nazw kolumn
- [[08-L1-vs-L2-Architecture]] — dlaczego L1 to generyczny notebook Spark, a L2 to dedykowane procedury T-SQL
- [[09-Capacity-and-Cost-Considerations]] — koszt Spark sesji: High Concurrency Mode vs `runMultiple()`/DAG, granularność Capacity Metrics

### Obserwowalność i eksploracja
- [[10-Monitoring-Observability-RTI]] — Fabric Events, Eventstream, KQL, Real-Time Dashboard, alerty
- [[11-Exploring-the-Accelerator]] — praktyczny spacer po akceleratorze (uruchomienie pipeline'ów, zapytania SQL)

## Struktura repozytorium (skrót)

```
workspace/
  elt-framework/controlDB.SQLDatabase/   # metadane ELT (control DB)
  lakehouse/                             # lh_bronze, lh_silver
  warehouse/dw_gold.Warehouse/           # gold DW + stored procedures L2
  pipeline/
    elt-framework/                       # wwi-elt-framework (utility)
    reusable/azure-sql/                  # ingest ASQL
    reusable/mirrordb/                   # ingest mirrored DB
    reusable/level1-transform/           # L1: bronze -> silver (Spark)
    reusable/level2-transform/           # L2: silver -> gold (T-SQL SP)
  rti/                                   # Eventstream, Eventhouse, Dashboard
ipynb/                                   # notebooki reużywalne (źródło .ipynb)
.github/workflows/deploy-fabric-accelerator.yml   # CI/CD (Fabric CLI)
```

## Źródła
- Wiki: https://github.com/bennyaustin/fabric-accelerator/wiki
- Wiki ELT Framework: https://github.com/bennyaustin/elt-framework/wiki
- Ten dokument zawiera też ustalenia z analizy kodu wykonanej w rozmowie z Copilotem (2026-07-13).
