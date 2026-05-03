Receive transfer updates from Wise and add to Airtable

https://n8nworkflows.xyz/workflows/receive-transfer-updates-from-wise-and-add-to-airtable-993


# Receive transfer updates from Wise and add to Airtable

### 1. Workflow Overview

This workflow automates the process of receiving transfer status updates from Wise (formerly TransferWise) and appending relevant transfer information to an Airtable base. It is designed for users who want to track their Wise transfers in real-time and maintain a synchronized record in Airtable for reporting, auditing, or notification purposes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger the workflow upon a transfer status change event from Wise.
- **1.2 Transfer Data Retrieval:** Fetch detailed information about the specific transfer that changed status.
- **1.3 Data Preparation:** Extract and format key transfer details to ensure only relevant data is passed forward.
- **1.4 Data Storage:** Append the prepared transfer information to a specified Airtable table.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for transfer status change events from Wise and triggers the workflow accordingly.

- **Nodes Involved:**  
  - Wise Trigger

- **Node Details:**  
  - **Wise Trigger**  
    - **Type:** Trigger node specialized for Wise events.  
    - **Configuration:**  
      - Event: `tranferStateChange` (note: the event name contains a likely typo; it should probably be `transferStateChange` but is used as-is).  
      - Profile ID: `16138858` (specific Wise profile to monitor).  
    - **Credentials:** Wise API credentials are required for authentication.  
    - **Input/Output:** No input; outputs event payload including transfer ID.  
    - **Version:** v1 of the Wise Trigger node.  
    - **Potential Failures:**  
      - Authentication errors if credentials expire or are invalid.  
      - Misspelled event name may cause trigger not to fire if Wise API expects correct spelling.  
      - Network timeout or webhook misconfiguration.  
    - **Sub-workflow:** None.

#### 1.2 Transfer Data Retrieval

- **Overview:**  
  Retrieves detailed information about the transfer whose status has changed.

- **Nodes Involved:**  
  - Wise

- **Node Details:**  
  - **Wise**  
    - **Type:** Resource node to interact with Wise API.  
    - **Configuration:**  
      - Resource: `transfer`  
      - Transfer ID: Dynamically set using expression `{{$json["data"]["resource"]["id"]}}` extracted from the trigger event data.  
    - **Credentials:** Uses the same Wise API credentials as the trigger node.  
    - **Input/Output:** Input from Wise Trigger node; outputs transfer details JSON.  
    - **Version:** v1.  
    - **Potential Failures:**  
      - Transfer ID might be missing or malformed, causing API call failure.  
      - API rate limits or connectivity issues.  
      - Authentication token expiry or invalid credentials.  
    - **Sub-workflow:** None.

#### 1.3 Data Preparation

- **Overview:**  
  Extracts only the required fields from the transfer data to pass to Airtable, ensuring data cleanliness and structure.

- **Nodes Involved:**  
  - Set

- **Node Details:**  
  - **Set**  
    - **Type:** Data transformation node that sets specific key-value pairs.  
    - **Configuration:**  
      - Keeps only the following fields:  
        - `Transfer ID` = `{{$json["id"]}}`  
        - `Date` = `{{$json["created"]}}`  
        - `Reference` = `{{$json["reference"]}}`  
        - `Amount` = `{{$json["sourceValue"]}}`  
      - Option `keepOnlySet` enabled to filter out all other data.  
    - **Input/Output:** Input from Wise node; outputs only the defined fields.  
    - **Version:** v1.  
    - **Potential Failures:**  
      - If any of these fields are missing in the input JSON, the output will have empty or undefined values.  
      - Expression errors if data paths change in Wise API.  
    - **Sub-workflow:** None.

#### 1.4 Data Storage

- **Overview:**  
  Appends the extracted transfer data as a new record to a specified Airtable table.

- **Nodes Involved:**  
  - Airtable

- **Node Details:**  
  - **Airtable**  
    - **Type:** Integration node for Airtable API.  
    - **Configuration:**  
      - Operation: `append` (inserts new records).  
      - Table: `Table 1` (target table in Airtable base).  
      - Application parameter is left blank (uses default or preconfigured base).  
    - **Credentials:** Airtable API credentials stored in n8n.  
    - **Input/Output:** Receives prepared data from Set node; outputs API response.  
    - **Version:** v1.  
    - **Potential Failures:**  
      - Authentication failure if API key is invalid or revoked.  
      - Schema mismatch if Airtable columns do not correspond to incoming data fields.  
      - Rate limits or network errors.  
      - If the table name is incorrect or missing.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name    | Node Type            | Functional Role          | Input Node(s) | Output Node(s) | Sticky Note                                                                                     |
|--------------|----------------------|-------------------------|---------------|----------------|------------------------------------------------------------------------------------------------|
| Wise Trigger | Wise Trigger         | Trigger on transfer status changes | None          | Wise           | This node will trigger the workflow when the status of your transfer changes.                  |
| Wise         | Wise                 | Fetch transfer details   | Wise Trigger  | Set            | This node will get the information about the transfer.                                        |
| Set          | Set                  | Data extraction/filtering | Wise          | Airtable       | We use the Set node to ensure that only the data that we set in this node gets passed on to the next nodes in the workflow. We set the value of `Transfer ID`, `Date`, `Reference`, and `Amount` in this node. |
| Airtable     | Airtable             | Append data to Airtable  | Set           | None           | This node will append the data that we set in the previous node to a table.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Wise Trigger Node**  
   - Add a new Wise Trigger node.  
   - Set **Event** to `tranferStateChange` (copy exactly as in original).  
   - Set **Profile ID** to `16138858` (or your own Wise profile ID).  
   - Set credentials using your Wise API credentials.  
   - Position at the start of the workflow.

2. **Create Wise Node**  
   - Add a Wise node next in sequence.  
   - Set **Resource** to `transfer`.  
   - For **Transfer ID**, enter the expression: `{{$json["data"]["resource"]["id"]}}` to dynamically use ID from the trigger.  
   - Use the same Wise API credentials as the trigger node.  
   - Connect Wise Trigger node output to this Wise node input.

3. **Create Set Node**  
   - Add a Set node after the Wise node.  
   - Enable option `Keep Only Set`.  
   - Add the following string fields with expressions:  
     - `Transfer ID` = `{{$json["id"]}}`  
     - `Date` = `{{$json["created"]}}`  
     - `Reference` = `{{$json["reference"]}}`  
     - `Amount` = `{{$json["sourceValue"]}}`  
   - Connect Wise node output to Set node input.

4. **Create Airtable Node**  
   - Add an Airtable node last in the sequence.  
   - Set **Operation** to `append`.  
   - Set **Table** to `Table 1` (or your target Airtable table).  
   - Set credentials with your Airtable API credentials.  
   - Connect Set node output to Airtable node input.

5. **Activate and Test Workflow**  
   - Save and activate the workflow.  
   - Test by triggering a transfer status change in Wise to verify data is appended to Airtable.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                   |
|-----------------------------------------------------------------------------------------------|---------------------------------|
| The workflow screenshot is referenced but not included here; it visually represents the nodes and connections. | Refer to original workflow visual. |
| Wise API event name appears to have a typo: `tranferStateChange` instead of `transferStateChange`. Verify in your environment. | Wise API documentation           |
| Airtable operation is set to append; ensure your target table schema matches the fields sent. | Airtable API documentation       |
| This workflow requires valid API credentials for both Wise and Airtable for proper operation. | n8n credential management docs   |