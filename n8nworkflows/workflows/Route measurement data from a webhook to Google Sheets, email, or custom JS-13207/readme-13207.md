Route measurement data from a webhook to Google Sheets, email, or custom JS

https://n8nworkflows.xyz/workflows/route-measurement-data-from-a-webhook-to-google-sheets--email--or-custom-js-13207


# Route measurement data from a webhook to Google Sheets, email, or custom JS

# 1. Workflow Overview

This workflow receives measurement data through an HTTP webhook and routes the processed payload to one of several destinations based on a configurable `outputMode`. It is designed for scenarios where a measurement-producing system, such as 3D inspection software or a custom device, needs to submit structured measurement records into downstream systems like Google Sheets, an HTML response, email as CSV, or custom JavaScript logic.

The workflow is organized into the following logical blocks:

## 1.1 Webhook Entry and Runtime Configuration
The workflow starts with a POST webhook endpoint at `/measurement-data`. Immediately after receiving the request, it injects configuration values such as `outputMode`, `googleSheetId`, and email settings through a Set node.

## 1.2 Payload Validation and Error Handling
The workflow checks whether the required fields `measurementId`, `timestamp`, and `values` exist in the webhook body. If any are missing, it returns a 400 response and also raises a workflow error.

## 1.3 Data Normalization and Initial Success Acknowledgment
If validation passes, the workflow extracts and normalizes measurement fields into a flatter structure with defaults for optional fields. It then returns a JSON success response to the caller.

## 1.4 Output Routing
After acknowledging receipt, the workflow branches based on `outputMode`:
- `sheets` → Google Sheets
- `html` → HTML report returned as a web page
- `csv` → CSV file sent by email
- `custom` → custom JavaScript logic

## 1.5 Destination-Specific Processing
Each branch performs output-specific actions:
- append or update a row in Google Sheets
- render HTML and respond with a browser-friendly page
- generate a CSV file and email it
- execute extensible JavaScript examples for APIs, storage, and workflow triggering

---

# 2. Block-by-Block Analysis

## 2.1 Webhook Entry and Runtime Configuration

### Overview
This block receives the incoming POST request and injects workflow-level settings used later for routing and destination behavior. The configuration is hardcoded in a Set node, making it easy to switch the workflow behavior without editing downstream nodes.

### Nodes Involved
- Webhook Trigger - Receive Measurement Data
- Workflow Configuration

### Node Details

#### Webhook Trigger - Receive Measurement Data
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Main entry point of the workflow. Accepts inbound HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `measurement-data`
  - Response mode: `lastNode`
- **Key expressions or variables used:**  
  No parameter expressions on the node itself. Downstream nodes reference:
  - `$('Webhook Trigger - Receive Measurement Data').item.json.body`
- **Input and output connections:**
  - Input: none, entry point
  - Output: `Workflow Configuration`
- **Version-specific requirements:**  
  Type version `2.1`. Response behavior depends on webhook/Respond to Webhook compatibility in the n8n version used.
- **Edge cases or potential failure types:**
  - Wrong HTTP method
  - Caller posting malformed JSON
  - Payload not available under `body` if caller sends unexpected content type
  - Because the node is configured with `responseMode: lastNode`, response handling can become ambiguous when multiple `Respond to Webhook` nodes exist downstream
- **Sub-workflow reference:** none

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Injects configurable runtime settings into the item stream.
- **Configuration choices:**
  - Adds:
    - `outputMode = "sheets"`
    - `googleSheetId = "YOUR_SHEET_ID_HERE - Please replace with your actual Google Sheet ID"`
    - `emailRecipient = "user@example.com"`
    - `emailSubject = "Measurement Data Export"`
  - `includeOtherFields: true`, so original webhook data remains available
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input: `Webhook Trigger - Receive Measurement Data`
  - Output: `Validate Required Fields1`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**
  - If `outputMode` is left invalid, routing later will produce no branch match
  - Placeholder `googleSheetId` will cause Google Sheets failure if not replaced
  - Placeholder email values can cause SMTP rejection or misdelivery
- **Sub-workflow reference:** none

---

## 2.2 Payload Validation and Error Handling

