Automated Revenue Predictions from Stripe Data with GPT-4 and Google Sheets

https://n8nworkflows.xyz/workflows/automated-revenue-predictions-from-stripe-data-with-gpt-4-and-google-sheets-4577


# Automated Revenue Predictions from Stripe Data with GPT-4 and Google Sheets

### 1. Workflow Overview

The **"Automated Revenue Predictions from Stripe Data with GPT-4 and Google Sheets"** workflow is designed to automate the daily forecasting of revenue based on Stripe payment data. It integrates Stripe for transactional data retrieval, uses OpenAI's GPT-4 model to analyze and predict future revenue trends, optionally indexes data in Pinecone for enhanced context retrieval, and stores/presents the forecast results in Supabase and Google Sheets for reporting and further analytics.

The workflow is logically divided into three main functional blocks:

- **1.1 Data Retrieval & Preprocessing:** Collects raw sales data from Stripe, aggregates daily revenue, and prepares a textual prompt for AI forecasting.
- **1.2 AI Agent Forecasting:** Uses GPT-4 via an agent to analyze the sales data, generate structured revenue forecasts, interpret trends, and optionally store embeddings and context in Pinecone vector database.
- **1.3 Storage & Reporting:** Saves the forecast results into Supabase for historical tracking and logs them into Google Sheets for visual human review.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Retrieval & Preprocessing

**Overview:**  
This block runs daily to fetch all successful Stripe charges, summarizes the daily revenue, and formats the data into a natural language prompt for AI consumption.

**Nodes Involved:**  
- Run Daily Forecast  
- Fetch Stripe Charges  
- Summarize Sales  
- Prepare data

**Node Details:**

- **Run Daily Forecast**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers the workflow daily at a specified hour (configured for 9 AM UTC).  
  - *Configuration:* Executes the workflow once per day to keep forecasting up-to-date.  
  - *Input:* None (trigger node)  
  - *Output:* Triggers `Fetch Stripe Charges`  
  - *Edge Cases:* Missed triggers if n8n host is down or scheduler fails.

- **Fetch Stripe Charges**  
  - *Type:* Stripe API Node  
  - *Role:* Retrieves all Stripe charges with status 'succeeded' over a defined timeframe (default is all recent charges).  
  - *Configuration:* Resource = Charge, Operation = Get All, Return All = true.  
  - *Input:* Trigger from schedule node  
  - *Output:* List of Stripe charge objects including amount, timestamp, customer data, etc.  
  - *Edge Cases:* API rate limits, authentication failures, empty response if no charges during timeframe.

- **Summarize Sales**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Aggregates Stripe charges by date, converting UNIX timestamps to `YYYY-MM-DD` and summing amounts (converted from cents to dollars).  
  - *Key Code Logic:*  
    - Maps all items to JSON charge objects.  
    - Reduces charges into a dictionary keyed by date with summed revenue values.  
  - *Input:* Stripe charges list  
  - *Output:* JSON object summarizing daily revenue totals  
  - *Edge Cases:* Empty input array, malformed charge data.

- **Prepare data**  
  - *Type:* Set Node  
  - *Role:* Converts the summary object into a string format suitable for prompt injection into the AI Agent.  
  - *Configuration:* Sets a string field `summary` with value from previous node’s JSON summary.  
  - *Input:* JSON summary from Code node  
  - *Output:* Single item with `summary` string property  
  - *Edge Cases:* If summary data is large, string length limits might apply downstream.

---

#### 1.2 AI Agent Forecasting

**Overview:**  
This block uses an intelligent agent powered by GPT-4 to forecast revenue trends and generate structured output, optionally enhancing the context with embeddings stored in Pinecone.

**Nodes Involved:**  
- Forecaster agent  
- OpenAI Chat Model  
- Structured Output Parser  
- Embeddings OpenAI (optional)  
- Pinecone Vector Store (optional)

**Node Details:**

