Track Shipments with DHL/Delhivery APIs & Send Google Sheets Updates to Customers via WhatsApp/Email

https://n8nworkflows.xyz/workflows/track-shipments-with-dhl-delhivery-apis---send-google-sheets-updates-to-customers-via-whatsapp-email-6650


# Track Shipments with DHL/Delhivery APIs & Send Google Sheets Updates to Customers via WhatsApp/Email

### 1. Workflow Overview

This workflow automates daily shipment tracking for orders managed via DHL and Delhivery courier services. It fetches shipment data from a Google Sheet, filters active shipments, queries courier APIs for tracking updates, normalizes and detects any status changes, updates the Google Sheet accordingly, and sends real-time notifications to customers via WhatsApp and Email. It concludes with logging an execution summary.

**Logical Blocks:**

- **1.1 Input Reception:** Daily trigger and shipment data retrieval from Google Sheets.
- **1.2 Shipment Filtering:** Filters out shipments already delivered or lacking tracking numbers.
- **1.3 Courier Routing:** Routes shipments to the appropriate courier API (Delhivery or DHL) based on courier info.
- **1.4 Courier API Tracking:** Calls respective courier APIs to obtain real-time status updates.
- **1.5 Data Parsing & Normalization:** Parses diverse courier API responses, normalizes status values, and detects status changes.
- **1.6 Status Change Filtering:** Processes only shipments with detected status changes.
- **1.7 Updates & Notifications:** Updates shipment status in Google Sheets and notifies customers via WhatsApp and Email.
- **1.8 Execution Summary:** Logs workflow performance and summary metrics.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Initiates workflow execution daily and retrieves shipment records from a Google Sheet.
- **Nodes Involved:** `Daily Trigger`, `Get Shipments List`

##### Node: Daily Trigger
- **Type:** Cron Trigger
- **Role:** Automatically triggers the workflow every day at 9 AM.
- **Configuration:** Default daily schedule with no manual parameter modifications.
- **Connections:** Output → `Get Shipments List`
- **Potential Failures:** Cron node misconfiguration or n8n instance downtime could prevent trigger.

##### Node: Get Shipments List
- **Type:** Google Sheets (Read)
- **Role:** Fetches all shipment data from the specified Google Sheet document and tab.
- **Configuration Highlights:**
  - Auth via Google Service Account credentials.
  - Document ID and Sheet Name provided (parameterized with internal IDs).
  - No filters applied at this stage.
- **Key Expressions:** None directly, but configured to read entire sheet.
- **Connections:** Input ← `Daily Trigger`, Output → `Filter Active Shipments`
- **Potential Failures:** Authentication errors, API quota limits, or incorrect document/sheet IDs.

#### 2.2 Shipment Filtering

- **Overview:** Filters out shipments that are already delivered or lack tracking numbers.
- **Nodes Involved:** `Filter Active Shipments`

##### Node: Filter Active Shipments
- **Type:** Filter
- **Role:** Excludes shipments with status `"delivered"` and those with empty tracking numbers.
- **Configuration:**
  - Conditions combined with AND:
    - Status not equal to `"delivered"`.
    - Tracking number is not empty.
- **Key Expressions:**  
  - `{{$json.status}} != "delivered"`  
  - `{{$json.tracking_number}} != ""`
- **Connections:** Input ← `Get Shipments List`, Output → `Route by Courier`
- **Potential Failures:** Expression evaluation errors if fields missing, or unexpected status values.

#### 2.3 Courier Routing

- **Overview:** Routes shipments to appropriate courier API nodes based on the courier field.
- **Nodes Involved:** `Route by Courier`

##### Node: Route by Courier
- **Type:** If Condition
- **Role:** Checks if the courier is `"delhivery"` to route to Delhivery API; else routes to DHL API.
- **Configuration:**
  - Condition: `{{$json.courier}} == "delhivery"`
  - True branch → `Track via Delhivery`
  - False branch → `Track via DHL`
- **Connections:** Input ← `Filter Active Shipments`, Output True → `Track via Delhivery`, Output False → `Track via DHL`
- **Potential Failures:** Missing or incorrect courier field may route incorrectly or cause data loss.

#### 2.4 Courier API Tracking

- **Overview:** Calls respective courier APIs to get current tracking info.
- **Nodes Involved:** `Track via Delhivery`, `Track via DHL`