### Overview
This block verifies that the required fields exist in the webhook body before any processing happens. On failure, it both returns a 400 JSON response and raises a workflow error to stop execution.

### Nodes Involved
- Validate Required Fields1
- Error - Invalid Payload
- Send Error Response

### Node Details

#### Validate Required Fields1
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional gate that checks mandatory input fields.
- **Configuration choices:**
  - Uses AND logic
  - Checks existence of:
    - `measurementId`
    - `timestamp`
    - `values`
  - Expressions reference the webhook body directly rather than current item fields
- **Key expressions or variables used:**
  - `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.measurementId }}`
  - `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.timestamp }}`
  - `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.values }}`
- **Input and output connections:**
  - Input: `Workflow Configuration`
  - True output: `Parse and Normalize Data1`
  - False output: `Error - Invalid Payload`, `Send Error Response`
- **Version-specific requirements:**  
  Type version `2.3`, condition format uses the newer IF condition structure.
- **Edge cases or potential failure types:**
  - `exists` only checks presence, not type or quality
  - Empty object `{}` in `values` still passes
  - Empty string values may pass existence checks depending on evaluation behavior
  - If payload structure differs from `body`, expressions return undefined
- **Sub-workflow reference:** none

#### Error - Invalid Payload
- **Type and technical role:** `n8n-nodes-base.stopAndError`  
  Explicitly fails the execution on invalid input.
- **Configuration choices:**
  - Error message: `Invalid payload: Missing required fields (measurementId, timestamp, values)`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input: false branch of `Validate Required Fields1`
  - Output: none
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**
  - In combination with `Send Error Response`, execution order on the same branch may matter operationally
  - Depending on runtime behavior, a stop/error may interfere with webhook response completion if triggered before response delivery
- **Sub-workflow reference:** none

#### Send Error Response
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns a client-facing 400 JSON error payload.
- **Configuration choices:**
  - Response code: `400`
  - Respond with JSON
  - Static body:
    - `success: false`
    - `error: Missing required fields: measurementId, timestamp, values`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input: false branch of `Validate Required Fields1`
  - Output: none
- **Version-specific requirements:**  
  Type version `1.5`
- **Edge cases or potential failure types:**
  - Multiple response nodes in one workflow require careful webhook response mode design
  - If another response is already sent, this node may fail
- **Sub-workflow reference:** none

---

## 2.3 Data Normalization and Initial Success Acknowledgment

### Overview
This block transforms the incoming payload into a consistent flat record and supplies defaults for optional fields. It then sends a success acknowledgment containing the `measurementId`.

### Nodes Involved
- Parse and Normalize Data1
- Send Success Response1

### Node Details

#### Parse and Normalize Data1
- **Type and technical role:** `n8n-nodes-base.set`  
  Maps the webhook payload to normalized fields used by every downstream output.
- **Configuration choices:**
  - Creates:
    - `measurementId`
    - `timestamp`
    - `deviceId` default `unknown`
    - `operator` default `N/A`
    - `partNumber` default `N/A`
    - `dimension_x` default `0`
    - `dimension_y` default `0`
    - `dimension_z` default `0`
    - `tolerance` default `0`
    - `status` default `received`
    - `notes` default empty string
- **Key expressions or variables used:**
  - `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.measurementId }}`
  - `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.values.x || 0 }}`
  - Similar expressions for all normalized fields
- **Input and output connections:**
  - Input: true branch of `Validate Required Fields1`
  - Output: `Send Success Response1`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**
  - `|| 0` treats valid zero values correctly, but also coerces other falsy values unexpectedly
  - If numeric fields are strings, type coercion may or may not behave as expected
  - Missing nested `values` members resolve to defaults silently
  - `status` comes from `body.status`, not `body.values.status`
- **Sub-workflow reference:** none

#### Send Success Response1
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends a success confirmation back to the caller after validation and normalization.
- **Configuration choices:**
  - HTTP status: `200`
  - JSON response body:
    - `success: true`
    - `message: Measurement data received successfully`
    - `measurementId: {{$json.measurementId}}`
  - `enableResponseOutput: true`, so data continues to downstream nodes
