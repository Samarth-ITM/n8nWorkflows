Enrich leads from Google Sheets with ScraperCity emails, phones, and validation

https://n8nworkflows.xyz/workflows/enrich-leads-from-google-sheets-with-scrapercity-emails--phones--and-validation-13854


# Enrich leads from Google Sheets with ScraperCity emails, phones, and validation

### 1. Workflow Overview

This workflow automates a comprehensive B2B lead enrichment process. It takes basic contact information (name and domain) from a Google Sheet and uses the ScraperCity API to find professional email addresses, mobile phone numbers, and verify email deliverability.

The logic is organized into five functional blocks:
*   **1.1 Configuration & Input:** Sets up environment variables and imports lead data.
*   **1.2 Email Discovery:** Submits leads to ScraperCity to find potential business emails and handles asynchronous job polling.
*   **1.3 Phone Discovery:** Takes discovered emails and searches for associated mobile numbers.
*   **1.4 Email Validation:** Verifies the deliverability and "catch-all" status of discovered emails.
*   **1.5 Data Consolidation & Export:** Merges results from all three ScraperCity services into a single record and updates the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration & Input
This block initializes the workflow and retrieves the source data.
*   **Nodes Involved:** `Start Enrichment`, `Configure Workflow`, `Read Contacts from Sheet`.
*   **Node Details:**
    *   **Configure Workflow (Set):** Defines the `inputDocumentId`, `inputSheetName`, `outputDocumentId`, and `outputSheetName`.
    *   **Read Contacts from Sheet (Google Sheets):** Reads all rows from the specified input sheet. Requires Google Sheets OAuth2 credentials.

#### 2.2 Email Discovery (ScraperCity Email Finder)
This block converts names and domains into emails.
*   **Nodes Involved:** `Format Contacts for Email Finder`, `Start Email Finder Job`, `Store Email Finder Run ID`, `Wait Before Checking Email Finder`, `Poll Email Finder Loop`, `Check Email Finder Status`, `Is Email Finder Complete?`, `Wait Before Next Email Finder Poll`, `Download Email Finder Results`, `Parse Email Finder CSV`, `Remove Duplicate Emails`, `Filter Valid Emails Only`.
*   **Node Details:**
    *   **Start Email Finder Job (HTTP Request):** Sends a POST request to `https://scrapercity.com/api/v1/scrape/email-finder`.
    *   **Polling Logic:** Uses `Split In Batches` (40 iterations) and `Wait` (60s) nodes to check the status at `/api/v1/scrape/status/{runId}` until it returns `SUCCEEDED`.
    *   **Data Parsing:** The CSV result is parsed into JSON. Duplicates are removed and empty email results are filtered out.

#### 2.3 Phone Discovery (ScraperCity Mobile Finder)
Searches for mobile numbers using the discovered emails.
*   **Nodes Involved:** `Format Emails for Mobile Finder`, `Start Mobile Finder Job`, `Store Mobile Finder Run ID`, `Wait Before Checking Mobile Finder`, `Poll Mobile Finder Loop`, `Check Mobile Finder Status`, `Is Mobile Finder Complete?`, `Wait Before Next Mobile Finder Poll`, `Download Mobile Finder Results`, `Parse Mobile Finder CSV`.
*   **Node Details:**
    *   **Start Mobile Finder Job (HTTP Request):** Sends a POST to `/api/v1/scrape/mobile-finder`.
    *   **Asynchronous Handling:** Follows the same 40-iteration polling pattern as the Email Finder.
    *   **Normalization:** The `Parse Mobile Finder CSV` node maps the API's `input` field back to `email` to facilitate later merging.

#### 2.4 Email Validation (ScraperCity Email Validator)
Checks the quality and safety of the discovered emails.
*   **Nodes Involved:** `Format Emails for Validator`, `Start Email Validator Job`, `Store Validator Run ID`, `Wait Before Checking Validator`, `Poll Validator Loop`, `Check Validator Status`, `Is Validator Complete?`, `Wait Before Next Validator Poll`, `Download Validator Results`, `Parse Validator CSV`.
*   **Node Details:**
    *   **Start Email Validator Job (HTTP Request):** POST to `/api/v1/scrape/email-validator`.
    *   **Verification:** Checks for MX records, deliverability, and catch-all status.

