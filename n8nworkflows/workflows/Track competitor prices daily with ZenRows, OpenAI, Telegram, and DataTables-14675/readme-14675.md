Track competitor prices daily with ZenRows, OpenAI, Telegram, and DataTables

https://n8nworkflows.xyz/workflows/track-competitor-prices-daily-with-zenrows--openai--telegram--and-datatables-14675


# Track competitor prices daily with ZenRows, OpenAI, Telegram, and DataTables

# Technical Analysis: Competitor Price Tracking Workflow

This documentation provides a comprehensive technical breakdown of the n8n workflow designed to monitor competitor product prices daily, identify changes, update a database, and notify the user via Telegram.

---

### 1. Workflow Overview
The primary purpose of this workflow is to automate the monitoring of product prices from external websites. It leverages a managed proxy service (ZenRows) to bypass anti-scraping measures, uses a data table for persistence, and employs AI (OpenAI) to summarize the findings for the end-user.

The logic is divided into four functional blocks:
- **1.1 Data Retrieval:** Triggering the process and fetching the list of target URLs and current prices from an internal database.
- **1.2 Scraping & Extraction:** Iterating through the product list, fetching live HTML via ZenRows, and extracting the current price.
- **1.3 Price Comparison & Persistence:** Comparing the live price against the stored price and updating the database if a change is detected.
- **1.4 AI Summarization & Notification:** Aggregating all changes and using an LLM to generate a human-readable summary sent via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Retrieval
**Overview:** This block initiates the workflow on a schedule and prepares the dataset of products to be tracked.
- **Nodes Involved:** `Schedule Trigger`, `Get row(price url)`, `set input data`.
- **Node Details:**
    - **Schedule Trigger:** (Trigger) Set to run daily.
    - **Get row(price url):** (n8n Data Table) Retrieves the list of product URLs and their last known prices.
    - **set input data:** (Set) Standardizes the input data structure before passing it to the loop.
    - **Potential Failures:** Data Table connectivity issues or empty tables resulting in no items to process.

#### 2.2 Scraping & Extraction
**Overview:** This block handles the "heavy lifting" of visiting websites and pulling specific data points.
- **Nodes Involved:** `Loop Over Items`, `Set Data in Loop`, `Zenrows check product`, `Extract data`.
- **Node Details:**
    - **Loop Over Items:** (Split in Batches) Ensures the workflow processes products sequentially to avoid rate limiting or memory crashes.
    - **Set Data in Loop:** (Set) Maps the specific URL for the current iteration.
    - **Zenrows check product:** (HTTP Request) Sends a request to the ZenRows API to fetch the target page. *Configured to continue on error* to prevent a single 404 or timeout from stopping the entire process.
    - **Extract data:** (Code) A JavaScript snippet used to parse the HTML response from ZenRows and isolate the price element.
    - **Potential Failures:** Changes in the competitor's website HTML structure (CSS selectors) causing the extraction code to fail.

#### 2.3 Price Comparison & Persistence
**Overview:** This block determines if the price has shifted and ensures the database remains the "source of truth."
- **Nodes Involved:** `Compare prices`, `Update row(price chenged)`.
- **Node Details:**
    - **Compare prices:** (Code) Logic that compares the "Stored Price" from the Data Table with the "Live Price" from the extraction node.
    - **Update row(price chenged):** (n8n Data Table) Updates the specific row in the database only if a price change was detected.
    - **Edge Case:** If the extracted price is non-numeric or null, the comparison logic must handle this to avoid updating the database with "undefined."

#### 2.4 AI Summarization & Notification
**Overview:** Instead of sending a notification for every single price change, this block aggregates results and creates a natural language report.
- **Nodes Involved:** `Filter`, `Aggregate`, `Sumary writer`, `OpenAI Chat Model`, `Send a text message`.
- **Node Details:**
    - **Filter:** Isolates only the items where a price change occurred.
    - **Aggregate:** Combines all changed items into a single array/string.
    - **Sumary writer:** (LLM Chain) Takes the aggregated list and prompts the AI to write a concise summary.
    - **OpenAI Chat Model:** (AI Model) The brain providing the linguistic capability for the summary.
    - **Send a text message:** (Telegram) Delivers the final AI-generated report to a specified chat ID.
    - **Potential Failures:** OpenAI API quota exhaustion or Telegram Bot API token expiration.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule Trigger | Workflow Initiation | - | Get row(price url) | |
| Get row(price url) | Data Table | Fetch targets | Schedule Trigger | set input data | |
| set input data | Set | Data normalization | Get row(price url) | Loop Over Items | |
| Loop Over Items | Split in Batches | Item iteration | set input data, Update row... | Filter, Set Data in Loop | |
| Filter | Filter | Detect changes | Loop Over Items | Aggregate | |
| Aggregate | Aggregate | Group changes | Filter | Sumary writer | |
| Set Data in Loop | Set | Variable mapping | Loop Over Items | Zenrows check product | |
| Zenrows check product | HTTP Request | Web scraping | Set Data in Loop | Extract data | |
| Extract data | Code | HTML Parsing | Zenrows check product | Compare prices | |
| Compare prices | Code | Logic validation | Extract data | Update row... | |
| Update row(price chenged) | Data Table | Database Update | Compare prices | Loop Over Items | |
| Sumary writer | LLM Chain | AI Text Generation | Aggregate | Send a text message | |
| OpenAI Chat Model | AI Model | LLM Provider | - | Sumary writer | |
| Send a text message | Telegram | Final Notification | Sumary writer | - | |
| Send a message | Gmail | (Disabled) Backup alert | - | - | |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Database Setup
1. Create an **n8n Data Table** with at least two columns: `URL` (String) and `Price` (Number/String).
2. Populate the table with the competitor product URLs you wish to track.

#### Step 2: Trigger & Initialization
1. Add a **Schedule Trigger** node (Set to Interval: Daily).
2. Connect it to a **Data Table** node configured to "Get Rows."
3. Add a **Set** node named `set input data` to ensure the data format is clean.

#### Step 3: The Processing Loop
1. Add a **Split in Batches** node to iterate through the products.
2. Connect the loop to two paths:
    - **Path A (Extraction):** `Set` $\rightarrow$ `HTTP Request` (ZenRows API) $\rightarrow$ `Code` (Extract Price) $\rightarrow$ `Code` (Compare Price) $\rightarrow$ `Data Table` (Update Row).
    - **Path B (Reporting):** `Filter` (Check if price changed) $\rightarrow$ `Aggregate` (Merge all changes).
3. Ensure the `Update row` node loops back to the `Split in Batches` node.

#### Step 4: AI & Notification Logic
1. Add a **Chain LLM** node (`Sumary writer`).
2. Connect the **OpenAI Chat Model** node to the `Sumary writer`.
3. Configure the prompt in the Chain node to: *"Summarize the following price changes into a short, professional Telegram message."*
4. Connect the output to a **Telegram** node using your Bot Token and Chat ID.

#### Step 5: Credentials Configuration
- **ZenRows:** API Key included in the HTTP Request Header.
- **OpenAI:** API Key via n8n Credentials.
- **Telegram:** Bot Token and Chat ID via n8n Credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| ZenRows is used to avoid CAPTCHAs and IP blocks. | [ZenRows Website](https://www.zenrows.com) |
| The Gmail node is present in the JSON but disabled. | Optional alternative for notifications. |
| Ensure the "Continue on Fail" option is active for the HTTP node. | Prevents workflow crash on 404 errors. |