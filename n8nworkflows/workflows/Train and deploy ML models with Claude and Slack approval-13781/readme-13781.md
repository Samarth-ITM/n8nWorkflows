Train and deploy ML models with Claude and Slack approval

https://n8nworkflows.xyz/workflows/train-and-deploy-ml-models-with-claude-and-slack-approval-13781


# Train and deploy ML models with Claude and Slack approval

This document provides a technical breakdown of the **Train and Deploy ML Models with Claude and Slack Approval** workflow. This system automates an end-to-end MLOps pipeline, from initial strategy planning to model training and human-in-the-loop (HITL) deployment authorization.

---

### 1. Workflow Overview

This workflow automates the lifecycle of a machine learning project. It is triggered via a Webhook containing a dataset URL and a business goal. The process is divided into five logical phases:

1.  **P1 — Orchestration & Strategy:** Claude AI (Sonnet) analyzes the request and generates a JSON-based execution plan.
2.  **P2 — Data Engineering:** The system fetches the raw CSV, performs data cleaning, handles missing values, and encodes categorical variables.
3.  **P3 — Feature Engineering:** Claude AI (Haiku) approves specific features, which are then calculated (e.g., family size, social titles).
4.  **P4 — Training & Evaluation:** Three models (Logistic Regression, Random Forest, XGBoost) are trained from scratch using JavaScript. Claude AI (Sonnet) then acts as a judge to pick the best model based on F1-score and accuracy.
5.  **P5 — HITL Deployment:** Claude AI generates a technical `MODEL_CARD.md`. A Slack message is sent to a human operator to approve or reject the deployment.

---

### 2. Block-by-Block Analysis

#### 2.1 Phase 1: Orchestration & Strategy
Defines the roadmap for the ML task.
*   **Nodes Involved:** `P1: Receive MLOps Job`, `P1: Plan ML Strategy`, `P1: Anthropic Sonnet`, `P1: Parse Strategy`, `P1: Log Strategy`.
*   **Node Details:**
    *   **Webhook:** Listens for POST requests with `dataset_url`, `target_variable`, and `business_goal`.
    *   **Chain LLM (Sonnet):** Uses a Lead Data Scientist prompt to output a structured JSON plan.
    *   **Code (Parse):** Regex-based extraction to ensure the LLM output is valid JSON; includes fallback defaults.
    *   **HTTP Request (Log):** Sends the strategy metadata to a Supabase audit table.

#### 2.2 Phase 2: Data Engineering
Fetches and prepares the raw data for consumption.
*   **Nodes Involved:** `P2: Fetch CSV`, `P2: Clean Data`, `P2: Log Data Eng`.
*   **Node Details:**
    *   **HTTP Request (Fetch):** Downloads the CSV from the provided URL.
    *   **Code (Cleaning):** A robust JS parser that handles quoted fields, drops rows with missing targets, converts numeric fields, and imputes the median age.
    *   **Edge Cases:** Handles malformed CSV lines and missing target values.

#### 2.3 Phase 3: Feature Engineering
Enhances the dataset with derived features.
*   **Nodes Involved:** `P3: Reason About Features`, `P3: Anthropic Haiku`, `P3: Parse Feature Plan`, `P3: Engineer Features`, `P3: Log Feature Eng`.
*   **Node Details:**
    *   **Chain LLM (Haiku):** Uses a lighter model to validate if the suggested features (e.g., `FamilySize`) are appropriate for the dataset.
    *   **Code (Engineering):** Implements logic to extract titles from names (Mr/Mrs/Dr) and calculates family metrics.

#### 2.4 Phase 4: Training & Evaluation
The core computational block where models are built and compared.
*   **Nodes Involved:** `P4: Setup Algorithms`, `P4: Train All Models`, `P4: LLM Judge Best Model`, `P4: Anthropic Sonnet Judge`, `P4: Parse Judge Verdict`, `P4: Log Training`.
*   **Node Details:**
    *   **Setup Algorithms:** Performs an 80/20 train/test split and prepares the feature matrices (X) and labels (y).
    *   **Train All Models (Code):** Implements manual Gradient Descent for Logistic Regression, Bagged Decision Stumps for Random Forest, and Gradient Boosting for XGBoost—all in pure JavaScript.
    *   **Chain LLM (Sonnet Judge):** Evaluates the resulting metrics (Accuracy, Precision, Recall, F1) against the business goal to select a "Winner."

#### 2.5 Phase 5: HITL Deployment
Finalizes documentation and seeks human approval.
*   **Nodes Involved:** `P5: Generate Model Card`, `P5: Anthropic Sonnet Card`, `P5: Assemble Deployment Payload`, `P5: Send Slack Approval`, `P5: Log Pipeline Complete`.
*   **Node Details:**
    *   **Chain LLM (Sonnet Card):** Generates a professional `MODEL_CARD.md` including sections on limitations and intended use.
    *   **Slack Node:** Posts a formatted message to a specific channel with the model's performance and a request for a "✅" or "❌" reaction.
    *   **Sub-workflow Note:** The Slack node is the current end-point; a production version requires an additional Webhook listener to handle the Slack interaction callback.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| P1: Receive MLOps Job | Webhook | Entry Point | - | P1: Plan ML Strategy | Phase 1 — Orchestration & Strategy |
