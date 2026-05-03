Enriching Brazilian Company Data with CNPJ API and Google Sheets

https://n8nworkflows.xyz/workflows/enriching-brazilian-company-data-with-cnpj-api-and-google-sheets-7457


# Enriching Brazilian Company Data with CNPJ API and Google Sheets

### 1. Workflow Overview

This workflow automates the enrichment of Brazilian company data by fetching detailed information from the MinhaReceita.org CNPJ API and updating a Google Sheets spreadsheet accordingly. It is designed to process batches of company identifiers (CNPJs) lacking detailed data in the sheet, retrieve up-to-date company details from a free, no-authentication API, and store the enriched data back in the spreadsheet. Upon completion, it sends a Telegram notification.

The workflow is logically divided into these blocks:

- **1.1 Initialization and Configuration**: Manual trigger and setting global parameters (Telegram ID, API base URL).  
- **1.2 Data Retrieval from Google Sheets**: Reading all CNPJs from the spreadsheet that lack company information.  
- **1.3 Batch Processing Loop**: Splitting the CNPJs into batches and iterating over each.  
- **1.4 External API Interaction**: Requesting detailed company data from MinhaReceita.org for each CNPJ.  
- **1.5 Data Update Back to Google Sheets**: Updating the corresponding spreadsheet rows with enriched data.  
- **1.6 Completion Notification**: Sending a Telegram message when all CNPJs have been processed, with a limiter to avoid notification spam.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Configuration

**Overview:**  
This block initiates the workflow manually and sets essential runtime parameters, including the Telegram chat ID for notifications and the base URL for the CNPJ API.

**Nodes Involved:**  
- Execute Workflow Manually  
- Settings  
- Sticky Note (Initial Setup)

**Node Details:**

- **Execute Workflow Manually**  
  - Type: Manual Trigger  
  - Role: Entry point, starts the workflow on user demand.  
  - Configuration: No parameters needed; simply triggers workflow execution.  
  - Connections: Outputs to "Settings".  
  - Edge Cases: None specific; user must trigger manually.

- **Settings**  
  - Type: Set Node  
  - Role: Defines global variables for Telegram ID and API base URL.  
  - Configuration:  
    - `telegram_id`: Placeholder string "YOUR_TELEGRAM_ID", to be replaced with actual Telegram chat ID.  
    - `api_base_url`: Set to "https://minhareceita.org/", the external API endpoint root.  
  - Connections: Inputs from "Execute Workflow Manually", outputs to "Get All CNPJs in Sheet".  
  - Edge Cases: Telegram ID must be valid for notifications; API base URL must be reachable.

- **Sticky Note (Initial Setup)**  
  - Type: Sticky Note  
  - Purpose: Provides user instructions to configure Telegram ID, verify API URL, and ensure Google Sheets credentials are active.  
  - Connections: None.  
  - Edge Cases: None.

---

#### 2.2 Data Retrieval from Google Sheets

**Overview:**  
Reads all company identifiers (CNPJs) from a specified Google Sheets document, filtering to only include rows where the 'razao_social' (company name) is empty, indicating missing data.

**Nodes Involved:**  
- Get All CNPJs in Sheet  
- Sticky Note (Data Source)

**Node Details:**

- **Get All CNPJs in Sheet**  
  - Type: Google Sheets Node  
  - Role: Reads rows from the Google Sheets document to obtain CNPJs needing enrichment.  
  - Configuration:  
    - Document ID and Sheet Name specified (Sheet GID=0).  
    - Filter applied on the column 'razao_social' to find empty entries.  
    - No row limit specified; retrieves all matching rows.  
  - Connections: Input from "Settings", output to "Loop CNPJs".  
  - Credentials: Uses configured Google Sheets OAuth2 credentials.  
  - Edge Cases: Permissions for Google Sheets must be valid; empty or malformed CNPJs in sheet could cause issues.

- **Sticky Note (Data Source)**  
  - Type: Sticky Note  
  - Content: Explains the data source, filtering by empty 'razao_social' to process only CNPJs without data.  
  - Connections: None.

---

#### 2.3 Batch Processing Loop

**Overview:**  
Splits the list of CNPJs into batches of 100 to manage API request load and iterates over each batch for further processing.

**Nodes Involved:**  
- Loop CNPJs  
- Sticky Note (API Details)

**Node Details:**

- **Loop CNPJs**  
  - Type: SplitInBatches  
  - Role: Controls batch size to process CNPJs in manageable groups (batch size = 100).  
  - Configuration: Batch size set to 100 items per execution cycle.  
  - Connections: Input from "Get All CNPJs in Sheet", outputs to both "Limit Notifier Interactions" and "Send API Request".  
  - Edge Cases: Large datasets handled efficiently; partial batches processed correctly.

- **Sticky Note (API Details)**  
  - Type: Sticky Note  
  - Content: Details about the API endpoint, data richness (+40 fields), free access, and updated official source.  
  - Connections: None.

