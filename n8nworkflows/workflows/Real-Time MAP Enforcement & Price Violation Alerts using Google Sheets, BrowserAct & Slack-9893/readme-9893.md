Real-Time MAP Enforcement & Price Violation Alerts using Google Sheets, BrowserAct & Slack

https://n8nworkflows.xyz/workflows/real-time-map-enforcement---price-violation-alerts-using-google-sheets--browseract---slack-9893


# Real-Time MAP Enforcement & Price Violation Alerts using Google Sheets, BrowserAct & Slack

### 1. Workflow Overview

This workflow automates the enforcement of MAP (Minimum Advertised Price) policies by monitoring reseller product pricing in real time. It is designed for brands or retailers who want to ensure their resellers comply with agreed pricing rules and to receive immediate alerts on violations or out-of-stock situations.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & Looping:**  
  This block triggers the workflow on a schedule, retrieves reseller and product data from Google Sheets, and iterates over each reseller entry one by one.

- **1.2 Price Scraping & Data Parsing:**  
  For each reseller, the workflow invokes a BrowserAct task to scrape the live product page for current pricing data, and then parses the scraped JSON output into usable data items.

- **1.3 Violation Logic & Alerting:**  
  This block evaluates the scraped price against the approved MAP price and detects "out of stock" conditions. It sends detailed Slack notifications for price violations or out-of-stock alerts, then loops back to process the next reseller.

This modular structure ensures efficient, real-time monitoring and clear, actionable alerts to facilitate MAP enforcement.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Looping

- **Overview:**  
  This block triggers the workflow on a scheduled interval, fetches the list of resellers and their product details from a Google Sheet, and splits the data so each reseller is processed individually in a loop.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get row(s) in sheet (Google Sheets)  
  - Loop Over Items (SplitInBatches)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node for scheduled execution  
    - Configuration: Runs at a regular interval (default: every hour) to start the workflow automatically  
    - Inputs: None (trigger only)  
    - Outputs: Triggers downstream nodes  
    - Edge Cases: Misconfiguration of interval may cause too frequent or infrequent runs  

  - **Get row(s) in sheet**  
    - Type: Google Sheets node  
    - Configuration: Reads all rows from the specified Google Sheet and Sheet tab ("MAP Violation Alerts", gid=0) containing reseller data (Reseller_URL, Reseller_Name, Product_SKU, AP_Price)  
    - Credentials: Google Sheets OAuth2 credentials required  
    - Inputs: Triggered by Schedule Trigger  
    - Outputs: Array of reseller entries as JSON items  
    - Edge Cases: Authentication failure, sheet access permissions, empty or malformed sheet data  

  - **Loop Over Items**  
    - Type: SplitInBatches node  
    - Configuration: Defaults to processing one item at a time to avoid data mix-up  
    - Inputs: Receives array of reseller entries from Google Sheets  
    - Outputs: Passes each reseller entry individually downstream  
    - Edge Cases: Large data sets may slow down processing; batch size configurable  

---

#### 2.2 Price Scraping & Data Parsing

- **Overview:**  
  For each reseller, this block runs a BrowserAct workflow task to scrape the live reseller product page to extract current pricing data. The workflow then fetches the task results and parses the JSON string output into discrete data items.

- **Nodes Involved:**  
  - Run a workflow task (BrowserAct)  
  - Get details of a workflow task (BrowserAct)  
  - Code in JavaScript

