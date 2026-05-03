Automated Daily Stock Market Report with Bright Data, GPT-4.1, Airtable and Gmail

https://n8nworkflows.xyz/workflows/automated-daily-stock-market-report-with-bright-data--gpt-4-1--airtable-and-gmail-8724


# Automated Daily Stock Market Report with Bright Data, GPT-4.1, Airtable and Gmail

### 1. Workflow Overview

This workflow automates the daily generation and distribution of a professional stock market report. It targets financial analysts, investors, or institutional clients who require timely, data-driven market summaries. The logic is organized into the following functional blocks:

- **1.1 Scheduled Trigger & Stock List Initialization:** Automatically starts the workflow daily and defines the list of stocks to monitor.
- **1.2 Stock Data Retrieval via Bright Data:** For each stock ticker, initiates a scraping job on Bright Data API, monitors job progress, and fetches detailed stock market data.
- **1.3 Data Aggregation & Storage:** Consolidates the scraped data into a single dataset and archives it into Airtable for historical reference.
- **1.4 AI-Powered Report Generation:** Uses an AI agent (GPT-4.1) to analyze aggregated stock data and produce a clean, HTML-formatted daily market digest.
- **1.5 Email Delivery:** Sends the AI-generated report via Gmail to a specified recipient.

Each block is tightly linked to the next, ensuring automated data flow from input to final email delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Stock List Initialization

**Overview:**  
Starts the workflow automatically every day and sets a fixed list of stocks to track.

**Nodes Involved:**  
- Daily Run Trigger  
- Set Stock List  
- Split Stocks  
- Prepare Stock Keyword

**Node Details:**

- **Daily Run Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow at a fixed daily interval (default every 1 day).  
  - *Parameters:* Interval set to daily without specific start time or timezone configured.  
  - *Connections:* Output → Set Stock List  
  - *Potential Failures:* None significant, but misconfiguration could prevent trigger activation.

- **Set Stock List**  
  - *Type:* Set  
  - *Role:* Defines a static array of stock objects with ticker, name, and approximate market cap.  
  - *Parameters:* Hardcoded JSON array with 10 stocks (e.g., SHEL, NESN.SW, SAP, etc.).  
  - *Connections:* Output → Split Stocks  
  - *Edge Cases:* Static data means no dynamic updating; outdated stock list if not maintained.

- **Split Stocks**  
  - *Type:* Split Out  
  - *Role:* Splits the array of stocks into individual items to process each stock separately.  
  - *Parameters:* Splits on the `json` field containing the stock list.  
  - *Connections:* Output → Prepare Stock Keyword  
  - *Edge Cases:* Empty or malformed stock list would halt further processing.

- **Prepare Stock Keyword**  
  - *Type:* Set  
  - *Role:* Adds a `keyword` field to each stock item, set to the stock ticker symbol.  
  - *Parameters:* Sets `keyword` = `{{ $json.ticker }}`.  
  - *Connections:* Output → Bright Data Scraper  
  - *Edge Cases:* Missing `ticker` values will cause incorrect scraping queries.

---

#### 2.2 Stock Data Retrieval via Bright Data

**Overview:**  
Launches scraping jobs for each stock via Bright Data API, checks job status in a loop, and fetches completed data.

**Nodes Involved:**  
- Bright Data Scraper  
- Check Scraper Progress  
- Scraper Status Switch  
- Wait for Data  
- Fetch Scraper Results

**Node Details:**

- **Bright Data Scraper**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to Bright Data API to trigger data scraping jobs per stock keyword.  
  - *Parameters:*  
    - URL: `https://api.brightdata.com/datasets/v3/trigger`  
    - Method: POST  
    - Body: Array of all stock keywords from previous node  
    - Query Params: `dataset_id`, `include_errors`, `type=discover_new`, `discover_by=keyword`  
    - Headers: Authorization Bearer token (must replace `YOUR_BRIGHTDATA_API_KEY` with valid key)  
  - *Connections:* Output → Check Scraper Progress  
  - *Edge Cases:* Authentication failure, API rate limits, network timeouts.

- **Check Scraper Progress**  
  - *Type:* HTTP Request  
  - *Role:* Polls Bright Data API to check the status (`running` or `ready`) of scraping jobs using `snapshot_id`.  
  - *Parameters:*  
    - URL templated with `snapshot_id` from previous response  
    - Method: GET  
    - Headers: Authorization Bearer token  
  - *Connections:* Output → Scraper Status Switch  
  - *Edge Cases:* Snapshot ID missing or invalid, API errors, rate limits.

