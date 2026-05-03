TwentyCRM event based updates on selective messaging channels with logs

https://n8nworkflows.xyz/workflows/twentycrm-event-based-updates-on-selective-messaging-channels-with-logs-2509


# TwentyCRM event based updates on selective messaging channels with logs

### 1. Workflow Overview

This workflow is designed to provide automated notifications and updates triggered by events in TwentyCRM, a CRM platform. It targets DevOps, Engineering, and Managed Service Provider professionals who require alerts routed through different messaging channels based on the severity or type of the event. The workflow leverages TwentyCRM’s webhook events, filters and logs received event data, and conditionally routes notifications via email or Slack according to the event type.

Logical blocks:

- **1.1 Event Reception:** Captures incoming webhook events from TwentyCRM.
- **1.2 Data Filtering:** Extracts and formats the essential event data fields for processing.
- **1.3 Event Logging:** Appends event details to a Google Sheets document for audit and tracking.
- **1.4 Channel Selection & Messaging:** Evaluates the event type to determine the appropriate messaging channel and sends notifications accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Event Reception

**Overview:**  
Receives real-time event data from TwentyCRM’s webhook trigger to initiate the workflow.

**Nodes Involved:**  
- on new twentycrm event (Webhook)

**Node Details:**

- **on new twentycrm event**  
  - *Type:* Webhook (Trigger)  
  - *Role:* Listens for POST requests from TwentyCRM’s webhook system.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Path: `8118bda9-0e4f-44cd-bf64-31020b6d5ab5` (unique webhook endpoint)  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Raw JSON payload from TwentyCRM webhook available in `$json.body`  
  - *Version Requirements:* Compatible with n8n v1.63.4 and later  
  - *Failure Modes:*  
    - Webhook misconfiguration in TwentyCRM (no data received)  
    - HTTP method mismatch  
    - Network or authentication errors  
  - *Sticky Notes:*  
    - "1. ☝️ Set up `On new TwentyCRM event` Trigger's url at webhook in TwentyCRM" (Sticky Note1)  
    - Workflow title and summary note (Sticky Note4)

#### 1.2 Data Filtering

**Overview:**  
Extracts only the relevant fields from the webhook payload needed downstream, ensuring the presence of mandatory `eventType` (mapped as `eventName`) for correct routing.

**Nodes Involved:**  
- filter required data #eventType mandatory (Set)

**Node Details:**

- **filter required data #eventType mandatory**  
  - *Type:* Set  
  - *Role:* Creates a simplified JSON object with key event properties for easy access.  
  - *Configuration:*  
    - Dot notation enabled for nested fields  
    - Assigns:  
      - `eventName` ← `$json.body.eventName` (event type string)  
      - `objectMetadata.id` ← `$json.body.objectMetadata.id` (object unique ID)  
      - `objectMetadata.nameSingular` ← `$json.body.objectMetadata.nameSingular` (object name)  
      - `record.id` ← `$json.body.record.id` (record unique ID)  
      - `record.__typename` ← `$json.body.record.__typename` (record type)  
  - *Inputs:* Raw webhook JSON data  
  - *Outputs:* Filtered JSON with only necessary fields  
  - *Potential Failures:*  
    - Missing or malformed webhook payload fields leading to undefined variables  
    - Expression evaluation errors if fields are null or undefined  
  - *Sticky Notes:*  
    - "Filter Data 👇 Change filter criteria here to determine what values are required for you but don't forget to include eventType as it is a functional requirement" (Sticky Note2)

#### 1.3 Event Logging

**Overview:**  
Logs each filtered event entry into a Google Sheet for record-keeping and auditing.

**Nodes Involved:**  
- events log (Google Sheets)

**Node Details:**

- **events log**  
  - *Type:* Google Sheets  
  - *Role:* Appends the filtered event data as a new row to a specified Google Sheets document.  
  - *Configuration:*  
    - Operation: Append  
    - Sheet name and Document ID must be configured (dynamic or static)  
  - *Inputs:* Filtered event data from the Set node  
  - *Outputs:* Confirmation of row append operation  
  - *Credentials:* Google OAuth2 with Gmail API and Google Sheets scopes required  
  - *Potential Failures:*  
    - Credential issues (expired or missing scopes)  
    - Missing or incorrect spreadsheet/document ID or sheet name  
    - API quota limits or timeouts  
  - *Sticky Notes:*  
    - "👈 event loggin All events are logged in the sheet with one entry per row" (Sticky Note)

#### 1.4 Channel Selection & Messaging

**Overview:**  
Selects the messaging channel based on event type and sends notifications. Specifically, `delete` events trigger an email via Gmail, while all other events send a Slack message.

