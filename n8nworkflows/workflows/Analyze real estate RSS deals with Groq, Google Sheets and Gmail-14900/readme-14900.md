Analyze real estate RSS deals with Groq, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/analyze-real-estate-rss-deals-with-groq--google-sheets-and-gmail-14900


# Analyze real estate RSS deals with Groq, Google Sheets and Gmail

# Workflow Analysis: Real Estate News RSS to AI Deal Opportunity Analyzer

### 1. Workflow Overview
This workflow automates the discovery, evaluation, and notification of real estate investment opportunities. It monitors specific Google News RSS feeds, deduplicates content against a Google Sheet database, utilizes a Large Language Model (LLM) via Groq to score the "opportunity value" of each article, and triggers alerts for high-scoring deals.

The logic is divided into the following functional blocks:
- **1.1 Ingestion & Deduplication:** Fetches target RSS feeds and compares them against existing records to ensure only new articles are processed.
- **1.2 Batch Processing & Rate Control:** Manages the flow of articles to prevent API rate-limiting during AI analysis.
- **1.3 AI Evaluation & Data Structuring:** Uses Groq LLM to analyze content and converts raw text responses into structured JSON.
- **1.4 Qualification & Alerting:** Filters for high-score opportunities ($\ge 8$), archives them in Google Sheets, and sends a Gmail notification.

---

### 2. Block-by-Block Analysis

#### 2.1 Ingestion & Deduplication
**Overview:** Initializes the data pipeline by defining search queries and ensuring no duplicate articles are analyzed.
- **Nodes Involved:** `Start Workflow`, `Fetch Existing Deals (Sheet)`, `RSS Sources`, `Loop Over Sources`, `Fetch RSS News Feed`, `Filter New Articles`.
- **Node Details:**
    - `Start Workflow`: Manual trigger to initiate the process.
    - `Fetch Existing Deals (Sheet)`: **Google Sheets node**. Retrieves all previously processed links from a specific sheet to serve as a deduplication list.
    - `RSS Sources`: **Code node**. Returns an array of Google News RSS search URLs (e.g., focusing on "Ahmedabad real estate price drop").
    - `Loop Over Sources`: **Split In Batches node**. Iterates through the list of RSS URLs.
    - `Fetch RSS News Feed`: **RSS Feed Read node**. Dynamically fetches the feed using `{{ $json.url }}`.
    - `Filter New Articles`: **Code node**. Compares the links from the RSS feed against the links retrieved from the Google Sheet. Only articles not present in the sheet are passed forward.
    - **Potential Failures:** Google Sheets API auth errors; RSS feeds returning 404 or empty results.

#### 2.2 Batch Processing & Rate Control
**Overview:** Manages the execution flow of filtered articles to ensure stability and compliance with AI API limits.
- **Nodes Involved:** `Process Articles Loop`, `Throttle API Calls`.
- **Node Details:**
    - `Process Articles Loop`: **Split In Batches node**. Processes the filtered list of new articles one by one (or in small batches).
    - `Throttle API Calls`: **Wait node**. Implements a 3-second delay between AI requests to avoid Groq API rate limits.
    - **Potential Failures:** Loop timeouts if the volume of new articles is extremely high.

#### 2.3 AI Evaluation & Data Structuring
**Overview:** Analyzes the snippet of the news article to determine if it represents a viable real estate deal.
- **Nodes Involved:** `Analyze Deal Opportunity`, `Groq LLM Engine`, `Parse AI Response JSON`, `Format & Enrich Deal Data`.
- **Node Details:**
    - `Groq LLM Engine`: **Chat Groq node**. Uses model `openai/gpt-oss-120b`.
    - `Analyze Deal Opportunity`: **Chain LLM node**. Uses a specific persona ("Real Estate Analyst") and requests a JSON output containing a `Score` (1-10) and a `Reason`.
    - `Parse AI Response JSON`: **Code node**. Uses a Regular Expression to extract the JSON block from the AI's text response and parses it into actual JSON fields.
    - `Format & Enrich Deal Data`: **Set node**. Combines the AI's `Score` and `Reason` with original RSS metadata (`title`, `link`, `pubDate`, `contentSnippet`).
    - **Potential Failures:** AI returning non-JSON text (handled by the Parse node); Groq API timeouts.

#### 2.4 Qualification & Alerting
**Overview:** Final filter to separate "noise" from "opportunities" and execute the notification sequence.
- **Nodes Involved:** `Filter High Opportunity (Score ≥ 8)`, `Save High-Value Deals (Sheet)`, `Send Deal Alert Email`.
- **Node Details:**
    - `Filter High Opportunity (Score ≥ 8)`: **If node**. Checks if `Score` is $\ge 8$.
    - `Save High-Value Deals (Sheet)`: **Google Sheets node**. Appends the high-value deal details (Link, Title, Reason, PubDate) back to the database.
    - `Send Deal Alert Email`: **Gmail node**. Sends a formatted email alert including the deal title, AI reason, and the source link.
    - **Potential Failures:** Gmail API quota limits; Google Sheet column mapping mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start Workflow | Manual Trigger | Initiation | - | Fetch Existing Deals (Sheet) | Workflow Overview: Real Estate Deal Opportunity Analyzer |
