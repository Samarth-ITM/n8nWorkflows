Automated B2B Lead Generation: Google Maps to Sheets with BrowserAct & Telegram Alerts

https://n8nworkflows.xyz/workflows/automated-b2b-lead-generation--google-maps-to-sheets-with-browseract---telegram-alerts-9890


# Automated B2B Lead Generation: Google Maps to Sheets with BrowserAct & Telegram Alerts

---

### 1. Workflow Overview

This workflow automates B2B lead generation by scraping local business data from Google Maps and saving it into Google Sheets, followed by real-time Telegram notifications for each new lead. It is designed for users aiming to gather local business leads efficiently and be instantly informed of new entries.

The workflow is logically divided into three main blocks:

- **1.1 Scraping Initiation and Monitoring:** Triggering a BrowserAct scraping task on Google Maps with specified search parameters and waiting for its completion.
- **1.2 Data Parsing and Splitting:** Parsing the raw JSON string output from the scraper and splitting it into individual business lead items.
- **1.3 Saving Leads and Notification:** Writing each parsed lead into a Google Sheet with deduplication based on business name and sending a Telegram message alert for every saved lead.

---

### 2. Block-by-Block Analysis

#### 1.1 Scraping Initiation and Monitoring

**Overview:**  
This block initiates a web scraping task using BrowserAct to extract business data from Google Maps according to user-specified criteria (location, category, number of leads). It then polls BrowserAct until the scraping task finishes.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Run a workflow task (BrowserAct Start Task)  
- Get details of a workflow task (BrowserAct Poll Task Status)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually.  
  - *Configuration:* No parameters; manual execution only.  
  - *Connections:* Outputs to "Run a workflow task".  
  - *Edge Cases:* If left running unattended, no automation occurs (replaceable with Cron).  

- **Run a workflow task**  
  - *Type:* BrowserAct Workflow Node  
  - *Role:* Initiates a BrowserAct scraping job using the Google Maps Local Lead Finder template.  
  - *Configuration:*  
    - Workflow ID set to BrowserAct template "Google Maps Local Lead Finder".  
    - Input parameters:  
      - Location: "Brooklyn" (default, editable)  
      - Bussines_Category: "Baby Care " (note trailing space)  
      - Extracted_Data: "15" (number of leads to retrieve)  
  - *Credentials:* Linked to a BrowserAct API account.  
  - *Connections:* Outputs task ID to "Get details of a workflow task".  
  - *Edge Cases:*  
    - Authentication failure if BrowserAct API key invalid.  
    - Invalid input parameters may cause empty or failed scraping.  
    - Network or service downtime may cause task start failure.  

- **Get details of a workflow task**  
  - *Type:* BrowserAct Workflow Node  
  - *Role:* Polls the status of the scraping task until completion (or timeout).  
  - *Configuration:*  
    - Task ID dynamically received from the previous node.  
    - Operation: "getTask"  
    - Max wait time: 600 seconds (10 minutes)  
    - Polling interval: 30 seconds  
    - Wait for finish: true (workflow pauses until done)  
  - *Credentials:* Same BrowserAct API account as above.  
  - *Connections:* Outputs raw scraping results to "Code in JavaScript".  
  - *Edge Cases:*  
    - Timeout if scraping takes longer than 10 minutes.  
    - Task failure reported by BrowserAct.  
    - Network issues during polling.  

---

#### 1.2 Data Parsing and Splitting

**Overview:**  
This block parses the raw stringified JSON containing multiple business leads and splits it into individual n8n items for further processing.

**Nodes Involved:**  
- Code in JavaScript

**Node Details:**

- **Code in JavaScript**  
  - *Type:* Code (JavaScript)  
  - *Role:* Converts the raw JSON string output from BrowserAct into an array of individual lead objects, each becoming a separate n8n item.  
  - *Configuration:*  
    - Reads the JSON string from the path `$input.first().json.output.string`.  
    - Validates the presence of the string; throws error if missing.  
    - Parses JSON string safely, throwing errors on malformed input.  
    - Confirms the parsed data is an array; errors if not.  
    - Maps each array element into `{ json: item }` format for splitting.  
  - *Input:* Output from "Get details of a workflow task".  
  - *Output:* Multiple items representing individual leads.  
  - *Edge Cases:*  
    - Missing or empty string leads to immediate error, stopping workflow.  
    - Malformed JSON throws parse error.  
    - Non-array JSON structure causes error.  
    - Errors here prevent downstream processing.  