- **Node Details:**

  - **Run a workflow task**  
    - Type: BrowserAct node for running a scraping task  
    - Configuration: Calls a specific BrowserAct workflow ("MAP Violation Alerts" template) with input parameter `Target_Link` set dynamically from the current reseller's `Reseller_URL`  
    - Credentials: BrowserAct API credentials required  
    - Inputs: Receives one reseller item from Loop Over Items  
    - Outputs: Returns a task object containing the task ID  
    - Edge Cases: API rate limits, network timeouts, invalid URLs, task creation failures  

  - **Get details of a workflow task**  
    - Type: BrowserAct node to fetch task results  
    - Configuration: Uses task ID from previous node, waits for task completion before returning result  
    - Inputs: Receives task info from Run a workflow task  
    - Outputs: Returns the scraping results, including a JSON string with pricing data  
    - Edge Cases: Task timeouts, incomplete results, API errors  

  - **Code in JavaScript**  
    - Type: Code node (JavaScript)  
    - Configuration: Parses the JSON string output (`$input.first().json.output.string`) from the scraping task, validates it, and converts the array of objects into individual n8n items for further processing  
    - Key Expressions: Uses `$input.first().json.output.string` to extract raw JSON string  
    - Inputs: Receives scraping results JSON string  
    - Outputs: Multiple items parsed from JSON array, each representing a price data point  
    - Edge Cases: Empty or missing JSON string, malformed JSON causing parse errors, non-array data structure triggering error  

---

#### 2.3 Violation Logic & Alerting

- **Overview:**  
  This block analyzes the parsed price data for each reseller product and determines if there are MAP violations or out-of-stock conditions. It sends tailored alerts to a Slack channel based on the detected condition, then merges processing branches to continue looping.

- **Nodes Involved:**  
  - If1 (Check for 'NoData' out-of-stock)  
  - Send a message1 (Slack alert for out-of-stock)  
  - If (Price violation check)  
  - Send a message (Slack alert for MAP violation)  
  - Merge (Combining branches back into loop)