**Nodes Involved:**  
- message channel evaluation (If)  
- email channel for delete eventType (Gmail)  
- message channel for all other eventTypes (Slack)

**Node Details:**

- **message channel evaluation**  
  - *Type:* If  
  - *Role:* Evaluates if the event type is a `delete` operation to route the message accordingly.  
  - *Configuration:*  
    - Expression: `{{$json.eventName.split(".")[1]}} == "delete"`  
    - Case sensitive, strict type validation  
  - *Inputs:* Filtered event data  
  - *Outputs:*  
    - True branch → `email channel for delete eventType`  
    - False branch → `message channel for all other eventTypes`  
  - *Potential Failures:*  
    - Expression errors if `eventName` is undefined or does not contain a period  
    - Logic errors if event naming conventions change  

- **email channel for delete eventType**  
  - *Type:* Gmail  
  - *Role:* Sends an HTML formatted email alert for delete events.  
  - *Configuration:*  
    - Subject: "Record Deleted in TwentyCRM"  
    - Message body includes:  
      - `objectMetadata_id` and `record_id` from filtered data  
    - Credentials: Google OAuth2 with Gmail API scope  
  - *Inputs:* True branch output from If node  
  - *Outputs:* Email sent confirmation  
  - *Potential Failures:*  
    - Authentication errors (OAuth token expired or missing)  
    - Email sending limits or quota exceeded  
    - Missing data in template expressions  

- **message channel for all other eventTypes**  
  - *Type:* Slack  
  - *Role:* Posts a message to a Slack channel for all non-delete events.  
  - *Configuration:*  
    - Text includes event name, event ID, and record ID  
    - Channel ID must be set (can be dynamic or static)  
    - Credentials: Slack OAuth2 with `chat:write` scope  
  - *Inputs:* False branch output from If node  
  - *Outputs:* Slack message confirmation  
  - *Potential Failures:*  
    - Slack OAuth issues  
    - Incorrect or missing channel ID  
    - Message formatting errors  

- *Sticky Notes:*  
  - "Evaluation 👇 Based on the conditions proper channel for messaging is selected" (Sticky Note3)

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                             | Input Node(s)                 | Output Node(s)                                 | Sticky Note                                                      |
|--------------------------------|--------------------------|---------------------------------------------|------------------------------|-----------------------------------------------|-----------------------------------------------------------------|
| on new twentycrm event          | Webhook                  | Receive TwentyCRM events                     | None                         | filter required data #eventType mandatory      | 1. ☝️ Set up `On new TwentyCRM event` Trigger's url at webhook in TwentyCRM; Workflow title note |
| filter required data #eventType mandatory | Set                      | Extract and simplify event data              | on new twentycrm event       | events log, message channel evaluation          | Filter Data 👇 Change filter criteria here to determine what values are required for you but don't forget to include eventType as it is a functional requirement |
| events log                     | Google Sheets            | Append event data to Google Sheets           | filter required data #eventType mandatory | message channel evaluation                      | 👈 event loggin All events are logged in the sheet with one entry per row |
| message channel evaluation     | If                       | Choose messaging channel based on event type| filter required data #eventType mandatory, events log | email channel for delete eventType, message channel for all other eventTypes | Evaluation 👇 Based on the conditions proper channel for messaging is selected |
| email channel for delete eventType | Gmail                    | Send email alerts for delete events          | message channel evaluation   | None                                          |                                                                 |
| message channel for all other eventTypes | Slack                    | Send Slack messages for other event types    | message channel evaluation   | None                                          |                                                                 |
| Sticky Note1                   | Sticky Note              | Instruction to set up webhook URL            | None                         | None                                          | 1. ☝️ Set up `On new TwentyCRM event` Trigger's url at webhook in TwentyCRM |
| Sticky Note2                   | Sticky Note              | Guidance on filter node setup                 | None                         | None                                          | Filter Data 👇 Change filter criteria here to determine what values are required for you but don't forget to include eventType as it is a functional requirement |
| Sticky Note                    | Sticky Note              | Note on event logging                          | None                         | None                                          | 👈 event loggin All events are logged in the sheet with one entry per row |
| Sticky Note3                   | Sticky Note              | Note on channel evaluation logic              | None                         | None                                          | Evaluation 👇 Based on the conditions proper channel for messaging is selected |
| Sticky Note4                   | Sticky Note              | Workflow title and summary                     | None                         | None                                          | ### Get event triggered notifications / updates on preferred messaging channels with TwentyCRM ### |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node:**  
   - Add a **Webhook** node named `on new twentycrm event`  
   - Set HTTP Method to `POST`  
   - Set Path to a unique identifier (e.g., `8118bda9-0e4f-44cd-bf64-31020b6d5ab5`)  
   - This node will receive incoming events from TwentyCRM’s webhook.

