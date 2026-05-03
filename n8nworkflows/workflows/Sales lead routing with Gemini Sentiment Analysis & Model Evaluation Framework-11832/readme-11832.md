Sales lead routing with Gemini Sentiment Analysis & Model Evaluation Framework

https://n8nworkflows.xyz/workflows/sales-lead-routing-with-gemini-sentiment-analysis---model-evaluation-framework-11832


# Sales lead routing with Gemini Sentiment Analysis & Model Evaluation Framework

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sales lead routing with Gemini Sentiment Analysis & Model Evaluation Framework  
**Workflow name (in JSON):** Smart sales lead routing with sentiment analysis

This workflow routes incoming sales leads (emails) based on **AI sentiment categorization** and includes a built-in **model evaluation framework** using n8n’s Evaluation Trigger + Data Table (“Golden Dataset”). It has two execution entry points:

### 1.1 Production ingestion (Gmail polling)
- Polls Gmail for new emails.
- Runs **Sentiment Analysis** on the email body text.
- Routes to one of three notification emails (Hot Lead / Follow-up / Marketing Insights).

### 1.2 Evaluation framework (Golden dataset + metrics)
- Pulls rows from an n8n Data Table dataset.
- Runs the same Sentiment Analysis on each row.
- Saves the model output back to the dataset and records categorization metrics (actual vs expected).
- Uses “Safety Gate” nodes to prevent sending real emails during evaluation runs.

### 1.3 AI model selection (Gemini models)
- Provides multiple Google Gemini chat model nodes to test different models.
- In the current wiring, **Gemini 2.5 Flash Lite** is connected as the language model for the Sentiment Analysis node; the other two are present but not connected.

---

## 2. Block-by-Block Analysis

### Block A — Production Input Reception (Gmail)
**Overview:** Polls Gmail every minute and emits email items into the workflow for classification.  
**Nodes involved:** `Gmail Trigger`

#### Node: Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — entry point that polls Gmail.
- **Key configuration:**
  - Poll schedule: **everyMinute**
  - `simple: false` (emits richer message structure than “simple” mode).
  - Filters not configured (empty object) → will poll broadly depending on node defaults/account settings.
- **Outputs / connections:**
  - Main output → `Sentiment Analysis`
- **Credentials:** Gmail OAuth2 (named `gmailOAuth2 Credential`)
- **Potential failures / edge cases:**
  - OAuth token expiration / insufficient Gmail scopes.
  - High volume mailbox → rate limits or delayed polling.
  - Message payload fields may differ by GmailTrigger version; downstream expressions rely on `item.json.text`.

---

### Block B — Core AI Sentiment Categorization
**Overview:** Extracts text from the incoming item and classifies it into exactly one of Positive/Neutral/Negative, producing detailed structured results.  
**Nodes involved:** `Sentiment Analysis`

#### Node: Sentiment Analysis
- **Type / role:** `@n8n/n8n-nodes-langchain.sentimentAnalysis` — sentiment classification using an attached chat model.
- **Key configuration choices:**
  - **Categories:** `Positive, Neutral, Negative`
  - **System prompt:** instructs strict single-category output and adds business intent logic:
    - Must pick exactly one category.
    - Evaluate sentiment toward “OUR company”.
    - Ignore negativity toward competitors; example implies such a case should be treated as a high-value lead, but the **actual categories remain only Positive/Neutral/Negative** (important mismatch risk—see edge cases).
  - `includeDetailedResults: true` (expects structured output including e.g., `sentimentAnalysis.category` plus detail fields)
  - `enableAutoFixing: false` (no automatic repair if output formatting is off)
  - Input text expression:
    - `={{ $json.text || $json.input }}`
- **Model wiring (AI input):**
  - Receives a language model via **AI connection**.
  - In this workflow, `Google Gemini 2.5 Flash Lite` is connected to its `ai_languageModel` input.
- **Outputs / connections:**
  - Main output branches into three parallel paths:
    - Output 0 → `Check Positive`
    - Output 1 → `Check Neutral`
    - Output 2 → `Check Negative`
  - (These are not “IF” branches; they are three concurrent outputs feeding “Safety Gate” checks.)