---

#### 1.3 Saving Leads and Notification

**Overview:**  
This block appends or updates each lead in a Google Sheets document to avoid duplicates and sends a Telegram notification for every saved lead.

**Nodes Involved:**  
- Append or update row in sheet (Google Sheets)  
- Send a text message (Telegram)

**Node Details:**

- **Append or update row in sheet**  
  - *Type:* Google Sheets Node  
  - *Role:* Inserts or updates lead data into a specified sheet, using "Name" as the unique matching key to avoid duplicates.  
  - *Configuration:*  
    - Operation: "appendOrUpdate"  
    - Document ID: Google Sheets document ID linked to lead storage.  
    - Sheet Name: Specific sheet tab named "Google Maps Local Lead Finder" (gid 1084488211).  
    - Columns mapped: Name, Phone, Category, Rating, Address, Url, LastSummary — all mapped from lead JSON fields.  
    - Matching column: Name (for deduplication).  
    - Type conversion disabled to preserve original formatting.  
  - *Credentials:* Google Sheets OAuth2 credentials.  
  - *Connections:* Outputs to "Send a text message".  
  - *Edge Cases:*  
    - Google Sheets API quota or permission issues.  
    - Name field missing or duplicated in source data may cause update anomalies.  
    - Network failures during write.  

- **Send a text message**  
  - *Type:* Telegram node  
  - *Role:* Sends a Telegram message containing formatted lead details to a Telegram channel or chat.  
  - *Configuration:*  
    - Text message built dynamically from current lead JSON fields: Name, Address, LastSummary, Rating, Url, Category.  
    - Chat ID set to "@shoaywbs" (Telegram channel or group).  
  - *Credentials:* Telegram API credentials.  
  - *Edge Cases:*  
    - Invalid chat ID or bot permissions may prevent messages.  
    - Telegram API rate limits or downtime.  
    - Formatting errors if any lead fields are missing or contain unexpected data.  

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                     | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                                                 |
|------------------------------|----------------------------------|-----------------------------------|---------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Workflow start trigger             | -                               | Run a workflow task                |                                                                                                                             |
| Run a workflow task           | BrowserAct Workflow Node          | Start Google Maps scraping task   | When clicking ‘Execute workflow’ | Get details of a workflow task     | ### 🌐 1. Scrape & Wait: Starts scraper with search criteria and initiates scraping job.                                     |
| Get details of a workflow task| BrowserAct Workflow Node          | Poll scraping task status          | Run a workflow task              | Code in JavaScript                 | ### 🌐 1. Scrape & Wait: Waits for scraping completion to continue.                                                          |
| Code in JavaScript            | Code Node                        | Parse and split raw JSON string    | Get details of a workflow task   | Append or update row in sheet      | ### 🧹 2. Parse & Split Data: Parses text JSON into individual lead items for processing.                                    |
| Append or update row in sheet | Google Sheets Node                | Save leads to Google Sheets        | Code in JavaScript               | Send a text message               | ### 💾 3. Save & Notify: Adds leads to sheet with deduplication, triggers Telegram notifications.                            |
| Send a text message           | Telegram Node                    | Notify via Telegram                | Append or update row in sheet    | -                                 | ### 💾 3. Save & Notify: Sends a Telegram message for every saved lead.                                                      |
| Sticky Note - Intro           | Sticky Note                     | Documentation and intro            | -                               | -                                 | ## Try It Out! Explanation of the full workflow, requirements, and links to Discord and blog.                                |
| Sticky Note - How to Use      | Sticky Note                     | Usage instructions                 | -                               | -                                 | ## How to use step-by-step instructions for credentials, setup, and execution.                                               |
| Sticky Note - Need Help       | Sticky Note                     | Support resources                  | -                               | -                                 | ### Need Help? Links to BrowserAct API key, n8n connection tutorials, and automation videos.                                  |
| Sticky Note - Scraping Stage  | Sticky Note                     | Documents Scraping block           | -                               | -                                 | ### 🌐 1. Scrape & Wait: Explains first two nodes' roles in scraping and waiting.                                            |
| Sticky Note - Processing Stage| Sticky Note                     | Documents Parsing block            | -                               | -                                 | ### 🧹 2. Parse & Split Data: Explains the code node's role in parsing JSON and splitting items.                              |
| Sticky Note - Output Stage    | Sticky Note                     | Documents Saving & Notification   | -                               | -                                 | ### 💾 3. Save & Notify: Explains Google Sheets and Telegram node roles in saving data and sending alerts.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking ‘Execute workflow’". No parameters needed. This triggers the workflow manually.