| Fetch Existing Deals (Sheet) | Google Sheets | Deduplication Base | Start Workflow | RSS Sources | RSS Source Setup & Ingestion |
| RSS Sources | Code | URL Definition | Fetch Existing Deals | Loop Over Sources | RSS Source Setup & Ingestion |
| Loop Over Sources | Split In Batches | URL Iteration | RSS Sources / Fetch RSS News Feed | Fetch RSS News Feed / Filter New Articles | RSS Source Setup & Ingestion |
| Fetch RSS News Feed | RSS Feed Read | Content Acquisition | Loop Over Sources | Loop Over Sources | RSS Source Setup & Ingestion |
| Filter New Articles | Code | Deduplication Logic | Loop Over Sources | Process Articles Loop | Filtering, Looping & Rate Control |
| Process Articles Loop | Split In Batches | Article Iteration | Filter New Articles / Send Deal Alert Email | Throttle API Calls | Filtering, Looping & Rate Control |
| Throttle API Calls | Wait | Rate Limiting | Process Articles Loop | Analyze Deal Opportunity | Filtering, Looping & Rate Control |
| Groq LLM Engine | Chat Groq | AI Model Provider | - | Analyze Deal Opportunity | AI Analysis & Response Structuring |
| Analyze Deal Opportunity | Chain LLM | Deal Evaluation | Throttle API Calls | Parse AI Response JSON | AI Analysis & Response Structuring |
| Parse AI Response JSON | Code | JSON Extraction | Analyze Deal Opportunity | Format & Enrich Deal Data | AI Analysis & Response Structuring |
| Format & Enrich Deal Data | Set | Data Standardization | Parse AI Response JSON | Filter High Opportunity | AI Analysis & Response Structuring |
| Filter High Opportunity | If | Quality Gate | Format & Enrich Deal Data | Save High-Value Deals / Process Articles Loop | Opportunity Filtering, Storage & Alerting |
| Save High-Value Deals (Sheet) | Google Sheets | Archiving | Filter High Opportunity | Send Deal Alert Email | Opportunity Filtering, Storage & Alerting |
| Send Deal Alert Email | Gmail | Notification | Save High-Value Deals | Process Articles Loop | Opportunity Filtering, Storage & Alerting |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Ingestion Setup**
1. Create a **Manual Trigger**.
2. Add a **Google Sheets** node (`Fetch Existing Deals (Sheet)`): Set operation to "Get Many", select your Spreadsheet and Sheet1.
3. Add a **Code** node (`RSS Sources`): Return an array of objects containing `url` strings (Google News RSS links).
4. Add a **Split In Batches** node (`Loop Over Sources`) connected to the Code node.
5. Add an **RSS Feed Read** node: Set URL to `{{ $json.url }}`. Connect this back to the `Loop Over Sources` node to create a cycle for all URLs.

**Step 2: Deduplication & Iteration**
6. Create a **Code** node (`Filter New Articles`): Write a script to filter the current RSS items by checking if their `link` exists in the data retrieved by the `Fetch Existing Deals (Sheet)` node.
7. Add a **Split In Batches** node (`Process Articles Loop`) to iterate through the filtered articles.

**Step 3: AI Analysis Pipeline**
8. Add a **Wait** node (`Throttle API Calls`): Set to 3 seconds.
9. Add a **Chain LLM** node (`Analyze Deal Opportunity`): 
    - Prompt: "Role: Real Estate Analyst... Format: { "Score": int, "Reason": string }"
10. Connect a **Chat Groq** node to the Chain LLM: Configure Groq API credentials and select model `openai/gpt-oss-120b`.
11. Add a **Code** node (`Parse AI Response JSON`): Use `text.match(/\{[\s\S]*\}/)` to extract the JSON and `JSON.parse()` to convert it.
12. Add a **Set** node (`Format & Enrich Deal Data`): Map the original RSS fields (from `Process Articles Loop`) and the AI fields (`Score`, `Reason`).

**Step 4: Final Actions**
13. Add an **If** node: Set condition `Score` $\ge 8$.
14. On the **True** path, add a **Google Sheets** node (`Save High-Value Deals`): Set operation to "Append" and map the columns (Title, Link, Reason, PubDate).
15. Add a **Gmail** node (`Send Deal Alert Email`): Use the `Score`, `Reason`, and `Link` in the email body. Connect this back to the `Process Articles Loop` to move to the next item.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow focuses on Ahmedabad real estate markets. | Domain Context |
| Uses Groq for high-speed, low-cost LLM inference. | Technology Stack |
| Deduplication is critical to prevent repeated AI costs and email spam. | Logic Warning |
| Requires Google OAuth2 credentials for Sheets and Gmail. | Authentication |