- **Execution settings:**
  - `executeOnce: true` (per workflow execution; in evaluation loop this can have implications if multiple items are processed—see note below)
  - `retryOnFail: true`
- **Potential failures / edge cases:**
  - **Category mismatch in prompt:** Prompt mentions “High-Value Lead” but categories are limited to Positive/Neutral/Negative. The model might try to output “High-Value Lead” and fail formatting requirements.
  - Input field variability: if neither `$json.text` nor `$json.input` exists, input becomes `undefined` → model output may be inconsistent or node may error.
  - `executeOnce: true` can cause only the first item in a multi-item run to be processed depending on node behavior/version; this is risky in evaluation mode where multiple dataset rows are expected.
  - Gemini API quota/latency; large inputs may hit token limits.

---

### Block C — Safety Gate + Routing (Production vs Evaluation)
**Overview:** Ensures that when running evaluation, the workflow does not send emails; when running normally, sends the appropriate email for the sentiment.  
**Nodes involved:** `Check Positive`, `Check Neutral`, `Check Negative`, `Send Hot Lead Email`, `Send Follow-up Notification`, `Send Marketing Insights Email`

#### Node: Check Positive
- **Type / role:** `n8n-nodes-base.evaluation` with operation `checkIfEvaluating` — gates production actions.
- **Configuration:**
  - `operation: checkIfEvaluating`
- **Behavior / outputs:**
  - Output 1 → production path (send email)  
  - Output 0 → evaluation path (save output)
- **Connections:**
  - Output 0 → `Save Output`
  - Output 1 → `Send Hot Lead Email`
- **Edge cases:**
  - If evaluation context isn’t detected as expected, may incorrectly allow production emails during tests (or block production runs).

#### Node: Check Neutral
- Same as above.
- **Connections:**
  - Output 0 → `Save Output`
  - Output 1 → `Send Follow-up Notification`

#### Node: Check Negative
- Same as above.
- **Connections:**
  - Output 0 → `Save Output`
  - Output 1 → `Send Marketing Insights Email`

#### Node: Send Hot Lead Email
- **Type / role:** `n8n-nodes-base.gmail` — sends notification email for “Positive” (hot lead).
- **Key configuration:**
  - To: `user@example.com`
  - Subject: `Hot Lead!`
  - Message expression: `={{ $('Gmail Trigger').item.json.text }}`
- **Connections:** terminal node (no outputs used).
- **Credentials:** Gmail OAuth2
- **Edge cases:**
  - Expression references **Gmail Trigger item**, which won’t exist in evaluation runs (but Safety Gate should prevent reaching here).
  - If Gmail Trigger output does not contain `.json.text`, message becomes blank/undefined.
  - Gmail send quota / spam filtering.

#### Node: Send Follow-up Notification
- **Role:** “Neutral” routing email.
- **Configuration:**
  - Subject: `Follow-up`
  - Same `sendTo` and `message` expression as above.
- **Edge cases:** same as above.

#### Node: Send Marketing Insights Email
- **Role:** “Negative” routing email (marketing insights).
- **Configuration:**
  - Subject: `Sales insights`
  - Same `sendTo` and `message` expression as above.
- **Edge cases:** same as above.

---

### Block D — Evaluation Entry + Batch Loop
**Overview:** Iterates through a Data Table dataset row-by-row, feeding each row into sentiment analysis and recording results and metrics.  
**Nodes involved:** `When fetching a dataset row`, `Loop Over Items`

#### Node: When fetching a dataset row
- **Type / role:** `n8n-nodes-base.evaluationTrigger` — evaluation entry point reading from a Data Table (Golden Dataset).
- **Key configuration:**
  - Data Table: “Sentiment Analysis Evaluation” (ID `LHUrQaGzxlnqJGRW`)
  - `onError: continueRegularOutput` (errors won’t hard-stop; may emit partial results)
- **Connections:**
  - Main output → `Loop Over Items`
- **Edge cases:**
  - Data Table missing/renamed, wrong permissions, empty dataset.
  - Row schema mismatch (expects fields like `input` or `text` and `expected` later).

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — processes dataset rows in batches (default batch size if not set).
- **Key configuration:**
  - No explicit batch size in parameters → n8n default (commonly 1 unless configured).