- **Forecaster agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Acts as the orchestrator AI agent that receives the sales summary prompt and uses the OpenAI GPT-4 model to generate forecasts. It integrates output parsing and optional tool use.  
  - *Configuration:*  
    - Prompt includes a CFO AI persona and the sales summary data.  
    - Uses structured output parsing for JSON-formatted forecasts.  
    - Calls OpenAI GPT-4 (via linked language model node) and optionally Pinecone vector store and embeddings.  
  - *Input:* `summary` string from Prepare data node  
  - *Output:* AI-generated structured forecast JSON including forecast, trend, confidence, insights  
  - *Edge Cases:* Prompt formatting errors, API auth failures, timeouts, unexpected AI response formats, large prompt length limits.

- **OpenAI Chat Model**  
  - *Type:* OpenAI Language Model Node  
  - *Role:* Executes GPT-4 model calls for natural language generation based on prompt from agent.  
  - *Configuration:*  
    - Model: "gpt-4o-mini" (a GPT-4 variant or mini model)  
    - Temperature: default (not explicitly set to ensure determinism)  
  - *Credentials:* OpenAI API key required  
  - *Input:* Prompt text from agent  
  - *Output:* GPT-4 completion text  
  - *Edge Cases:* API rate limiting, quota exhaustion, network errors.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser Node  
  - *Role:* Parses GPT-4 raw text output into structured JSON using a predefined JSON schema example.  
  - *Configuration:* JSON schema example defines keys: forecast (monthly forecasts), trend, confidence, insights.  
  - *Input:* GPT-4 text response  
  - *Output:* Parsed JSON object with forecast data  
  - *Edge Cases:* Parsing failures if GPT-4 output does not match schema, malformed JSON, incomplete responses.

- **Embeddings OpenAI** *(Optional)*  
  - *Type:* OpenAI Embeddings Node  
  - *Role:* Converts text data into vector embeddings for semantic storage or retrieval.  
  - *Input:* Text data (likely sales summary or forecast text)  
  - *Output:* Vector embeddings  
  - *Edge Cases:* API failures, text too long.

- **Pinecone Vector Store** *(Optional)*  
  - *Type:* Vector Store Node (Pinecone)  
  - *Role:* Stores or retrieves vector embeddings for retrieval-augmented generation (RAG) or long-term memory.  
  - *Configuration:* Index named "new", tool name "Sales_data" describing Stripe sales data context.  
  - *Input:* Embeddings from OpenAI Embeddings node  
  - *Output:* Indexed vector data or query results  
  - *Edge Cases:* Authentication issues, index unavailability, network failures.

---

#### 1.3 Storage & Reporting

**Overview:**  
The final block stores the structured forecast results into a Supabase database for archival and app integration, and logs the same data into Google Sheets for easy visualization and manual review.

**Nodes Involved:**  
- Save Forecast to Supabase  
- Log Forecast in Google Sheets

**Node Details:**

- **Save Forecast to Supabase**  
  - *Type:* Supabase Node  
  - *Role:* Automatically stores forecast data into a Supabase table (likely named `forecasts`).  
  - *Configuration:* Set to auto-map all input data fields to columns.  
  - *Input:* Parsed forecast JSON from Forecaster agent  
  - *Output:* Confirmation or inserted record info  
  - *Credentials:* Supabase API key and project info required  
  - *Edge Cases:* API failures, schema mismatches, network issues.