| P1: Plan ML Strategy | Chain LLM | Strategy Design | P1: Receive MLOps Job | P1: Parse Strategy | Phase 1 — Orchestration & Strategy |
| P1: Anthropic Sonnet | Anthropic Chat | AI Brain (P1) | - | P1: Plan ML Strategy | Phase 1 — Orchestration & Strategy |
| P1: Parse Strategy | Code | JSON Sanitization | P1: Plan ML Strategy | P1: Log Strategy, P2: Fetch CSV | Phase 1 — Orchestration & Strategy |
| P1: Log Strategy | HTTP Request | Audit Logging | P1: Parse Strategy | - | Phase 1 — Orchestration & Strategy |
| P2: Fetch CSV | HTTP Request | Data Acquisition | P1: Parse Strategy | P2: Clean Data | Phase 2 — Data Engineering |
| P2: Clean Data | Code | Data Preprocessing | P2: Fetch CSV | P2: Log Data Eng, P3: Reason About Features | Phase 2 — Data Engineering |
| P2: Log Data Eng | HTTP Request | Audit Logging | P2: Clean Data | - | Phase 2 — Data Engineering |
| P3: Reason About Features | Chain LLM | Feature Logic | P2: Clean Data | P3: Parse Feature Plan | Phase 3 — Feature Engineering |
| P3: Anthropic Haiku | Anthropic Chat | AI Brain (P3) | - | P3: Reason About Features | Phase 3 — Feature Engineering |
| P3: Parse Feature Plan | Code | Schema Validation | P3: Reason About Features | P3: Engineer Features | Phase 3 — Feature Engineering |
| P3: Engineer Features | Code | Feature Calc | P3: Parse Feature Plan | P3: Log Feature Eng, P4: Setup Algorithms | Phase 3 — Feature Engineering |
| P3: Log Feature Eng | HTTP Request | Audit Logging | P3: Engineer Features | - | Phase 3 — Feature Engineering |
| P4: Setup Algorithms | Code | Train/Test Split | P3: Engineer Features | P4: Train All Models | Phase 4 — Training & Evaluation |
| P4: Train All Models | Code | Training Engine | P4: Setup Algorithms | P4: LLM Judge Best Model | Phase 4 — Training & Evaluation |
| P4: LLM Judge Best Model | Chain LLM | Model Selection | P4: Train All Models | P4: Parse Judge Verdict | Phase 4 — Training & Evaluation |
| P4: Anthropic Sonnet Judge | Anthropic Chat | AI Brain (P4) | - | P4: LLM Judge Best Model | Phase 4 — Training & Evaluation |
| P4: Parse Judge Verdict | Code | Result Parsing | P4: LLM Judge Best Model | P4: Log Training, P5: Generate Model Card | Phase 4 — Training & Evaluation |
| P4: Log Training | HTTP Request | Audit Logging | P4: Parse Judge Verdict | - | Phase 4 — Training & Evaluation |
| P5: Generate Model Card | Chain LLM | Documentation | P4: Parse Judge Verdict | P5: Assemble Deployment Payload | Phase 5 — HITL Deployment |
| P5: Anthropic Sonnet Card | Anthropic Chat | AI Brain (P5) | - | P5: Generate Model Card | Phase 5 — HITL Deployment |
| P5: Assemble Deployment Payload | Code | Base64 Encoding | P5: Generate Model Card | P5: Send Slack Approval, P5: Log Pipeline | Phase 5 — HITL Deployment |
| P5: Send Slack Approval | Slack | Human Interface | P5: Assemble Deployment Payload | - | Phase 5 — HITL Deployment |
| P5: Log Pipeline Complete | HTTP Request | Audit Logging | P5: Assemble Deployment Payload | - | Phase 5 — HITL Deployment |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Webhook Node** (POST). Set the path to `mlops-v2`.
2.  **Strategy Phase:**
    *   Add a **Basic LLM Chain** node. Connect an **Anthropic Chat Model** node (select `claude-3-5-sonnet`).
    *   Configure the prompt to request a JSON ML plan.
    *   Add a **Code Node** to parse the LLM text output into a JSON object using `JSON.parse()`.
3.  **Data Phase:**
    *   Add an **HTTP Request** node to GET the `dataset_url` from the P1 strategy.
    *   Add a **Code Node** for cleaning. You must include a CSV parser function and logic to impute missing values (like `Age = 29` for the Titanic dataset) and encode strings (Male/Female to 0/1).
4.  **Engineering Phase:**
    *   Add another **LLM Chain** with `claude-3-haiku` to approve features.
    *   Add a **Code Node** to compute `FamilySize` (`SibSp + Parch + 1`) and extract titles from the `Name` column.
5.  **Training Phase:**
    *   Add a **Code Node** to split data. Calculate a `splitIdx` at 80% of the array length.
    *   Add the "Training Engine" **Code Node**. This node requires JS functions for `sigmoid`, `dot product`, and the specific training loops for Logistic Regression and Gradient Boosting.
    *   Add an **LLM Chain** (Sonnet) to act as the `Judge`. Pass the performance metrics of all three models.
6.  **Deployment Phase:**
    *   Add an **LLM Chain** (Sonnet) to generate the `MODEL_CARD.md`.
    *   Add a **Slack Node**. Configure it to send a message to your chosen `channelId`. Use expressions to include the `winner`, `f1_score`, and `justification`.
7.  **Logging (Optional):** At the end of every phase, add an **HTTP Request** node pointing to your Supabase/database URL to track `run_id`, `phase`, and `status`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Supabase Audit Schema** | Create a table `mlops_audit_log` with columns: `run_id`, `workflow_step`, `phase`, `status`. |
| **Anthropic Models** | Uses Sonnet for complex reasoning (P1, P4, P5) and Haiku for speed/cost (P3). |
| **Slack Permissions** | Bot needs `chat:write` scope and must be invited to the channel via `/invite @botname`. |
| **CSV Parsing** | Built-in JS code handles commas within quotes (e.g., "Braund, Mr. Owen Harris"). |
| **Tested Dataset** | Optimized for the Titanic Survival dataset (Binary Classification). |