Save Hotmart events to Google Sheets

https://n8nworkflows.xyz/workflows/save-hotmart-events-to-google-sheets-2702


# Save Hotmart events to Google Sheets

### 1. Workflow Overview

This workflow automates the tracking and recording of Hotmart events into Google Sheets, enabling creators and businesses to monitor sales, refunds, subscription changes, and cart abandonment in a centralized, organized manner. It listens for incoming Hotmart webhook events, categorizes them by event type, processes timestamps, formats data accordingly, and appends the relevant information into a Google Sheets document for further analysis.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives Hotmart event data via a webhook.
- **1.2 Event Classification:** Uses a switch node to route events based on their type.
- **1.3 Timestamp Conversion:** Converts event timestamps into a standardized format.
- **1.4 Data Preparation:** Sets and formats data fields for Google Sheets insertion.
- **1.5 Data Storage:** Saves the processed event data into Google Sheets.
- **1.6 Execution Logging:** Records execution metadata for debugging or auditing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives incoming HTTP POST requests from Hotmart containing event data.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook (HTTP endpoint)  
    - Configuration: Listens for Hotmart events on a unique webhook URL (webhookId: `439a86fe-819c-4516-9ebf-54e779b560c3`).  
    - Key Expressions/Variables: None explicitly configured; expects JSON payload from Hotmart.  
    - Input: External HTTP POST request from Hotmart.  
    - Output: Passes event data to the Switch node.  
    - Edge Cases:  
      - Invalid or malformed payloads may cause downstream processing errors.  
      - Network or authentication issues with Hotmart webhook delivery.  
    - Notes: This is the workflow’s entry point.

#### 2.2 Event Classification

- **Overview:**  
  Routes incoming events based on their type to handle different Hotmart event categories such as purchases, refunds, subscription updates, and cart abandonment.

- **Nodes Involved:**  
  - Switch

- **Node Details:**  
  - **Switch**  
    - Type: Switch (conditional routing)  
    - Configuration: Routes based on event type with cases including:  
      - `PURCHASE_PROTEST` (initial refund request)  
      - `SUBSCRIPTION_CANCELLATION` (subscription cancellation during guarantee period)  
      - `PURCHASE_REFUNDED` (confirmed refund)  
      - `PURCHASE_COMPLETE` (purchase finalized after guarantee period)  
      - Other event types likely handled similarly (7 outputs total).  
    - Key Expressions/Variables: Uses event type field from webhook payload to determine route.  
    - Input: Event data from Webhook node.  
    - Output: All routes lead to the "Convert timestamp" node.  
    - Edge Cases:  
      - Unknown or new event types not covered by switch cases will not be processed correctly.  
      - Misclassification if event type field is missing or malformed.  
    - Notes: Sticky note clarifies event type meanings and timing nuances.

#### 2.3 Timestamp Conversion

- **Overview:**  
  Converts event timestamps into a consistent, usable date-time format for logging and storage.

- **Nodes Involved:**  
  - Convert timestamp

- **Node Details:**  
  - **Convert timestamp**  
    - Type: DateTime  
    - Configuration: Default parameters (likely converts from UNIX timestamp or ISO string to formatted date-time).  
    - Key Expressions/Variables: Uses timestamp field from event data.  
    - Input: Routed event data from Switch node.  
    - Output: Passes converted timestamp data to "Set some data" node.  
    - Edge Cases:  
      - Invalid or missing timestamps could cause conversion errors or null values.  
      - Timezone considerations may affect accuracy if not explicitly handled.  
    - Version: Uses DateTime node version 2.

#### 2.4 Data Preparation

- **Overview:**  
  Prepares and formats the event data fields to match the structure expected by Google Sheets.

- **Nodes Involved:**  
  - Set some data

- **Node Details:**  
  - **Set some data**  
    - Type: Set  
    - Configuration: Defines key-value pairs for Google Sheets columns, such as product name, buyer info, event type, converted timestamp, payment amount, etc.  
    - Key Expressions/Variables: Uses expressions to extract and format data from previous nodes, including the converted timestamp.  
    - Input: Data with converted timestamp from "Convert timestamp" node.  
    - Output: Passes formatted data to "Google Sheets" node.  
    - Edge Cases:  
      - Missing fields in event data may result in incomplete rows.  
      - Expression errors if expected fields are absent or renamed.  
    - Version: Set node version 3.4.

#### 2.5 Data Storage

- **Overview:**  
  Appends the prepared event data as new rows into a specified Google Sheets spreadsheet.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Google Sheets**  
    - Type: Google Sheets  
    - Configuration: Appends rows to a configured spreadsheet and worksheet. Requires OAuth2 credentials for Google API access.  
    - Key Expressions/Variables: Uses data fields set in the previous node to populate columns.  
    - Input: Formatted data from "Set some data" node.  
    - Output: None (end of data flow).  
    - Edge Cases:  
      - Authentication errors if OAuth2 token expires or is invalid.  
      - API rate limits or quota exceeded errors.  
      - Spreadsheet or worksheet not found or access denied.  
    - Version: Google Sheets node version 4.5.

