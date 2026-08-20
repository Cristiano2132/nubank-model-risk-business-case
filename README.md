# Compass Model Risk Architecture & KRI Scorecards

## 1. Overview: Executive KRI Architecture

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Inter, sans-serif', 'fontSize': '13px'}, 'flowchart': {'nodeSpacing': 35, 'rankSpacing': 10, 'padding': 4, 'subGraphPadding': 6}}}%%
flowchart LR
    ROOT["🎯 <b>Compass Risk Score</b><br>2LoD Multi-Dimension<br>Evaluation"]

    ROOT -->|"60%"| QUANTI_HDR
    ROOT -->|"40%"| QUALI_HDR

    subgraph DIM_QUANTI ["Empirical Evidence (Live Testing)"]
        direction TB
        QUANTI_HDR["📊 <b>Quantitative Dimension</b> (60% Weight)"]
        K1["<b>KRI 1: Feature Parity Skew</b><br>Divergence between batch (training) and streaming (serving) calculations"]
        K2["<b>KRI 2: Population Drift (KS)</b><br>Statistical distribution shift between offline baseline and online requests"]
        K3["<b>KRI 3: Silent Staleness</b><br>Frequency of expired customer profiles served beyond the 6h TTL"]
        
        QUANTI_HDR -->|"35%"| K1
        QUANTI_HDR -->|"35%"| K2
        QUANTI_HDR -->|"30%"| K3
    end

    subgraph DIM_QUALI ["Operational Maturity & Safeguards"]
        direction TB
        QUALI_HDR["🛡️ <b>Qualitative Dimension</b> (40% Weight)"]
        K4["<b>KRI 4: Feature Contracts</b><br>Standardized metadata coverage: schema, SLA, owner, and valid bounds"]
        K5["<b>KRI 5: CI/CD Validation Gates</b><br>Automated pull-request checks for parity, schema changes, and drift"]
        K6["<b>KRI 6: Confidence Envelope</b><br>Real-time circuit breakers triggering human fallback on abnormal features"]
        K7["<b>KRI 7: Semantic Observability</b><br>Monitoring data health and distribution drift alongside uptime and latency"]
        
        QUALI_HDR -->|"25%"| K4
        QUALI_HDR -->|"25%"| K5
        QUALI_HDR -->|"25%"| K6
        QUALI_HDR -->|"25%"| K7
    end

    classDef rootStyle fill:#2d0a4e,stroke:#a855f7,stroke-width:2px,color:#ffffff;
    classDef headerQuanti fill:#1e3a8a,stroke:#3b82f6,stroke-width:2px,color:#ffffff;
    classDef headerQuali fill:#4c1d95,stroke:#8b5cf6,stroke-width:2px,color:#ffffff;
    classDef quantiStyle fill:#0f2942,stroke:#3b82f6,stroke-width:1.5px,color:#ffffff;
    classDef qualiStyle fill:#2e1065,stroke:#8b5cf6,stroke-width:1.5px,color:#ffffff;

    class ROOT rootStyle;
    class QUANTI_HDR headerQuanti;
    class QUALI_HDR headerQuali;
    class K1,K2,K3 quantiStyle;
    class K4,K5,K6,K7 qualiStyle;