- **Scraper Status Switch**  
  - *Type:* Switch  
  - *Role:* Routes workflow based on scraper status:  
    - If `running`, loops to Wait for Data  
    - If `ready`, proceeds to Fetch Scraper Results  
  - *Parameters:* Checks `status` field in JSON response.  
  - *Connections:*  
    - Running → Wait for Data  
    - Ready → Fetch Scraper Results  
  - *Edge Cases:* Unexpected status values, missing `status` field.

- **Wait for Data**  
  - *Type:* Wait  
  - *Role:* Pauses workflow for 20 seconds before rechecking scraper progress.  
  - *Parameters:* Fixed wait time: 20 seconds.  
  - *Connections:* Output → Check Scraper Progress  
  - *Edge Cases:* Delay too short/long may cause rate limit or slow response issues.

- **Fetch Scraper Results**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves completed stock data JSON from Bright Data API using `snapshot_id`.  
  - *Parameters:*  
    - URL templated with `snapshot_id`  
    - Method: GET  
    - Query Parameter: `format=json`  
    - Headers: Authorization Bearer token  
  - *Connections:* Output → Aggregate Stock Data, Save to Airtable (Daily Stocks)  
  - *Edge Cases:* Invalid `snapshot_id`, empty or malformed data.

---

#### 2.3 Data Aggregation & Storage

**Overview:**  
Consolidates individual stock data into a single dataset and archives the data into Airtable.

**Nodes Involved:**  
- Aggregate Stock Data  
- Save to Airtable (Daily Stocks)

**Node Details:**

- **Aggregate Stock Data**  
  - *Type:* Aggregate  
  - *Role:* Combines multiple stock JSON items into one array/object to feed AI node.  
  - *Parameters:* Aggregates all incoming data into a single collection.  
  - *Connections:* Output → Generate Daily Summary (AI)  
  - *Edge Cases:* Empty input produces empty dataset, which may cause AI generation failures.

- **Save to Airtable (Daily Stocks)**  
  - *Type:* Airtable  
  - *Role:* Creates records in Airtable “Daily Stocks” table for historical tracking.  
  - *Parameters:*  
    - Base: `appqPdjbR45flhdgT` (Airtable base ID)  
    - Table: `tblvIkbZriGaCrfzP` (Daily Stocks)  
    - Fields mapped: Price, Ticker, Company, Change %, Sentiment, Market Cap  
    - Operation: Create Record (no update or upsert)  
  - *Connections:* Input from Fetch Scraper Results  
  - *Credentials:* Airtable Personal Access Token required  
  - *Edge Cases:* API limits, invalid base/table IDs, missing required fields.

---

#### 2.4 AI-Powered Report Generation

**Overview:**  
Analyzes aggregated stock data using GPT-4.1 to produce a detailed, professionally formatted HTML stock market digest ready for email.

**Nodes Involved:**  
- Generate Daily Summary (AI)  
- Simple Memory  
- OpenAI Chat Model

**Node Details:**

- **Generate Daily Summary (AI)**  
  - *Type:* LangChain Agent Node (AI agent)  
  - *Role:* Uses AI to:  
    - Parse JSON stock data  
    - Detect market trends, top gainers/losers  
    - Create an HTML email digest with professional formatting  
  - *Parameters:*  
    - Model: GPT-4.1-mini  
    - Prompt: Detailed instructions for generating an HTML stock market report including tables, insights, and event sections  
  - *Connections:*  
    - Input from Aggregate Stock Data  
    - AI Memory input from Simple Memory  
    - AI Language Model input from OpenAI Chat Model  
    - AI Tool output to Send Report via Gmail  
  - *Edge Cases:* Model API errors, prompt formatting issues, empty input data.

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Stores conversational or context memory for AI interactions (though minimal in this use case).  
  - *Connections:* Provides AI memory input to Generate Daily Summary (AI).  
  - *Edge Cases:* Memory overflow or stale data; generally low risk here.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides AI language model interface (GPT-4.1-mini).  
  - *Parameters:* Model set to GPT-4.1-mini, no special options.  
  - *Credentials:* Requires valid OpenAI API credentials.  
  - *Connections:* Supplies language model input to Generate Daily Summary (AI).  
  - *Edge Cases:* API key invalid/expired, rate limits, model unavailability.

---

#### 2.5 Email Delivery

**Overview:**  
Sends the AI-generated HTML stock market report via Gmail to a configured recipient.

**Nodes Involved:**  
- Send Report via Gmail

**Node Details:**

- **Send Report via Gmail**  
  - *Type:* Gmail Tool  
  - *Role:* Sends an email containing the AI-generated HTML report.  
  - *Parameters:*  
    - To: `Baptiste.fort.pro@gmail.com` (recipient email)  
    - Subject: `Daily Stock Market Digest`  
    - Message: HTML from Generate Daily Summary (AI) node output  
    - Append Attribution: false (no footer added by Gmail node)  
  - *Credentials:* Gmail OAuth2 credentials required  
  - *Edge Cases:* Authentication failure, quota limits, email delivery issues, malformed HTML causing rendering problems.

