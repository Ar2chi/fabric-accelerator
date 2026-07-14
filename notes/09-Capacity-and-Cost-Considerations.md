---
title: Koszt capacity — High Concurrency vs DAG/runMultiple
tags: [fabric-accelerator, analysis, cost, spark, capacity]
---

# Koszt Capacity: sesje Spark, High Concurrency Mode i `runMultiple()`

> Zapis wniosków z analizy/dyskusji z Copilotem (2026-07-13) na temat optymalizacji kosztów Spark
> w architekturze L1 tego akceleratora.

Powiązane: [[05-Data-Factory-Pipelines]] · [[06-Spark-Notebooks]] · [[08-L1-vs-L2-Architecture]]

## Problem wyjściowy

`Master Level1 Transform.DataPipeline` używa `ForEach` z `isSequential: false, batchCount: 4` —
do 4 tabel przetwarzanych równolegle. Każda z nich wywołuje osobną `TridentNotebook` activity.
**W trybie standardowym każda notebook activity domyślnie startuje własną, nową sesję Spark** —
4 równoległe = 4 osobne sesje/rozliczenia jednocześnie. W tym repo **nigdzie nie jest skonfigurowany
High Concurrency Mode** (brak `spark.highConcurrency.max` w kodzie/pipeline'ach).

## Opcja A: High Concurrency Mode (natywny mechanizm Fabric)

- Wiele Notebook Activity **współdzieli jedną uruchomioną sesję Spark** zamiast tworzyć nową za każdym razem.
- Warunki współdzielenia sesji: te same ustawienia Lakehouse domyślnego + te same ustawienia Spark compute.
  Jeśli się różnią — Fabric **automatycznie** tworzy osobną sesję (wbudowany filtr kompatybilności).
- **Billing**: rozliczana jest tylko aktywność **inicjująca** sesję. Kolejne (2..N) nie są billowane
  osobno — i **nie pojawiają się osobno w Capacity Metrics** (patrz sekcja Capacity Metrics niżej).
- Domyślny limit współdzielenia sesji: **5** notebooków. Można podnieść do **50**
  (`spark.highConcurrency.max = <2-50>` w Environment).
- Executory alokowane FAIR schedulingiem między REPL-ami, by zredukować ryzyko "głodzenia" (starvation).
- **Włączenie**: Environment → Spark Properties →
  ```
  spark.highConcurrency.max = 20
  ```
  Wg wiki Set-up (krok 6): *"po zakończeniu deploymentu włącz high concurrency dla notebooków i dla
  pipeline'ów uruchamiających wiele notebooków"*.

## Opcja B: `notebookutils.notebook.runMultiple()` (DAG w kodzie)

Alternatywny pomysł: zamiast wielu Notebook Activities w pipeline, jeden notebook-orchestrator
buduje dynamicznie DAG z metadanych i woła `runMultiple()`.

### Limity i ograniczenia (z dokumentacji Microsoft)

| Parametr | Wartość |
|---|---|
| Maks. concurrent notebooks w jednym `runMultiple()` | **50** |
| Domyślna współbieżność (`concurrency`) | 3× liczba dostępnych rdzeni CPU sesji (konfigurowalne, `0`=unlimited) |
| Domyślny timeout per cela | 90 sekund (`timeoutPerCellInSeconds`) |
| Domyślny timeout całego DAG | 12h (`timeoutInSeconds`) |
| Zasoby | Wszystkie notebooki w DAG dzielą **jedną** sesję/driver |

### Dlaczego nie jest to "lepsze" rozwiązanie tego samego problemu

- `runMultiple()` nie skaluje zasobów niezależnie — wszystkie child-runy są ograniczone rozmiarem
  **jednej** sesji Spark (nie ma automatycznego rozłożenia na osobne sesje jak w standardowym trybie).
- Wymagałoby przeniesienia całej logiki orkiestracji/audytu (dziś realizowanej przez pipeline +
  stored procedury `InsertTransformInstance_L1`, `UpdateTransformInstance_L1`) do kodu Python
  wołającego SQL z wnętrza notebooka — utrata natywnego monitoringu ADF, integracji z alertami
  Fabric Events (patrz [[10-Monitoring-Observability-RTI]]).
- L2 (T-SQL stored procedures) i ingest (Copy Activity, Mirroring) **w ogóle nie mieszczą się**
  w modelu `runMultiple()` — to nie jest Spark.

### Porównanie: `runMultiple(concurrency=4)` vs HC Mode (limit≥4)

Przy **równych, świadomie ustawionych limitach** różnica w ryzyku "niestabilności drivera" praktycznie
zanika — to ten sam mechanizm współdzielenia sesji (REPL cores). Realne różnice:

| Aspekt | High Concurrency Mode | `runMultiple()` + DAG |
|---|---|---|
| Warunek wejścia do współdzielonej sesji | Automatyczny filtr: ten sam Lakehouse + Spark compute | Brak — trzeba pilnować ręcznie |
| Widoczność w Capacity Metrics | Też ukryta pod aktywnością inicjującą (patrz niżej) | Też ukryta pod root-notebookiem |
| Integracja z natywnym monitoringiem ADF/pipeline | Tak, każda aktywność nadal osobno widoczna w historii pipeline'u | Nie — child-runy widoczne tylko w snapshotach notebooka |
| Wymaga przepisania orkiestracji | Nie — zero zmian w istniejących pipeline'ach | Tak — trzeba zbudować notebook-orchestrator |

## Capacity Metrics — granularność kosztu

Kluczowy, zaskakujący wniosek: **ani HC Mode, ani DAG nie dają pełnej granularności kosztu per tabela w Capacity Metrics.**

> *"Only the initiating notebook or pipeline activity that starts the shared Spark application is
> billed. [...] This behavior is also reflected in Capacity Metrics, where usage is reported against
> the initiating notebook."* — dokumentacja Microsoft, High Concurrency Mode.

| Scenariusz | Widoczność kosztu w Capacity Metrics |
|---|---|
| Bez HC, bez DAG (obecny stan) — każda notebook activity = osobna sesja | Pełna granularność — każdy notebook to osobny "item" z własnym CU |
| HC Mode włączony | Koszt widoczny tylko przy notebooku inicjującym sesję; pozostałe "ukryte" |
| `runMultiple()` / DAG | Koszt widoczny tylko przy notebooku-orchestratorze (root); child-aktywności "ukryte" |

**Wniosek:** obie metody redukcji kosztu (HC i DAG) mają identyczny efekt uboczny — utratę
możliwości rozbicia rachunku "ile kosztowała tabela X" wyłącznie z poziomu Capacity Metrics.
Jeśli potrzebna jest pełna granularność kosztowa per tabela, jedyną opcją jest pozostanie przy
osobnych sesjach (bez HC, bez DAG) i zaakceptowanie wyższego kosztu.

Dodatkowe miejsca do wglądu w granularność (poza Capacity Metrics):
- **Timepoint item detail page** (Capacity Metrics app) — operacje wewnątrz jednego itemu, filtr po
  Operation ID/user/CU threshold.
- Dla DAG: własny snapshot/log wykonania child-notebooka (czas trwania per aktywność w wyniku
  `runMultiple()`), ale to czas, nie koszt CU wprost.

## Rekomendacja

1. **Pierwszy krok**: włącz High Concurrency Mode (`spark.highConcurrency.max` ≥ `batchCount` z ForEach,
   np. 10-20) — rozwiązuje problem kosztowy przy zerowym nakładzie na przebudowę frameworka.
2. Zostaw pipeline'y (Data Factory) jako warstwę orkiestracji/audytu/control-plane — to jedyny
   mechanizm obsługujący ingest (Copy/Mirroring) i L2 (T-SQL) natywnie.
3. Rozważ `runMultiple()`/DAG tylko jeśli po HC Mode nadal występuje wąskie gardło (bardzo duża
   liczba bardzo małych, krótkotrwałych transformacji, gdzie narzut startowy REPL dominuje) —
   i tylko dla samego kroku L1, nie całej orkiestracji.
4. Jeśli budujesz coś od zera: pipeline'y jako szkielet (audyt, retry, monitoring, integracja
   multi-engine) + HC Mode od początku dla kroków Spark, zamiast czekać z optymalizacją "na później".