---

#### 2.4 External API Interaction

**Overview:**  
For each CNPJ in the batch, an HTTP GET request is made to MinhaReceita.org to obtain comprehensive company data.

**Nodes Involved:**  
- Send API Request  
- Sticky Note (Insert New Data)

**Node Details:**

- **Send API Request**  
  - Type: HTTP Request  
  - Role: Queries the API with the CNPJ appended to the base URL to retrieve company data.  
  - Configuration:  
    - URL dynamically constructed as `{{ $('Settings').item.json.api_base_url }}{{ $json.cnpj }}`.  
    - No authentication needed.  
    - HTTP method: GET (default).  
  - Connections: Input from "Loop CNPJs", output to "Update CNPJ Info in Sheet".  
  - Edge Cases: API downtime, invalid CNPJs, or malformed responses could cause failures; no retry logic configured.

- **Sticky Note (Insert New Data)**  
  - Type: Sticky Note  
  - Content: Explains that data from the API response is inserted back into the spreadsheet corresponding to each CNPJ.  
  - Connections: None.

---

#### 2.5 Data Update Back to Google Sheets

**Overview:**  
Updates the Google Sheets spreadsheet rows with the enriched data obtained from the API for each CNPJ.

**Nodes Involved:**  
- Update CNPJ Info in Sheet

**Node Details:**

- **Update CNPJ Info in Sheet**  
  - Type: Google Sheets Node  
  - Role: Updates existing rows in the sheet with the new company data fields from the API response.  
  - Configuration:  
    - Operation: Update.  
    - Sheet and Document ID match the input sheet.  
    - Matching column: `row_number` used to identify the exact row to update.  
    - Columns mapped explicitly to numerous fields including address, contact, company nature, classification codes, and fiscal info.  
    - Converts fields as defined, preserving type integrity.  
  - Credentials: Uses Google Sheets OAuth2 credentials.  
  - Connections: Input from "Send API Request", output loops back to "Loop CNPJs" to continue processing next batch.  
  - Edge Cases: If row_number is missing or mismatched, updates may fail or overwrite wrong rows; API data fields may be incomplete.

---

#### 2.6 Completion Notification

**Overview:**  
After all batches and CNPJs are processed, a Telegram notification is sent once to confirm the completion of the data enrichment process.

**Nodes Involved:**  
- Limit Notifier Interactions  
- Notify When Finish  
- Sticky Note (Notifications)

**Node Details:**

- **Limit Notifier Interactions**  
  - Type: Limit  
  - Role: Ensures only one notification is sent per complete execution to avoid spamming Telegram.  
  - Configuration: Default limit of one.  
  - Connections: Inputs from "Loop CNPJs" (parallel output), output to "Notify When Finish".  
  - Edge Cases: Ensures no duplicate notifications even if workflow is retried.

- **Notify When Finish**  
  - Type: Telegram Node  
  - Role: Sends a text message to the configured Telegram chat ID notifying completion.  
  - Configuration:  
    - Text: "CNPJ info extraction complete!"  
    - Chat ID: dynamically set from the Settings node variable `telegram_id`.  
  - Credentials: Uses Telegram Bot API credentials.  
  - Edge Cases: Invalid Telegram ID or revoked bot permissions cause notification failure.

- **Sticky Note (Notifications)**  
  - Type: Sticky Note  
  - Content: Explains notification setup and limiter purpose.  
  - Connections: None.

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                     | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                                                  |
|-----------------------------|------------------------|-----------------------------------|--------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| Execute Workflow Manually    | Manual Trigger         | Starts the workflow manually      |                          | Settings                      |                                                                                                              |
| Settings                    | Set Node               | Defines Telegram ID and API URL   | Execute Workflow Manually | Get All CNPJs in Sheet        | ⚙️ INITIAL SETUP - Configure your Telegram ID for notifications. API URL configured. Google Sheets creds active. |
| Get All CNPJs in Sheet       | Google Sheets          | Reads CNPJs lacking data          | Settings                  | Loop CNPJs                   | 📊 DATA SOURCE - Reads CNPJs from sheet, filters empty 'razao_social' only.                                   |
| Loop CNPJs                  | SplitInBatches         | Processes CNPJs in batches        | Get All CNPJs in Sheet     | Limit Notifier Interactions, Send API Request | 🌐 API DETAILS - API returns +40 fields, free, no auth needed.                                                |
| Limit Notifier Interactions  | Limit                  | Limits notifications to once      | Loop CNPJs                | Notify When Finish            | 📱 NOTIFICATIONS - Telegram notifies when finished, limiter prevents spam.                                   |
| Send API Request             | HTTP Request           | Calls MinhaReceita API for data   | Loop CNPJs                | Update CNPJ Info in Sheet     | 📄 INSERT NEW DATA - Retrieves data from API, updates sheet rows with new info.                              |
| Update CNPJ Info in Sheet    | Google Sheets          | Updates spreadsheet rows          | Send API Request          | Loop CNPJs                   |                                                                                                              |
| Notify When Finish           | Telegram               | Sends completion notification     | Limit Notifier Interactions|                            |                                                                                                              |
| Sticky Note                  | Sticky Note            | User guidance and explanations    |                          |                               | Multiple sticky notes cover setup, data source, API details, notifications, and workflow logic explanations. |
| Sticky Note1                 | Sticky Note            | Data source explanation           |                          |                               |                                                                                                              |
| Sticky Note2                 | Sticky Note            | API details explanation           |                          |                               |                                                                                                              |
| Sticky Note3                 | Sticky Note            | Notifications explanation         |                          |                               |                                                                                                              |
| Sticky Note4                 | Sticky Note            | Data insertion explanation        |                          |                               |                                                                                                              |
| Sticky Note6                 | Sticky Note            | Author LinkedIn link              |                          |                               | # 📌 Follow me: [LinkedIn](https://www.linkedin.com/in/vikthyr)                                              |
| Sticky Note7                 | Sticky Note            | Workflow logic overview           |                          |                               | # 📋 HOW THE WORKFLOW WORKS - Stepwise detailed explanation.                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `Execute Workflow Manually`  
   - Purpose: Entry point to manually start the workflow.