- **Log Forecast in Google Sheets**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends forecast data (trend, forecast, confidence, insights) into a Google Sheet for dashboard reporting.  
  - *Configuration:*  
    - Operation: Append  
    - Document ID and Sheet Name predefined (spreadsheet URL provided)  
    - Columns mapped with expressions referencing structured JSON keys from AI output.  
  - *Input:* Forecast output JSON  
  - *Output:* Row appended confirmation  
  - *Credentials:* Google OAuth2 with Sheets API scope  
  - *Edge Cases:* API quota limits, permission errors, sheet structure changes.

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                             | Input Node(s)           | Output Node(s)                         | Sticky Note                                     |
|---------------------------|-------------------------------------|---------------------------------------------|-------------------------|---------------------------------------|------------------------------------------------|
| Run Daily Forecast         | Schedule Trigger                    | Triggers workflow daily                      | None                    | Fetch Stripe Charges                  | Triggers daily run at 9 AM UTC                  |
| Fetch Stripe Charges       | Stripe API Node                    | Retrieves successful Stripe charges          | Run Daily Forecast      | Summarize Sales                      | Retrieves all succeeded Stripe charges          |
| Summarize Sales           | Code Node (JavaScript)              | Aggregates daily revenue from charges        | Fetch Stripe Charges    | Prepare data                        | Converts timestamps and sums amounts             |
| Prepare data              | Set Node                          | Prepares prompt string with summary          | Summarize Sales         | Forecaster agent                    | Converts summary object to string for prompt    |
| Forecaster agent          | LangChain Agent Node                | AI agent generating revenue forecasts        | Prepare data            | Save Forecast to Supabase, Log Forecast in Google Sheets | Uses GPT-4 and structured output parsing         |
| OpenAI Chat Model         | OpenAI Language Model               | Calls GPT-4 to generate forecast text        | Forecaster agent        | Forecaster agent                    | GPT-4 model for natural language generation      |
| Structured Output Parser  | LangChain Structured Output Parser | Parses GPT-4 output into structured JSON     | OpenAI Chat Model       | Forecaster agent                    | Parses AI response to forecast JSON              |
| Embeddings OpenAI         | OpenAI Embeddings Node             | (Optional) Creates vector embeddings          | Forecaster agent        | Pinecone Vector Store               | Converts text to vector for Pinecone             |
| Pinecone Vector Store     | Vector Store (Pinecone)            | (Optional) Stores/retrieves vectors           | Embeddings OpenAI       | Forecaster agent                    | Indexes sales data for retrieval-augmented use   |
| Save Forecast to Supabase | Supabase Node                     | Stores forecast results in Supabase           | Forecaster agent        | None                              | Stores historical forecast data                   |
| Log Forecast in Google Sheets | Google Sheets Node               | Appends forecast data to Google Sheets        | Forecaster agent        | None                              | Logs forecast for dashboard and human review     |
| Sticky Note               | Sticky Note                       | Documentation and explanation notes           | None                    | None                              | Covers Data Retrieval & Preprocessing block      |
| Sticky Note1              | Sticky Note                       | Documentation and explanation notes           | None                    | None                              | Covers AI Agent Forecasting block                 |
| Sticky Note2              | Sticky Note                       | Documentation and explanation notes           | None                    | None                              | Covers Storage & Reporting block                  |
| Sticky Note4              | Sticky Note                       | Full workflow documentation summary           | None                    | None                              | Detailed comprehensive documentation              |
| Sticky Note9              | Sticky Note                       | Support and contact information                | None                    | None                              | Workflow assistance and external links            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run daily at 9:00 AM UTC (or desired hour).  
   - Name it `Run Daily Forecast`.

2. **Create a Stripe Node**  
   - Type: Stripe  
   - Resource: Charge  
   - Operation: Get All  
   - Return All: True  
   - Filters: Add `status: succeeded`, optionally add `created[gte]` filter for last 30 days if desired.  
   - Connect `Run Daily Forecast` main output to this node’s input.  
   - Name it `Fetch Stripe Charges`.  
   - Provide Stripe API credentials.

3. **Add a Code Node for Summarization**  
   - Type: Code (JavaScript)  
   - Paste the code to aggregate charges by date, converting UNIX timestamps to `YYYY-MM-DD`, summing amounts, and converting cents to dollars:  
   ```js
   const charges = items.map(item => item.json);
   const summary = charges.reduce((acc, charge) => {
     const date = new Date(charge.created * 1000).toISOString().split("T")[0];
     acc[date] = (acc[date] || 0) + charge.amount / 100;
     return acc;
   }, {});
   return [{ json: { summary } }];
   ```  
   - Connect `Fetch Stripe Charges` output to this node’s input.  
   - Name it `Summarize Sales`.

4. **Add a Set Node to Prepare Prompt Data**  
   - Type: Set  
   - Add a string field named `summary` with value `={{ $json.summary }}` (injects the summarized sales JSON as a string).  
   - Connect `Summarize Sales` output to this node’s input.  
   - Name it `Prepare data`.

