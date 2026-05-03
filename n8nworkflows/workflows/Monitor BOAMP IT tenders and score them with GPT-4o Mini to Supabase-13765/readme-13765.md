Monitor BOAMP IT tenders and score them with GPT-4o Mini to Supabase

https://n8nworkflows.xyz/workflows/monitor-boamp-it-tenders-and-score-them-with-gpt-4o-mini-to-supabase-13765


# Monitor BOAMP IT tenders and score them with GPT-4o Mini to Supabase

# Reference Document: Monitor BOAMP IT Tenders and Score with GPT-4o Mini to Supabase

## 1. Workflow Overview
This workflow automates the monitoring of French public sector IT tenders via the BOAMP (Bulletin Officiel des Annonces des Marchés Publics) API. It is designed for IT consultancies, agencies, and ESNs (Entreprises de Services du Numérique) to streamline lead generation.

The workflow fetches the latest IT-related tenders, utilizes OpenAI's GPT-4o Mini to analyze the technical stack and estimate budget/relevance, and then routes the data into a Supabase database. Based on an AI-generated score, opportunities are categorized as "Hot" (high relevance) or "Archived" for later review.

### Functional Blocks:
1.  **Input Reception:** Triggers the workflow via schedule, manual action, or webhook, and sets global configuration parameters.
2.  **Fetch & Normalize:** Retrieves raw JSON data from the BOAMP OpenData API and transforms it into a standard format.
3.  **AI Analysis:** Sends the tender description to OpenAI to extract structured data (score, stack, summary).
4.  **Routing & Storage:** Evaluates the AI score against a threshold and inserts the record into specific Supabase tables/statuses.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception
*   **Overview:** Handles how the workflow starts and defines the global scoring threshold.
*   **Nodes Involved:** Schedule Every 4 Hours, Manual Trigger, Webhook Trigger, Config.
*   **Node Details:**
    *   **Triggers:** Three entry points (Cron expression for every 4 hours, UI button, or HTTP POST request to `/devops-radar-fetch`).
    *   **Config (Set Node):** Sets a variable `SCORE_THRESHOLD` (default: 75). This value is passed down the chain to determine what constitutes a "Hot" lead.
    *   **Edge Cases:** Multiple simultaneous triggers may cause database locks if Supabase is not configured for high concurrency.

### 2.2 Fetch & Normalize
*   **Overview:** Connects to the external API and prepares the text for AI processing.
*   **Nodes Involved:** Fetch BOAMP IT Tenders, Normalize Tender Data.
*   **Node Details:**
    *   **Fetch BOAMP IT Tenders (HTTP Request):** Performs a GET request to the `boamp-datadila` API filtering for the keyword "informatique". It limits results to 10 records per run to manage API tokens.
    *   **Normalize Tender Data (Code):** A JavaScript snippet that maps raw API fields (like `idweb`, `objet`, `descripteur_libelle`) into a cleaner object. It creates a `body` string combining the title and description to provide context for the AI.
    *   **Potential Failure:** API downtime or schema changes in the BOAMP OpenData portal.

### 2.3 AI Analysis
*   **Overview:** Uses LLM intelligence to evaluate the quality of the tender.
*   **Nodes Involved:** AI Score Tender, Parse AI Response.
*   **Node Details:**
    *   **AI Score Tender (HTTP Request):** Calls the OpenAI API (`gpt-4o-mini`). It uses a specific system prompt to force a JSON response containing: `pertinence` (0-100), `budget_estimated`, `stack_keywords`, `summary`, and a `recommendation`.
    *   **Parse AI Response (Code):** Extracts the JSON from the AI's string response. It includes a `try/catch` block to handle malformed JSON, defaulting the score to 50 if the AI fails. It also assigns the final `status` ('notified' vs 'archived') based on the threshold.
    *   **Variables:** Uses `$json.title` and `$json.body` for analysis.

### 2.4 Routing & Storage
*   **Overview:** Directs the processed data to the appropriate database destination.
*   **Nodes Involved:** Is Hot Opportunity?, Supabase Insert Hot, Supabase Insert Archived.
*   **Node Details:**
    *   **Is Hot Opportunity? (If):** Compares the AI-generated `score` against the `SCORE_THRESHOLD`.
    *   **Supabase Nodes:** Uses the "Auto-map" feature to insert data into the `opportunities` table. The `SCORE_THRESHOLD` field is ignored during insertion as it is a workflow-only logic variable.
    *   **Failure Types:** Authentication errors with Supabase or unique constraint violations if the same tender ID is fetched multiple times.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Every 4 Hours | Schedule Trigger | Workflow Entry | (None) | Config | **Step 1 - Triggers** |
