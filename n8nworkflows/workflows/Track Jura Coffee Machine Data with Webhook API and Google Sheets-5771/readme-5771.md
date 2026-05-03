Track Jura Coffee Machine Data with Webhook API and Google Sheets

https://n8nworkflows.xyz/workflows/track-jura-coffee-machine-data-with-webhook-api-and-google-sheets-5771


# Track Jura Coffee Machine Data with Webhook API and Google Sheets

### 1. Workflow Overview

This workflow, titled **"Track Jura Coffee Machine Data with Webhook API and Google Sheets"**, is designed to collect coffee consumption data from Jura coffee machines (or compatible BLE devices like ESP32/ESP8266), store this data into Google Sheets, and provide an API endpoint to retrieve the latest coffee consumption records. It targets use cases such as real-time monitoring, logging, and dashboarding of coffee machine usage.

The workflow is logically divided into two main blocks:

- **1.1 Coffee Data Reception and Storage (POST webhook flow)**  
  Receives coffee consumption counts via a POST webhook, enriches the data with timestamp details, and appends it to a Google Sheet.

- **1.2 Coffee Data Retrieval (GET webhook flow)**  
  Provides a GET webhook API to fetch the latest coffee consumption records from the Google Sheet for use in dashboards or reports.

Supporting these are setup and instructional sticky notes for user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Coffee Data Reception and Storage (POST webhook flow)

**Overview:**  
This block handles incoming coffee count data sent as POST requests from Jura-compatible devices. It records the current date and time, formats the data into a row, and appends it to a specified Google Sheet.

**Nodes Involved:**  
- Receive Coffee Count (POST) (Webhook node)  
- Generate Timestamp (Date & Time node)  
- Prepare Row Data (Set node)  
- Append to Google Sheet (Google Sheets node)  
- Sticky Note - Input Setup (Sticky Note)  
- Sticky Note - Setup Instructions (Sticky Note)

**Node Details:**

- **Receive Coffee Count (POST)**  
  - *Type:* Webhook (HTTP POST endpoint)  
  - *Role:* Entry point for coffee machine data via POST requests.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Path: Configured via environment variable `{{WEBHOOK_POST_PATH}}` (dynamic)  
  - *Input:* HTTP POST JSON body, expected to contain `total_coffees` integer.  
  - *Output:* JSON object with body data forwarded downstream.  
  - *Failure Modes:* Invalid/missing JSON body, malformed `total_coffees` field, webhook path conflicts.  
  - *Version:* n8n v2+ webhook node.  
  - *Notes:* This node receives data from Jura BLE devices or ESP32/ESP8266 BLE devices.

- **Generate Timestamp**  
  - *Type:* Date & Time node  
  - *Role:* Attaches the current date-time to the workflow data.  
  - *Configuration:* Default settings to generate current timestamp at execution.  
  - *Input:* JSON from webhook node.  
  - *Output:* Adds `currentDate` field with ISO8601 datetime string.  
  - *Failure Modes:* Node execution failure unlikely; time service issues rare.

- **Prepare Row Data**  
  - *Type:* Set node  
  - *Role:* Formats and maps data fields to prepare the row for Google Sheets.  
  - *Configuration:*  
    - Extracts date and time from `currentDate` by splitting the ISO string:  
      - `data` = date part (YYYY-MM-DD)  
      - `time` = time part (HH:MM:SS)  
    - Maps `coffee counter` from the webhook JSON body’s `total_coffees` field.  
  - *Input:* Data with `currentDate` and webhook JSON body.  
  - *Output:* JSON with three fields: `data`, `time`, `coffee counter` (number).  
  - *Expressions:*  
    - `={{ $json.currentDate.split('T')[0] }}` for date  
    - `={{ $json.currentDate.split('T')[1].split('.')[0] }}` for time  
    - `={{ $('Webhook').item.json.body.total_coffees }}` for coffee count  
  - *Failure Modes:* Expression errors if fields missing; JSON path errors if webhook data malformed.

- **Append to Google Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends the prepared data row into the configured Google Sheet.  
  - *Configuration:*  
    - Operation: Append  
    - Spreadsheet ID: dynamic via `{{SHEET_ID}}` environment variable  
    - Sheet Name: dynamic via `{{SHEET_NAME}}` environment variable (default "Sheet1")  
    - Columns: auto-mapped from input data fields `currentDate` (mapped as `date`), `time`, and `coffee counter`  
    - Credentials: Google Sheets OAuth2 with appropriate scopes for append access  
  - *Input:* Prepared row data JSON.  
  - *Output:* Confirmation of append operation.  
  - *Failure Modes:* Authentication issues, API quota limits, incorrect spreadsheet/sheet ID, schema mismatch.

- **Sticky Note - Input Setup**  
  - *Type:* Sticky Note  
  - *Content:* Detailed instructions on the POST webhook input flow, expected JSON body, and Google Sheet column requirements.  
  - *Placement:* Near POST webhook flow for quick user reference.

- **Sticky Note - Setup Instructions**  
  - *Type:* Sticky Note  
  - *Content:* High-level setup steps for Jura device, Google Sheet placeholders, and webhook paths.  
  - *Placement:* General setup area.

