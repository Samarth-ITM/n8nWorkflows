WhatsApp Message Auto-Logger for Vtiger CRM with Lead Relation

https://n8nworkflows.xyz/workflows/whatsapp-message-auto-logger-for-vtiger-crm-with-lead-relation-6595


# WhatsApp Message Auto-Logger for Vtiger CRM with Lead Relation

### 1. Workflow Overview

This workflow automates the process of logging inbound WhatsApp messages into Vtiger CRM, specifically associating messages with existing Leads or creating new Leads when necessary. It is designed for businesses that use WhatsApp for customer communication and want to maintain comprehensive, up-to-date records in their CRM system without manual input.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception**: Captures inbound WhatsApp messages via a webhook.
- **1.2 Message Filtering**: Filters out group messages and outbound messages to focus only on relevant inbound individual messages.
- **1.3 Data Extraction**: Extracts key details from the message payload such as customer name, phone number, and message content.
- **1.4 Lead Search**: Queries Vtiger CRM to check if a Lead exists with the extracted phone number.
- **1.5 Lead Existence Decision**: Decides the path based on whether a matching Lead was found.
- **1.6 Lead Creation & Logging**: Creates a new Lead if none was found and logs the message.
- **1.7 Message Logging for Existing Lead**: Logs the message associated with an existing Lead.
- **1.8 No-Operation**: A fallback for ignored messages (group or outbound).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives inbound WhatsApp messages via an HTTP POST webhook endpoint.
- **Nodes Involved:** `Webhook: WhatsApp Listen`
- **Node Details:**
  - Type: Webhook
  - Role: Entry point for inbound WhatsApp messages.
  - Configuration: POST method on path `/whatsAppListen`.
  - Inputs: External HTTP POST from WhatsApp messaging service.
  - Outputs: JSON payload containing message details.
  - Edge cases/failures: Webhook misconfiguration, invalid or unexpected payload formats, HTTP errors.
  - Sub-workflow: None.

#### 1.2 Message Filtering

- **Overview:** Filters incoming messages to exclude group messages and outbound messages sent by self.
- **Nodes Involved:** `If message not from group and not outbound`
- **Node Details:**
  - Type: If (conditional)
  - Role: Filters messages by checking two conditions:
    - Message is not from the user itself (`fromMe === false`).
    - Message is not from a group (remoteJid does not include '@g.us').
  - Uses expressions checking JSON properties: `$json.body.data.key.fromMe` and `$json.body.data.key.remoteJid`.
  - Input: Output of webhook node.
  - Output: Two branches — pass filtered messages forward; others to no operation.
  - Edge cases: Unexpected JSON structure or missing keys.
  - Version: 2.2 (supports complex conditions).
  - Sub-workflow: None.

#### 1.3 Data Extraction

- **Overview:** Extracts and structures relevant data (customer name, phone, and message) from the filtered inbound message.
- **Nodes Involved:** `Set: Extract Name, Phone & Message`
- **Node Details:**
  - Type: Set
  - Role: Creates a simplified data object with keys `custName`, `phone`, and `message`.
  - Extracts:
    - `custName` from `$json.body.data.pushName`.
    - `phone` by removing WhatsApp domain suffix from `$json.body.data.key.remoteJid`.
    - `message` from `$json.body.data.message.conversation`.
  - Input: Filtered inbound message.
  - Output: Cleaned data for querying CRM.
  - Edge cases: Missing or malformed message data fields.
  - Sub-workflow: None.

#### 1.4 Lead Search

- **Overview:** Queries Vtiger CRM for an existing Lead matching the extracted phone number.
- **Nodes Involved:** `Search Lead by Phone`
- **Node Details:**
  - Type: Vtiger CRM node
  - Role: Executes a SQL-like SELECT query on Leads using phone or mobile number equal to extracted phone.
  - Query: `SELECT id FROM Leads WHERE phone='${phone}' OR mobile='${phone}' LIMIT 1;`
  - Credentials: Uses stored Vtiger API credentials.
  - Input: Extracted phone data.
  - Output: Result array containing Lead id if found.
  - Edge cases: CRM API errors, authentication failures, empty results, malformed query.
  - Version: 1.
  - Sub-workflow: None.

#### 1.5 Lead Existence Decision

- **Overview:** Checks whether the Lead search returned a valid Lead id to choose the next action.
- **Nodes Involved:** `If Lead existing`
- **Node Details:**
  - Type: If (conditional)
  - Role: Checks if the first Lead search result has a non-empty id.
  - Expression: Checks if `$json.result[0].id` is not empty.
  - Input: Output from Lead search.
  - Output: Two branches:
    - True: Lead exists, proceed to log message.
    - False: Lead does not exist, proceed to create Lead.
  - Edge cases: Empty or undefined search results.
  - Version: 2.2.
  - Sub-workflow: None.

#### 1.6 Lead Creation & Logging