2. **Add a BrowserAct node** named "Run a workflow task":
   - Select operation to run a workflow task by specifying a workflow ID.
   - Enter the BrowserAct workflow ID for the Google Maps Local Lead Finder template.
   - Set input parameters:
     - Location: e.g., "Brooklyn" (editable)
     - Bussines_Category: e.g., "Baby Care " (note trailing space)
     - Extracted_Data: number of leads to extract, e.g., 15
   - Attach BrowserAct API credentials.
   - Connect this node output to the next node.

3. **Add another BrowserAct node** named "Get details of a workflow task":
   - Operation: "getTask"
   - Set `taskId` from the previous node output (`={{ $json.id }}`).
   - Enable "Wait for finish" with a max wait time of 600 seconds.
   - Set polling interval to 30 seconds.
   - Use the same BrowserAct API credentials.
   - Connect output to the next node.

4. **Add a Code node** named "Code in JavaScript":
   - Paste the following code to parse the raw JSON string:
     ```javascript
     const jsonString = $input.first().json.output.string;

     if (!jsonString) {
         throw new Error("Input string is empty or missing at the specified path: $input.first().json.output.string");
     }

     let parsedData;
     try {
         parsedData = JSON.parse(jsonString);
     } catch (error) {
         throw new Error(`Failed to parse JSON string: ${error.message}`);
     }

     if (!Array.isArray(parsedData)) {
         throw new Error('Parsed data is not an array. It cannot be split into multiple items.');
     }

     return parsedData.map(item => ({ json: item }));
     ```
   - Connect output to the next node.

5. **Add a Google Sheets node** named "Append or update row in sheet":
   - Operation: "appendOrUpdate".
   - Select Google Sheets document by ID or from list.
   - Select sheet/tab named "Google Maps Local Lead Finder" (or your own).
   - Map columns with respective JSON fields: Name, Phone, Category, Rating, Address, Url, LastSummary.
   - Set matching column as "Name" to avoid duplicates.
   - Use Google Sheets OAuth2 credentials.
   - Connect output to the next node.

6. **Add a Telegram node** named "Send a text message":
   - Set Chat ID to your Telegram channel or group, e.g., "@shoaywbs".
   - Compose the message text using expressions to include lead details:
     ```
     {{$json.Name}}
     {{$json.Address}}
     {{$json.LastSummary}}
     {{$json.Rating}}
     {{$json.Url}}
     {{$json.Category}}
     ```
   - Use your Telegram API credentials.
   - No output connection needed.

7. **Connect nodes sequentially**:
   - Manual Trigger → Run a workflow task → Get details of a workflow task → Code in JavaScript → Append or update row in sheet → Send a text message.

8. **Credential Setup**:
   - BrowserAct: API key with access to the "Google Maps Local Lead Finder" template.
   - Google Sheets: OAuth2 credentials with edit access to the target spreadsheet.
   - Telegram: Bot token with permission to send messages to the specified chat ID.

9. **Optional**: Replace the manual trigger with a Cron node to schedule regular automated runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automates lead generation by scraping Google Maps and notifies users via Telegram in real-time. Requires BrowserAct, Google Sheets, and Telegram credentials.                 | Introductory description in Sticky Note - Intro                                                  |
| For setup assistance, watch videos on BrowserAct API key retrieval, n8n integration, BrowserAct templates customization, and local lead generation automation.                              | Links in Sticky Note - Need Help                                                                  |
| Join the BrowserAct Discord community or read their blog for updates, examples, and support.                                                                                                | Discord: https://discord.com/invite/UpnCKd7GaU Blog: https://www.browseract.com/blog              |
| Use the BrowserAct template named “Google Maps Local Lead Finder” for scraping tasks to avoid building scraping logic from scratch.                                                        | Mentioned in Sticky Note - How to Use                                                             |
| When customizing search parameters, watch out for trailing spaces in category names which may affect scraping results.                                                                      | Observed trailing space in “Baby Care ” input                                                     |
| Telegram chat ID must be a valid channel or group where the bot is added with permissions to post messages.                                                                                 | Telegram node configuration                                                                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This workflow respects all current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.