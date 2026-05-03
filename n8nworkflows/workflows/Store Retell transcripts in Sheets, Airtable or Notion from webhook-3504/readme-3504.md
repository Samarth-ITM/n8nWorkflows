Store Retell transcripts in Sheets, Airtable or Notion from webhook

https://n8nworkflows.xyz/workflows/store-retell-transcripts-in-sheets--airtable-or-notion-from-webhook-3504


# Store Retell transcripts in Sheets, Airtable or Notion from webhook

### 1. Workflow Overview

This workflow automates the storage of analyzed call data from Retell AI voice calls into popular productivity tools: Airtable, Google Sheets, and Notion. It listens for webhook events from Retell, specifically the `call_analyzed` event, extracts relevant call metadata and analysis results, and saves this structured data into the user’s preferred database or spreadsheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives webhook POST requests from Retell containing call event data.
- **1.2 Event Filtering:** Filters incoming webhook events to process only `call_analyzed` events.
- **1.3 Data Preparation:** Extracts and formats key call fields for export.
- **1.4 Data Storage:** Saves the prepared data into Airtable, Google Sheets, and Notion databases, depending on user configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives incoming webhook POST requests from Retell containing call event data.

- **Nodes Involved:**  
  - `Webhook`

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook (HTTP POST listener)  
    - Configuration: Listens on path `poc-retell-analysis` for POST requests.  
    - Key expressions: None (raw webhook data received).  
    - Input: External HTTP POST from Retell.  
    - Output: JSON payload containing call event data.  
    - Edge cases:  
      - Invalid or malformed webhook payloads may cause downstream expression failures.  
      - Unauthorized or unexpected HTTP methods are ignored by configuration.  
      - Network or connectivity issues may cause webhook delivery failures on Retell side.  
    - Version: n8n Webhook node v2.

#### 2.2 Event Filtering

- **Overview:**  
  Filters incoming webhook events to process only those where the event type is `call_analyzed`, ignoring other Retell events like `call_started` or `call_ended`.

- **Nodes Involved:**  
  - `Filter - only call ended`

- **Node Details:**  
  - **Filter - only call ended**  
    - Type: Filter node  
    - Configuration: Checks if `body.event` field equals `call_analyzed`.  
    - Key expression: `{{$json.body.event}} === 'call_analyzed'`  
    - Input: Output from `Webhook` node.  
    - Output: Passes data only if condition matches; otherwise, stops workflow execution.  
    - Edge cases:  
      - If the event field is missing or different, data is discarded.  
      - Expression failures if `body.event` is undefined.  
    - Version: Filter node v2.2.

#### 2.3 Data Preparation

- **Overview:**  
  Extracts relevant call data fields from the webhook payload, formats timestamps to local ISO strings, converts cost from cents to dollars, and selects the appropriate phone number based on call direction.

- **Nodes Involved:**  
  - `Set fields to export`

- **Node Details:**  
  - **Set fields to export**  
    - Type: Set node (data transformation)  
    - Configuration: Assigns new fields with expressions extracting and formatting data from the webhook JSON:  
      - Call ID  
      - Start and End datetime (converted from timestamps to local ISO strings)  
      - Duration in seconds  
      - Transcript text  
      - Call summary  
      - User sentiment  
      - Phone number (conditional on call direction: outbound uses `to_number`, inbound uses `from_number`)  
      - Total cost converted from cents to dollars  
    - Key expressions examples:  
      - `={{ $('Webhook').item.json.body.call.call_id }}`  
      - `={{ $('Webhook').item.json.body.call.start_timestamp.toDateTime('ms').toLocal().toISO() }}`  
      - `={{ $if($('Webhook').item.json.body.call.direction == 'outbound', $('Webhook').item.json.body.call.to_number, $('Webhook').item.json.body.call.from_number) }}`  
      - `={{ $json.body.call.call_cost.combined_cost/100 }}`  
    - Input: Output from `Filter - only call ended`.  
    - Output: Structured JSON with simplified fields for export.  
    - Edge cases:  
      - Missing or malformed timestamps may cause conversion errors.  
      - Missing cost or call direction fields may cause incorrect or missing data.  
      - Expression failures if referenced fields do not exist.  
    - Version: Set node v3.4.

#### 2.4 Data Storage

- **Overview:**  
  Saves the prepared call data into three possible destinations: Airtable, Google Sheets, and Notion. Each destination node is configured to map the extracted fields into the respective database or spreadsheet schema.

- **Nodes Involved:**  
  - `Save to Airtable`  
  - `Save to Excel` (Google Sheets)  
  - `Save to Notion`

