Inventory Reconciliation between Notion & Airtable with GPT-4o Slack Alerts

https://n8nworkflows.xyz/workflows/inventory-reconciliation-between-notion---airtable-with-gpt-4o-slack-alerts-11479


# Inventory Reconciliation between Notion & Airtable with GPT-4o Slack Alerts

---
### 1. Workflow Overview

**Purpose:**  
This workflow performs inventory reconciliation between two systems: Notion (physical inventory counts) and Airtable (system counts). Its aim is to identify discrepancies item-by-item, update Airtable records when mismatches occur, and notify the operations team via Slack with AI-generated concise summaries. Invalid or incomplete data payloads are logged for audit and troubleshooting.

**Target Use Cases:**  
- Inventory managers needing automated synchronization and validation between physical and system counts.  
- Real-time alerting of stock mismatches to operations teams.  
- Maintaining data integrity across Notion and Airtable without manual cross-checks.

**Logical Blocks:**  
- **1.1 Trigger & Data Fetching:** Manual trigger initiates fetching inventory data from Notion and Airtable.  
- **1.2 Data Merge & Validation:** Merges datasets into a combined payload and validates structure. Logs invalid payloads.  
- **1.3 Inventory Comparison:** Compares individual Notion records against Airtable entries to detect discrepancies.  
- **1.4 Conditional Update:** Determines if Airtable needs updating; branches accordingly.  
- **1.5 System Update:** Updates Airtable records with corrected counts when discrepancies found.  
- **1.6 AI Slack Notification:** Generates Slack messages via GPT-4o — different messages for matched counts vs updates. Sends notifications to operations user.  
- **1.7 Logging Invalid Requests:** Logs malformed payloads in Google Sheets for review.

---
### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Fetching  
**Overview:** Initiates workflow manually and retrieves inventory records from Notion and Airtable.  
**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Fetch Records from Notion Database  
- Fetch Records from Airtable  

**Node Details:**  
- *When clicking ‘Execute workflow’*  
  - Type: Manual Trigger  
  - Role: Entry point; requires manual execution to start workflow.  
  - Inputs: None  
  - Outputs: Data flow to Notion and Airtable fetch nodes  
  - Edge Cases: No trigger means no execution; manual step may delay automation if unattended.

- *Fetch Records from Notion Database*  
  - Type: Notion Node (databasePage resource)  
  - Role: Retrieves all pages (records) from specified Notion Inventory database representing physical stock counts.  
  - Config: Database ID set to the Inventory database; fetches all pages without filtering.  
  - Inputs: Trigger node output  
  - Outputs: Array of Notion records  
  - Edge Cases: API rate limits, authentication issues, empty database.

- *Fetch Records from Airtable*  
  - Type: Airtable Node  
  - Role: Obtains records from Airtable Inventory table representing system stock counts.  
  - Config: Base and table selected; operation is “search” (fetch all).  
  - Inputs: Trigger node output  
  - Outputs: Array of Airtable records  
  - Edge Cases: API token expiry, network failures, empty table.

---

#### 2.2 Data Merge & Validation  
**Overview:** Merges Notion and Airtable datasets into a unified payload. Validates payload structure to ensure presence of critical identifiers before proceeding. Invalid payloads are logged.  
**Nodes Involved:**  
- Merge Notion + Airtable Inputs  
- Validate Payload Structure  
- Build Combined Notion + Airtable Payload  
- Log Invalid Versioning Requests to Google Sheets  

**Node Details:**  
- *Merge Notion + Airtable Inputs*  
  - Type: Merge Node  
  - Role: Combines data streams from Notion and Airtable fetch nodes into a single stream for comparison.  
  - Inputs: Outputs from Notion and Airtable fetch nodes  
  - Outputs: Merged data to validation node  
  - Edge Cases: Mismatched or missing data may cause merge issues.

- *Validate Payload Structure*  
  - Type: If Node  
  - Role: Checks if merged payload contains non-empty ‘id’ to validate record integrity.  
  - Inputs: Merged data  
  - Outputs: True branch to Build Combined Payload node; False branch to Logging node  
  - Edge Cases: Missing or malformed ‘id’ fields causing false negatives.

- *Build Combined Notion + Airtable Payload*  
  - Type: Code Node  
  - Role: Extracts relevant fields from Notion and Airtable datasets into a combined JSON structure for comparison.  
  - Inputs: Validated merged data  
  - Outputs: Payload with arrays of Notion properties and Airtable records  
  - Edge Cases: Unexpected data structure, missing fields.

- *Log Invalid Versioning Requests to Google Sheets*  
  - Type: Google Sheets Node  
  - Role: Appends invalid or malformed payload data to a Google Sheets document for audit and manual review.  
  - Inputs: Invalid payloads from validation failure  
  - Outputs: None  
  - Edge Cases: Google Sheets API quota, document permission errors.