---

### 3. Summary Table

| Node Name              | Node Type                                | Functional Role                                      | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                     |
|------------------------|----------------------------------------|-----------------------------------------------------|--------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Daily Run Trigger      | Schedule Trigger                       | Starts workflow daily                               | -                        | Set Stock List                 | Starts the workflow automatically at fixed intervals. Ensures daily execution.                 |
| Set Stock List         | Set                                   | Defines static list of stocks                        | Daily Run Trigger         | Split Stocks                  | Defines the list of stocks to track; acts as seed data for scraping.                           |
| Split Stocks           | Split Out                             | Splits stock array into individual items            | Set Stock List            | Prepare Stock Keyword          | Splits stock list array for separate processing of each stock.                                |
| Prepare Stock Keyword  | Set                                   | Adds keyword field with stock ticker                | Split Stocks              | Bright Data Scraper            | Adds a `keyword` field used by Bright Data scraper.                                           |
| Bright Data Scraper    | HTTP Request                          | Triggers Bright Data scraping jobs                   | Prepare Stock Keyword     | Check Scraper Progress         | Sends POST request to start scraping jobs on Bright Data API.                                |
| Check Scraper Progress | HTTP Request                          | Polls Bright Data for job status                      | Bright Data Scraper, Wait for Data | Scraper Status Switch        | Checks if scraping job is `running` or `ready`. Loops accordingly.                           |
| Scraper Status Switch  | Switch                               | Routes workflow based on scraper status              | Check Scraper Progress    | Fetch Scraper Results, Wait for Data | Routes to fetch results or wait for completion based on job status.                       |
| Wait for Data          | Wait                                 | Pauses workflow before rechecking scraper status     | Scraper Status Switch     | Check Scraper Progress         | Waits a fixed time (20 seconds) to avoid rate limits and allow job completion.                |
| Fetch Scraper Results  | HTTP Request                          | Retrieves scraped stock data                          | Scraper Status Switch     | Aggregate Stock Data, Save to Airtable (Daily Stocks) | Retrieves completed scraped data in JSON format.                                            |
| Aggregate Stock Data   | Aggregate                            | Combines multiple stock data items into one dataset  | Fetch Scraper Results     | Generate Daily Summary (AI)    | Merges individual stock data into single JSON for AI analysis.                               |
| Save to Airtable (Daily Stocks) | Airtable                      | Saves stock data records in Airtable                  | Fetch Scraper Results     | -                              | Stores daily stock data in Airtable for historical tracking.                                 |
| Generate Daily Summary (AI) | LangChain Agent                | Generates professional HTML stock market report      | Aggregate Stock Data, Simple Memory, OpenAI Chat Model | Send Report via Gmail          | AI analyzes data and outputs formatted HTML email body.                                      |
| Simple Memory          | LangChain Memory Buffer Window       | Provides context memory for AI agent                  | -                        | Generate Daily Summary (AI)    | Maintains AI interaction context (limited use here).                                         |
| OpenAI Chat Model      | LangChain OpenAI Chat Model           | Provides GPT-4.1 language model interface             | -                        | Generate Daily Summary (AI)    | AI language model backend for report generation.                                            |
| Send Report via Gmail  | Gmail Tool                          | Sends the generated HTML report via email             | Generate Daily Summary (AI) | -                              | Sends final report email to recipient inbox.                                                |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Schedule Trigger** node named `Daily Run Trigger`  
- Set it to trigger every 1 day (interval).  
- No specific start time or timezone needed unless desired.

**Step 2:** Create a **Set** node named `Set Stock List`  
- Configure to set a single JSON array field `json` with stock objects: ticker, name, market_cap (use the provided 10 stocks).  
- Connect `Daily Run Trigger` output to this node.

**Step 3:** Create a **Split Out** node named `Split Stocks`  
- Set to split the array on the field `json` (the stock list).  
- Connect `Set Stock List` output to this node.

**Step 4:** Create a **Set** node named `Prepare Stock Keyword`  
- Add a new field `keyword` with value `={{ $json.ticker }}`.  
- Connect `Split Stocks` output to this node.

**Step 5:** Create an **HTTP Request** node named `Bright Data Scraper`  
- Method: POST  
- URL: `https://api.brightdata.com/datasets/v3/trigger`  
- Query Parameters:  
  - `dataset_id`: `gd_lmrpz3vxmz972ghd7`  
  - `include_errors`: `true`  
  - `type`: `discover_new`  
  - `discover_by`: `keyword`  
