Monitor Shopify Stores for New Products with BrowserAct and Slack Alerts

https://n8nworkflows.xyz/workflows/monitor-shopify-stores-for-new-products-with-browseract-and-slack-alerts-9895


# Monitor Shopify Stores for New Products with BrowserAct and Slack Alerts

### 1. Workflow Overview

This workflow automates the monitoring of multiple competitor Shopify stores to detect newly added products and immediately notify a team via Slack. It is designed for competitive intelligence use cases, where real-time awareness of product launches by competitors is valuable.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & Loop Setup:** Periodically triggers the workflow, reads a master list of competitor stores from a Google Sheet, and processes competitors one-by-one.
- **1.2 Scrape & Store Data:** For each competitor, creates or accesses a dedicated Google Sheet tab, triggers a BrowserAct scraper workflow to fetch the current product list, parses the results, and stores them.
- **1.3 Compare & Alert:** Compares the newly scraped product list against the previously stored list for that competitor, detects any new products, and sends a Slack alert if new items are found.

Supporting the main flow are sticky notes with instructions and help resources.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Loop Setup

**Overview:**  
This block initiates the workflow on a schedule and obtains the list of competitor stores from a Google Sheet. It then loops over each competitor item to process them individually.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet  
- Loop Over Items

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger / Scheduler  
  - Configuration: Runs on a default interval (every minute by default in n8n when empty interval)  
  - Input/Output: No input; triggers the workflow outputting current timestamp and metadata  
  - Failure Cases: Misconfiguration of interval; if disabled, workflow will not run automatically  
  - Version: 1.2

- **Get row(s) in sheet**  
  - Type: Google Sheets node  
  - Configuration: Reads all rows from the sheet named "Competitor Store List" in the document with ID `1A_SG0aNjbRs9Sdc2uowXyNCI0-h3W4uEpPogD_B3ZM4`  
  - Credentials: Google Sheets OAuth2  
  - Output: Array of competitor store objects with columns like `Name`, `Link`, `Pagination Type`  
  - Failure Cases: Credential issues, sheet not found, empty sheet  
  - Version: 4.7

- **Loop Over Items**  
  - Type: SplitInBatches node  
  - Configuration: Default batch size (processes competitors one at a time)  
  - Input: Array of competitors from previous node  
  - Output: Outputs one competitor item per iteration for downstream processing  
  - Failure Cases: If input is empty, no iterations run  
  - Version: 3

---

#### 2.2 Scrape & Store Data

**Overview:**  
For each competitor, this block ensures a dedicated sheet exists, runs a BrowserAct scraping workflow to fetch the competitor’s current product list, parses the scraper output, and stores the results in the competitor’s dedicated sheet.

**Nodes Involved:**  
- Create sheet  
- Run a workflow (BrowserAct Scraper)  
- Get workflow Data (BrowserAct task result)  
- Parse Json (Code node)  
- Store Data (Google Sheets appendOrUpdate)

**Node Details:**

- **Create sheet**  
  - Type: Google Sheets node  
  - Configuration: Creates a new sheet/tab titled with the competitor's `Name` inside the main spreadsheet document  
  - Credentials: Google Sheets OAuth2  
  - Output: Confirmation of sheet creation, always outputs data regardless of success (useful for downstream node)  
  - Failure Cases: Sheet already exists (may error or ignore depending on Google Sheets API behavior), permissions issues  
  - Version: 4.7

- **Run a workflow**  
  - Type: BrowserAct node (runs a BrowserAct workflow)  
  - Configuration: Runs an external BrowserAct workflow with ID `57142458383023994` (the scraping template)  
  - Input Parameters:  
    - `Competitor_Store_Link`: URL of competitor store  
    - `Pagination_Type`: How pagination works on the competitor site  
    - `Total_Product`: Number of products to scrape (fixed at 10 in this setup)  
  - Credentials: BrowserAct API key  
  - Output: A task ID to fetch results later  
  - Failure Cases: API authentication failure, invalid workflow ID, scraping failures due to site changes or anti-bot measures  
  - Version: 1

- **Get workflow Data**  
  - Type: BrowserAct node  
  - Configuration: Gets the task result from BrowserAct based on the task ID from the previous node, waits for completion  
  - Credentials: Same BrowserAct API key  
  - Output: Contains scraper output data as JSON string in `output.string`  
  - Failure Cases: Task failure, timeout, API failure  
  - Version: 1

- **Parse Json**  
  - Type: Code node (JavaScript)  
  - Configuration: Parses the JSON string from the scraper output to convert it into an array of items for n8n processing  
  - Key Expressions: Reads `input.first().json.output.string`  
  - Failure Cases: Empty or missing JSON string, malformed JSON, data not an array  
  - Version: 2