##### Node: Track via Delhivery
- **Type:** HTTP Request
- **Role:** Calls Delhivery tracking API with tracking number as query parameter.
- **Configuration:**
  - URL: `https://track.delhivery.com/api/v1/packages/json/`
  - Auth: HTTP Header Auth (generic credential)
  - Query Param: `waybill = {{$json.tracking_number}}`
- **Connections:** Input ← `Route by Courier` (True), Output → `Parse Tracking Data`
- **Potential Failures:** API authentication failures, network issues, invalid tracking numbers.

##### Node: Track via DHL
- **Type:** HTTP Request
- **Role:** Calls DHL tracking API with tracking number.
- **Configuration:**
  - URL: `https://api-eu.dhl.com/track/shipments`
  - Auth: HTTP Header Auth (generic credential)
  - Query Param: `trackingNumber = {{$json.tracking_number}}`
- **Connections:** Input ← `Route by Courier` (False), Output → `Parse Tracking Data`
- **Potential Failures:** Similar to Delhivery node — auth, API limits, invalid tracking numbers.

#### 2.5 Data Parsing & Normalization

- **Overview:** Parses the JSON responses from courier APIs, extracts relevant shipment status info, normalizes various status terms, and detects if status changed.
- **Nodes Involved:** `Parse Tracking Data`

##### Node: Parse Tracking Data
- **Type:** Code (JavaScript)
- **Role:** Processes API responses, extracts fields like status, location, timestamps; normalizes status labels; compares with previous status to detect changes.
- **Key Logic:**
  - For Delhivery: Reads `ShipmentData` array, extracts `Status`, `StatusLocation`, `StatusDateTime`, `ExpectedDeliveryDate`.
  - For DHL: Reads `shipments` array, extracts `status.statusCode`, `location`, `timestamp`.
  - Normalizes statuses into a set of standardized terms (`delivered`, `in_transit`, etc.).
  - Marks `status_changed` as boolean if current differs from previous.
- **Connections:** Input ← `Track via Delhivery` or `Track via DHL`, Output → `Check Status Change`
- **Potential Failures:** Unexpected/missing fields in API response, malformed JSON, code runtime errors.

#### 2.6 Status Change Filtering

- **Overview:** Allows only shipments with detected status changes to proceed to update and notification steps.
- **Nodes Involved:** `Check Status Change`

##### Node: Check Status Change
- **Type:** Filter
- **Role:** Passes only items where `status_changed == true`.
- **Configuration:**
  - Condition: `{{$json.status_changed}} == true`
- **Connections:** Input ← `Parse Tracking Data`, Output → `Update Google Sheet`, `Send WhatsApp Update`, `Send Email Update`
- **Potential Failures:** If `status_changed` field missing or invalid, may filter incorrectly.

#### 2.7 Updates & Notifications

- **Overview:** Updates the shipment status in Google Sheets and sends notifications via WhatsApp and Email to customers.
- **Nodes Involved:** `Update Google Sheet`, `Send WhatsApp Update`, `Send Email Update`

##### Node: Update Google Sheet
- **Type:** Google Sheets (Update)
- **Role:** Updates shipment status and other info in the Google Sheet row.
- **Configuration:**
  - Uses service account credentials.
  - Document and sheet IDs specified.
  - Auto-maps input data fields to sheet columns.
- **Connections:** Input ← `Check Status Change`, Output → `Execution Summary`
- **Potential Failures:** Sheet access or permission errors, row matching failures.

##### Node: Send WhatsApp Update
- **Type:** HTTP Request
- **Role:** Sends WhatsApp message to customer phone with shipment update details.
- **Configuration:**
  - URL: `https://api.whatsapp.com/send`
  - Body Parameters:
    - `phone`: Customer's phone number from JSON.
    - `text`: Message template including order ID, tracking number, status, location, last update, estimated delivery.
  - Sends POST body.
- **Connections:** Input ← `Check Status Change`, Output → `Execution Summary`
- **Potential Failures:** WhatsApp API usage limits, invalid phone numbers, network issues.

##### Node: Send Email Update
- **Type:** Email Send
- **Role:** Sends email notification to customer with shipment update.
- **Configuration:**
  - SMTP credentials configured.
  - Subject: Includes order ID.
  - To: Customer email from JSON.
  - From: `noreply@yourcompany.com`
- **Connections:** Input ← `Check Status Change`, Output → `Execution Summary`
- **Potential Failures:** SMTP auth errors, invalid email addresses, email quota limits.

#### 2.8 Execution Summary