| Manual Trigger | Manual Trigger | Workflow Entry | (None) | Config | **Step 1 - Triggers** |
| Webhook Trigger | Webhook | Workflow Entry | (None) | Config | **Step 1 - Triggers** |
| Config | Set | Parameter Setup | Triggers | Fetch BOAMP | (None) |
| Fetch BOAMP IT Tenders | HTTP Request | Data Fetching | Config | Normalize Tender | **Step 2 - Fetch & Normalize** |
| Normalize Tender Data | Code | Data Formatting | Fetch BOAMP | AI Score Tender | **Step 2 - Fetch & Normalize** |
| AI Score Tender | HTTP Request | AI Processing | Normalize | Parse AI Response | **Step 3–5 - AI & Route** |
| Parse AI Response | Code | Logic/Parsing | AI Score Tender | Is Hot? | **Step 3–5 - AI & Route** |
| Is Hot Opportunity? | If | Decision Logic | Parse AI | Supabase Nodes | **Step 3–5 - AI & Route** |
| Supabase Insert Hot | Supabase | Data Storage | Is Hot? (True) | (None) | **Step 3–5 - AI & Route** |
| Supabase Insert Archived | Supabase | Data Storage | Is Hot? (False) | (None) | **Step 3–5 - AI & Route** |

---

## 4. Reproducing the Workflow from Scratch

1.  **Database Setup:**
    *   Go to Supabase SQL Editor and run the SQL schema provided in the "General Notes" section to create the `opportunities` table.
2.  **Triggers:**
    *   Add a **Schedule Trigger** (Interval: 4 hours).
    *   Add a **Manual Trigger**.
    *   Add a **Webhook Trigger** (Method: POST, Path: `devops-radar-fetch`).
3.  **Global Config:**
    *   Connect triggers to a **Set Node** named "Config". Create a Number variable `SCORE_THRESHOLD` = `75`.
4.  **Fetch Data:**
    *   Add an **HTTP Request Node**. Method: `GET`. URL: `https://boamp-datadila.opendatasoft.com/api/explore/v2.1/catalog/datasets/boamp/records?limit=10&refine=objet:informatique`.
5.  **Normalization:**
    *   Add a **Code Node**. Use JavaScript to map `results` array. Combine `objet` and `descripteur_libelle` into a single `body` field.
6.  **AI Integration:**
    *   Add an **HTTP Request Node** for OpenAI. 
    *   URL: `https://api.openai.com/v1/chat/completions`. Method: `POST`. 
    *   Header: Predefined Credential (OpenAI).
    *   Body (JSON): Prompt the model `gpt-4o-mini` to analyze the `title` and `body` and return a strict JSON object.
7.  **Data Parsing:**
    *   Add a **Code Node** to parse the OpenAI response string into a JSON object and merge it with the original tender data.
8.  **Routing Logic:**
    *   Add an **If Node**. Condition: `score` >= `SCORE_THRESHOLD`.
9.  **Storage:**
    *   Add two **Supabase Nodes** (Table: `opportunities`, Action: `Insert`). 
    *   One for the "True" path (High Score) and one for the "False" path (Low Score). Ensure "Auto-map" is on and `SCORE_THRESHOLD` is excluded from the mapping.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Database Schema:** `CREATE TABLE IF NOT EXISTS opportunities (id UUID DEFAULT gen_random_uuid() PRIMARY KEY, source TEXT NOT NULL, external_id TEXT NOT NULL, title TEXT, raw_data JSONB, stack_keywords TEXT[], budget_estimated TEXT, score INT DEFAULT 0, summary TEXT, status TEXT DEFAULT 'new' CHECK (status IN ('new','notified','archived')), created_at TIMESTAMPTZ DEFAULT NOW(), UNIQUE(source, external_id));` | Run this in Supabase SQL Editor before starting the workflow. |
| **API Reference:** BOAMP OpenData API | [BOAMP Datadila Explorer](https://boamp-datadila.opendatasoft.com/explore/dataset/boamp/api/) |
| **AI Model:** GPT-4o-Mini | Chosen for low cost and high speed for classification tasks. |