- **Store Data**  
  - Type: Google Sheets node  
  - Configuration: Appends or updates rows in the competitor’s dedicated sheet, matching on `Name` column to update existing products or add new ones  
  - Columns Mapped: `Name`, `Price`  
  - Credentials: Google Sheets OAuth2  
  - Failure Cases: Sheet not found, permission issues, data format issues  
  - Version: 4.7

---

#### 2.3 Compare & Alert

**Overview:**  
This block fetches both the newly scraped product list and the previously stored product list from the competitor’s sheet, compares them for new product entries, and sends a Slack alert if differences are detected.

**Nodes Involved:**  
- Get row(s) for Compare  
- Get Data For Compare  
- Merge  
- Compare Datas (Code node)  
- Check New Product (If node)  
- Send a message (Slack node)

**Node Details:**

- **Get row(s) for Compare**  
  - Type: Google Sheets node  
  - Configuration: Reads all rows from the competitor’s dedicated sheet to get the previous product list  
  - Credentials: Google Sheets OAuth2  
  - Execute Once: true (fetches all data once per loop iteration)  
  - Failure Cases: Sheet not found, empty sheet  
  - Version: 4.7

- **Get Data For Compare**  
  - Type: Google Sheets node  
  - Configuration: Reads all rows from the competitor’s dedicated sheet again to get the newly stored product list after scraping  
  - Credentials: Google Sheets OAuth2  
  - Execute Once: true  
  - Failure Cases: Same as above  
  - Version: 4.7

- **Merge**  
  - Type: Merge node  
  - Configuration: Combines input data streams from `Get row(s) for Compare` and `Get Data For Compare` nodes to feed into the comparison code  
  - Mode: Choose Branch  
  - Use Data of Input 2  
  - Failure Cases: No data on one branch, data mismatch  
  - Version: 3.2

- **Compare Datas**  
  - Type: Code node (JavaScript)  
  - Configuration: Compares two arrays of product objects by checking if any product in the newly scraped list does not exist in the previously stored list based on product `Name`  
  - Logic:  
    - Creates a Set of names from the old list  
    - Checks if any name in the new list is missing in old list  
    - Outputs a single item with `{ Alert: true/false, Message: string }`  
  - Failure Cases: Null or missing fields; empty lists; data type mismatches  
  - Version: 2

- **Check New Product**  
  - Type: If node  
  - Configuration: Checks if `Alert` is `true` in the code node output  
  - If true: proceeds to send Slack message  
  - Else: loops back to next competitor for processing  
  - Failure Cases: Missing Alert field, type mismatch  
  - Version: 2.2

