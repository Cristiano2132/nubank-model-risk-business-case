# Compass Feature Pipeline — Risk Assessment & Governance Framework

> **Role:** Model Risk Specialist — ML/AI Data & Infrastructure Risks (IC5)
> **Candidate:** Cristiano Oliveira

This repository contains the final deliverable for the Solara Digital Bank business case, focusing on the infrastructure and data risk assessment of the **Compass** feature store and serving pipeline.

## Deliverable Structure

As requested by the case guidelines, the analysis and framework design are divided into four main areas. All technical evidence, formal testing, and governance architecture are documented and executed within the main notebook.

### 1. Risk Landscape & Assessment Approach
*Context: Question 1 — Infrastructure & Data Risk Assessment (hands-on)*

The notebook contains a deep dive into the live system, simulating the batch and streaming pathways to uncover hidden regressions. We've mapped the formal tests to specific risks (Training/Serving Skew, Silent Staleness, Drift, etc.). 

### 2. Position on Proposed Changes
*Context: Q1.3*

Based on the quantitative findings, I present a clear, independent view on the two proposed changes (real-time credit-line top-ups with no human review, and lightening the feature code review process). 

### 3. Governance Foundations
*Context: Q1.4*

A proposal of the most critical controls and monitoring mechanisms that Solara must implement to safely operate Compass (e.g., Confidence Envelopes, Feature Contracts, and Semantic Observability).

### 4. Governance Framework Design (Org-Wide)
*Context: Question 2*

A broader strategy for a Data Mesh environment, detailing what to standardize centrally (Model Risk guards) vs. what to leave to domain autonomy, how to implement KRIs without noise, and how to scale this across teams with varying maturity levels.

---

## Tools & Technologies Used

To deliver a reproducible and data-driven analysis, the following stack was utilized in this case:

- **Python (Pandas, NumPy):** Used to simulate production feature data, compute KS statistics (drift), measure parity (training/serving skew), and build the scoring engine.
- **Jupyter Notebook:** Acts as the primary medium to seamlessly integrate the executive narrative with the executable Python code and empirical evidence.
- **Matplotlib & Seaborn:** Used to dynamically plot the PMBOK-style Risk Matrix and the framework architecture directly from the KRI outputs.
- **Git:** Version control of the repository and scripts.
- **Antigravity AI Assistant:** Leveraged as a pair-programming partner to assist with Python refactoring, logic calibration (e.g. dynamic risk matrices), and markdown documentation translation/structuring.


## Instructions & Local Reproduction

To execute the notebook and reproduce all empirical tests and scorecards:

### 1. Clone the Compass Feature Pipeline (Prerequisite)
The data and backend pipeline are hosted in the Compass repository:
```bash
# Clone the Compass pipeline repository locally
git clone https://github.com/lennon-cruz/compass-feature-pipeline.git ~/Documents/compass-feature-pipeline
```
*(Ensure the Compass data is populated in `compass-feature-pipeline/data/` with `offline_store.db`, `online_store.db`, and `transactions.csv`)*.

### 2. Configure Path & Run the Notebook
1. **Open the Notebook:** `business_case_answers.ipynb`
2. In **Cell 2 (Setup)**, verify that `DATA_DIR` and `COMPASS_SRC` point to your local Compass repo path (by default, it searches `~/Documents/compass-feature-pipeline`).
3. Run all cells (`Run All`) to dynamically execute the quantitative tests, compute KRIs, and render the visual PMBOK Risk Matrix.

### 3. Additional Architecture Baseline
- **`ARCHITECTURE.md`**: Contains UML diagrams, data flows, and schema analysis used as the baseline for this risk assessment.