- **Connections:**
  - Output 1 (the “Next batch” loop output) → `Sentiment Analysis`
  - After metrics are set, `Set Metrics` routes back to `Loop Over Items` to request the next batch.
- **Edge cases:**
  - If Sentiment Analysis is `executeOnce: true`, looping may not evaluate all rows correctly.
  - If batch size > 1, downstream nodes must support multiple items cleanly.

---

### Block E — Saving Outputs + Metric Recording
**Overview:** Stores the model’s predicted category into the dataset, then records a categorization metric comparing expected vs actual labels.  
**Nodes involved:** `Save Output`, `Set Metrics`

#### Node: Save Output
- **Type / role:** `n8n-nodes-base.evaluation` — writes an output field back into the Data Table row.
- **Key configuration:**
  - Data Table: “Sentiment Analysis Evaluation”
  - Adds output field:
    - `result = {{ $('Sentiment Analysis').item.json.sentimentAnalysis.category }}`
- **Connections:**
  - Main output → `Set Metrics`
- **Edge cases:**
  - If Sentiment Analysis output structure differs (e.g., no `sentimentAnalysis.category`), expression fails or saves blank.
  - If multiple items are in-flight, using `$('Sentiment Analysis').item` can reference the wrong item unless n8n keeps item pairing consistent.

#### Node: Set Metrics
- **Type / role:** `n8n-nodes-base.evaluation` — computes/stores evaluation metric(s).
- **Key configuration:**
  - `operation: setMetrics`
  - `metric: categorization`
  - `actualAnswer = {{ $json.sentimentAnalysis.category }}`
  - `expectedAnswer = {{ $json.expected }}`
- **Connections:**
  - Main output → `Loop Over Items` (closes the evaluation loop)
- **Edge cases:**
  - Requires the evaluation item to contain:
    - `sentimentAnalysis.category` (actual)
    - `expected` (ground truth label)
  - If dataset uses different column names (e.g., `label`), metrics will be wrong/empty.
  - Expected labels must match the category set exactly (`Positive|Neutral|Negative`) or accuracy will drop due to label mismatch/casing.

---

### Block F — Gemini Model Nodes (Model candidates)
**Overview:** Provides switchable Gemini chat models for experimentation; only one is currently wired into Sentiment Analysis.  
**Nodes involved:** `Google Gemini 2.5 Flash Lite`, `Google Gemini 2.5 Flash`, `Google Gemini 3 PRO`

#### Node: Google Gemini 2.5 Flash Lite
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — chat LLM provider for LangChain nodes.
- **Key configuration:**
  - Model: `models/gemini-2.5-flash-lite`
- **Connections:**
  - AI output → `Sentiment Analysis` (ai_languageModel input)
- **Credentials:** Google PaLM / Gemini API credential (`googlePalmApi Credential`)
- **Edge cases:** API key restrictions, model availability by region/project, quota limits.

#### Node: Google Gemini 2.5 Flash
- **Role:** alternative model provider for A/B testing.
- **Configuration:** modelName not explicitly set in JSON (likely defaults or UI-selected model).
- **Connections:** not connected to Sentiment Analysis in the provided workflow.
- **Edge cases:** if connected later, ensure the correct model is selected and available.

#### Node: Google Gemini 3 PRO
- **Role:** alternative “Pro” model provider for higher quality evaluation.
- **Configuration:** `models/gemini-3-pro-preview`
- **Connections:** not connected.
- **Edge cases:** preview model instability/availability; potentially higher latency/cost.

---