- Body: JSON array of keywords from `Prepare Stock Keyword` (use expression to map all items)  
- Headers: Authorization: `Bearer YOUR_BRIGHTDATA_API_KEY` (replace with valid token)  
- Connect `Prepare Stock Keyword` output to this node.

**Step 6:** Create an **HTTP Request** node named `Check Scraper Progress`  
- Method: GET  
- URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
- Headers: Authorization: `Bearer YOUR_BRIGHTDATA_API_KEY`  
- Connect `Bright Data Scraper` output to this node. Also connect from the Wait node later.

**Step 7:** Create a **Switch** node named `Scraper Status Switch`  
- Check `status` field in JSON response.  
- Two rules:  
  - Equals `running` → output `running`  
  - Equals `ready` → output `ready`  
- Connect `Check Scraper Progress` output to this node.

**Step 8:** Create a **Wait** node named `Wait for Data`  
- Wait for 20 seconds (fixed time).  
- Connect `Scraper Status Switch` output `running` to this node.  
- Connect `Wait for Data` output to `Check Scraper Progress` (loop).

**Step 9:** Create an **HTTP Request** node named `Fetch Scraper Results`  
- Method: GET  
- URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
- Query Parameter: `format=json`  
- Headers: Authorization: `Bearer YOUR_BRIGHTDATA_API_KEY`  
- Connect `Scraper Status Switch` output `ready` to this node.

**Step 10:** Create an **Aggregate** node named `Aggregate Stock Data`  
- Aggregate all incoming items into one JSON array/object.  
- Connect `Fetch Scraper Results` output to this node.

**Step 11:** Create an **Airtable** node named `Save to Airtable (Daily Stocks)`  
- Set operation to `Create Record`.  
- Configure your Airtable base and table (e.g., `appqPdjbR45flhdgT`, `Daily Stocks`).  
- Map fields: Ticker, Company, Price, Change %, Sentiment, Market Cap from the JSON data fields.  
- Credentials: Set up Airtable Personal Access Token with write permissions.  
- Connect `Fetch Scraper Results` output to this node.

**Step 12:** Create a **LangChain Agent** node named `Generate Daily Summary (AI)`  
- Model: GPT-4.1-mini or equivalent  
- Input: JSON from `Aggregate Stock Data` node.  
- Prompt: Use detailed prompt instructing AI to analyze stock data and produce a clean HTML email digest, including market trends, top movers, tables, insights, and upcoming events.  
- Connect `Aggregate Stock Data` output to this node.  
- Connect AI memory input from a **Simple Memory** node (optional for context).  
- Connect AI language model input from an **OpenAI Chat Model** node configured with OpenAI credentials.

**Step 13:** Create a **Simple Memory** node  
- Default settings for LangChain memory.  
- Connect to `Generate Daily Summary (AI)` AI memory input.

**Step 14:** Create an **OpenAI Chat Model** node  
- Set model to GPT-4.1-mini.  
- Use OpenAI API credentials.  
- Connect to `Generate Daily Summary (AI)` AI language model input.

**Step 15:** Create a **Gmail Tool** node named `Send Report via Gmail`  
- Operation: Send Email  
- To: recipient’s email (e.g., `Baptiste.fort.pro@gmail.com`)  
- Subject: `Daily Stock Market Digest`  
- Message: Set to output of `Generate Daily Summary (AI)` node (HTML content)  
- Credentials: OAuth2 Gmail account set up  
- Connect AI tool output of `Generate Daily Summary (AI)` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Replace all `YOUR_BRIGHTDATA_API_KEY` placeholders with valid Bright Data API tokens before running.        | Bright Data API Documentation: https://brightdata.com/docs/api-reference                          |
| Airtable base and table IDs must be adjusted to your own Airtable configuration with correct field mapping.| Airtable API Docs: https://airtable.com/api                                                        |
| OpenAI credentials require proper subscription and API key with access to GPT-4.1 or equivalent models.     | OpenAI API Docs: https://platform.openai.com/docs                                                  |
| Email recipient address in Gmail node must be updated to the desired recipient.                             | Gmail OAuth2 Setup Guide: https://developers.google.com/gmail/api/quickstart                       |
| Prompt in AI node is critical to output proper HTML; do not modify the prompt structure without testing.   | LangChain Node Reference: https://docs.n8n.io/nodes/agents/langchain-agent/                        |
| The workflow respects API rate limits by polling with wait intervals to avoid rapid requests.              | Rate limit best practices for Bright Data and OpenAI should be reviewed.                           |

---

**Disclaimer:**  
The text provided is exclusively generated from an automated workflow created with n8n, a powerful integration and automation tool. All content complies strictly with content policies and contains no illegal or offensive material. All data processed is legal and publicly available.