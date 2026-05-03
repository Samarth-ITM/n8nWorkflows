Generate Indian stock investment ideas using Groq AI, Google Sheets and stock API

https://n8nworkflows.xyz/workflows/generate-indian-stock-investment-ideas-using-groq-ai--google-sheets-and-stock-api-14898


# Generate Indian stock investment ideas using Groq AI, Google Sheets and stock API

### 1. Workflow Overview

The **Simple Investment Idea Generator** is an automated pipeline designed to generate high-quality, non-redundant investment ideas for the Indian stock market. The workflow operates on a daily schedule, synthesizing real-time market data and news to provide actionable insights.

The logic is divided into five primary functional blocks:
- **1.1 Trigger & Data Collection:** Scheduled activation and fetching of trending stocks and market news.
- **1.2 Data Preparation:** Cleaning, filtering, and structuring raw data into a format optimized for LLM consumption.
- **1.3 AI Idea Generation & Validation:** Utilizing a Groq-powered AI agent with memory and external tool access to generate unique investment ideas.
- **1.4 Data Formatting & Storage:** Parsing AI JSON output and appending validated ideas to a Google Sheet.
- **1.5 Error Handling & Alerts:** A centralized system to catch failures at the API, AI, or formatting stages and notify the user via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Collection
**Overview:** Initiates the process daily and gathers the raw materials (stocks and news) needed for analysis.
- **Nodes Involved:** `Executes Workflow Everyday at 9 AM`, `Fetch Trending Stocks`, `Fetch Indian Market News`, `getting latest 5 news`, `Handle API Failure`.
- **Node Details:**
    - **Executes Workflow Everyday at 9 AM (Schedule Trigger):** Triggers the workflow daily at 09:00.
    - **Fetch Trending Stocks (HTTP Request):** Calls `https://stock.indianapi.in/trending`. Uses `httpMultipleHeadersAuth` for API access.
    - **Fetch Indian Market News (RSS Feed Read):** Reads Google News RSS for keywords "Indian stock market OR NSE OR BSE".
    - **getting latest 5 news (Limit):** Restricts the news feed to the 5 most recent items to avoid token overflow in the AI prompt.
    - **Handle API Failure (If):** Checks if `trending_stocks` is undefined. If true, it routes to the error block.

#### 2.2 Data Preparation
**Overview:** Merges divergent data streams and transforms them into a concise summary for the AI.
- **Nodes Involved:** `Combine Market Data`, `Format Data for AI`.
- **Node Details:**
    - **Combine Market Data (Merge):** Combines the stock data and the filtered news data by position.
    - **Format Data for AI (Set):** Uses a JavaScript expression to map only the top 3 gainers and top 3 losers (name, price, change, trends, rating) and cleans HTML tags from news snippets using regex (`/<[^>]*>/g`).

#### 2.3 AI Idea Generation & Validation
**Overview:** The "brain" of the workflow. It uses an AI Agent to reason through data while ensuring no duplicate ideas are generated based on historical data.
- **Nodes Involved:** `Generate Investment Ideas`, `Groq Chat Model`, `Simple Memory`, `get_existing_ideas`, `Handle AI Failure`.
- **Node Details:**
    - **Generate Investment Ideas (AI Agent):** A professional equity research analyst agent. It is strictly instructed to avoid generic terms and use sector-based reasoning. It requires a JSON array output.
    - **Groq Chat Model (Chat Model):** Uses the `openai/gpt-oss-120b` model via Groq for high-speed inference.
    - **Simple Memory (Window Buffer Memory):** Maintains a session key `investment-session` to remember recent ideas within the current execution window.
    - **get_existing_ideas (Google Sheets Tool):** A tool that allows the AI to read the current Google Sheet to see what ideas were already posted, preventing historical duplication.
    - **Handle AI Failure (If):** Validates if the AI output is empty, an empty array `[]`, or null.