#### 2.6 Execution Logging

- **Overview:**  
  Saves execution metadata for monitoring, debugging, or auditing purposes.

- **Nodes Involved:**  
  - Save execution data

- **Node Details:**  
  - **Save execution data**  
    - Type: Execution Data  
    - Configuration: Default; captures execution details of the workflow run.  
    - Input: Connected in parallel after the Switch node, triggered on every webhook event.  
    - Output: None (terminal node for logging).  
    - Edge Cases:  
      - Logging failures generally do not affect main workflow but may reduce traceability.  
    - Version: Execution Data node version 1.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role              | Input Node(s)      | Output Node(s)         | Sticky Note                                                                                          |
|---------------------|--------------------|-----------------------------|--------------------|------------------------|----------------------------------------------------------------------------------------------------|
| Webhook             | Webhook            | Receives Hotmart events     | -                  | Switch                 |                                                                                                    |
| Switch              | Switch             | Routes events by type       | Webhook            | Convert timestamp, Save execution data | - PURCHASE_PROTEST is first refund notification; SUBSCRIPTION_CANCELLATION occurs with refund request; PURCHASE_REFUNDED is actual refund; PURCHASE_COMPLETE means guarantee period ended and funds belong to seller. |
| Save execution data  | Execution Data     | Logs execution metadata     | Switch             | -                      |                                                                                                    |
| Convert timestamp    | DateTime           | Converts event timestamps   | Switch             | Set some data          |                                                                                                    |
| Set some data        | Set                | Prepares data for Sheets    | Convert timestamp   | Google Sheets           |                                                                                                    |
| Google Sheets       | Google Sheets       | Appends data to spreadsheet | Set some data      | -                      |                                                                                                    |
| Sticky Note          | Sticky Note        | Notes                       | -                  | -                      |                                                                                                    |
| Sticky Note1         | Sticky Note        | Notes                       | -                  | -                      |                                                                                                    |
| Sticky Note2         | Sticky Note        | Notes                       | -                  | -                      |                                                                                                    |
| Sticky Note3         | Sticky Note        | Notes                       | -                  | -                      |                                                                                                    |
| Sticky Note4         | Sticky Note        | Notes                       | -                  | -                      |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Configure to receive HTTP POST requests.  
   - Set a unique webhook path or ID.  
   - No authentication required (unless Hotmart requires).  
   - This node is the entry point for Hotmart events.

2. **Create Switch Node**  
   - Type: Switch  
   - Add multiple cases based on the event type field from the webhook payload, for example:  
     - `PURCHASE_PROTEST`  
     - `SUBSCRIPTION_CANCELLATION`  
     - `PURCHASE_REFUNDED`  
     - `PURCHASE_COMPLETE`  
     - Additional cases as needed for other Hotmart event types.  
   - Connect Webhook node output to Switch node input.

3. **Create Execution Data Node**  
   - Type: Execution Data  
   - Connect Switch node output (any) to this node to log execution metadata.  
   - This node does not affect data flow.

4. **Create DateTime Node (Convert timestamp)**  
   - Type: DateTime  
   - Configure to convert the event timestamp field (e.g., UNIX timestamp or ISO string) into a standardized date-time format.  
   - Connect all Switch node outputs to this node (fan-out).  
   - Use version 2 of the DateTime node.

5. **Create Set Node (Set some data)**  
   - Type: Set  
   - Define key-value pairs to prepare data for Google Sheets, such as:  
     - Product name  
     - Buyer name and contact  
     - Event type  
     - Converted timestamp  
     - Payment amount  
     - Subscription status  
     - Any other relevant fields from the event payload.  
   - Use expressions to extract data from the previous node output.  
   - Connect DateTime node output to this node.

6. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Configure credentials with OAuth2 for Google API access.  
   - Set operation to "Append" rows.  
   - Select the target spreadsheet and worksheet.  
   - Map the fields from the Set node to the corresponding columns in the sheet.  
   - Connect Set node output to Google Sheets node input.

7. **Add Sticky Notes**  
   - Add notes to document event type meanings, workflow purpose, and any other relevant information for maintainers.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Track all your Hotmart events in one place and keep your data organized for analysis.                      | Workflow purpose description                         |
| No need to manually transfer data – this workflow automates data logging from Hotmart to Google Sheets.   | Workflow benefit                                    |
| Event type explanations: PURCHASE_PROTEST, SUBSCRIPTION_CANCELLATION, PURCHASE_REFUNDED, PURCHASE_COMPLETE | Sticky note on Switch node                           |
| Check out other templates by the creator Solomon.                                                         | https://n8n.io/creators/solomon/                     |

---

This documentation provides a detailed understanding of the workflow structure, node configurations, and reproduction steps, enabling advanced users and AI agents to maintain, extend, or troubleshoot the Hotmart event logging automation effectively.