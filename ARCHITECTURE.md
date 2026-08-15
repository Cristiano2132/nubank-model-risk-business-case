# Compass — Architecture & Business Case Analysis

> **Context**: Solara Digital Bank's feature store and pipeline. Computes features from customer transaction data and serves them to downstream credit models — in batch (underwriting) and near-real-time (in-app credit line adjustments).

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Component Architecture](#2-component-architecture)
3. [Data Flow — Batch Path](#3-data-flow--batch-path)
4. [Data Flow — Streaming Path](#4-data-flow--streaming-path)
5. [Database Schema (ER Diagram)](#5-database-schema-er-diagram)
6. [UML Class & Module Diagram](#6-uml-class--module-diagram)
7. [Execution Sequence — End-to-End](#7-execution-sequence--end-to-end)
8. [The Core Risk: Training/Serving Skew](#8-the-core-risk-trainingserving-skew)
9. [Risk Mapping to Business Case](#9-risk-mapping-to-business-case)
10. [Proposed Solution Architecture](#10-proposed-solution-architecture)

---

## 1. System Overview

```mermaid
graph TB
    subgraph Inputs["📥 Inputs"]
        TXN["transactions.csv\n(32,929 rows · 300 customers)"]
    end

    subgraph Pipelines["⚙️ Feature Pipelines"]
        BATCH["Batch Pipeline\n(nightly · full recompute)"]
        STREAM["Streaming Pipeline\n(incremental · event-by-event)"]
    end

    subgraph Stores["🗄️ Feature Stores - SQLite"]
        OFFLINE["Offline Store\noffline_store.db\n(point-in-time keyed)"]
        ONLINE["Online Store\nonline_store.db\n(latest value per customer)"]
    end

    subgraph API["🔌 Serving API"]
        FAPI["feature_api.py\nget_offline_features()\nget_online_features()"]
    end

    subgraph Consumers["🏦 Downstream Consumers"]
        UNDER["Underwriting Model\n(batch · full review)"]
        INAPP["In-App Credit Line\n(near-real-time · automated)"]
    end

    subgraph Monitoring["📊 Monitoring"]
        DASH["dashboard.py\nuptime + avg latency"]
    end

    TXN --> BATCH
    TXN --> STREAM
    BATCH -->|"exact 30d mean"| OFFLINE
    STREAM -->|"EWMA alpha=0.2"| ONLINE
    OFFLINE --> FAPI
    ONLINE --> FAPI
    FAPI --> UNDER
    FAPI --> INAPP
    FAPI --> DASH

    style BATCH fill:#3b82f6,color:#fff
    style STREAM fill:#f59e0b,color:#fff
    style OFFLINE fill:#10b981,color:#fff
    style ONLINE fill:#ef4444,color:#fff
```

---

## 2. Component Architecture

```mermaid
graph LR
    subgraph src_compass["src/compass/"]
        CONFIG["config.py\n── paths\n── NUM_CUSTOMERS=300\n── HISTORY_DAYS=180\n── TXN_AVG_WINDOW=30d\n── EWMA_ALPHA=0.2\n── FRESHNESS_TTL=6h"]

        DATAGEN["data_generation.py\n── generate_transactions()\n── _generate_event_days()\n── _amounts_for_day()\nPatterns: daily/weekly/sparse"]

        subgraph features_dir["features/"]
            BATCHPIPE["batch_pipeline.py\n── run_batch_pipeline()\n── compute_txn_amount_avg_30d()\n── _load_transactions()"]
            STREAMPIPE["streaming_pipeline.py\n── run_streaming_pipeline()\n── _load_running_averages()\n── _load_ordered_events()\nBATCH_SIZE=50"]
        end

        subgraph store_dir["store/"]
            OFFSTORE["offline_store.py\nclass OfflineStore\n── write_features()\n── get_features()"]
            ONSTORE["online_store.py\nclass OnlineStore\n── upsert_feature()\n── get_features()\n── get_all_features()\n── get_state()\n── set_state()"]
        end

        subgraph serving_dir["serving/"]
            FEATAPI["feature_api.py\n── get_offline_features()\n── get_online_features()"]
            QUERYCLI["query_cli.py\nCLI: query-features"]
        end

        subgraph model_dir["model/"]
            CREDIT["credit_score.py\n── score()\n── decide()\nTHRESHOLD=0.5"]
        end

        subgraph monitoring_dir["monitoring/"]
            DASHBOARD["dashboard.py\n── run_health_check()\n── _sample_customer_ids()\n── _time_call()\nSAMPLE_SIZE=50"]
        end
    end

    CONFIG --> BATCHPIPE
    CONFIG --> STREAMPIPE
    CONFIG --> DATAGEN
    BATCHPIPE --> OFFSTORE
    STREAMPIPE --> ONSTORE
    OFFSTORE --> FEATAPI
    ONSTORE --> FEATAPI
    FEATAPI --> QUERYCLI
    FEATAPI --> CREDIT
    FEATAPI --> DASHBOARD
```

---

## 3. Data Flow — Batch Path

> **Trigger**: nightly cron / manual `uv run batch-pipeline`
> **Formula**: `mean(amount)` where `event_timestamp ∈ [as_of - 30d, as_of]`

```mermaid
flowchart LR
    A["transactions.csv"] -->|read_csv| B["DataFrame\nall 32929 rows"]
    B -->|"filter: last 30d\nfrom as_of"| C["Windowed DataFrame\nper customer"]
    C -->|"groupby customer_id\n.mean()"| D["Dict: customer_id to avg"]
    D -->|for each customer| E["OfflineStore\n.write_features()"]
    E -->|INSERT OR REPLACE| F[("offline_store.db\noffline_features\nPK: customer_id, as_of, feature_name")]

    style F fill:#10b981,color:#fff
```

**Key properties of the batch path:**
- ✅ **Exact**: arithmetic mean over a precise time window
- ✅ **Reproducible**: same input → same output always
- ✅ **Point-in-time safe**: keyed by `as_of`, never overwrites history
- ❌ **Expensive**: full table scan on every run
- ❌ **Latency**: hours-old (nightly cadence)

---

## 4. Data Flow — Streaming Path

> **Trigger**: continuous / `uv run streaming-pipeline`
> **Formula**: EWMA — `α × new_amount + (1 - α) × prev_avg` where `α = 0.2`

```mermaid
flowchart LR
    A["transactions.csv"] -->|sort by timestamp| B["Ordered Events\nwith seq index"]
    B -->|"resume from\nlast_processed_seq"| C["Pending Events\nstart_seq onwards"]
    C -->|batches of 50| D{"customer_id\nin running_avg?"}
    D -->|Yes| E["EWMA Update\n0.2 x amount + 0.8 x prev"]
    D -->|No| F["Seed\nrunning_avg = amount"]
    E --> G["OnlineStore\n.upsert_feature()"]
    F --> G
    G -->|"UPDATE SET value, computed_at"| H[("online_store.db\nonline_features\nPK: customer_id, feature_name")]
    G -->|UPDATE| I[("pipeline_state\nlast_processed_seq")]

    style H fill:#ef4444,color:#fff
    style I fill:#fbbf24
```

**Key properties of the streaming path:**
- ✅ **Fast**: O(1) per event, no full table scan
- ✅ **Incremental**: resumes from last position via `pipeline_state`
- ❌ **Approximate**: EWMA is an exponential smoother, NOT a 30d window
- ❌ **Order-sensitive**: result depends on event arrival order
- ❌ **Silent staleness**: no alert if `computed_at` exceeds `FRESHNESS_TTL` (6h)

---

## 5. Database Schema (ER Diagram)

```mermaid
erDiagram
    OFFLINE_FEATURES {
        TEXT customer_id PK
        TEXT as_of PK
        TEXT feature_name PK
        REAL value
        TEXT computed_at
    }

    ONLINE_FEATURES {
        TEXT customer_id PK
        TEXT feature_name PK
        REAL value
        TEXT computed_at
    }

    PIPELINE_STATE {
        TEXT key PK
        TEXT value
    }

    OFFLINE_FEATURES ||--o{ ONLINE_FEATURES : "same customer_id - no FK"
    ONLINE_FEATURES ||--|| PIPELINE_STATE : "state: last_processed_seq"
```

**Schema observations relevant to the business case:**

| Property | Offline Store | Online Store |
|---|---|---|
| **File** | `offline_store.db` | `online_store.db` |
| **Table** | `offline_features` | `online_features` |
| **Primary Key** | `(customer_id, as_of, feature_name)` | `(customer_id, feature_name)` |
| **History** | ✅ Keeps all `as_of` snapshots | ❌ One row per customer (overwritten) |
| **Staleness detection** | N/A — batch is always fresh | ❌ No TTL enforcement in schema |
| **Formula** | Exact 30d mean | EWMA α=0.2 |
| **`FRESHNESS_TTL`** | Not applicable | Defined in config (6h), **not enforced** |

---

## 6. UML Class & Module Diagram

```mermaid
classDiagram
    class OfflineStore {
        +db_path: Path
        -_conn: sqlite3.Connection
        +__init__(db_path: Path)
        +write_features(customer_id, as_of, features, computed_at)
        +get_features(customer_id, as_of) dict
        +close()
    }

    class OnlineStore {
        +db_path: Path
        -_conn: sqlite3.Connection
        +__init__(db_path: Path)
        +upsert_feature(customer_id, feature_name, value, computed_at)
        +get_features(customer_id) dict
        +get_all_features() dict
        +get_state(key) str
        +set_state(key, value)
        +close()
    }

    class BatchPipeline {
        <<module>>
        +run_batch_pipeline(as_of, store) OfflineStore
        +compute_txn_amount_avg_30d(df, as_of) dict
        -_load_transactions() DataFrame
    }

    class StreamingPipeline {
        <<module>>
        +run_streaming_pipeline(store, stop_after_seconds) int
        -_load_ordered_events() DataFrame
        -_load_running_averages(store) dict
        EWMA_ALPHA = 0.2
        _BATCH_SIZE = 50
    }

    class FeatureAPI {
        <<module>>
        +get_offline_features(customer_id, as_of) dict
        +get_online_features(customer_id) dict
    }

    class CreditScore {
        <<module>>
        +score(features) float
        +decide(features) dict
        DECISION_THRESHOLD = 0.5
        _TXN_AMOUNT_AVG_WEIGHT = -0.01
    }

    class Dashboard {
        <<module>>
        +run_health_check() dict
        -_sample_customer_ids(limit) list
        -_time_call(fn, args) tuple
        SAMPLE_SIZE = 50
    }

    BatchPipeline --> OfflineStore : writes to
    StreamingPipeline --> OnlineStore : writes to
    FeatureAPI --> OfflineStore : reads from
    FeatureAPI --> OnlineStore : reads from
    CreditScore ..> FeatureAPI : consumes features
    Dashboard ..> FeatureAPI : probes health
```

---

## 7. Execution Sequence — End-to-End

```mermaid
sequenceDiagram
    actor User
    participant DG as data_generation
    participant BP as batch_pipeline
    participant SP as streaming_pipeline
    participant OFFSTORE as OfflineStore
    participant ONSTORE as OnlineStore
    participant API as feature_api
    participant CS as credit_score
    participant MON as dashboard

    User->>DG: "uv run generate-data"
    DG->>DG: "generate_transactions(300 customers, 180 days)"
    DG-->>User: "transactions.csv (32,929 rows)"

    User->>BP: "uv run batch-pipeline"
    BP->>BP: "_load_transactions()"
    BP->>BP: "compute_txn_amount_avg_30d(df, as_of)"
    Note over BP: EXACT mean over last 30d window
    BP->>OFFSTORE: "write_features(customer_id, as_of, features)"
    OFFSTORE-->>User: "offline_store.db updated"

    User->>SP: "uv run streaming-pipeline"
    SP->>ONSTORE: "get_state('last_processed_seq')"
    ONSTORE-->>SP: "last_seq (resume point)"
    SP->>SP: "_load_ordered_events()"
    loop For each event batch
        SP->>SP: "EWMA: 0.2 x amount + 0.8 x prev"
        SP->>ONSTORE: "upsert_feature(customer_id, feature_name, value)"
        SP->>ONSTORE: "set_state('last_processed_seq', seq)"
    end
    ONSTORE-->>User: "online_store.db updated"

    User->>API: "uv run query-features --customer-id C0001"
    API->>OFFSTORE: "get_features(C0001, as_of=now)"
    OFFSTORE-->>API: "txn_amount_avg_30d = 142.4050"
    API->>ONSTORE: "get_features(C0001)"
    ONSTORE-->>API: "txn_amount_avg_30d = 76.8235"
    API-->>User: "DIVERGENCE VISIBLE: offline=142.4 vs online=76.8"

    User->>MON: "uv run monitor"
    MON->>MON: "_sample_customer_ids(50)"
    MON->>API: "get_offline_features() x 50 customers"
    MON->>API: "get_online_features() x 50 customers"
    MON-->>User: "uptime=100%, avg_latency=Xms"
    Note over User,MON: No staleness check! No feature drift alert!
```

---

## 8. The Core Risk: Training/Serving Skew

> This is the central problem the Business Case asks you to identify and solve.

```mermaid
graph TB
    subgraph Training["Model Training - Underwriting"]
        T1["Training Data Source:\nOffline Store"]
        T2["Feature: txn_amount_avg_30d\n= EXACT arithmetic mean\nover rolling 30d window"]
        T3["Model learns weights\nagainst EXACT mean values"]
        T1 --> T2 --> T3
    end

    subgraph Serving["Model Serving - In-App Real-Time"]
        S1["Serving Data Source:\nOnline Store"]
        S2["Feature: txn_amount_avg_30d\n= EWMA with alpha=0.2\nexponential smoother"]
        S3["Model receives EWMA values\na DIFFERENT distribution!"]
        S1 --> S2 --> S3
    end

    subgraph Impact["Impact"]
        I1["Training/Serving Skew\nhidden covariate shift"]
        I2["Model scores are miscalibrated\nfor real-time decisions"]
        I3["Automatic credit top-ups\napproved or denied incorrectly"]
    end

    T3 -->|"SKEW"| I1
    S3 -->|"SKEW"| I1
    I1 --> I2 --> I3

    style I1 fill:#ef4444,color:#fff
    style I2 fill:#f59e0b,color:#fff
    style I3 fill:#dc2626,color:#fff
```

### Quantifying the skew (from our run)

| Metric | Offline (Batch) | Online (Streaming) | Delta |
|---|---|---|---|
| `txn_amount_avg_30d` for C0001 | **142.41** | **76.82** | **~46% lower** |
| Formula | Exact 30d mean | EWMA α=0.2 | Different math |
| Freshness guarantee | Computed at run time | May be stale (6h TTL not enforced) | No enforcement |

### Why EWMA ≠ 30d mean

```
Exact 30d mean:  sum(all amounts in window) / count
EWMA:            α × latest_amount + (1-α) × previous_EWMA
                 where α = 0.2 (heavy discount on old values)

EWMA is "recency-biased" — it decays historical values exponentially.
For customers with high recent spending but lower historical amounts,
EWMA will be HIGHER than the exact mean.
For customers with declining activity, EWMA will be LOWER.
Neither is wrong in isolation — but using one for training and the
other for serving is a fundamental model governance violation.
```

---

## 9. Risk Mapping to Business Case

### Incident Analysis

| Incident (from PDF) | Root Cause in Code | File | Risk Category |
|---|---|---|---|
| Customer complaint: credit limit based on "outdated transaction pattern" | `FRESHNESS_TTL = 6h` defined in `config.py` but never enforced at read time | `online_store.py`, `config.py` | **Silent Staleness** |
| Credit line adjustments inconsistent with underwriting score | Batch uses `mean()`, streaming uses EWMA | `batch_pipeline.py` L28 vs `streaming_pipeline.py` L72 | **Training/Serving Skew** |
| Monitoring shows 100% uptime but model quality degraded | `dashboard.py` measures only API latency and uptime — no feature drift, no staleness alert | `monitoring/dashboard.py` | **Blind Monitoring** |

### Business Case Questions → Code Evidence

**Q1: What are the risks of removing human review from the credit top-up flow?**

1. `feature_api.py` has NO staleness guard on `get_online_features()`. A customer with no transactions in 3 months returns a feature value — just stale.
2. `credit_score.py decide()` has no confidence bounds or data freshness check. It scores with stale `76.82` as happily as fresh `142.41`.
3. `monitoring/dashboard.py` measures API health (latency/uptime) but NEVER checks if online store data is fresh. A model on 7-day-old features shows 100% uptime.

---

## 10. Proposed Solution Architecture

```mermaid
graph TB
    subgraph FIX1["Fix 1: Unify Feature Logic"]
        F1A["Extract shared compute_txn_amount_avg_30d()"]
        F1B["Streaming approximates window\nwith a sliding buffer"]
        F1C["OR: document EWMA explicitly\nand train the online model on EWMA features"]
        F1A --> F1B
        F1A --> F1C
    end

    subgraph FIX2["Fix 2: Enforce Freshness TTL"]
        F2A["OnlineStore.get_features() checks computed_at"]
        F2B["If age > FRESHNESS_TTL then raise StaleFeatureError"]
        F2C["feature_api falls back to offline store\nor rejects the request"]
        F2A --> F2B --> F2C
    end

    subgraph FIX3["Fix 3: Monitoring with Drift Detection"]
        F3A["Add feature_drift_monitor()"]
        F3B["Compare online vs offline feature distributions"]
        F3C["Alert if mean delta > threshold (e.g. 15%)"]
        F3D["Track P50/P95 of feature staleness age"]
        F3A --> F3B --> F3C
        F3A --> F3D
    end

    subgraph FIX4["Fix 4: Human Review Gate"]
        F4A["Automated approval only if:\nfeature_age < 1h\nonline vs offline delta < 15%\ncustomer has > 30 days history"]
        F4B["Otherwise queue for human review"]
        F4A --> F4B
    end

    FIX1 --> FIX2 --> FIX3 --> FIX4
```

### Before vs After

```mermaid
graph LR
    subgraph BEFORE["Current State"]
        B1["Batch: exact mean\nOnline: EWMA\ndifferent formulas"]
        B2["Staleness: config 6h\nbut never checked"]
        B3["Monitoring: API uptime only\nno feature quality"]
        B4["Decision: automatic\nno guards"]
    end

    subgraph AFTER["Target State"]
        A1["Unified feature definition\nSame math, same result"]
        A2["TTL enforced at read time\nStaleFeatureError on expiry"]
        A3["Drift monitor: online vs offline\nStaleness P95 alerting"]
        A4["Auto-approval only if\nfreshness + consistency pass"]
    end

    B1 -->|"Fix 1"| A1
    B2 -->|"Fix 2"| A2
    B3 -->|"Fix 3"| A3
    B4 -->|"Fix 4"| A4
```

---

## How to Run Analyses

```bash
# 1. Full pipeline setup
uv run generate-data
uv run batch-pipeline
uv run streaming-pipeline

# 2. Query and compare feature values (the skew is visible here)
uv run query-features --customer-id C0001

# 3. Health check (shows uptime/latency - does NOT check staleness)
uv run monitor

# 4. Run tests
uv run pytest -v

# 5. Explore raw data directly
sqlite3 data/raw/offline_store.db "SELECT * FROM offline_features WHERE customer_id='C0001';"
sqlite3 data/raw/online_store.db "SELECT * FROM online_features WHERE customer_id='C0001';"
sqlite3 data/raw/online_store.db "SELECT * FROM pipeline_state;"
```

---

*Generated for the Solara Digital Bank / Compass Feature Pipeline Business Case Analysis.*