- **Send a message**  
  - Type: Slack node  
  - Configuration: Sends a formatted Slack message to channel ID `C09LWT82KHN` (named `new_product_added`)  
  - Message content dynamically uses competitor `Name` and date/time from schedule trigger  
  - Authentication: OAuth2 with Slack credentials  
  - Failure Cases: Slack API auth failure, invalid channel ID, message formatting errors  
  - Version: 2.3

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                                  | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                          |
|-------------------------|-------------------------------|-------------------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger              | Periodic trigger to start workflow               |                        | Get row(s) in sheet      | "Schedule Trigger kicks off the entire process automatically." (Sticky Note - Input & Loop)                                          |
| Get row(s) in sheet     | Google Sheets                 | Reads master competitor store list                | Schedule Trigger        | Loop Over Items           | "Fetches your master list of competitor stores to be monitored." (Sticky Note - Input & Loop)                                        |
| Loop Over Items         | SplitInBatches                | Processes each competitor individually            | Get row(s) in sheet     | Create sheet (branch 2), else end branch | "Processes each competitor one-by-one." (Sticky Note - Input & Loop)                                                                 |
| Create sheet            | Google Sheets                 | Creates competitor-specific sheet if not exists  | Loop Over Items         | Run a workflow           | "Creates a dedicated sheet for each competitor." (Sticky Note - Scrape & Store)                                                      |
| Run a workflow          | BrowserAct Workflow           | Runs BrowserAct scraper for competitor products   | Create sheet            | Get workflow Data        | "Runs scraper to get fresh product list from competitor." (Sticky Note - Scrape & Store)                                              |
| Get workflow Data       | BrowserAct API                | Retrieves scraper results                          | Run a workflow          | Parse Json               | "Fetches completed scraper results." (Sticky Note - Scrape & Store)                                                                  |
| Parse Json              | Code                         | Parses scraper JSON output into items             | Get workflow Data       | Get row(s) for Compare, Store Data | "Parses JSON string output from scraper into array of product items." (Sticky Note - Scrape & Store)                                  |
| Store Data              | Google Sheets                 | Appends or updates competitor product data        | Parse Json              | Get Data For Compare      | "Stores freshly scraped product list for historical tracking." (Sticky Note - Scrape & Store)                                         |
| Get Data For Compare    | Google Sheets                 | Retrieves newly stored data for comparison        | Store Data              | Merge                    |                                                                                                                                      |
| Get row(s) for Compare  | Google Sheets                 | Retrieves old stored data for comparison          |                        | Merge                    |                                                                                                                                      |
| Merge                   | Merge                        | Combines old and new product lists                 | Get row(s) for Compare, Get Data For Compare | Compare Datas             | "Fetches both new and old product lists for comparison." (Sticky Note - Compare & Alert)                                              |
| Compare Datas           | Code                         | Compares product lists, detects new products      | Merge                   | Check New Product         | "Compares lists and detects new product names." (Sticky Note - Compare & Alert)                                                      |
| Check New Product       | If                           | Checks if new products were found                  | Compare Datas            | Send a message (true), Loop Over Items (false) | "If new products detected, send Slack alert." (Sticky Note - Compare & Alert)                                                          |
| Send a message          | Slack                        | Sends Slack alert about new products               | Check New Product        | Loop Over Items           | "Sends Slack alert for new products detected." (Sticky Note - Compare & Alert)                                                        |
| Sticky Note - Intro     | Sticky Note                  | Workflow overview and introduction                  |                        |                          | "This template is an advanced competitive intelligence tool..."                                                                      |
| Sticky Note - How to Use| Sticky Note                  | Instructions for setup and usage                     |                        |                          | "How to use: Credentials, BrowserAct template, Google Sheet setup, Slack Channel, Activate workflow."                                  |
| Sticky Note - Need Help | Sticky Note                  | Help resources and video links                       |                        |                          | "Video tutorials and help links for BrowserAct and n8n integration."                                                                 |
| Sticky Note - Input & Loop | Sticky Note                | Notes on schedule trigger, Google Sheets, looping  |                        |                          | "Setup & Loop: Schedule trigger, fetch competitor list, loop over items."                                                             |
| Sticky Note - Scrape & Store | Sticky Note              | Notes on scraping and storage logic                  |                        |                          | "Scrape & Store: Create sheet, BrowserAct scrape, parse, store data."                                                                 |
| Sticky Note - Compare & Alert | Sticky Note              | Notes on comparison and alert logic                   |                        |                          | "Compare & Alert: Fetch old/new data, compare, alert on new products."                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set interval to your desired frequency (e.g., every hour).  
   - This node will start the workflow automatically.

3. **Add a Google Sheets node ("Get row(s) in sheet"):**  
   - Operation: Read rows  
   - Document ID: Your Google Sheet storing the competitor list  
   - Sheet Name: `Competitor Store List`  
   - Credentials: Configure with your Google Sheets OAuth2 credentials  
   - Connect Schedule Trigger → Get row(s) in sheet

4. **Add a SplitInBatches node ("Loop Over Items"):**  
   - Default batch size = 1 (process one competitor at a time)  
   - Connect Get row(s) in sheet → Loop Over Items

5. **Add a Google Sheets node ("Create sheet"):**  
   - Operation: Create sheet  
   - Document ID: Same as before  
   - Title: Set expression to competitor name: `={{ $json.Name }}`  
   - Credentials: Google Sheets OAuth2  
   - Connect Loop Over Items → Create sheet

6. **Add a BrowserAct node ("Run a workflow"):**  
   - Operation: Run a workflow  
   - Workflow ID: Enter your BrowserAct workflow ID for Shopify scraping  
   - Input Parameters (array of objects):  
     - Name: `Competitor_Store_Link`, Value: `={{ $json.Link }}`  
     - Name: `Pagination_Type`, Value: `={{ $json["Pagination Type"] }}`  
     - Name: `Total_Product`, Value: `10` (or adjust as needed)  
   - Credentials: BrowserAct API  
   - Connect Create sheet → Run a workflow

7. **Add a BrowserAct node ("Get workflow Data"):**  
   - Operation: Get task result  
   - Task ID: Expression: `={{ $json.id }}` (from previous node output)  
   - Wait for finish: true  
   - Credentials: BrowserAct API  
   - Connect Run a workflow → Get workflow Data