5. **Add a LangChain Agent Node for Forecasting**  
   - Type: LangChain Agent Node  
   - Configure prompt with text:  
     ```
     You are a CFO AI Agent. Based on the following Stripe sales data:

     {{ $json.summary }}

     Analyze the trends, identify any patterns (growth, decline, seasonality), and forecast expected daily or weekly revenue for the next 3 months.
     ```  
   - Enable structured output parsing.  
   - Connect from `Prepare data` main output to this node’s input.  
   - Name it `Forecaster agent`.

6. **Add an OpenAI Chat Model Node**  
   - Type: OpenAI  
   - Model: `gpt-4o-mini` (or `gpt-4`/`gpt-4-turbo` depending on availability)  
   - Temperature: 0.2 (optional for deterministic output)  
   - Provide OpenAI API credentials.  
   - Connect this node as AI Language Model in `Forecaster agent` node configuration.

7. **Add a Structured Output Parser Node**  
   - Type: LangChain Structured Output Parser  
   - Provide JSON schema example for forecast output with keys: forecast (monthly forecasts), trend, confidence, insights.  
   - Connect this node as AI Output Parser in `Forecaster agent` configuration.

8. **(Optional) Add OpenAI Embeddings Node**  
   - Type: OpenAI Embeddings  
   - Provide OpenAI credentials.  
   - Connect `Forecaster agent` output to this node for generating embeddings.

9. **(Optional) Add Pinecone Vector Store Node**  
   - Type: Pinecone Vector Store  
   - Configure Pinecone index (e.g., "new") and tool description "Sales_data".  
   - Provide Pinecone API credentials.  
   - Connect Embeddings node output to this node.  
   - Connect this node as AI Tool in `Forecaster agent` configuration.

10. **Add a Supabase Node to Store Forecast**  
    - Type: Supabase  
    - Configure to insert data into `forecasts` table, auto-mapping input fields.  
    - Provide Supabase credentials.  
    - Connect `Forecaster agent` main output to this node’s input.  
    - Name it `Save Forecast to Supabase`.

11. **Add a Google Sheets Node for Reporting**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: specify your Google Sheet document ID.  
    - Sheet Name: specify the target sheet (e.g., `gid=0`).  
    - Map columns:  
      - `forecast` → `{{ $json.output.forecast }}`  
      - `trend` → `{{ $json.output.trend }}`  
      - `confidence` → `{{ $json.output.confidence }}`  
      - `insights` → `{{ $json.output.insights }}`  
    - Provide Google Sheets OAuth2 credentials.  
    - Connect `Forecaster agent` main output to this node’s input.  
    - Name it `Log Forecast in Google Sheets`.

12. **Connect the Flow**  
    - `Run Daily Forecast` → `Fetch Stripe Charges` → `Summarize Sales` → `Prepare data` → `Forecaster agent` → (`Save Forecast to Supabase` and `Log Forecast in Google Sheets`)  
    - Within `Forecaster agent`, configure AI model, output parser, and optional tools as detailed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                             |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Workflow automatically forecasts revenue trends using Stripe sales data and GPT-4 AI for CFO-level insights. | Workflow purpose and business context                        |
| Contact for workflow assistance: Yaron@nofluff.online                                                        | Support email                                               |
| Additional tips and tutorials available on YouTube and LinkedIn:                                             | YouTube: https://www.youtube.com/@YaronBeen/videos          |
|                                                                                                               | LinkedIn: https://www.linkedin.com/in/yaronbeen/            |
| The workflow uses structured output parsing to ensure reliable JSON forecast extraction from GPT-4 responses. | Important for AI integration robustness                      |
| Pinecone integration is optional but recommended for enhanced historical context and retrieval-augmented generation. | Enhances AI memory and accuracy                              |
| Google Sheets provides a user-friendly dashboard for manual review and sharing of forecasts.                  | Reporting and visualization                                  |
| Supabase stores structured forecast data for analytics, versioning, and history tracking.                     | Data persistence and integration                             |

---

This document provides a detailed, structured reference for understanding, modifying, or recreating the "Automated Revenue Predictions from Stripe Data with GPT-4 and Google Sheets" workflow, supporting both advanced users and AI-driven automation agents.