---

#### 2.2 Coffee Data Retrieval (GET webhook flow)

**Overview:**  
This block provides a GET webhook endpoint that reads the most recent rows from the Google Sheet and returns them as a JSON response for dashboard or monitoring usage.

**Nodes Involved:**  
- Webhook2 (GET webhook node)  
- Fetch Sheet Rows (Google Sheets node)  
- Limit to Last Row (Limit node)  
- Respond with Sheet Data (Respond to Webhook node)  
- Sticky Note - Output Setup (Sticky Note)  
- Sticky Note - Webhooks (Sticky Note)

**Node Details:**

- **Webhook2 (GET endpoint)**  
  - *Type:* Webhook (HTTP GET endpoint)  
  - *Role:* Entry point for retrieving coffee data records.  
  - *Configuration:*  
    - HTTP Method: GET (default)  
    - Path: dynamic via `{{WEBHOOK_GET_PATH}}` environment variable  
    - Response Mode: Uses downstream node to respond (`Respond with Sheet Data`)  
  - *Input:* HTTP GET requests.  
  - *Output:* Triggers downstream nodes.  
  - *Failure Modes:* Path conflicts, unauthorized requests if not secured externally.

- **Fetch Sheet Rows**  
  - *Type:* Google Sheets node  
  - *Role:* Reads all rows from the configured Google Sheet.  
  - *Configuration:*  
    - Operation: Read rows (default)  
    - Spreadsheet ID: dynamic via `{{SHEET_ID}}`  
    - Sheet Name: dynamic via `{{SHEET_NAME}}` ("Sheet1")  
    - Credentials: Google Sheets OAuth2 with read access  
  - *Input:* Triggered by webhook.  
  - *Output:* Array of rows as JSON objects.  
  - *Failure Modes:* Authentication failure, spreadsheet access issues, API limits.

- **Limit to Last Row**  
  - *Type:* Limit node  
  - *Role:* Restricts output to the last (most recent) row(s) of data.  
  - *Configuration:*  
    - Keep: "lastItems" (default 1, or can be configured)  
  - *Input:* Entire sheet rows.  
  - *Output:* Latest rows only.  
  - *Failure Modes:* Empty sheet results in empty output.

- **Respond with Sheet Data**  
  - *Type:* Respond to Webhook node  
  - *Role:* Sends the limited sheet rows as HTTP response to client.  
  - *Configuration:*  
    - Respond with: all incoming items (the limited rows)  
  - *Input:* Latest rows from Limit node.  
  - *Output:* HTTP response with JSON body.  
  - *Failure Modes:* Network issues, large payloads.

- **Sticky Note - Output Setup**  
  - *Type:* Sticky Note  
  - *Content:* Explanation of the GET webhook endpoint usage for dashboards and real-time stats.  
  - *Placement:* Near GET webhook flow.

- **Sticky Note - Webhooks**  
  - *Type:* Sticky Note  
  - *Content:* Summarizes POST webhook role (receiving data) and GET webhook role (providing data).  
  - *Placement:* General webhook area.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                        | Input Node(s)            | Output Node(s)              | Sticky Note                                                                                       |
|---------------------------|---------------------|-------------------------------------|--------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| Receive Coffee Count (POST)| Webhook             | Receive coffee count data via POST  | (Start)                  | Generate Timestamp          | ☕ Coffee Input Workflow: POST webhook receives `total_coffees` from BLE device, adds timestamp. |
| Generate Timestamp        | Date & Time         | Add current date-time                | Receive Coffee Count (POST)| Prepare Row Data            |                                                                                                 |
| Prepare Row Data          | Set                 | Format data row for Google Sheets   | Generate Timestamp       | Append to Google Sheet      |                                                                                                 |
| Append to Google Sheet    | Google Sheets       | Append coffee data row               | Prepare Row Data         | (End)                      |                                                                                                 |
| Fetch Sheet Rows          | Google Sheets       | Retrieve all rows from Google Sheet | Webhook2 (GET)           | Limit to Last Row           |                                                                                                 |
| Limit to Last Row         | Limit               | Keep only most recent row(s)         | Fetch Sheet Rows         | Respond with Sheet Data     |                                                                                                 |
| Respond with Sheet Data   | Respond to Webhook  | Send sheet data as HTTP response     | Limit to Last Row        | (End)                      |                                                                                                 |
| Webhook2                  | Webhook             | GET endpoint to retrieve data        | (Start)                  | Fetch Sheet Rows            | 📊 Dashboard Data Endpoint: GET webhook returns rows for live views                             |
| Sticky Note - Setup       | Sticky Note         | Setup instructions                   |                          |                            | Setup instructions: Configure Jura device, Google Sheets placeholders, webhook paths           |
| Sticky Note - Input Setup | Sticky Note         | Input POST webhook flow explanation |                          |                            | ☕ Coffee Input Workflow: details on POST webhook and Google Sheet columns                      |
| Sticky Note - Output Setup| Sticky Note         | Output GET webhook flow explanation  |                          |                            | 📊 Dashboard Data Endpoint: GET webhook usage for real-time stats                              |
| Sticky Note - Webhooks    | Sticky Note         | Summary of webhook roles             |                          |                            | 🧩 POST webhook: receives data from Jura ESP; 🧩 GET webhook: provides last rows for live views|