- **Overview:** Creates a new Lead in Vtiger CRM using extracted data and logs the inbound message.
- **Nodes Involved:** `Create Lead`, `Log to WhatsAppLog (New Lead)`
- **Node Details:**

  - **Create Lead**
    - Type: Vtiger CRM node
    - Role: Creates a new Lead with:
      - `firstname` set to `-` (placeholder).
      - `lastname` set to extracted customer name.
      - `phone` set to extracted phone number.
      - `assigned_user_id` hard-coded to "19x1".
    - Input: From the false branch of Lead existence check.
    - Output: Newly created Lead id.
    - Edge cases: CRM validation errors, duplicate Leads, API errors.

  - **Log to WhatsAppLog (New Lead)**
    - Type: Vtiger CRM node
    - Role: Creates a new record in custom module `WhatsaAppLog` with:
      - Message text.
      - Customer name.
      - Direction set to "Inbound".
      - Phone number.
      - Lead id linked to the newly created Lead.
      - Assigned user id "19x1".
    - Input: Output from `Create Lead`.
    - Edge cases: CRM API failures, missing Lead id propagation.

#### 1.7 Message Logging for Existing Lead

- **Overview:** Logs the inbound WhatsApp message into the `WhatsAppLog` custom module linked to the existing Lead.
- **Nodes Involved:** `Log to WhatsAppLog (Existing Lead)`
- **Node Details:**
  - Type: Vtiger CRM node
  - Role: Creates a WhatsaAppLog record linked to the found Lead id.
  - Fields populated:
    - Message text.
    - Customer name.
    - Direction "Inbound".
    - Phone number.
    - Lead id from existing Lead.
    - Assigned user id "19x1".
  - Input: True branch of Lead existence check.
  - Edge cases: CRM API errors, missing Lead id.
  - Version: 1.

#### 1.8 No-Operation

- **Overview:** Handles ignored messages (group or outbound) by doing nothing, preventing further processing.
- **Nodes Involved:** `No Operation, do nothing`
- **Node Details:**
  - Type: NoOp
  - Role: Terminates flow silently for filtered-out messages.
  - Input: False branch of message filtering.
  - Output: None.
  - Edge cases: None.

---

### 3. Summary Table

| Node Name                          | Node Type                   | Functional Role                                              | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                                  |
|-----------------------------------|-----------------------------|--------------------------------------------------------------|----------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook: WhatsApp Listen           | Webhook                     | Receives inbound WhatsApp messages                            | External HTTP POST               | If message not from group and not outbound | ## Summary This workflow: - **Listens** for inbound WhatsApp messages. - **Ignores** group messages and those sent by yourself (Outbound). - **Searches** Vtiger CRM for an existing lead by phone. - If found, **logs** the message in the `WhatsAppLog` module and relate with Lead. - If not found, **creates a new lead** and logs the message.  Use case: - Ideal for businesses using WhatsApp for customer communication who want to automate CRM lead management and ensure no conversation is missed.|
| If message not from group and not outbound | If                         | Filters out group and outbound messages                       | Webhook: WhatsApp Listen         | Set: Extract Name, Phone & Message; No Operation, do nothing | See above                                                                                                   |
| Set: Extract Name, Phone & Message | Set                         | Extracts customer name, phone, and message content            | If message not from group and not outbound | Search Lead by Phone               | See above                                                                                                   |
| Search Lead by Phone               | Vtiger CRM node             | Searches for existing Lead by phone number                    | Set: Extract Name, Phone & Message | If Lead existing                  | See above                                                                                                   |
| If Lead existing                  | If                          | Checks if Lead found                                          | Search Lead by Phone             | Log to WhatsAppLog (Existing Lead); Create Lead | See above                                                                                                   |
| Log to WhatsAppLog (Existing Lead) | Vtiger CRM node             | Logs WhatsApp message linked to existing Lead                 | If Lead existing (true branch)  | (none)                           | See above                                                                                                   |
| Create Lead                      | Vtiger CRM node             | Creates new Lead in CRM                                       | If Lead existing (false branch) | Log to WhatsAppLog (New Lead)     | See above                                                                                                   |
| Log to WhatsAppLog (New Lead)     | Vtiger CRM node             | Logs WhatsApp message linked to newly created Lead            | Create Lead                     | (none)                           | See above                                                                                                   |
| No Operation, do nothing           | NoOp                        | Terminates processing for ignored messages                    | If message not from group and not outbound (false branch) | (none)                           | See above                                                                                                   |
| Sticky Note                      | Sticky Note                 | Provides summary and use case explanation                      | None                           | None                            | See above                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Name: `Webhook: WhatsApp Listen`
   - Type: Webhook
   - HTTP Method: POST
   - Path: `whatsAppListen`
   - Purpose: Receive inbound WhatsApp messages.