- **Overview:** Logs a summary of workflow execution including total processed shipments and those updated.
- **Nodes Involved:** `Execution Summary`

##### Node: Execution Summary
- **Type:** Code (JavaScript)
- **Role:** Counts total processed shipments, how many had status updates, logs info with timestamp.
- **Connections:** Input ← `Update Google Sheet`, `Send WhatsApp Update`, `Send Email Update`
- **Potential Failures:** None expected; only logging.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                         | Input Node(s)             | Output Node(s)                       | Sticky Note                                                                                                                                        |
|-----------------------|--------------------|---------------------------------------|---------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger         | Cron Trigger       | Starts workflow daily at 9 AM          |                           | Get Shipments List                  | ## How it works * **Daily Trigger**: Cron node runs workflow every day at 9 AM                                                                     |
| Get Shipments List    | Google Sheets      | Fetches all shipment records           | Daily Trigger             | Filter Active Shipments             | * **Get Shipments List**: Fetches all shipment data from Google Sheet                                                                              |
| Filter Active Shipments| Filter             | Filters out delivered/empty tracking   | Get Shipments List        | Route by Courier                   | * **Filter Active Shipments**: Excludes delivered orders and empty tracking numbers                                                               |
| Route by Courier      | If Condition       | Routes shipments to courier APIs       | Filter Active Shipments   | Track via Delhivery, Track via DHL | * **Route by Courier**: Directs shipments to appropriate API (Delhivery or DHL)                                                                     |
| Track via Delhivery   | HTTP Request       | Calls Delhivery tracking API           | Route by Courier (true)   | Parse Tracking Data                | * **Track via APIs**: Makes real-time tracking calls to courier services                                                                           |
| Track via DHL         | HTTP Request       | Calls DHL tracking API                  | Route by Courier (false)  | Parse Tracking Data                | * **Track via APIs**: Makes real-time tracking calls to courier services                                                                           |
| Parse Tracking Data   | Code               | Parses and normalizes courier responses| Track via Delhivery, Track via DHL | Check Status Change               | * **Parse Tracking Data**: Normalizes different API responses and detects status changes                                                          |
| Check Status Change   | Filter             | Filters shipments with status changes  | Parse Tracking Data       | Update Google Sheet, Send WhatsApp Update, Send Email Update | * **Check Status Change**: Only processes shipments with actual status updates                                                                     |
| Update Google Sheet   | Google Sheets      | Updates shipment data in sheet         | Check Status Change       | Execution Summary                 | * **Update & Notify**: Simultaneously updates Google Sheet, sends WhatsApp message, and email notification                                         |
| Send WhatsApp Update  | HTTP Request       | Sends shipment update via WhatsApp     | Check Status Change       | Execution Summary                 | * **Update & Notify**: Simultaneously updates Google Sheet, sends WhatsApp message, and email notification                                         |
| Send Email Update     | Email Send         | Sends shipment update via email        | Check Status Change       | Execution Summary                 | * **Update & Notify**: Simultaneously updates Google Sheet, sends WhatsApp message, and email notification                                         |
| Execution Summary     | Code               | Logs execution metrics                  | Update Google Sheet, Send WhatsApp Update, Send Email Update |                                    | * **Execution Summary**: Logs workflow performance metrics                                                                                        |
| Sticky Note           | Sticky Note        | Documentation and overview              |                           |                                    | ## How it works * **Daily Trigger**: Cron node runs workflow every day at 9 AM * **Get Shipments List**: Fetches all shipment data from Google Sheet * **Filter Active Shipments**: Excludes delivered orders and empty tracking numbers * **Route by Courier**: Directs shipments to appropriate API (Delhivery or DHL) * **Track via APIs**: Makes real-time tracking calls to courier services * **Parse Tracking Data**: Normalizes different API responses and detects status changes * **Check Status Change**: Only processes shipments with actual status updates * **Update & Notify**: Simultaneously updates Google Sheet, sends WhatsApp message, and email notification * **Execution Summary**: Logs workflow performance metrics |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Cron Trigger** node named `Daily Trigger`.  
- Set it to execute daily at 9 AM (default daily cron configuration).  
- No credentials required.

**Step 2:** Add a **Google Sheets** node named `Get Shipments List`.  
- Operation: Read all rows from a sheet.  
- Set Document ID to your Google Sheet document ID containing shipments.  
- Set Sheet Name to the tab containing shipment data.  
- Authentication: Use Google Service Account credentials configured in n8n.  
- Connect output of `Daily Trigger` to this node.