### Block G — Sticky Notes (Documentation in-canvas)
**Overview:** Non-executable notes that describe sections: evaluation trigger, safety gate, core logic, and metric recording.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note5`

No runtime effect; content is captured in the Summary Table.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | gmailTrigger | Production entry point: poll Gmail for new emails | — | Sentiment Analysis |  |
| Sentiment Analysis | langchain.sentimentAnalysis | Classify email/dataset text into Positive/Neutral/Negative | Gmail Trigger; Loop Over Items; (AI) Google Gemini 2.5 Flash Lite | Check Positive; Check Neutral; Check Negative | ## CORE LOGIC |
| Check Positive | evaluation | Safety gate: route to evaluation path vs production email (positive) | Sentiment Analysis | Save Output; Send Hot Lead Email | ## SAFETY GATE |
| Check Neutral | evaluation | Safety gate: route to evaluation path vs production email (neutral) | Sentiment Analysis | Save Output; Send Follow-up Notification | ## SAFETY GATE |
| Check Negative | evaluation | Safety gate: route to evaluation path vs production email (negative) | Sentiment Analysis | Save Output; Send Marketing Insights Email | ## SAFETY GATE |
| Send Hot Lead Email | gmail | Production notification for positive sentiment | Check Positive | — | ## SAFETY GATE |
| Send Follow-up Notification | gmail | Production notification for neutral sentiment | Check Neutral | — | ## SAFETY GATE |
| Send Marketing Insights Email | gmail | Production notification for negative sentiment | Check Negative | — | ## SAFETY GATE |
| When fetching a dataset row | evaluationTrigger | Evaluation entry point: read Golden Dataset rows | — | Loop Over Items | ## EVALUATION TRIGGER |
| Loop Over Items | splitInBatches | Batch/loop through dataset rows | When fetching a dataset row; Set Metrics | Sentiment Analysis | ## EVALUATION TRIGGER |
| Save Output | evaluation | Write model output (“result”) back to Data Table row | Check Positive/Neutral/Negative | Set Metrics | ## METRIC RECORDING AND EVALUATION |
| Set Metrics | evaluation | Record categorization metric: actual vs expected | Save Output | Loop Over Items | ## METRIC RECORDING AND EVALUATION |
| Google Gemini 2.5 Flash Lite | lmChatGoogleGemini | LLM provider used by Sentiment Analysis | — | (AI) Sentiment Analysis |  |
| Google Gemini 2.5 Flash | lmChatGoogleGemini | Alternative LLM provider (not wired) | — | — |  |
| Google Gemini 3 PRO | lmChatGoogleGemini | Alternative LLM provider (not wired) | — | — |  |
| Sticky Note | stickyNote | Canvas note | — | — | ## EVALUATION TRIGGER |
| Sticky Note1 | stickyNote | Canvas note | — | — | ## SAFETY GATE |
| Sticky Note2 | stickyNote | Canvas note | — | — | ## METRIC RECORDING AND EVALUATION |
| Sticky Note5 | stickyNote | Canvas note | — | — | ## CORE LOGIC |
| Sticky Note3 | stickyNote | Canvas note (overall description) | — | — | # Smart Sales Lead Routing with Evaluation Framework  /  This workflow automates the categorization and routing of sales leads while providing a built-in sandbox for model evaluation. It uses Google Gemini to analyze the sentiment of incoming emails and routes them based on their business value.  /  ## How it works  /  * Production Path: The Gmail Trigger ingests new emails, which are sent to the Sentiment Analysis node. Based on the result, leads are routed to specific Gmail notification nodes.  /  * Evaluation Path: The Evaluation Trigger pulls a "Golden Dataset" from an n8n Data Table. The Evaluation nodes then compare the AI's actual output against your ground-truth labels to calculate accuracy.  /  * The Safety Gate: "Check if Evaluating" nodes ensure that during a test run, the workflow follows the evaluation logic and never sends actual emails to the sales team.  /  ## How to use  /  * Configure Credentials: Set up your Gmail OAuth2 and Google Gemini API credentials.  /  * Prepare Data: Create an n8n Data Table with columns for input text and an expected sentiment label.  /  * Test Models: Connect different Gemini model nodes (Flash Lite, Flash, or Pro) to the Sentiment Analysis node to compare latency and accuracy.  /  Analyze Results: Run the Evaluation Trigger to populate the Data Table with success metrics. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Add **Gmail OAuth2** credentials in n8n (Google Cloud project, OAuth consent, scopes for Gmail send + read).
   2. Add **Google Gemini/PaLM API** credentials (API key / configured credential supported by the `googlePalmApi` credential type).

2) **Create the Production entry node**
   1. Add node: **Gmail Trigger**
   2. Set polling to **Every Minute**.
   3. Leave filters empty (or configure label/from/subject as desired).
   4. Select your **Gmail OAuth2** credential.

3) **Create the LLM provider node (active model)**
   1. Add node: **Google Gemini Chat Model** (`lmChatGoogleGemini`)
   2. Select model: `models/gemini-2.5-flash-lite`
   3. Attach your **googlePalmApi** credential.

4) **Create the Sentiment Analysis node**
   1. Add node: **Sentiment Analysis** (LangChain)
   2. Set **Input Text** to: `{{ $json.text || $json.input }}`
   3. In options:
      - Categories: `Positive, Neutral, Negative`
      - Include detailed results: enabled
      - Auto-fixing: disabled
      - System prompt: paste the provided prompt (ensure it still enforces exactly one of the categories).
   4. Connect **Gmail Trigger → Sentiment Analysis** (main connection).
   5. Connect **Google Gemini 2.5 Flash Lite → Sentiment Analysis** using the **AI language model** connection.

5) **Create the Safety Gate nodes**
   1. Add three nodes of type **Evaluation** and set operation **Check if evaluating**:
      - `Check Positive`
      - `Check Neutral`
      - `Check Negative`
   2. Connect **Sentiment Analysis** main outputs to each check node (three outgoing connections).

6) **Create the three Gmail notification nodes**
   1. Add node: **Gmail** named `Send Hot Lead Email`
      - To: `user@example.com`
      - Subject: `Hot Lead!`
      - Message: `{{ $('Gmail Trigger').item.json.text }}`
      - Credential: Gmail OAuth2
   2. Add node: **Gmail** named `Send Follow-up Notification`
      - Subject: `Follow-up`
      - Same To/Message/credential
   3. Add node: **Gmail** named `Send Marketing Insights Email`
      - Subject: `Sales insights`
      - Same To/Message/credential
   4. Connect **Check Positive (production output) → Send Hot Lead Email**
   5. Connect **Check Neutral (production output) → Send Follow-up Notification**
   6. Connect **Check Negative (production output) → Send Marketing Insights Email**

7) **Create the Golden Dataset (Data Table)**
   1. In n8n, create a **Data Table** named “Sentiment Analysis Evaluation”.
   2. Add columns that the workflow expects:
      - One column for the input text: either `text` or `input`
      - One column for the ground truth label: `expected`
      - (Optional) a column for stored model output; the workflow writes output named `result`.

8) **Create the Evaluation trigger + loop**
   1. Add node: **Evaluation Trigger** named `When fetching a dataset row`
      - Select the “Sentiment Analysis Evaluation” Data Table.
      - Set error handling to “continue regular output” if desired.
   2. Add node: **Loop Over Items** (`Split in Batches`)
      - Leave default batch behavior (or set batch size to 1 for deterministic evaluation).
   3. Connect **Evaluation Trigger → Loop Over Items**
   4. Connect **Loop Over Items (items output) → Sentiment Analysis**

9) **Create output saving + metric recording**
   1. Add node: **Evaluation** named `Save Output`
      - Configure to save outputs to the same Data Table.
      - Add output:
        - Name: `result`
        - Value: `{{ $('Sentiment Analysis').item.json.sentimentAnalysis.category }}`
   2. Add node: **Evaluation** named `Set Metrics`
      - Operation: `Set metrics`
      - Metric type: `categorization`
      - Actual: `{{ $json.sentimentAnalysis.category }}`
      - Expected: `{{ $json.expected }}`
   3. Connect **each Safety Gate node’s evaluation output → Save Output**
   4. Connect **Save Output → Set Metrics**
   5. Connect **Set Metrics → Loop Over Items** (to fetch next row)

10) **(Optional) Add alternative Gemini model nodes**
   - Add `Google Gemini 2.5 Flash` and `Google Gemini 3 PRO` model nodes.
   - To compare models, connect only one model node at a time into Sentiment Analysis’ **AI language model** input.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Smart Sales Lead Routing with Evaluation Framework” note explaining production path, evaluation path, safety gate, and how to use | Embedded in canvas sticky note (overall workflow description) |
| Safety Gate concept: “Check if Evaluating” prevents real emails during evaluation runs | Sticky note: **## SAFETY GATE** |
| Dataset-driven evaluation via Evaluation Trigger and Data Table (“Golden Dataset”) | Sticky note: **## EVALUATION TRIGGER** |
| Metric recording and evaluation nodes store outputs and compute categorization metrics | Sticky note: **## METRIC RECORDING AND EVALUATION** |