---

#### 2.3 Inventory Comparison  
**Overview:** Compares each Notion inventory record against Airtable records to detect quantity mismatches and prepare a comparison result.  
**Nodes Involved:**  
- Compare Notion Record With Airtable Record  

**Node Details:**  
- *Compare Notion Record With Airtable Record*  
  - Type: Code Node  
  - Role: For each Notion item, finds matching Airtable item by name, compares ‘Quantity in Stock’, calculates difference, and flags if update needed.  
  - Inputs: Combined payload from previous block  
  - Outputs: JSON with status (match/no match), counts, difference, record ID, and update flag  
  - Key Variables: notionName, notionCount, airtableCount, diff, needsUpdate  
  - Edge Cases: No match found in Airtable, missing fields, multiple matches.

---

#### 2.4 Conditional Update  
**Overview:** Evaluates if the comparison indicates a need to update Airtable record and routes workflow accordingly.  
**Nodes Involved:**  
- Check If Record Requires Update  

**Node Details:**  
- *Check If Record Requires Update*  
  - Type: If Node  
  - Role: Checks if ‘needsUpdate’ flag is true.  
  - Inputs: Output from comparison node  
  - Outputs: True branch triggers Airtable update and update notification; False branch triggers no-update notification.  
  - Edge Cases: Incorrect flag setting, logical errors causing wrong branch selection.

---

#### 2.5 System Update  
**Overview:** Updates Airtable record with corrected quantity from Notion when discrepancy is detected.  
**Nodes Involved:**  
- Update Airtable Record With Corrected Count  

**Node Details:**  
- *Update Airtable Record With Corrected Count*  
  - Type: Airtable Node  
  - Role: Performs upsert operation to correct ‘Quantity in Stock’ in Airtable for identified record.  
  - Config: Uses record ID from comparison; updates quantity to Notion count.  
  - Inputs: True branch from Conditional Update node  
  - Outputs: Updated Airtable record data  
  - Edge Cases: API update failures, record locking, invalid record ID.

---

#### 2.6 AI Slack Notification  
**Overview:** Generates and sends Slack messages to operations user about reconciliation results using GPT-4o AI model. Distinct messages are generated for “no action needed” and “update performed” scenarios.  
**Nodes Involved:**  
- Configure GPT-4o – Slack Summary Model  
- Generate Slack Summary  
- Slack – Send Summary Notification  
- Configure GPT-4o – Slack Summary Model2  
- Generate Slack Summary1  
- Slack – Send Update Notification  

**Node Details:**  
- *Configure GPT-4o – Slack Summary Model*  
  - Type: Langchain Azure OpenAI Node  
  - Role: Configures GPT-4o model for generating Slack messages when no update needed.  
  - Credentials: Azure OpenAI account  
  - Inputs: From Conditional Update false branch  
  - Outputs: Message generation node  

- *Generate Slack Summary*  
  - Type: Langchain Agent Node  
  - Role: Uses GPT-4o to produce concise Slack summary indicating “no action needed” or matched counts.  
  - Key Prompt: Includes JSON comparison result and instructions to keep message short and clear.  
  - Outputs: Slack message text  

- *Slack – Send Summary Notification*  
  - Type: Slack Node  
  - Role: Sends generated message to specified user/channel on Slack.  
  - Inputs: AI-generated message from Generate Slack Summary  
  - Credentials: Slack OAuth2 user token  
  - Edge Cases: Slack API rate limits, invalid user ID.

- *Configure GPT-4o – Slack Summary Model2*  
  - Type: Langchain Azure OpenAI Node  
  - Role: Configures GPT-4o model for generating Slack messages when update is performed.  
  - Inputs: From Conditional Update true branch after Airtable update  

- *Generate Slack Summary1*  
  - Type: Langchain Agent Node  
  - Role: Generates Slack message highlighting discrepancy details and confirming Airtable update.  
  - Key Prompt: Includes item name, old and new counts, difference, and confirmation of update.  
  - Outputs: Slack message text  

- *Slack – Send Update Notification*  
  - Type: Slack Node  
  - Role: Sends update notification message to Slack user/channel.  
  - Inputs: AI-generated message from Generate Slack Summary1  
  - Edge Cases: Same as above.

---

#### 2.7 Logging Invalid Requests  
**Overview:** Captures any invalid or malformed payloads that fail validation and appends them to a Google Sheet for manual auditing.  
**Nodes Involved:**  
- Log Invalid Versioning Requests to Google Sheets (also described in block 2.2)  

---

### 3. Summary Table