2. **Create If Node for Message Filtering**
   - Name: `If message not from group and not outbound`
   - Type: If
   - Conditions (AND combinator):
     - Expression equals `false` for `$json.body.data.key.fromMe` (message is inbound).
     - Expression equals `false` for `$json.body.data.key.remoteJid.includes('@g.us')` (not group).
   - Connect webhook node output to this node.

3. **Create No Operation Node**
   - Name: `No Operation, do nothing`
   - Type: NoOp
   - Connect from the false output of the filtering If node.

4. **Create Set Node for Data Extraction**
   - Name: `Set: Extract Name, Phone & Message`
   - Type: Set
   - Set fields:
     - `custName` = `{{$json.body.data.pushName}}`
     - `phone` = `{{$json.body.data.key.remoteJid.replace('@s.whatsapp.net', '')}}`
     - `message` = `{{$json.body.data.message.conversation}}`
   - Connect from the true output of the filtering If node.

5. **Create Vtiger CRM Node for Lead Search**
   - Name: `Search Lead by Phone`
   - Type: Vtiger CRM
   - Operation: Query
   - Query field:  
     ```
     SELECT id FROM Leads WHERE phone='{{$json.phone}}' OR mobile='{{$json.phone}}' LIMIT 1;
     ```
   - Credentials: Add your Vtiger API credentials.
   - Connect from Set node output.

6. **Create If Node for Lead Existence Check**
   - Name: `If Lead existing`
   - Type: If
   - Condition: Check if `$json.result[0].id` is not empty.
   - Connect from Lead Search node output.

7. **Create Vtiger CRM Node for Existing Lead Message Logging**
   - Name: `Log to WhatsAppLog (Existing Lead)`
   - Type: Vtiger CRM
   - Operation: Create
   - Element Type: `WhatsaAppLog`
   - Fields:
     - `cf_1100` (message) = `{{$('Set: Extract Name, Phone & Message').item.json.message}}`
     - `name` = `{{$('Set: Extract Name, Phone & Message').item.json.custName}}`
     - `cf_1098` = `Inbound`
     - `cf_1102` (phone) = `{{$('Set: Extract Name, Phone & Message').item.json.phone}}`
     - `leadid` = `{{$json.result[0].id}}`
     - `assigned_user_id` = `19x1`
   - Credentials: Use Vtiger credentials.
   - Connect from true output of Lead existence If node.

8. **Create Vtiger CRM Node for Lead Creation**
   - Name: `Create Lead`
   - Type: Vtiger CRM
   - Operation: Create
   - Element Type: `Leads`
   - Fields:
     - `firstname` = `-`
     - `lastname` = `{{$('Set: Extract Name, Phone & Message').item.json.custName}}`
     - `phone` = `{{$('Set: Extract Name, Phone & Message').item.json.phone}}`
     - `assigned_user_id` = `19x1`
   - Credentials: Use Vtiger credentials.
   - Connect from false output of Lead existence If node.

9. **Create Vtiger CRM Node for New Lead Message Logging**
   - Name: `Log to WhatsAppLog (New Lead)`
   - Type: Vtiger CRM
   - Operation: Create
   - Element Type: `WhatsaAppLog`
   - Fields:
     - `cf_1100` (message) = `{{$('Set: Extract Name, Phone & Message').item.json.message}}`
     - `name` = `{{$('Set: Extract Name, Phone & Message').item.json.custName}}`
     - `cf_1098` = `Inbound`
     - `cf_1102` (phone) = `{{$('Set: Extract Name, Phone & Message').item.json.phone}}`
     - `leadid` = `{{$json.result.id}}` (from newly created Lead)
     - `assigned_user_id` = `19x1`
   - Credentials: Use Vtiger credentials.
   - Connect from output of Create Lead node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates logging of WhatsApp inbound messages to Vtiger CRM Leads and is ideal for businesses wanting to automate CRM lead management from WhatsApp conversations.                                                   | Sticky Note node content within the workflow                                                  |
| For proper operation, ensure Vtiger CRM API credentials have sufficient permissions to query Leads, create Leads, and create records in the custom `WhatsaAppLog` module.                                                           | Credential configuration in Vtiger nodes                                                       |
| The workflow ignores group messages and outbound messages to avoid cluttering logs with irrelevant data.                                                                                                                           | Filtering logic in the If node "If message not from group and not outbound"                    |
| Phone number extraction assumes the WhatsApp JID format ending with '@s.whatsapp.net'. Adjust if WhatsApp changes JID format.                                                                                                      | Set node expression for extracting phone                                                      |
| Assigned user id "19x1" is hardcoded and should be customized to match your Vtiger CRM user IDs.                                                                                                                                   | Throughout Vtiger CRM create nodes                                                            |
| Vtiger CRM custom fields referenced (`cf_1100`, `cf_1098`, `cf_1102`) correspond to WhatsAppLog module fields. Ensure these fields exist and are correctly configured in your CRM.                                                  | Vtiger CRM module field mapping                                                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.