#### 2.5 Data Consolidation & Export
*   **Nodes Involved:** `Prepare Merge Inputs`, `Merge All Enrichment Data`, `Write Enriched Leads to Sheet`.
*   **Node Details:**
    *   **Prepare Merge Inputs (Set):** Uses the `.all()` method to collect all items from the three distinct parsing nodes (`Parse Email Finder CSV`, `Parse Mobile Finder CSV`, and `Parse Validator CSV`) into JSON strings.
    *   **Merge All Enrichment Data (Code):** Uses a Javascript Map to join the three datasets based on the email address as the primary key.
    *   **Write Enriched Leads to Sheet (Google Sheets):** Uses the `appendOrUpdate` operation. It matches existing leads by email and updates/adds columns for confidence scores, phone numbers, and validation statuses.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start Enrichment | manualTrigger | Trigger | (None) | Configure Workflow | |
| Configure Workflow | set | Variable Setup | Start Enrichment | Read Contacts from Sheet | The Configure Workflow node is the single place to set all user-configurable variables... |
| Read Contacts from Sheet | googleSheets | Input Data | Configure Workflow | Format Contacts for Email Finder | Set your input and output Google Sheets document IDs here |
| Format Contacts for Email Finder | code | Data Prep | Read Contacts from Sheet | Start Email Finder Job | |
| Start Email Finder Job | httpRequest | API Call | Format Contacts for Email Finder | Store Email Finder Run ID | Add your ScraperCity API key credential here |
| Store Email Finder Run ID | set | Data Storage | Start Email Finder Job | Wait Before Checking Email Finder | |
| Wait Before Checking Email Finder | wait | Delay | Store Email Finder Run ID | Poll Email Finder Loop | |
| Poll Email Finder Loop | splitInBatches | Loop Control | Wait Before Checking... | Check Email Finder Status | Each polling loop retries up to 40 times... |
| Check Email Finder Status | httpRequest | Status Check | Poll Email Finder Loop | Is Email Finder Complete? | |
| Is Email Finder Complete? | if | Logic Gate | Check Email Finder... | Download Email Finder Results, Wait Before Next... | |
| Wait Before Next Email Finder Poll | wait | Delay | Is Email Finder Complete? | Poll Email Finder Loop | |
| Download Email Finder Results | httpRequest | Data Fetch | Is Email Finder Complete? | Parse Email Finder CSV | |
| Parse Email Finder CSV | code | Data Parsing | Download Email Finder Results | Remove Duplicate Emails | |
| Remove Duplicate Emails | removeDuplicates | Cleanup | Parse Email Finder CSV | Filter Valid Emails Only | |
| Filter Valid Emails Only | filter | Cleanup | Remove Duplicate Emails | Format Emails for Mobile Finder | |
| Format Emails for Mobile Finder | code | Data Prep | Filter Valid Emails Only | Start Mobile Finder Job | |
| Start Mobile Finder Job | httpRequest | API Call | Format Emails for Mobile Finder | Store Mobile Finder Run ID | |
| Store Mobile Finder Run ID | set | Data Storage | Start Mobile Finder Job | Wait Before Checking Mobile Finder | |
| Wait Before Checking Mobile Finder | wait | Delay | Store Mobile Finder Run ID | Poll Mobile Finder Loop | |
| Poll Mobile Finder Loop | splitInBatches | Loop Control | Wait Before Checking... | Check Mobile Finder Status | |
| Check Mobile Finder Status | httpRequest | Status Check | Poll Mobile Finder Loop | Is Mobile Finder Complete? | |
| Is Mobile Finder Complete? | if | Logic Gate | Check Mobile Finder... | Download Mobile Finder Results, Wait Before Next... | |
| Wait Before Next Mobile Finder Poll | wait | Delay | Is Mobile Finder Complete? | Poll Mobile Finder Loop | |
| Download Mobile Finder Results | httpRequest | Data Fetch | Is Mobile Finder Complete? | Parse Mobile Finder CSV | |
| Parse Mobile Finder CSV | code | Data Parsing | Download Mobile Finder Results | Format Emails for Validator | |
| Format Emails for Validator | code | Data Prep | Parse Mobile Finder CSV | Start Email Validator Job | |
| Start Email Validator Job | httpRequest | API Call | Format Emails for Validator | Store Validator Run ID | |
| Store Validator Run ID | set | Data Storage | Start Email Validator Job | Wait Before Checking Validator | |
| Wait Before Checking Validator | wait | Delay | Store Validator Run ID | Poll Validator Loop | |
| Poll Validator Loop | splitInBatches | Loop Control | Wait Before Checking Validator | Check Validator Status | |
| Check Validator Status | httpRequest | Status Check | Poll Validator Loop | Is Validator Complete? | |
| Is Validator Complete? | if | Logic Gate | Check Validator Status | Download Validator Results, Wait Before Next... | |
| Wait Before Next Validator Poll | wait | Delay | Is Validator Complete? | Poll Validator Loop | |
| Download Validator Results | httpRequest | Data Fetch | Is Validator Complete? | Parse Validator CSV | |
| Parse Validator CSV | code | Data Parsing | Download Validator Results | Prepare Merge Inputs | |
| Prepare Merge Inputs | set | Data Aggregation | Parse Validator CSV | Merge All Enrichment Data | Prepare Merge Inputs collects all three parsed result sets... |
| Merge All Enrichment Data | code | Logic/Merge | Prepare Merge Inputs | Write Enriched Leads to Sheet | Merge All Enrichment Data joins them by email address into one row... |
| Write Enriched Leads to Sheet | googleSheets | Output Data | Merge All Enrichment Data | (None) | ...appends or updates the output Google Sheet with all enriched columns... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Credentials:**
    *   Create **Google Sheets OAuth2** credentials.
    *   Create **HTTP Header Auth** credentials named "ScraperCity API Key" (Header Name: `Authorization`, Value: `Bearer YOUR_API_KEY`).