- **Node Details:**

  - **If1 (Out of Stock Check)**  
    - Type: Conditional node  
    - Configuration: Checks if scraped `Price` field equals the string `"NoData"`, indicating the product is out of stock or data not found  
    - Inputs: Parsed price data from Code node  
    - Outputs:  
      - True branch: Out-of-stock alert path  
      - False branch: Passes to price violation check  
    - Edge Cases: Price field missing or null may cause false negatives  

  - **Send a message1 (Slack Out-of-Stock Alert)**  
    - Type: Slack node for sending messages  
    - Configuration: Sends a formatted Slack message noting the reseller is out of stock for the product, including reseller name, SKU, URL, and date from Schedule Trigger  
    - Credentials: Slack OAuth2 credentials required  
    - Inputs: True branch of If1  
    - Outputs: Goes to Merge node  
    - Edge Cases: Slack API rate limits, invalid channel ID, authentication failures  

  - **If (Price Violation Check)**  
    - Type: Conditional node  
    - Configuration: Two conditions combined with AND:  
      1. Checks that scraped `Price` exists (is defined).  
      2. Compares scraped `Price` against approved price `AP_Price` from Google Sheets; triggers if scraped price is less than approved price.  
    - Inputs: False branch of If1  
    - Outputs:  
      - True branch: Price violation alert  
      - False branch: Ends processing for this item (no violation)  
    - Edge Cases: Price data type mismatches (strings vs numbers), missing AP_Price, comparison errors  

  - **Send a message (Slack MAP Violation Alert)**  
    - Type: Slack node  
    - Configuration: Sends a detailed Slack alert about the reseller breaking MAP rules, including reseller name, SKU, approved price, current price, product URL, and date  
    - Credentials: Slack OAuth2 credentials required  
    - Inputs: True branch of If  
    - Outputs: Goes to Merge node  
    - Edge Cases: Same as Send a message1  

  - **Merge**  
    - Type: Merge node  
    - Configuration: Combines three branches:  
      1. False branch of price violation (no action)  
      2. Slack MAP violation alert branch  
      3. Slack out-of-stock alert branch  
    - Outputs: Loops back to Loop Over Items for next reseller  
    - Edge Cases: None significant; ensures workflow continuity  

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                          | Input Node(s)              | Output Node(s)                 | Sticky Note                                                  |
|-------------------------|---------------------------------|----------------------------------------|----------------------------|-------------------------------|--------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                 | Starts workflow on schedule             | None                       | Get row(s) in sheet            | See "Input & Loop" note about automatic scheduled start      |
| Get row(s) in sheet    | Google Sheets                   | Reads reseller data from Google Sheet   | Schedule Trigger           | Loop Over Items                | See "Input & Loop" note about reading master reseller list  |
| Loop Over Items         | SplitInBatches                  | Processes each reseller entry individually | Get row(s) in sheet       | Run a workflow task            | See "Input & Loop" note about looping per reseller          |
| Run a workflow task     | BrowserAct                      | Launches BrowserAct scraping task       | Loop Over Items            | Get details of a workflow task | See "Price Scraping" note on BrowserAct scraping             |
| Get details of a workflow task | BrowserAct              | Fetches scraping results                 | Run a workflow task        | Code in JavaScript             | See "Price Scraping" note on fetching results                |
| Code in JavaScript      | Code                           | Parses JSON string output into items    | Get details of a workflow task | If1                       | See "Price Scraping" note about parsing scraper output       |
| If1                     | If                             | Checks if product is out of stock ("NoData") | Code in JavaScript       | Send a message1 (true), If (false) | See "Logic & Alerting" note on out-of-stock check           |
| Send a message1         | Slack                          | Sends out-of-stock Slack notification   | If1 (true)                 | Merge                         | See "Logic & Alerting" note on out-of-stock Slack alert      |
| If                      | If                             | Checks for MAP price violation           | If1 (false)                | Send a message (true), Merge (false) | See "Logic & Alerting" note on price violation check        |
| Send a message          | Slack                          | Sends MAP violation Slack notification  | If (true)                  | Merge                         | See "Logic & Alerting" note on MAP violation Slack alert     |
| Merge                   | Merge                          | Combines all alerting branches and continues loop | Send a message1, Send a message, If (false) | Loop Over Items | See "Logic & Alerting" note on merging branches             |
| Sticky Note - Intro     | Sticky Note                    | Intro and overview of workflow           | None                       | None                         | Introductory explanation and usage overview                  |
| Sticky Note - How to Use| Sticky Note                    | Instructions for setup and use           | None                       | None                         | Setup instructions for credentials, sheets, Slack, BrowserAct|
| Sticky Note - Need Help | Sticky Note                    | Provides help links and resources        | None                       | None                         | Video tutorials and documentation links                      |
| Sticky Note - Input & Loop | Sticky Note                  | Explains input and looping logic         | None                       | None                         | Describes schedule trigger, Google Sheets, and looping nodes |
| Sticky Note - Data Collection | Sticky Note               | Describes price scraping and parsing     | None                       | None                         | Explains BrowserAct scraping and code node parsing           |
| Sticky Note - Logic & Alerting | Sticky Note               | Explains violation logic and alerts      | None                       | None                         | Details alerting logic with If nodes and Slack notifications |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to desired frequency (e.g., every hour) to automate checking.

2. **Create Google Sheets node ("Get row(s) in sheet"):**  
   - Type: Google Sheets  
   - Set Document ID to your Google Sheet containing reseller data.  
   - Set Sheet Name to the correct tab (gid=0).  
   - Use Google Sheets OAuth2 credentials.  
   - Configure to read rows with columns: `Reseller_URL`, `Reseller_Name`, `Product_SKU`, `AP_Price`.

3. **Create SplitInBatches node ("Loop Over Items"):**  
   - Type: SplitInBatches  
   - Connect it to the Google Sheets node.  
   - Default batch size (1) to process each reseller individually.

4. **Create BrowserAct node ("Run a workflow task"):**  
   - Type: BrowserAct (n8n community node)  
   - Set Workflow ID to the BrowserAct template workflow ID for MAP alerts.  
   - Set input parameter `Target_Link` to `={{ $json.Reseller_URL }}` dynamically.  
   - Use BrowserAct API credentials.