#### 2.4 Data Formatting & Storage
**Overview:** Converts the AI's text response into structured rows for a spreadsheet.
- **Nodes Involved:** `Format for Sheets`, `Json Format Error Check`, `Append row in sheet`.
- **Node Details:**
    - **Format for Sheets (Code):** A JS node that strips markdown (e.g., ```json) from the AI response and parses the string into a JSON object. If parsing fails, it returns a `FORMAT_ERROR`.
    - **Json Format Error Check (If):** Verifies that the previous node did not produce a `FORMAT_ERROR`.
    - **Append row in sheet (Google Sheets):** Appends the final data (Date, Stock, Title, Reason, Horizon) to the "AI Investment Ideas – India" spreadsheet.

#### 2.5 Error Handling & Alerts
**Overview:** A safety net that aggregates errors from various stages and sends a single notification.
- **Nodes Involved:** `API Failed`, `AI Failed`, `Combine all the errors`, `Send Error Message on Gmail`.
- **Node Details:**
    - **API Failed / AI Failed (Set):** Standardizes error messages with `error_type`, `message`, and `stage`.
    - **Combine all the errors (Merge):** Collects errors from the API, AI, and JSON parsing stages.
    - **Send Error Message on Gmail (Gmail):** Sends a formatted email to the administrator containing the error type, stage, and timestamp.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Executes Workflow... | Schedule Trigger | Workflow Entry | - | Fetch Trending Stocks, Fetch Indian Market News | Trigger & Data Collection |
| Fetch Trending Stocks | HTTP Request | Data Acquisition | Executes Workflow... | Handle API Failure | Trigger & Data Collection |
| Fetch Indian Market News| RSS Feed Read | Data Acquisition | Executes Workflow... | getting latest 5 news | Trigger & Data Collection |
| getting latest 5 news | Limit | Data Filtering | Fetch Indian Market News | Combine Market Data | Trigger & Data Collection |
| Handle API Failure | If | Validation | Fetch Trending Stocks | API Failed, Combine Market Data | Trigger & Data Collection |
| Combine Market Data | Merge | Data Integration | Handle API Failure, getting latest 5 news | Format Data for AI | Data Preparation |
| Format Data for AI | Set | Data Structuring | Combine Market Data | Generate Investment Ideas | Data Preparation |
| Generate Investment Ideas| AI Agent | Intelligence/Logic | Format Data for AI | Handle AI Failure | AI Idea Generation & Validation |
| Groq Chat Model | Chat Model | LLM Provider | - | Generate Investment Ideas | AI Idea Generation & Validation |
| Simple Memory | Buffer Memory | Context Retention | - | Generate Investment Ideas | AI Idea Generation & Validation |
| get_existing_ideas | GSheets Tool | Data Retrieval | - | Generate Investment Ideas | AI Idea Generation & Validation |
| Handle AI Failure | If | Validation | Generate Investment Ideas | AI Failed, Format for Sheets | AI Idea Generation & Validation |
| Format for Sheets | Code | Data Parsing | Handle AI Failure | Json Format Error Check | Data Formatting & Storage |
| Json Format Error Check | If | Validation | Format for Sheets | Combine all the errors, Append row in sheet | Data Formatting & Storage |
| Append row in sheet | Google Sheets | Data Persistence | Json Format Error Check | - | Data Formatting & Storage |
| API Failed | Set | Error Definition | Handle API Failure | Combine all the errors | Error Handling & Alerts |
| AI Failed | Set | Error Definition | Handle AI Failure | Combine all the errors | Error Handling & Alerts |
| Combine all the errors | Merge | Error Aggregation | API Failed, AI Failed, Json Format Error Check | Send Error Message on Gmail | Error Handling & Alerts |
| Send Error Message... | Gmail | Notification | Combine all the errors | - | Error Handling & Alerts |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Data Collection Setup
1. Create a **Schedule Trigger** set to daily at 9:00 AM.
2. Add an **HTTP Request** node. URL: `https://stock.indianapi.in/trending`. Configure `httpMultipleHeadersAuth` credentials.
3. Add an **RSS Feed Read** node. URL: `https://news.google.com/rss/search?q=Indian stock market OR NSE OR BSE`.
4. Add a **Limit** node after the RSS feed, set to `5` items.
5. Add an **If** node after the HTTP Request to check if `trending_stocks` is undefined. Route "True" to a **Set** node (API Failed) and "False" to a **Merge** node.

#### Step 2: Data Formatting
1. Create a **Merge** node (Mode: Combine, by Position) to join the HTTP Request output and the Limit node output.
2. Add a **Set** node. Use the "Raw" mode and enter the JavaScript mapping logic to extract top 3 gainers/losers and clean news headlines using regex.

#### Step 3: AI Agent Configuration
1. Create an **AI Agent** node.
2. Attach a **Chat Model** (Groq) using the `openai/gpt-oss-120b` model.
3. Attach a **Window Buffer Memory** node (Session Key: `investment-session`, Window: 15).
4. Attach a **Google Sheets Tool** node. Connect it to your "AI Investment Ideas – India" sheet to allow the agent to read existing data.
5. In the Agent's prompt, define the persona (Equity Research Analyst) and the strict JSON output format (title, stock, reason, risk, horizon).

#### Step 4: Output Processing & Storage
1. Add an **If** node after the Agent to verify that the `output` is not empty or `[]`.
2. Add a **Code** node to clean the AI's markdown response and `JSON.parse()` the result.
3. Add an **If** node to verify the `error_type` is not `FORMAT_ERROR`.
4. Add a **Google Sheets** node (Operation: Append). Map the fields: Date (`$today`), Stock, Title, Reason, and Horizon.

#### Step 5: Error Notification
1. Create a **Merge** node that accepts inputs from:
    - The "API Failed" Set node.
    - The "AI Failed" Set node.
    - The "Json Format Error Check" (True path).
2. Connect this Merge node to a **Gmail** node. Use a template: `Type: {{ $json.error_type }} | Stage: {{ $json.stage }} | Message: {{ $json.message }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires Stock API credentials for trending data | `httpMultipleHeadersAuth` |
| Requires Google Sheets OAuth2 credentials | Read/Append access to Sheet |
| Requires Groq Cloud API Key | `openai/gpt-oss-120b` model |
| Targeted for the Indian Market (NSE/BSE) | Market-specific logic |