| Node Name                               | Node Type                         | Functional Role                               | Input Node(s)                                 | Output Node(s)                               | Sticky Note                                                                                                    |
|----------------------------------------|----------------------------------|-----------------------------------------------|-----------------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’       | Manual Trigger                   | Workflow entry point                          | None                                          | Fetch Records from Notion Database, Fetch Records from Airtable |                                                                                                                |
| Fetch Records from Notion Database      | Notion Node                     | Fetch physical inventory counts from Notion  | When clicking ‘Execute workflow’              | Merge Notion + Airtable Inputs                 |                                                                                                                |
| Fetch Records from Airtable             | Airtable Node                   | Fetch system inventory counts from Airtable  | When clicking ‘Execute workflow’              | Merge Notion + Airtable Inputs                 |                                                                                                                |
| Merge Notion + Airtable Inputs          | Merge Node                     | Combines Notion and Airtable data streams    | Fetch Records from Notion Database, Fetch Records from Airtable | Validate Payload Structure                      | Sticky Note1: Data Intake & Merge — Notion + Airtable                                                               |
| Validate Payload Structure              | If Node                        | Validates presence of critical payload fields| Merge Notion + Airtable Inputs                 | Build Combined Notion + Airtable Payload (true), Log Invalid Versioning Requests to Google Sheets (false) |                                                                                                                |
| Build Combined Notion + Airtable Payload| Code Node                     | Prepares combined JSON payload for comparison| Validate Payload Structure (true)              | Compare Notion Record With Airtable Record    |                                                                                                                |
| Log Invalid Versioning Requests to Google Sheets | Google Sheets Node             | Logs invalid payloads for audit               | Validate Payload Structure (false)             | None                                         |                                                                                                                |
| Compare Notion Record With Airtable Record | Code Node                     | Compares Notion and Airtable stock quantities| Build Combined Notion + Airtable Payload       | Check If Record Requires Update                | Sticky Note2: Reconciliation Logic — Count Comparison                                                                |
| Check If Record Requires Update        | If Node                        | Determines if Airtable update is required    | Compare Notion Record With Airtable Record     | Update Airtable Record With Corrected Count (true), Generate Slack Summary (false) |                                                                                                                |
| Update Airtable Record With Corrected Count | Airtable Node                 | Updates Airtable stock count for discrepancies| Check If Record Requires Update (true)          | Generate Slack Summary1                        | Sticky Note3: System Update — Fixing Airtable Record                                                                |
| Configure GPT-4o – Slack Summary Model | Langchain Azure OpenAI Node    | Configures GPT-4o for no-update Slack message| Check If Record Requires Update (false)         | Generate Slack Summary                         | Sticky Note4: Notification Engine — AI Slack Messaging                                                              |
| Generate Slack Summary                  | Langchain Agent Node           | Generates Slack message for no updates       | Configure GPT-4o – Slack Summary Model          | Slack – Send Summary Notification              |                                                                                                                |
| Slack – Send Summary Notification       | Slack Node                    | Sends Slack message for no updates            | Generate Slack Summary                           | None                                         |                                                                                                                |
| Configure GPT-4o – Slack Summary Model2| Langchain Azure OpenAI Node    | Configures GPT-4o for update notification     | Update Airtable Record With Corrected Count     | Generate Slack Summary1                        | Sticky Note4: Notification Engine — AI Slack Messaging                                                              |
| Generate Slack Summary1                 | Langchain Agent Node           | Generates Slack message after Airtable update| Configure GPT-4o – Slack Summary Model2         | Slack – Send Update Notification               |                                                                                                                |
| Slack – Send Update Notification         | Slack Node                    | Sends Slack message after Airtable update    | Generate Slack Summary1                          | None                                         |                                                                                                                |
| Sticky Note                            | Sticky Note                    | Documentation summary of entire workflow     | None                                          | None                                         | Contains detailed descriptive documentation                                                                         |
| Sticky Note1                           | Sticky Note                    | Documentation for data intake & merging block| None                                          | None                                         |                                                                                                                |
| Sticky Note2                           | Sticky Note                    | Documentation for reconciliation logic       | None                                          | None                                         |                                                                                                                |
| Sticky Note3                           | Sticky Note                    | Documentation for system update block         | None                                          | None                                         |                                                                                                                |
| Sticky Note4                           | Sticky Note                    | Documentation for notification engine         | None                                          | None                                         |                                                                                                                |

---
### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: Start workflow manually.  
   - No parameters needed.

2. **Create Notion Node to Fetch Inventory Records**  
   - Node Type: Notion  
   - Resource: databasePage  
   - Operation: getAll  
   - Database ID: Set to your Notion Inventory database ID  
   - Credentials: Connect with valid Notion API credentials.  
   - Connect Trigger output to this node.

3. **Create Airtable Node to Fetch Inventory Records**  
   - Node Type: Airtable  
   - Operation: search (fetch all records)  
   - Base ID: Your Airtable base containing inventory  
   - Table ID: Inventory table  
   - Credentials: Airtable Personal Access Token with read access  
   - Connect Trigger output to this node.