- **Node Details:**  

  - **Save to Airtable**  
    - Type: Airtable node  
    - Configuration:  
      - Base and Table selected from user’s Airtable workspace (example base: "Retell sample", table: "Transcripts").  
      - Columns mapped explicitly from prepared fields (e.g., Call ID, Transcript, Call Summary, Start/End Datetime, Phone Number, User Sentiment, Duration, Total Cost).  
      - Operation: Create new record.  
    - Credentials: Airtable API token configured.  
    - Input: Output from `Set fields to export`.  
    - Output: Airtable record creation response.  
    - Edge cases:  
      - Authentication failures with Airtable API.  
      - Schema mismatch if Airtable table columns are missing or renamed.  
      - API rate limits or network errors.  
    - Version: Airtable node v2.1.

  - **Save to Excel** (Google Sheets)  
    - Type: Google Sheets node  
    - Configuration:  
      - Document ID and Sheet Name selected (example sheet: "Transcripts" tab).  
      - Columns mapped from prepared fields, with some fields forced as strings (e.g., Phone Number prefixed with `'` to preserve formatting).  
      - Operation: Append new row.  
    - Credentials: Google Sheets OAuth2 configured.  
    - Input: Output from `Set fields to export`.  
    - Output: Append operation response.  
    - Edge cases:  
      - OAuth token expiration or permission issues.  
      - Sheet or tab not found or renamed.  
      - API quota limits or network errors.  
    - Version: Google Sheets node v4.5.

  - **Save to Notion**  
    - Type: Notion node  
    - Configuration:  
      - Database ID selected (example database: "UserDB - Transcripts").  
      - Page title set to Call Summary.  
      - Properties mapped to Notion database columns with appropriate types (rich_text, number, date).  
    - Credentials: Notion API token configured.  
    - Input: Output from `Set fields to export`.  
    - Output: Created page response.  
    - Edge cases:  
      - Notion API authentication failures.  
      - Database schema mismatch or missing columns.  
      - API rate limits or network issues.  
    - Version: Notion node v2.2.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                          | Input Node(s)           | Output Node(s)                          | Sticky Note                                                                                                          |
|-----------------------|--------------------|----------------------------------------|------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Webhook               | Webhook            | Receive Retell webhook POST events     | -                      | Filter - only call ended              | POST Webhook receiving your Retell events                                                                            |
| Filter - only call ended | Filter             | Filter only `call_analyzed` events     | Webhook                | Set fields to export                  | Only keep the `call_analyzed` events (it contains all data points)                                                   |
| Set fields to export   | Set                | Extract and format call data fields    | Filter - only call ended | Save to Excel, Save to Airtable, Save to Notion | Prepare your data to be sent to your preferred database. If you add more data or post call analytics, add fields here. |
| Save to Airtable       | Airtable           | Save call data to Airtable              | Set fields to export    | -                                     | Save all fields from Retell to Airtable                                                                              |
| Save to Excel          | Google Sheets      | Save call data to Google Sheets         | Set fields to export    | -                                     | Save all fields from Retell to Google Sheets                                                                         |
| Save to Notion         | Notion             | Save call data to Notion database       | Set fields to export    | -                                     | Save all fields from Retell to Notion                                                                                |
| Sticky Note1           | Sticky Note        | Workflow overview and instructions      | -                      | -                                     | ## Automatically store Retell transcripts in Google Sheets/Airtable/Notion from webhook ... (full content)           |
| Sticky Note            | Sticky Note        | Webhook node comment                    | -                      | -                                     | POST Webhook receiving your Retell events                                                                            |
| Sticky Note2           | Sticky Note        | Filter node comment                     | -                      | -                                     | Only keep the `call_analyzed` events (it contains all data points)                                                   |
| Sticky Note3           | Sticky Note        | Set node comment                        | -                      | -                                     | Prepare your data to be sent to your preferred database. If you add more data or post call analytics, add fields here. |
| Sticky Note4           | Sticky Note        | Airtable node comment                   | -                      | -                                     | Save all fields from Retell to Airtable                                                                              |
| Sticky Note5           | Sticky Note        | Google Sheets node comment              | -                      | -                                     | Save all fields from Retell to Google Sheets                                                                         |
| Sticky Note6           | Sticky Note        | Notion node comment                     | -                      | -                                     | Save all fields from Retell to Notion                                                                                |
| Sticky Note7           | Sticky Note        | General instruction                     | -                      | -                                     | Remove the unnecessary tools # 👇                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `poc-retell-analysis`  
   - Purpose: Receive webhook POST requests from Retell AI.

2. **Create Filter Node**  
   - Name: `Filter - only call ended`  
   - Type: Filter  
   - Condition: Check if `{{$json.body.event}}` equals `call_analyzed`  
   - Connect Webhook node output to this node input.