2.  **Initial Setup:**
    *   Create a **Manual Trigger** node.
    *   Connect a **Set** node ("Configure Workflow") with four string variables: `inputDocumentId`, `inputSheetName`, `outputDocumentId`, `outputSheetName`.
3.  **Data Ingestion:**
    *   Add a **Google Sheets** node ("Read Contacts from Sheet"). Set operation to `Read All Rows` using expressions to map the IDs from the previous Set node.
4.  **Enrichment Services (Repeat for Email Finder, Mobile Finder, and Validator):**
    *   **Format Node:** Add a **Code** node to transform data into an array (e.g., `contacts` for Email Finder, `inputs` for Mobile Finder).
    *   **Request Node:** Add an **HTTP Request** node. Method: POST. Body: JSON.
    *   **Polling Loop:** 
        *   Add a **Set** node to store the `runId`.
        *   Add a **Wait** node (30s).
        *   Add a **Split In Batches** node (Batch size 1, Max iterations 40).
        *   Add an **HTTP Request** node to GET the status from the ScraperCity status URL.
        *   Add an **If** node to check if `status === "SUCCEEDED"`.
        *   If **False**: Connect a **Wait** node (60s) back to the Split In Batches node.
        *   If **True**: Connect an **HTTP Request** node to GET the results from the ScraperCity downloads URL.
    *   **Parsing Node:** Add a **Code** node to parse the CSV string from the response into individual JSON items.
5.  **Merge & Output:**
    *   Add a **Set** node ("Prepare Merge Inputs") using expressions like `{{ JSON.stringify($('Node Name').all().map(i => i.json)) }}` for each of the three parsed outputs.
    *   Add a **Code** node ("Merge All Enrichment Data") to loop through the Email Finder list and attach properties from the Mobile Finder and Validator lists using email as a key.
    *   Add a **Google Sheets** node ("Write Enriched Leads to Sheet"). Set operation to `Append or Update`. Map the email as the matching column.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Async Jobs** | ScraperCity jobs can take 10-60 minutes. Polling is essential. |
| **Input Columns** | Source sheet requires: `first_name`, `last_name`, `domain`. |
| **Output Columns** | The workflow creates: `email`, `phone`, `phone_type`, `deliverability`, `email_status`, etc. |
| **ScraperCity API** | [ScraperCity Documentation](https://scrapercity.com) |