- **Key expressions or variables used:**
  - `{{ $json.measurementId }}`
- **Input and output connections:**
  - Input: `Parse and Normalize Data1`
  - Output: `Route to Output Options1`
- **Version-specific requirements:**  
  Type version `1.5`
- **Edge cases or potential failure types:**
  - Because routing continues after responding, downstream failures will not be visible to the webhook caller
  - Combined with other `Respond to Webhook` nodes, this may create conflicting response logic
- **Sub-workflow reference:** none

---

## 2.4 Output Routing

### Overview
This block chooses one downstream destination according to `outputMode` from the configuration node. It is the main dispatch mechanism for the workflow.

### Nodes Involved
- Route to Output Options1

### Node Details

#### Route to Output Options1
- **Type and technical role:** `n8n-nodes-base.switch`  
  Multi-branch router selecting one destination path.
- **Configuration choices:**
  - Branch 1: `sheets` → output labeled `Google Sheets`
  - Branch 2: `html` → output labeled `HTML Page`
  - Branch 3: `csv` → output labeled `CSV Email`
  - Branch 4: `custom` → output labeled `Custom Code`
  - Each condition compares `$('Workflow Configuration').item.json.outputMode`
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').item.json.outputMode }}`
- **Input and output connections:**
  - Input: `Send Success Response1`
  - Outputs:
    - `Save to Google Sheets1`
    - `Generate HTML Table1`
    - `Convert to CSV File1`
    - `Custom JavaScript Logic1`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**
  - If `outputMode` is not one of the four expected values, nothing executes after this node
  - Since it references the configuration node by name, renaming that node breaks expressions unless updated
- **Sub-workflow reference:** none

---

## 2.5 Google Sheets Output

### Overview
This branch persists the normalized measurement record to a Google Sheet. It uses append-or-update behavior keyed by `measurementId`.

### Nodes Involved
- Save to Google Sheets1

### Node Details

#### Save to Google Sheets1
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Writes data to a Google Sheets document.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Sheet name: `Measurements`
  - Document ID from configuration:
    - `{{ $('Workflow Configuration').item.json.googleSheetId }}`
  - Matching column: `measurementId`
  - `useAppend: true`
  - Explicit field mapping for all normalized fields
- **Key expressions or variables used:**
  - `{{ $json.measurementId }}`
  - `{{ $json.timestamp }}`
  - `{{ $('Workflow Configuration').item.json.googleSheetId }}`