3. **Create Set Node**  
   - Name: `Set fields to export`  
   - Type: Set  
   - Add fields with expressions:  
     - Call ID: `={{ $('Webhook').item.json.body.call.call_id }}`  
     - Start Datetime: `={{ $('Webhook').item.json.body.call.start_timestamp.toDateTime('ms').toLocal().toISO() }}`  
     - End Datetime: `={{ $('Webhook').item.json.body.call.end_timestamp.toDateTime('ms').toLocal().toISO() }}`  
     - Duration in seconds: `={{ $('Webhook').item.json.body.call.call_cost.total_duration_seconds }}`  
     - Transcript: `={{ $('Webhook').item.json.body.call.transcript }}`  
     - Call Summary: `={{ $('Webhook').item.json.body.call.call_analysis.call_summary }}`  
     - User Sentiment: `={{ $('Webhook').item.json.body.call.call_analysis.user_sentiment }}`  
     - Phone Number: `={{ $if($('Webhook').item.json.body.call.direction == 'outbound', $('Webhook').item.json.body.call.to_number, $('Webhook').item.json.body.call.from_number) }}`  
     - Total Cost in Dollars: `={{ $json.body.call.call_cost.combined_cost/100 }}`  
   - Connect Filter node output to this node input.

4. **Create Airtable Node**  
   - Name: `Save to Airtable`  
   - Type: Airtable  
   - Credentials: Configure Airtable API token with access to your base.  
   - Base: Select your Airtable base (e.g., "Retell sample")  
   - Table: Select your table (e.g., "Transcripts")  
   - Operation: Create  
   - Map columns to Set node fields: Call ID, Transcript, Call Summary, Start Datetime, End Datetime, Phone Number, User Sentiment, Duration in seconds, Total Cost in Dollars.  
   - Connect Set node output to this node input.

5. **Create Google Sheets Node**  
   - Name: `Save to Excel`  
   - Type: Google Sheets  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Document ID: Select your Google Sheet document (e.g., "Retell sample")  
   - Sheet Name: Select the tab (e.g., "Transcripts")  
   - Operation: Append  
   - Map columns similarly to Airtable node, ensuring phone number is prefixed with `'` to preserve formatting.  
   - Connect Set node output to this node input.

6. **Create Notion Node**  
   - Name: `Save to Notion`  
   - Type: Notion  
   - Credentials: Configure Notion API token with access to your database.  
   - Database ID: Select your Notion database (e.g., "UserDB - Transcripts")  
   - Resource: Database Page  
   - Title: Map to Call Summary  
   - Map properties to fields with correct types (rich_text, number, date): Call ID, Duration in seconds, End Datetime, Phone Number, Start Datetime, Total Cost in Dollars, Transcript, User Sentiment.  
   - Connect Set node output to this node input.

7. **Connect Nodes**  
   - Webhook → Filter - only call ended → Set fields to export → Save to Airtable  
   - Set fields to export → Save to Excel  
   - Set fields to export → Save to Notion

8. **Optional Adjustments**  
   - Remove any storage nodes (Airtable, Google Sheets, Notion) if not used.  
   - Ensure all credentials are properly set up and tested.  
   - Confirm database/table/sheet schemas match the mapped fields.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow stores Retell call transcripts and analysis automatically in Airtable, Google Sheets, or Notion based on webhook events.                                                                                             | Workflow purpose overview.                                                                               |
| Retell sends multiple webhook events (`call_started`, `call_ended`, `call_analyzed`); this workflow processes only `call_analyzed` events, which contain complete analysis data.                                                     | Retell webhook documentation: https://docs.retellai.com/features/webhook-overview#webhook-overview       |
| Phone numbers are extracted conditionally based on call direction (`from_number` for inbound, `to_number` for outbound).                                                                                                           | Data preparation logic.                                                                                   |
| Cost values are converted from cents to dollars before saving.                                                                                                                                                                      | Data preparation logic.                                                                                   |
| Dates are converted from timestamps to local ISO 8601 strings for consistency and readability.                                                                                                                                     | Data preparation logic.                                                                                   |
| Users can extend the workflow by adding custom post-call analysis fields from `call.call_analysis.custom_analysis_data` and mapping them in the Set node and storage nodes accordingly.                                            | Extension note.                                                                                           |
| Templates for Airtable, Google Sheets, and Notion databases are provided to ensure schema compatibility.                                                                                                                           | Airtable: https://airtable.com/appN4jeIrD8waWCfr/shrsPtQLeqt8Sp3UZ  
Google Sheets: https://docs.google.com/spreadsheets/d/1TYgk8PK5w2l8Q5NtepdyLvgtuHXBHcODy-2hXOPP6AU/edit?usp=sharing  
Notion: https://www.notion.so/1cea19b9d4848089bda6f3d7e05a818d?v=1cea19b9d48481ea97ef000ccd20f210&pvs=4 |
| Reach out to the workflow authors at hello@agentstudio.io for assistance or advanced Retell Agent conversation analysis services.                                                                                                   | Contact email: hello@agentstudio.io                                                                       |

---

This documentation provides a complete, structured understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.