---

### 4. Reproducing the Workflow from Scratch

1. **Create POST Webhook Node**  
   - Type: Webhook  
   - Name: "Receive Coffee Count (POST)"  
   - HTTP Method: POST  
   - Path: Use environment variable `{{WEBHOOK_POST_PATH}}` or a fixed string (e.g., `jura-coffee-post`)  
   - No authentication configured (consider adding for security)  

2. **Add Date & Time Node**  
   - Type: Date & Time  
   - Name: "Generate Timestamp"  
   - Use default settings to generate current timestamp when triggered  

3. **Add Set Node to Prepare Row Data**  
   - Type: Set  
   - Name: "Prepare Row Data"  
   - Add fields:  
     - `data` (string) = expression: `{{$json.currentDate.split('T')[0]}}`  
     - `time` (string) = expression: `{{$json.currentDate.split('T')[1].split('.')[0]}}`  
     - `coffee counter` (number) = expression: `{{$('Receive Coffee Count (POST)').item.json.body.total_coffees}}`  

4. **Add Google Sheets Append Node**  
   - Type: Google Sheets  
   - Name: "Append to Google Sheet"  
   - Operation: Append  
   - Spreadsheet ID: Use `{{SHEET_ID}}` environment variable or paste your sheet ID  
   - Sheet Name: Use `{{SHEET_NAME}}` or actual sheet name (e.g., "Sheet1")  
   - Columns: Use auto-mapping for the fields: `data`, `time`, `coffee counter`  
   - Credentials: Set up and select Google Sheets OAuth2 credentials with write access  

5. **Connect Nodes for POST Flow**:  
   - "Receive Coffee Count (POST)" → "Generate Timestamp" → "Prepare Row Data" → "Append to Google Sheet"  

6. **Create GET Webhook Node**  
   - Type: Webhook  
   - Name: "Webhook2"  
   - HTTP Method: GET (default)  
   - Path: Use environment variable `{{WEBHOOK_GET_PATH}}` or fixed string (e.g., `jura-coffee-get`)  
   - Set Response Mode to "Respond with Node"  

7. **Add Google Sheets Read Node**  
   - Type: Google Sheets  
   - Name: "Fetch Sheet Rows"  
   - Operation: Read rows (default)  
   - Spreadsheet ID: Same as above (`{{SHEET_ID}}`)  
   - Sheet Name: Same as above (`{{SHEET_NAME}}`)  
   - Credentials: Same Google Sheets OAuth2 credentials with read access  

8. **Add Limit Node**  
   - Type: Limit  
   - Name: "Limit to Last Row"  
   - Keep: Last items (default to 1 or configure number of rows to return)  

9. **Add Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Name: "Respond with Sheet Data"  
   - Respond With: All incoming items  

10. **Connect Nodes for GET Flow**:  
    - "Webhook2" → "Fetch Sheet Rows" → "Limit to Last Row" → "Respond with Sheet Data"  

11. **Add Sticky Notes for User Guidance** (optional but recommended):  
    - Setup Instructions: Explain device configuration, environment variables, and Google Sheets setup  
    - Input Setup: Explain POST webhook expectations and test example JSON body  
    - Output Setup: Explain GET webhook usage and integration for dashboards  
    - Webhooks Summary: Clarify roles of POST and GET webhook endpoints  

12. **Environment Variables and Credentials:**  
    - Define `WEBHOOK_POST_PATH` and `WEBHOOK_GET_PATH` for webhook URLs  
    - Define `SHEET_ID` as the Google Sheet document ID for data storage  
    - Define `SHEET_NAME` as the target sheet/tab name (usually "Sheet1")  
    - Configure Google Sheets OAuth2 credentials with proper scopes for reading and appending data  

13. **Testing:**  
    - Test POST webhook with JSON body: `{ "total_coffees": 123 }` via Postman or curl  
    - Test GET webhook to verify latest coffee count entries are returned  

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Setup Instructions: Configure Jura BLE device to send coffee count data to POST webhook endpoint. Configure Google Sheet and credentials.| See Sticky Note - Setup Instructions node in workflow.                                         |
| Example POST JSON body for testing: `{ "total_coffees": 123 }`                                                                          | Included in Sticky Note - Input Setup node.                                                    |
| Use environment variables to dynamically manage webhook paths and Google Sheet IDs for flexibility and security.                      | Workflow uses `{{WEBHOOK_POST_PATH}}`, `{{WEBHOOK_GET_PATH}}`, `{{SHEET_ID}}`, `{{SHEET_NAME}}`.|
| Google Sheets OAuth2 credentials must have scopes to read and append rows to the target spreadsheet.                                   | Ensure OAuth2 credential setup in n8n with proper Google API scopes.                           |
| The GET webhook endpoint can be used to feed live dashboard visualizations or other monitoring tools with recent coffee count data.    | Described in Sticky Note - Output Setup node.                                                 |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a no-code integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.