4. **Add Merge Node to Combine Notion and Airtable Data**  
   - Node Type: Merge  
   - Inputs: Connect outputs of Notion and Airtable fetch nodes  
   - No special parameters (default merge).

5. **Add If Node to Validate Payload Structure**  
   - Node Type: If  
   - Condition: Check if `$json.id` is not empty (string not empty)  
   - Connect Merge node output to this node.

6. **Set True Branch to Code Node to Build Combined Payload**  
   - Node Type: Code  
   - JavaScript: Extract Notion properties and Airtable records into arrays and return combined JSON.  
   - Input: Valid merged payload  
   - Output: Combined payload for comparison.

7. **Set False Branch to Google Sheets Node to Log Invalid Payloads**  
   - Node Type: Google Sheets (Append operation)  
   - Document ID: Your Google Sheets document for logging  
   - Sheet Name: Sheet where to append invalid records  
   - Credentials: Google Sheets OAuth2 with write access  
   - Input: Invalid payloads from validation node.

8. **Create Code Node to Compare Notion Record With Airtable Record**  
   - Node Type: Code  
   - Logic:  
     - Extract Notion item name and quantity  
     - Find matching Airtable record by item name  
     - Calculate difference in quantity  
     - Output an object indicating status, counts, difference, and update requirement.  
   - Input: Output of combined payload builder.

9. **Add If Node to Check If Update Is Needed**  
   - Node Type: If  
   - Condition: Boolean equals `$json.needsUpdate` to `true`  
   - Input: Output of comparison code node.

10. **True Branch: Create Airtable Node to Update Record**  
    - Node Type: Airtable  
    - Operation: upsert  
    - Base/Table: Same as fetch node  
    - Use record ID from comparison to update ‘Quantity in Stock’ to Notion count  
    - Credentials: Airtable Personal Access Token with write access  
    - Input: True branch of update check node.

11. **True Branch: Configure GPT-4o Node for Slack Update Message**  
    - Node Type: Langchain Azure OpenAI  
    - Model: gpt-4o  
    - Credentials: Azure OpenAI API key  
    - Input: Airtable update node output.

12. **True Branch: Create Langchain Agent Node to Generate Slack Update Message**  
    - Node Type: Langchain Agent  
    - Prompt: Instruct GPT-4o to generate short Slack message showing discrepancy details and confirming Airtable update.  
    - Input: Output of GPT model configuration node.

13. **True Branch: Slack Node to Send Update Notification**  
    - Node Type: Slack  
    - Text: Use message from Langchain Agent node  
    - User: Operations user ID  
    - Credentials: Slack OAuth2 with chat permissions  
    - Input: Output of Slack message generation node.

14. **False Branch: Configure GPT-4o Node for Slack No-Update Message**  
    - Node Type: Langchain Azure OpenAI  
    - Model: gpt-4o  
    - Credentials: Azure OpenAI API key  
    - Input: False branch of update check node.

15. **False Branch: Langchain Agent Node to Generate Slack No-Update Message**  
    - Node Type: Langchain Agent  
    - Prompt: Generate short message indicating “No action needed” or matching counts.  
    - Input: Output of GPT model configuration node.

16. **False Branch: Slack Node to Send No-Update Notification**  
    - Node Type: Slack  
    - Text: Use message from Langchain Agent node  
    - User: Operations user ID  
    - Credentials: Slack OAuth2  
    - Input: Output of Slack message generation node.

17. **Connect all nodes according to the described flow:**  
    - Manual Trigger → Notion & Airtable fetch nodes  
    - Both fetch nodes → Merge  
    - Merge → Validate Payload Structure  
    - Validate True → Build Combined Payload → Compare → Check Update → True branch to Airtable Update and update Slack notifications, False branch to Slack no-update notifications  
    - Validate False → Log to Google Sheets

---
### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow compares Notion physical inventory counts against Airtable system counts to maintain stock accuracy.           | Full workflow purpose                                                                                     |
| Uses GPT-4o Azure OpenAI model to generate concise, professional Slack messages for operations team notifications.       | AI-generated messaging                                                                                     |
| Logs invalid or malformed payloads into Google Sheets for audit and manual correction.                                  | Error handling and data integrity monitoring                                                              |
| Slack notifications are sent to a specific user for real-time visibility on inventory status.                          | Slack integration details                                                                                  |
| Airtable upsert operation ensures system-of-record counts stay synchronized with physical counts.                      | Inventory system update logic                                                                               |
| Sticky notes in workflow document block purposes and logic for maintainability and easier understanding by collaborators.| Workflow embedded documentation                                                                             |

---

**Disclaimer:**  
The provided content is generated from an automated n8n workflow export and adheres strictly to content policies. No illegal or protected data is involved. All data handled is lawful and public.