5. **Create BrowserAct node ("Get details of a workflow task"):**  
   - Type: BrowserAct  
   - Set operation to "getTask" with parameter `taskId` from the previous node's output (`={{ $json.id }}`).  
   - Enable "waitForFinish" to get completed results.  
   - Use same BrowserAct credentials.

6. **Create Code node ("Code in JavaScript"):**  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript code that:  
     - Extracts the JSON string from the scraping output (`$input.first().json.output.string`)  
     - Parses it into an array  
     - Throws errors if missing or malformed  
     - Outputs each item as a separate n8n item.

7. **Create If node ("If1") to check for out-of-stock:**  
   - Type: If  
   - Condition: Check if `Price` equals `"NoData"` (string match, case sensitive).  
   - True branch leads to out-of-stock alert.  
   - False branch leads to price violation check.

8. **Create Slack node ("Send a message1") for out-of-stock alert:**  
   - Type: Slack  
   - Configure OAuth2 Slack credentials.  
   - Set message text with dynamic fields for reseller name, product SKU, URL, and current date from Schedule Trigger.  
   - Set channel ID to your alert Slack channel.

9. **Create If node ("If") to check for MAP price violation:**  
   - Type: If  
   - Conditions (AND):  
     - `Price` exists (not empty)  
     - `Price` < `AP_Price` (price from Google Sheet)  
   - True branch leads to MAP violation alert.  
   - False branch continues without alert.

10. **Create Slack node ("Send a message") for MAP violation alert:**  
    - Type: Slack  
    - Configure OAuth2 Slack credentials.  
    - Set message text to include reseller name, SKU, approved price, current price, URL, and date.  
    - Set channel ID same as above.

11. **Create Merge node:**  
    - Type: Merge  
    - Mode: Choose Branch  
    - Number of inputs: 3  
    - Connect:  
      - False branch of price violation If node (no alert)  
      - MAP violation Slack node output  
      - Out-of-stock Slack node output  
    - Output connects back to "Loop Over Items" to continue processing resellers.

12. **Connect all nodes as described:**  
    - Schedule Trigger → Google Sheets → Loop Over Items → Run a workflow task → Get details of a workflow task → Code in JavaScript → If1 → (True) Send a message1 → Merge  
    - If1 (False) → If → (True) Send a message → Merge  
    - If (False) → Merge → Loop Over Items (continue)  

13. **Add sticky notes for documentation and usage instructions:**  
    - Include notes about setup, credentials, BrowserAct template, Slack channel, and help resources as per original workflow.

14. **Activate the workflow:**  
    - Test with real data to ensure proper scraping, parsing, and alerting.  
    - Adjust schedule frequency or batch size as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| BrowserAct n8n Community Node is required: [n8n Nodes BrowserAct](https://www.npmjs.com/package/n8n-nodes-browseract-workflows) | Required for scraping reseller product pages                                                        |
| BrowserAct template needed: “MAP (Minimum Advertised Price) Violation Alerts”                                                   | Template workflow in BrowserAct for scraping MAP pricing                                           |
| Slack OAuth2 credentials must have permission to post messages in the target channel                                           | Setup Slack app and OAuth2 token with chat:write permissions                                       |
| Google Sheets OAuth2 credentials must have read access to your price list sheet                                                | Ensure the sheet has columns: Reseller_URL, Reseller_Name, Product_SKU, AP_Price                   |
| Helpful video tutorials for setup and customization:                                                                            | https://www.youtube.com/watch?v=pDjoZWEsZlE, https://www.youtube.com/watch?v=RoYMdJaRdcQ, and others |
| Join BrowserAct Discord for support:                                                                                           | https://discord.com/invite/UpnCKd7GaU                                                              |
| BrowserAct blog with additional tips:                                                                                          | https://www.browseract.com/blog                                                                     |

---

**Disclaimer:** The provided content is a detailed analysis and structured documentation of an n8n workflow designed for MAP enforcement automation. It complies with all current content policies and handles only publicly available, legal data.