- **Input and output connections:**
  - Input: `Route to Output Options1` on the `Google Sheets` branch
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`; field mapping UI and append/update behavior depend on the installed node version.
- **Edge cases or potential failure types:**
  - Missing or invalid Google credentials
  - Invalid sheet ID
  - Sheet `Measurements` does not exist
  - Matching column absent or mismatched in the target sheet
  - API quota limits or permission errors
- **Sub-workflow reference:** none

---

## 2.6 HTML Output

### Overview
This branch generates a styled HTML table containing the normalized measurement data and returns it as an HTTP response suitable for a browser.

### Nodes Involved
- Generate HTML Table1
- Return HTML Page1

### Node Details

#### Generate HTML Table1
- **Type and technical role:** `n8n-nodes-base.html`  
  Produces HTML markup using item data interpolation.
- **Configuration choices:**
  - Static HTML template with embedded expressions for all measurement fields
  - Includes inline CSS styling
- **Key expressions or variables used:**
  - `{{ $json.measurementId }}`
  - `{{ $json.timestamp }}`
  - Similar expressions for all normalized fields
- **Input and output connections:**
  - Input: `Route to Output Options1` on the `HTML Page` branch
  - Output: `Return HTML Page1`
- **Version-specific requirements:**  
  Type version `1.2`
- **Edge cases or potential failure types:**
  - Unescaped field values could affect rendering if they contain HTML-sensitive content
  - Large text in `notes` may break table layout
- **Sub-workflow reference:** none

#### Return HTML Page1
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the generated HTML back to the requester.
- **Configuration choices:**
  - Response code: `200`
  - Header: `Content-Type: text/html`
  - Response body: `{{ $json.html }}`
  - Response mode: text
- **Key expressions or variables used:**
  - `{{ $json.html }}`
- **Input and output connections:**
  - Input: `Generate HTML Table1`
  - Output: none
- **Version-specific requirements:**  
  Type version `1.5`
- **Edge cases or potential failure types:**
  - This node conflicts conceptually with the earlier success response node because only one webhook response should usually be sent
  - If a response has already been sent by `Send Success Response1`, this branch may not behave as intended
- **Sub-workflow reference:** none

---

## 2.7 CSV Email Output

### Overview
This branch converts the item data into a file and emails it as an attachment. It is suitable when measurement records need to be forwarded to a recipient as a CSV export.

### Nodes Involved
- Convert to CSV File1
- Send Email with CSV1

### Node Details

#### Convert to CSV File1
- **Type and technical role:** `n8n-nodes-base.convertToFile`  
  Converts incoming JSON data into a file for downstream attachment handling.
- **Configuration choices:**
  - File name expression:
    - `measurement-data-{{ current timestamp }}.csv`
- **Key expressions or variables used:**
  - `{{ 'measurement-data-' + $now.format('yyyy-MM-dd-HHmmss') + '.csv' }}`
- **Input and output connections:**
  - Input: `Route to Output Options1` on the `CSV Email` branch
  - Output: `Send Email with CSV1`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**
  - Output format assumptions depend on node defaults
  - If the binary property name differs from what the email node expects, attachments may fail
- **Sub-workflow reference:** none

#### Send Email with CSV1
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends an email with the generated file attached.
- **Configuration choices:**
  - To: `{{ $('Workflow Configuration').item.json.emailRecipient }}`
  - Subject: `{{ $('Workflow Configuration').item.json.emailSubject }}`
  - Text body references parsed timestamp
  - Attachment source: binary property `data`
  - From email: `user@example.com`
  - Email format: `text`
- **Key expressions or variables used:**
  - `{{ $('Parse and Normalize Data1').item.json.timestamp }}`
  - `{{ $('Workflow Configuration').item.json.emailSubject }}`
  - `{{ $('Workflow Configuration').item.json.emailRecipient }}`
- **Input and output connections:**
  - Input: `Convert to CSV File1`
  - Output: none
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**
  - SMTP credentials not configured
  - Sender address rejected by mail server
  - Missing binary property `data`
  - Invalid recipient or provider rate limits
- **Sub-workflow reference:** none

---

## 2.8 Custom JavaScript Output

### Overview
This branch is an extensibility point. It contains example JavaScript for API posting, storage preparation, triggering another workflow, and simple analytics, but its sample schema does not match the normalized measurement structure used elsewhere in this workflow.

### Nodes Involved
- Custom JavaScript Logic1

### Node Details

#### Custom JavaScript Logic1
- **Type and technical role:** `n8n-nodes-base.code`  
  Executes arbitrary JavaScript over incoming items.
- **Configuration choices:**
  - Reads all input items using `$input.all()`
  - Builds a `measurements` array expecting fields like:
    - `sensorId`
    - `temperature`
    - `humidity`
    - `pressure`
  - Makes sample external HTTP requests with `$http.request`
  - Prepares CSV content for S3-like storage
  - Triggers another workflow via webhook
  - Computes statistics on temperatures
  - Returns a summary object
- **Key expressions or variables used:**
  - `$input.all()`
  - `$http.request(...)`
  - `item.json.timestamp`
  - `item.json.sensorId`
  - `item.json.temperature`
- **Input and output connections:**
  - Input: `Route to Output Options1` on the `Custom Code` branch
  - Output: none
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**
  - The example code expects a different schema than this workflow provides; normalized data contains `deviceId`, `dimension_x`, etc., not `sensorId`, `temperature`, `humidity`, `pressure`
  - This mismatch can lead to `undefined` values, empty calculations, or invalid API payloads
  - The average temperature calculation can produce `NaN` if no valid temperatures exist
  - Uses placeholder URLs and tokens that must be replaced
  - External HTTP calls may fail due to auth, network, timeout, or DNS errors
- **Sub-workflow reference:**  
  Not an n8n sub-workflow node, but it includes an example external call to another n8n webhook:
  - `https://avikadam.app.n8n.cloud/webhook/alert-workflow`

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger - Receive Measurement Data | webhook | Receives POST measurement data at `/measurement-data` |  | Workflow Configuration | # Measurement Data Routing Workflow<br>This workflow receives measurement data via webhook and routes it to multiple output destinations based on configuration.<br>## How it works:<br>1. **Webhook receives** POST requests with measurement data (ID, timestamp, dimensions, tolerance, metadata)<br>2. **Validation** ensures required fields are present; returns error if missing<br>3. **Data normalization** extracts and structures all measurement fields with defaults for optional values<br>4. **Success response** confirms receipt immediately (200 OK)<br>5. **Routing** directs data to one of four outputs based on the outputMode setting:<br>- **sheets**: Appends to Google Sheets for tracking and analysis<br>- **html**: Returns formatted HTML table for browser viewing<br>- **csv**: Emails CSV file attachment to specified recipient<br>- **custom**: Runs JavaScript code for API calls, cloud storage, or custom logic<br>## Setup steps:<br>1. **Configure output mode**: Edit "Workflow Configuration" node and set outputMode to your preferred destination<br>2. **Google Sheets** (if using sheets mode): Add Google Sheets credentials and update googleSheetId with your sheet ID<br>3. **Email** (if using csv mode): Add SMTP credentials and set emailRecipient to the destination address<br>4. **Custom code** (if using custom mode): Modify the JavaScript node with your integration logic<br>5. **Test**: Send a POST request to the webhook URL with sample measurement data<br>## How to get the Webhook URL:<br>- **Double click** on the 1st node **Webhook Trigger - Receive Measurement Data**<br>- You can see Test / Production URL section at the top in drop down ⌄ Webhook URLs<br>**Ex.** Webhook URL format: https://{your-cloud-name}.app.n8n.cloud/webhook/measurement-data<br>## How to take measurements:<br>- Import and activate the workflow in n8n.<br>- Copy the **Webhook URL** from the Webhook Trigger node.<br>- Open the **3D Measure Up Web Application**: https://3dmeasureup.ai/<br>- Log in and navigate to **Settings → Webhooks**.<br>- Paste the webhook URL into the **Webhook URL** field and click **Save**.<br>### Now, whenever you take a measurement with 3D Measure Up, it will be available in your n8n workflow<br>## Webhook Entry Point<br>Receives POST requests with measurement data at:<br>`/measurement-data`<br>**Expected payload:**<br>- measurementId (required)<br>- timestamp (required)<br>- values (required)<br>- deviceId, operator, partNumber (optional) |
| Workflow Configuration | set | Injects output mode and destination settings | Webhook Trigger - Receive Measurement Data | Validate Required Fields1 | ## Workflow Configuration<br>**Key Settings:**<br>- outputMode: 'sheets' | 'html' | 'csv' | 'custom'<br>- googleSheetId: Your Google Sheet ID<br>- emailRecipient: Email address for CSV<br>- emailSubject: Email subject line<br>Modify these values to customize behavior. |
| Validate Required Fields1 | if | Validates required webhook body fields | Workflow Configuration | Parse and Normalize Data1; Error - Invalid Payload; Send Error Response | ## Validation & Error Handling<br>**Checks for required fields:**<br>- measurementId<br>- timestamp<br>- values<br>**If validation fails:**<br>- Returns 400 error response<br>- Stops workflow execution |
| Error - Invalid Payload | stopAndError | Stops execution on invalid payload | Validate Required Fields1 |  | ## Validation & Error Handling<br>**Checks for required fields:**<br>- measurementId<br>- timestamp<br>- values<br>**If validation fails:**<br>- Returns 400 error response<br>- Stops workflow execution |
| Send Error Response | respondToWebhook | Returns a 400 JSON error response | Validate Required Fields1 |  | ## Validation & Error Handling<br>**Checks for required fields:**<br>- measurementId<br>- timestamp<br>- values<br>**If validation fails:**<br>- Returns 400 error response<br>- Stops workflow execution |
| Parse and Normalize Data1 | set | Flattens and normalizes payload fields | Validate Required Fields1 | Send Success Response1 | ## Data Processing<br>**Parse and Normalize:**<br>Extracts and structures measurement data including dimensions (x, y, z), tolerance, status, and metadata.<br>**Success Response:**<br>Returns 200 OK with confirmation message and measurementId. |
| Send Success Response1 | respondToWebhook | Sends success acknowledgment and passes data onward | Parse and Normalize Data1 | Route to Output Options1 | ## Data Processing<br>**Parse and Normalize:**<br>Extracts and structures measurement data including dimensions (x, y, z), tolerance, status, and metadata.<br>**Success Response:**<br>Returns 200 OK with confirmation message and measurementId. |
| Route to Output Options1 | switch | Routes execution by configured output mode | Send Success Response1 | Save to Google Sheets1; Generate HTML Table1; Convert to CSV File1; Custom JavaScript Logic1 | ## Output Routing<br>Routes data based on `outputMode` configuration:<br>- **sheets** → Google Sheets<br>- **html** → HTML Page<br>- **csv** → CSV Email<br>- **custom** → Custom JavaScript<br>## Set outputMode in Workflow Configuration node. |
| Save to Google Sheets1 | googleSheets | Appends or updates a row in Google Sheets | Route to Output Options1 |  | ## Google Sheets Output<br>Appends/updates measurement data to Google Sheets.<br>⚠️ **Setup Required:**<br>- Configure Google Sheets credentials<br>- Set googleSheetId in Workflow Configuration<br>- Sheet name: "Measurements" |
| Generate HTML Table1 | html | Builds a styled HTML report from measurement data | Route to Output Options1 | Return HTML Page1 | ## 📄 HTML Report Output<br>Generates formatted HTML table with measurement data and returns as web page.<br>## Useful for viewing data directly in browser. |
| Return HTML Page1 | respondToWebhook | Returns generated HTML to the requester | Generate HTML Table1 |  | ## 📄 HTML Report Output<br>Generates formatted HTML table with measurement data and returns as web page.<br>## Useful for viewing data directly in browser. |
| Convert to CSV File1 | convertToFile | Converts item data into a CSV file | Route to Output Options1 | Send Email with CSV1 | ## CSV Email Output<br>Converts data to CSV file and sends via email.<br>⚠️ **Setup Required:**<br>- Configure SMTP credentials<br>- Set emailRecipient in Workflow Configuration<br>- Customize emailSubject if needed |
| Send Email with CSV1 | emailSend | Emails the generated CSV attachment | Convert to CSV File1 |  | ## CSV Email Output<br>Converts data to CSV file and sends via email.<br>⚠️ **Setup Required:**<br>- Configure SMTP credentials<br>- Set emailRecipient in Workflow Configuration<br>- Customize emailSubject if needed |
| Custom JavaScript Logic1 | code | Runs custom extensible JavaScript logic | Route to Output Options1 |  | ## Custom JavaScript Logic<br>Extensible code node with examples for:<br>- External API calls<br>- S3/cloud storage uploads<br>- Triggering other workflows<br>- Custom calculations<br>- Data transformations<br>## Modify the code to fit your needs! |
| Sticky Note | stickyNote | Canvas documentation note |  |  |  |
| Note - Webhook Entry Point | stickyNote | Canvas documentation note |  |  |  |
| Note - Validation & Error Handling | stickyNote | Canvas documentation note |  |  |  |
| Note - Data Processing | stickyNote | Canvas documentation note |  |  |  |
| Note - Output Routing | stickyNote | Canvas documentation note |  |  |  |
| Note - Google Sheets Output | stickyNote | Canvas documentation note |  |  |  |
| Note - HTML Output | stickyNote | Canvas documentation note |  |  |  |
| Note - CSV Email Output | stickyNote | Canvas documentation note |  |  |  |
| Note - Custom JavaScript | stickyNote | Canvas documentation note |  |  |  |
| Note - Configuration | stickyNote | Canvas documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Route measurement data from a webhook to Google Sheets, email, or custom JS`.

2. **Add a Webhook node** named `Webhook Trigger - Receive Measurement Data`.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `measurement-data`
   - Response Mode: `Last Node`
   - Save the node.
   - This creates an endpoint like `/webhook/measurement-data`.

3. **Add a Set node** named `Workflow Configuration`.
   - Connect it after the webhook node.
   - Enable keeping other incoming fields.
   - Add these fields:
     - `outputMode` as string: `sheets`
     - `googleSheetId` as string: your Google Sheet ID
     - `emailRecipient` as string: recipient email address
     - `emailSubject` as string: `Measurement Data Export`

4. **Add an IF node** named `Validate Required Fields1`.
   - Connect it after `Workflow Configuration`.
   - Configure AND conditions checking that these expressions exist:
     - `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.measurementId }}`
     - `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.timestamp }}`
     - `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.values }}`

5. **Add a Set node** named `Parse and Normalize Data1`.
   - Connect it to the **true** output of `Validate Required Fields1`.
   - Add these fields:
     - `measurementId` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.measurementId }}`
     - `timestamp` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.timestamp }}`
     - `deviceId` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.deviceId || 'unknown' }}`
     - `operator` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.operator || 'N/A' }}`
     - `partNumber` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.partNumber || 'N/A' }}`
     - `dimension_x` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.values.x || 0 }}`
     - `dimension_y` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.values.y || 0 }}`
     - `dimension_z` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.values.z || 0 }}`
     - `tolerance` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.values.tolerance || 0 }}`
     - `status` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.status || 'received' }}`
     - `notes` → `{{ $('Webhook Trigger - Receive Measurement Data').item.json.body.notes || '' }}`

6. **Add a Respond to Webhook node** named `Send Success Response1`.
   - Connect it after `Parse and Normalize Data1`.
   - Respond With: `JSON`
   - HTTP Status Code: `200`
   - Response Body:
     - `success: true`
     - `message: Measurement data received successfully`
     - `measurementId: {{ $json.measurementId }}`
   - Enable response output so the workflow continues after responding.

7. **Add a Switch node** named `Route to Output Options1`.
   - Connect it after `Send Success Response1`.
   - Add four rules based on:
     - `{{ $('Workflow Configuration').item.json.outputMode }}`
   - Configure outputs:
     - equals `sheets`
     - equals `html`
     - equals `csv`
     - equals `custom`

8. **Create the Google Sheets branch**.
   - Add a `Google Sheets` node named `Save to Google Sheets1`.
   - Connect it to the `sheets` output of the switch.
   - Configure credentials: Google Sheets OAuth2 or Service Account as supported by your n8n setup.
   - Operation: `Append or Update`
   - Document ID: `{{ $('Workflow Configuration').item.json.googleSheetId }}`
   - Sheet Name: `Measurements`
   - Matching Column: `measurementId`
   - Map fields:
     - measurementId
     - timestamp
     - deviceId
     - operator
     - partNumber
     - dimension_x
     - dimension_y
     - dimension_z
     - tolerance
     - status
     - notes
   - Ensure the target sheet contains matching column headers.

9. **Create the HTML branch**.
   - Add an `HTML` node named `Generate HTML Table1`.
   - Connect it to the `html` output of the switch.
   - Paste an HTML template containing a table with all normalized fields:
     - measurementId
     - timestamp
     - deviceId
     - operator
     - partNumber
     - dimension_x
     - dimension_y
     - dimension_z
     - tolerance
     - status
     - notes
   - Use `{{ $json.fieldName }}` inside the HTML.

10. **Add a Respond to Webhook node** named `Return HTML Page1`.
    - Connect it after `Generate HTML Table1`.
    - Respond With: `Text`
    - Response code: `200`
    - Header:
      - `Content-Type: text/html`
    - Response body:
      - `{{ $json.html }}`

11. **Create the CSV email branch**.
    - Add a `Convert to File` node named `Convert to CSV File1`.
    - Connect it to the `csv` output of the switch.
    - Configure a dynamic file name:
      - `{{ 'measurement-data-' + $now.format('yyyy-MM-dd-HHmmss') + '.csv' }}`
    - Ensure the node outputs a binary file property, typically `data`.

12. **Add an Email Send node** named `Send Email with CSV1`.
    - Connect it after `Convert to CSV File1`.
    - Configure SMTP credentials in n8n.
    - Set:
      - To: `{{ $('Workflow Configuration').item.json.emailRecipient }}`
      - Subject: `{{ $('Workflow Configuration').item.json.emailSubject }}`
      - From: a valid sender address allowed by your SMTP provider
      - Format: `Text`
      - Body: `{{ 'Please find attached the measurement data export from ' + $('Parse and Normalize Data1').item.json.timestamp }}`
      - Attachments: `data`

13. **Create the custom code branch**.
    - Add a `Code` node named `Custom JavaScript Logic1`.
    - Connect it to the `custom` output of the switch.
    - Paste the custom JavaScript logic.
    - Important: adapt the example code so it matches this workflow’s normalized fields, because the sample script expects fields like `sensorId` and `temperature` rather than `deviceId` and dimensional values.
    - If using external APIs:
      - replace sample URLs
      - replace placeholder bearer tokens
      - validate response and timeout behavior

14. **Add invalid payload handling**.
    - Add a `Stop and Error` node named `Error - Invalid Payload`.
    - Connect it to the **false** output of `Validate Required Fields1`.
    - Error message:
      - `Invalid payload: Missing required fields (measurementId, timestamp, values)`

15. **Add a Respond to Webhook node** named `Send Error Response`.
    - Also connect it to the **false** output of `Validate Required Fields1`.
    - Configure:
      - Response code: `400`
      - Respond with JSON
      - Response body indicating required fields are missing

16. **Set credentials**.
    - For Google Sheets:
      - create or assign a valid Google Sheets credential
      - confirm the sheet is accessible by the authenticated account
    - For Email:
      - create SMTP credentials
      - ensure the sender address is authorized
    - For custom code:
      - no credential object is directly configured in this node, so secrets should be handled carefully; ideally move API calls to dedicated credential-aware nodes if possible

17. **Prepare the Google Sheet if using `sheets` mode**.
    - Create a spreadsheet.
    - Create a tab named `Measurements`.
    - Add headers:
      - `measurementId`
      - `timestamp`
      - `deviceId`
      - `operator`
      - `partNumber`
      - `dimension_x`
      - `dimension_y`
      - `dimension_z`
      - `tolerance`
      - `status`
      - `notes`

18. **Set the desired output mode** in `Workflow Configuration`.
    - `sheets` for Google Sheets storage
    - `html` for HTML response
    - `csv` for emailed CSV
    - `custom` for custom JavaScript

19. **Test with a sample POST payload**, for example:
    - `measurementId`: unique string
    - `timestamp`: ISO date-time string
    - `values`: object with `x`, `y`, `z`, `tolerance`
    - optional: `deviceId`, `operator`, `partNumber`, `status`, `notes`

20. **Activate the workflow** after testing.

### Reproduction Notes and Constraints
- The workflow has **one formal entry point**: the webhook node.
- It does **not** use n8n Execute Workflow / sub-workflow nodes.
- The custom code node includes an example call to another workflow webhook, but that downstream target is external and optional.
- If you want HTML mode to be the only HTTP response, you should redesign the response strategy because the current workflow also sends an earlier success response.
- A more robust design would usually choose one of:
  - immediate JSON acknowledgment plus asynchronous processing
  - or a branch-specific final webhook response, but not both

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| 3D Measure Up Web Application | https://3dmeasureup.ai/ |
| Webhook URL format example: `https://{your-cloud-name}.app.n8n.cloud/webhook/measurement-data` | Use this pattern to configure the measurement sender |
| To find the webhook URL, open the webhook node and use the Test / Production URL dropdown in the Webhook URLs section | n8n editor usage note |
| The workflow is designed so 3D Measure Up can send measurement payloads directly into n8n | Integration context |
| The custom JavaScript example includes a sample downstream webhook call to `https://avikadam.app.n8n.cloud/webhook/alert-workflow` | Example only; replace or remove in production |
| The custom JavaScript sample schema does not match the normalized measurement schema used elsewhere in the workflow | Important implementation note |
| The workflow includes multiple Respond to Webhook nodes; response behavior should be reviewed carefully before production use | Architecture note |