2. **Add a Set Node for Data Filtering:**  
   - Add a **Set** node named `filter required data #eventType mandatory`  
   - Enable “Dot notation” and “Ignore conversion errors” options  
   - Assign the following values from the webhook JSON (`$json.body`):  
     - `eventName` ← expression: `{{$json.body.eventName}}`  
     - `objectMetadata.id` ← `{{$json.body.objectMetadata.id}}`  
     - `objectMetadata.nameSingular` ← `{{$json.body.objectMetadata.nameSingular}}`  
     - `record.id` ← `{{$json.body.record.id}}`  
     - `record.__typename` ← `{{$json.body.record.__typename}}`  
   - Connect the output of the webhook node to this Set node.

3. **Add Google Sheets Node for Logging:**  
   - Add a **Google Sheets** node named `events log`  
   - Set operation to `Append`  
   - Configure credentials with Google OAuth2 that includes Gmail API and Google Sheets scopes  
   - Specify the **Document ID** and **Sheet Name** where logs will be stored  
   - Connect the output of the Set node to this Google Sheets node.

4. **Add If Node for Channel Evaluation:**  
   - Add an **If** node named `message channel evaluation`  
   - Configure condition as:  
     - Expression: `{{$json.eventName.split(".")[1]}}`  
     - Operator: equals  
     - Value: `"delete"`  
     - Case sensitive: true  
     - Type validation: strict  
   - Connect outputs from both the Set node and the Google Sheets node into this If node (multiple inputs supported).

5. **Add Gmail Node for Delete Event Email:**  
   - Add a **Gmail** node named `email channel for delete eventType`  
   - Configure with Google OAuth2 credentials including Gmail API scope  
   - Set subject to `"Record Deleted in TwentyCRM"`  
   - Use HTML message body with variables:  
     ```
     <h2>Please find below the attached record details</h2><br/><br/>
     <ul>
       <li>objectMetadata_id: {{ $json.objectMetadata.id }}</li>
       <li>record_id: {{ $json.record.id }}</li>
     </ul>
     ```  
   - Connect the **true** output of the If node to this Gmail node.

6. **Add Slack Node for Other Event Types:**  
   - Add a **Slack** node named `message channel for all other eventTypes`  
   - Configure with Slack OAuth2 credentials with `chat:write` scope  
   - Set the channel ID for message posting (static or dynamic URL)  
   - Set message text with variables:  
     ```
     event: {{ $json.eventName }}
     event_id: {{ $json.objectMetadata.id }}
     record_id: {{ $json.record.id }}
     ```  
   - Connect the **false** output of the If node to this Slack node.

7. **Connect Nodes:**  
   - `on new twentycrm event` → `filter required data #eventType mandatory`  
   - `filter required data #eventType mandatory` → `events log`  
   - Both `filter required data #eventType mandatory` and `events log` → `message channel evaluation`  
   - `message channel evaluation` (true) → `email channel for delete eventType`  
   - `message channel evaluation` (false) → `message channel for all other eventTypes`

8. **Credentials Setup:**  
   - Configure Google OAuth2 credentials with scopes: `https://www.googleapis.com/auth/gmail.send` and `https://www.googleapis.com/auth/spreadsheets`  
   - Configure Slack OAuth2 credentials with `chat:write` scope  
   - Assign credentials to the respective Gmail, Google Sheets, and Slack nodes.

9. **Final Steps:**  
   - Set up the webhook URL from the `on new twentycrm event` node inside TwentyCRM webhook settings for your app.  
   - Test the workflow with sample events.  
   - Once confirmed, move the webhook URL in TwentyCRM from test to production.  
   - Activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow template was created in n8n version `v1.63.4`.                                        | Workflow version compatibility                                                                                   |
| Set up Google OAuth2 credentials with Gmail API and Google Sheets scopes.                           | Required for Gmail and Google Sheets nodes                                                                        |
| Set up Slack OAuth2 credentials with `chat:write` scope.                                           | Required for Slack message posting                                                                                |
| Configure the webhook URL from the `on new twentycrm event` node inside TwentyCRM's webhook system. | Ensures event delivery to this workflow                                                                           |
| For more on using TwentyCRM webhooks see: https://www.twentycrm.com/docs/webhooks                   | External documentation link                                                                                        |
| Slack API documentation for chat.postMessage: https://api.slack.com/methods/chat.postMessage       | For customizing Slack message node parameters                                                                     |
| Google Sheets API reference: https://developers.google.com/sheets/api/reference/rest                 | For advanced Google Sheets node configurations                                                                    |