8. **Add a Code node ("Parse Json"):**  
   - JavaScript code (parse JSON string from BrowserAct output):  
     ```js
     const jsonString = $input.first().json.output.string;
     if (!jsonString) throw new Error("Missing JSON string");
     const parsedData = JSON.parse(jsonString);
     if (!Array.isArray(parsedData)) throw new Error("Parsed data not an array");
     return parsedData.map(item => ({ json: item }));
     ```
   - Connect Get workflow Data → Parse Json

9. **Add a Google Sheets node ("Store Data"):**  
   - Operation: Append or update rows  
   - Document ID: Same as before  
   - Sheet Name: Use competitor name (pass dynamically or reuse from Loop Over Items context)  
   - Columns: Map `Name` and `Price` columns automatically  
   - Credentials: Google Sheets OAuth2  
   - Connect Parse Json → Store Data

10. **Add another Google Sheets node ("Get Data For Compare"):**  
    - Operation: Read rows  
    - Document ID and Sheet Name: same as Store Data node  
    - Credentials: Google Sheets OAuth2  
    - Execute Once: true  
    - Connect Store Data → Get Data For Compare

11. **Add a Google Sheets node ("Get row(s) for Compare"):**  
    - Operation: Read rows  
    - Document ID and Sheet Name: same competitor sheet  
    - Credentials: Google Sheets OAuth2  
    - Execute Once: true  
    - Connect from Loop Over Items (or relevant prior node) → Get row(s) for Compare

12. **Add a Merge node:**  
    - Mode: Choose Branch  
    - Use Data of Input 2  
    - Connect Get row(s) for Compare and Get Data For Compare into Merge node inputs

13. **Add a Code node ("Compare Datas"):**  
    - JavaScript code to compare arrays and detect new products:  
      ```js
      const oldList = $input.all()[0].json;
      const newList = $input.all()[1].json;

      const oldNames = new Set($input.all()[0].json.map(item => item.Name));
      const isNewProduct = $input.all()[1].json.some(item => !oldNames.has(item.Name));

      return [{
        json: {
          Alert: isNewProduct,
          Message: isNewProduct ? "New products found." : "No new products."
        }
      }];
      ```
    - Connect Merge → Compare Datas

14. **Add an If node ("Check New Product"):**  
    - Condition: Check if `Alert` equals `true`  
    - Connect Compare Datas → If

15. **Add a Slack node ("Send a message"):**  
    - Authentication: OAuth2 Slack credentials  
    - Channel ID: your Slack channel ID for alerts  
    - Message text:  
      ```
      New Product Added to {{ $('Loop Over Items').first().json.Name }}
      Website Please Check it out
      ------------------------------------------------------
      {{ $('Schedule Trigger').first().json['Readable date'] }}
      ```
    - Connect If (true) → Send a message

16. **Connect the Send a message node back to Loop Over Items to continue processing competitors after alert.**

17. **Connect If (false) output back to Loop Over Items to continue processing without alert.**

18. **Add sticky notes with instructions and links as needed for user guidance.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                         | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This template is an advanced competitive intelligence tool that automatically monitors competitor Shopify stores and alerts you when they launch a new product. It requires BrowserAct API, Google Sheets, and Slack credentials.                                   | Sticky Note - Intro                                                                                       |
| Use the BrowserAct template named "Competitors Shopify Website New Product Monitor" in your BrowserAct account.                                                                                                                                                     | Sticky Note - How to Use                                                                                  |
| Prepare a Google Sheet with a sheet named `Competitor Store List` containing columns: `Name`, `Link`, and `Pagination Type`.                                                                                                                                          | Sticky Note - How to Use                                                                                  |
| Slack channel ID must be updated to your desired alert channel in the Slack node configuration.                                                                                                                                                                      | Sticky Note - How to Use                                                                                  |
| Helpful video tutorials and guides are available for BrowserAct API key retrieval, BrowserAct n8n integration, BrowserAct template usage, and competitive monitoring automation.                                                                                      | Sticky Note - Need Help                                                                                   |
| BrowserAct n8n Community Node package: https://www.npmjs.com/package/n8n-nodes-browseract-workflows                                                                                                                                                                  | From description                                                                                          |
| Join BrowserAct Discord for help: https://discord.com/invite/UpnCKd7GaU                                                                                                                                                                                             | From description                                                                                          |
| BrowserAct blog for extended tutorials and updates: https://www.browseract.com/blog                                                                                                                                                                                  | From description                                                                                          |

---

**Disclaimer:**  
The provided content is derived exclusively from an n8n workflow automation. It complies with all applicable content policies and contains no illegal, offensive, or protected elements. All data handled is publicly accessible and legal.