**Step 3:** Add a **Filter** node named `Filter Active Shipments`.  
- Set conditions (AND):  
  - `status` field not equal to `"delivered"` (string, case sensitive).  
  - `tracking_number` field is not empty (string not empty).  
- Connect input from `Get Shipments List`.

**Step 4:** Add an **If** node named `Route by Courier`.  
- Condition: `courier` field equals `"delhivery"` (string, case sensitive).  
- True branch: connect to `Track via Delhivery` node.  
- False branch: connect to `Track via DHL` node.  
- Input from `Filter Active Shipments`.

**Step 5:** Add two **HTTP Request** nodes:  
- `Track via Delhivery`:  
  - URL: `https://track.delhivery.com/api/v1/packages/json/`  
  - Authentication: HTTP Header Auth using credentials for Delhivery API.  
  - Query Parameters: `waybill = {{$json.tracking_number}}`  
  - Input: True output of `Route by Courier`.
- `Track via DHL`:  
  - URL: `https://api-eu.dhl.com/track/shipments`  
  - Authentication: HTTP Header Auth using DHL API credentials.  
  - Query Parameters: `trackingNumber = {{$json.tracking_number}}`  
  - Input: False output of `Route by Courier`.

**Step 6:** Add a **Code** node named `Parse Tracking Data`.  
- Write JavaScript to:  
  - Parse JSON responses from either courier.  
  - Extract status, location, last updated timestamp, estimated delivery.  
  - Normalize statuses to fixed set (`delivered`, `in_transit`, etc.).  
  - Determine if status changed from previous row data.  
- Input from both `Track via Delhivery` and `Track via DHL`.

**Step 7:** Add a **Filter** node named `Check Status Change`.  
- Condition: `status_changed == true` (boolean).  
- Input from `Parse Tracking Data`.  
- Output to three parallel nodes: `Update Google Sheet`, `Send WhatsApp Update`, `Send Email Update`.

**Step 8:** Add a **Google Sheets** node named `Update Google Sheet`.  
- Operation: Update row with new shipment status data.  
- Use same Document ID and Sheet Name as in Step 2.  
- Authentication: Google Service Account.  
- Input from `Check Status Change`.

**Step 9:** Add an **HTTP Request** node named `Send WhatsApp Update`.  
- URL: `https://api.whatsapp.com/send`  
- Method: POST  
- Body Parameters:  
  - `phone`: `{{$json.customer_phone}}`  
  - `text`: A templated text message with order ID, tracking number, status, location, last updated, estimated delivery.  
- Input from `Check Status Change`.

**Step 10:** Add an **Email Send** node named `Send Email Update`.  
- SMTP credentials configured for your mail server.  
- To: `{{$json.customer_email}}`  
- From: `noreply@yourcompany.com` (or your domain email)  
- Subject: `"Shipment Update - Order #{{$json.order_id}}"`  
- Input from `Check Status Change`.

**Step 11:** Add a **Code** node named `Execution Summary`.  
- JavaScript to count total processed shipments and how many had status changes.  
- Log the counts and timestamp to console.  
- Input from `Update Google Sheet`, `Send WhatsApp Update`, and `Send Email Update`.

**Step 12:** Connect outputs from `Update Google Sheet`, `Send WhatsApp Update`, and `Send Email Update` to `Execution Summary` node.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| The workflow runs daily at 9 AM, ensuring customers receive timely shipment updates via preferred channels.     | Cron scheduling detail                                  |
| Google Sheets integration uses Service Account for secure, automated access without user intervention.           | Google Sheets API and credentials                       |
| WhatsApp messages use the official API endpoint with phone and templated text parameters for dynamic messaging.  | WhatsApp API documentation                              |
| Email notifications require SMTP credentials; ensure sender email is authorized to avoid spam filtering.         | SMTP setup and domain email configuration               |
| Status normalization enables consistent tracking state representation despite different courier API formats.     | Code node parsing logic                                 |
| For troubleshooting, check authentication credentials, API rate limits, and proper response JSON structures.     | Common failure points                                   |
| Sticky Note in workflow provides summary of logic and flow, useful for onboarding and maintenance.                | Visible inside n8n workflow editor                      |

---

**Disclaimer:**  
The provided documentation is based exclusively on an automated workflow created with n8n, an integration and automation tool. The workflow strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.