```

---

## 2. Quantitative Dimension Deep Dive (Empirical Data Health)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Inter, sans-serif', 'fontSize': '13px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 12, 'padding': 6}}}%%
flowchart LR
    Q_ROOT["📊 <b>Quantitative Score</b><br>Overall Weight: <b>60%</b><br>Score: <b style='color:#ef4444;'>0.0 / 100</b> (Critical)"]

    Q_ROOT -->|"35% Weight"| Q1
    Q_ROOT -->|"35% Weight"| Q2
    Q_ROOT -->|"30% Weight"| Q3

    subgraph CARDS ["Empirical Tests & Scoring Rules"]
        direction TB
        Q1["<b>KRI 1: Feature Parity (Training/Serving Skew)</b><br>• <b>Measurement:</b> Mean Absolute % Delta between Batch & Streaming<br>• <b>Scoring Rule:</b> &lt;5% = 100 pts | 5-15% = 50 pts | &gt;15% = 0 pts<br>• <b>Live Test Result:</b> 29.9% Delta (Exceeds tolerance) ➔ <b style='color:#ef4444;'>0 pts</b>"]
        
        Q2["<b>KRI 2: Statistical Drift (KS Statistic)</b><br>• <b>Measurement:</b> Max ECDF distance between Offline & Online distributions<br>• <b>Scoring Rule:</b> &lt;1% = 100 pts | &lt;5% = 80 pts | &lt;7% = 50 pts | &ge;7% = 0 pts<br>• <b>Live Test Result:</b> KS = 7.1% (Severe distribution shift) ➔ <b style='color:#ef4444;'>0 pts</b>"]
        
        Q3["<b>KRI 3: Silent Staleness</b><br>• <b>Measurement:</b> % of requests serving feature age exceeding 6h TTL<br>• <b>Scoring Rule:</b> 0% = 100 pts | 0.1-5% = 50 pts | &gt;5% = 0 pts<br>• <b>Live Test Result:</b> 4.3% stale + no TTL validation in code ➔ <b style='color:#f59e0b;'>50 pts</b>"]
    end

    classDef qRoot fill:#1e3a8a,stroke:#3b82f6,stroke-width:2px,color:#ffffff;
    classDef qCard fill:#0f2942,stroke:#3b82f6,stroke-width:1.5px,color:#ffffff;

    class Q_ROOT qRoot;
    class Q1,Q2,Q3 qCard;
```

---

## 3. Qualitative Dimension Deep Dive (Governance & Controls)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Inter, sans-serif', 'fontSize': '13px'}, 'flowchart': {'nodeSpacing': 30, 'rankSpacing': 12, 'padding': 6}}}%%
flowchart LR
    C_ROOT["🛡️ <b>Qualitative Score</b><br>Overall Weight: <b>40%</b><br>Score: <b style='color:#f59e0b;'>25.0 / 100</b> (High Risk)"]

    C_ROOT -->|"25% Weight"| C1
    C_ROOT -->|"25% Weight"| C2
    C_ROOT -->|"25% Weight"| C3
    C_ROOT -->|"25% Weight"| C4

    subgraph GOV_CARDS ["Control Assessments & Scoring Rules"]
        direction TB
        C1["<b>KRI 4: Feature Contract Completeness</b><br>• <b>Measurement:</b> Mandatory tags (name, formula, owner, version, bounds, SLA)<br>• <b>Scoring Rule:</b> (Implemented Tags / 6) × 100<br>• <b>Audit Status:</b> 2 of 6 tags present (name, formula only) ➔ <b style='color:#f59e0b;'>33.3 pts</b>"]
        
        C2["<b>KRI 5: CI/CD Validation Gates</b><br>• <b>Measurement:</b> Automated gates (Parity, KS check, Schema, Shadow deploy)<br>• <b>Scoring Rule:</b> (Active Automated Gates / 4) × 100<br>• <b>Audit Status:</b> 0 of 4 active (100% manual 5-day review) ➔ <b style='color:#ef4444;'>0.0 pts</b>"]
        
        C3["<b>KRI 6: Confidence Envelope Safeguards</b><br>• <b>Measurement:</b> Active circuit breakers (age &lt; TTL, |delta| &lt; limit, history &ge; 30d)<br>• <b>Scoring Rule:</b> (Implemented Rules / 3) × 100<br>• <b>Audit Status:</b> 0 of 3 implemented (unguarded scoring) ➔ <b style='color:#ef4444;'>0.0 pts</b>"]
        
        C4["<b>KRI 7: Semantic Observability</b><br>• <b>Measurement:</b> Monitored pillars (Latency, Uptime availability, Drift)<br>• <b>Scoring Rule:</b> (Monitored Pillars / 3) × 100<br>• <b>Audit Status:</b> 2 of 3 active (Latency & Uptime only; blind to Drift) ➔ <b style='color:#f59e0b;'>66.7 pts</b>"]
    end

    classDef cRoot fill:#4c1d95,stroke:#8b5cf6,stroke-width:2px,color:#ffffff;
    classDef cCard fill:#2e1065,stroke:#8b5cf6,stroke-width:1.5px,color:#ffffff;

    class C_ROOT cRoot;
    class C1,C2,C3,C4 cCard;
```