2. **Add a Set Node for Configuration**  
   - Name: `Settings`  
   - Add two string fields:  
     - `telegram_id`: Set to your Telegram chat ID (replace "YOUR_TELEGRAM_ID").  
     - `api_base_url`: Set to `https://minhareceita.org/`.  
   - Connect `Execute Workflow Manually` → `Settings`.

3. **Add Google Sheets Node to Read CNPJs**  
   - Name: `Get All CNPJs in Sheet`  
   - Operation: Read rows (default).  
   - Configure with Google Sheets OAuth2 credentials.  
   - Set Document ID to your target spreadsheet ID.  
   - Set Sheet Name or GID to the sheet where CNPJs are stored.  
   - Add filter to only retrieve rows where `razao_social` is empty (blank).  
   - Connect `Settings` → `Get All CNPJs in Sheet`.

4. **Add SplitInBatches Node**  
   - Name: `Loop CNPJs`  
   - Configure batch size to 100 for scalable processing.  
   - Connect `Get All CNPJs in Sheet` → `Loop CNPJs`.

5. **Add Limit Node to Control Notifications**  
   - Name: `Limit Notifier Interactions`  
   - Default limit (1) to allow one notification per run.  
   - Connect one output of `Loop CNPJs` → `Limit Notifier Interactions`.

6. **Add Telegram Node for Notifications**  
   - Name: `Notify When Finish`  
   - Set text to: `CNPJ info extraction complete!`  
   - Set chat ID to: `={{$('Settings').item.json.telegram_id}}`.  
   - Add Telegram Bot API credentials.  
   - Connect `Limit Notifier Interactions` → `Notify When Finish`.

7. **Add HTTP Request Node to Call API**  
   - Name: `Send API Request`  
   - HTTP Method: GET  
   - URL: `={{$('Settings').item.json.api_base_url + $json.cnpj}}`  
   - No authentication required.  
   - Connect another output of `Loop CNPJs` → `Send API Request`.

8. **Add Google Sheets Node to Update Rows**  
   - Name: `Update CNPJ Info in Sheet`  
   - Operation: Update existing rows.  
   - Configure with same Google Sheets OAuth2 credentials, document ID, and sheet as the read node.  
   - Set matching column as `row_number` to locate the row to update.  
   - Map columns explicitly with API response fields (e.g., `uf`, `cep`, `razao_social`, etc.).  
   - Connect `Send API Request` → `Update CNPJ Info in Sheet`.  
   - Connect `Update CNPJ Info in Sheet` → `Loop CNPJs` (to continue batch processing).

9. **Add Sticky Notes for Documentation** (Optional but recommended)  
   - Add notes describing initial setup, data source, API details, notification logic, and workflow overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow created by Vikthyr. Follow on LinkedIn: https://www.linkedin.com/in/vikthyr                                                  | Author credit and professional profile                          |
| API used: MinhaReceita.org - Free, no authentication, provides detailed official Brazilian company data via CNPJ identifier          | API documentation implicit from node configuration             |
| Telegram notification configured to alert user after full data enrichment completes, with limiter to prevent spam                    | Notification best practice                                      |
| Google Sheets OAuth2 credentials must have read/write access to the target spreadsheet                                               | Credential prerequisite for Google Sheets nodes                 |
| Ensure the Telegram Bot has permissions to send messages to the specified chat ID                                                    | Telegram bot configuration requirement                          |
| Empty 'razao_social' column used as filter to select only incomplete entries                                                        | Data filtering logic for incremental enrichment                 |

---

**Disclaimer:** This documentation is derived exclusively from the provided n8n workflow JSON. The workflow respects content policies and handles only legal, public data.