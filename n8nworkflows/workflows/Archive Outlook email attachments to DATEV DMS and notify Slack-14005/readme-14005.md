Archive Outlook email attachments to DATEV DMS and notify Slack

https://n8nworkflows.xyz/workflows/archive-outlook-email-attachments-to-datev-dms-and-notify-slack-14005


# Archive Outlook email attachments to DATEV DMS and notify Slack

# 1. Workflow Overview

This workflow monitors a Microsoft Outlook mailbox for new emails, extracts attachments, archives each attachment into DATEV DMS as a document, optionally posts a Slack notification, and finally updates the original Outlook email with a category indicating it has been archived.

Its main use case is semi-automated document intake from email into DATEV DMS for accounting, client recordkeeping, or document management processes. The current template includes a placeholder-style client lookup that simply selects the first active DATEV client, so it is intended to be customized before production use.

## 1.1 Email Reception and Attachment Filtering

The workflow starts from an Outlook trigger that polls for new emails. It then checks whether the message actually has attachments before continuing.

## 1.2 DATEV Client Resolution

Once a qualifying email is found, the workflow performs a DATEV master data lookup to retrieve a client record. In this template, the lookup is a demo configuration and does not yet match the client based on email metadata.

## 1.3 Attachment Splitting and Context Preparation

The email’s binary attachments are split into separate items so each attachment can be processed independently. In parallel, the workflow also retrieves the current DATEV user for metadata assignment.

## 1.4 DATEV DMS File Upload and Document Creation

Each attachment file is uploaded into DATEV DMS as a document file. The workflow then merges the uploaded file result, attachment metadata, and current DATEV user data to create a formal DATEV document entry with document structure and metadata.

## 1.5 Notification and Mailbox Update

After successful document creation, the workflow attempts to send a Slack message and then updates the Outlook email by assigning a category that marks it as archived. Both notification and mailbox update steps are configured to continue even if they encounter errors.

---

# 2. Block-by-Block Analysis

## Block 1 — Email Reception and Attachment Filtering

### Overview

This block detects incoming Outlook emails and ensures only emails with attachments are processed further. It serves as the workflow’s entry point and first guardrail.

### Nodes Involved

- New Outlook emal trigger
- If we have attachments

### Node Details

#### 1. New Outlook emal trigger

- **Type and technical role:** `n8n-nodes-base.microsoftOutlookTrigger`  
  Polling trigger for new Microsoft Outlook messages.
- **Configuration choices:**
  - Polls every minute.
  - Downloads attachments automatically.
  - Uses attachment binary keys prefixed with `attachment_`.
  - A filter object is present with `hasAttachments: false`, but the actual workflow logic relies on the downstream IF node to validate attachment presence.
- **Key expressions or variables used:**
  - Emits fields such as:
    - `id` for the Outlook message ID
    - `from`
    - `subject`
    - `hasAttachments`
  - Emits binaries under `item.binary`, one entry per attachment.
- **Input and output connections:**
  - Entry point node, no input.
  - Outputs to **If we have attachments**.
- **Version-specific requirements:**
  - Uses node type version 1.
  - Requires Microsoft Outlook OAuth2 credentials.
- **Edge cases or potential failure types:**
  - OAuth2 authorization failure or expired token.
  - Polling delays or trigger duplication depending on mailbox behavior.
  - Attachment download failures.
  - The node name contains a typo (`emal`), which does not break execution but matters because expressions reference the exact node name.
- **Sub-workflow reference:** None.

#### 2. If we have attachments

- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional gate to pass through only messages with attachments.
- **Configuration choices:**
  - Checks whether `{{$json.hasAttachments}}` is `true`.
  - `alwaysOutputData` is enabled, but only the true branch is connected.
- **Key expressions or variables used:**
  - `={{ $json.hasAttachments }}`
- **Input and output connections:**
  - Input from **New Outlook emal trigger**
  - True output to **Client lookup**
  - False output unused
- **Version-specific requirements:**
  - Uses IF node version 2.3 with condition format version 3.
- **Edge cases or potential failure types:**
  - If Outlook metadata is inconsistent and `hasAttachments` is false while binaries exist, the email will be skipped.
  - If the trigger configuration changes and attachments are no longer downloaded, later nodes will fail even if this condition passes.
- **Sub-workflow reference:** None.

---

## Block 2 — DATEV Client Resolution

### Overview

This block retrieves a DATEV client record that will later be used as the document’s correspondence partner. In the provided workflow, this is a demo setup that selects the first active client rather than performing real matching logic.

### Nodes Involved

- Client lookup

### Node Details

#### 3. Client lookup

- **Type and technical role:** `@klardaten/n8n-nodes-datevconnect.masterData`  
  DATEV master data lookup for client records.
- **Configuration choices:**
  - Limits results to 1 item.
  - Applies filter `status eq active`.
  - No dynamic matching based on sender, subject, or extracted identifiers.
- **Key expressions or variables used:**
  - No dynamic expression in parameters.
  - Later nodes use:
    - `$('Client lookup').item.json.id`
- **Input and output connections:**
  - Input from **If we have attachments**
  - Output to **Split attachments**
- **Version-specific requirements:**
  - Uses custom DATEVconnect node version 1.
  - Requires DATEVconnect credentials.
- **Edge cases or potential failure types:**
  - Returns an arbitrary active client rather than the correct one.
  - Zero results would cause downstream expression failure when referencing `.item.json.id`.
  - Authentication or API permission errors to DATEV master data.
- **Sub-workflow reference:** None.

---

## Block 3 — Attachment Splitting and Context Preparation

### Overview

This block transforms a single Outlook email containing multiple binary attachments into one item per attachment. It also starts retrieval of the current DATEV user so that document ownership metadata can be attached during creation.

### Nodes Involved

- Split attachments
- Get DATEV user

### Node Details

#### 4. Split attachments

- **Type and technical role:** `n8n-nodes-base.code`  
  Converts one email item with multiple binary properties into multiple items, each carrying exactly one attachment in `binary.data`.
- **Configuration choices:**
  - JavaScript iterates over all items from **New Outlook emal trigger**.
  - For each binary key, it creates a new item with:
    - copied JSON from the original email
    - a single binary property named `data`
- **Key expressions or variables used:**
  - Uses `$("New Outlook emal trigger").all()`
  - Reads `item.binary ?? {}`
  - Rewrites each attachment to:
    - `binary.data`
- **Input and output connections:**
  - Input from **Client lookup**
  - Outputs to:
    - **Upload a document file**
    - **Merge** input 1
    - **Get DATEV user**
- **Version-specific requirements:**
  - Uses Code node version 2.
  - Assumes JavaScript execution is enabled in n8n.
- **Edge cases or potential failure types:**
  - If no binaries are present, returns no items.
  - If attachment binary objects are malformed, downstream upload can fail.
  - Because it explicitly references **New Outlook emal trigger** instead of the direct input item, unusual execution patterns could become harder to debug.
- **Sub-workflow reference:** None.

#### 5. Get DATEV user

- **Type and technical role:** `@klardaten/n8n-nodes-datevconnect.identityAndAccessManagement`  
  Fetches the current DATEV user for use in document metadata.
- **Configuration choices:**
  - Resource is `currentUser`.
- **Key expressions or variables used:**
  - Later referenced as:
    - `$('Get DATEV user').item.json.id`
- **Input and output connections:**
  - Input from **Split attachments**
  - Output to **Merge** input 0
- **Version-specific requirements:**
  - Uses custom DATEVconnect node version 1.
  - Requires DATEVconnect credentials.
- **Edge cases or potential failure types:**
  - Authorization failure in DATEV API.
  - If the API does not return a valid `id`, document creation will fail.
  - This node executes per attachment path because it is downstream of the split, which may be unnecessary overhead.
- **Sub-workflow reference:** None.

---

## Block 4 — DATEV DMS File Upload and Document Creation

### Overview

This block uploads each attachment binary into DATEV DMS and then creates a document record linked to the uploaded file. It combines file upload output, attachment metadata, and DATEV user data before creating the final document.

### Nodes Involved

- Upload a document file
- Merge
- Create a document

### Node Details

#### 6. Upload a document file

- **Type and technical role:** `@klardaten/n8n-nodes-datevconnect.documentManagement`  
  Uploads the binary attachment as a DATEV document file object.
- **Configuration choices:**
  - Resource: `documentFile`
  - Operation: `upload`
  - Binary property: `data`
- **Key expressions or variables used:**
  - Relies on the Code node having normalized the binary property to `binary.data`.
- **Input and output connections:**
  - Input from **Split attachments**
  - Output to **Merge** input 2
- **Version-specific requirements:**
  - Uses custom DATEVconnect node version 1.
  - Requires DATEVconnect credentials.
- **Edge cases or potential failure types:**
  - Missing binary `data`.
  - Unsupported file format or upload size restrictions.
  - DATEV API timeouts or upload errors.
- **Sub-workflow reference:** None.

#### 7. Merge

- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines three parallel streams by item position before document creation.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
  - Number of inputs: 3
- **Key expressions or variables used:**
  - No explicit expression, but downstream nodes expect merged data from:
    - DATEV user
    - attachment metadata
    - uploaded file response
- **Input and output connections:**
  - Input 0 from **Get DATEV user**
  - Input 1 from **Split attachments**
  - Input 2 from **Upload a document file**
  - Output to **Create a document**
- **Version-specific requirements:**
  - Uses Merge node version 3.2.
- **Edge cases or potential failure types:**
  - Position-based combining can misalign data if one branch emits a different number of items.
  - If any input branch fails or produces fewer items, later expressions may reference wrong or missing data.
- **Sub-workflow reference:** None.

#### 8. Create a document

- **Type and technical role:** `@klardaten/n8n-nodes-datevconnect.documentManagement`  
  Creates a DATEV DMS document entry linked to the previously uploaded document file.
- **Configuration choices:**
  - Operation: `create`
  - Uses a structured JSON expression to define document metadata.
  - Important fixed values:
    - Class: `Dokument`, ID `1`
    - Domain: `Mandanten`, ID `1`
    - Folder: ID `2`, name `<Keine Angabe>`
  - Year and month are generated from the current date.
  - A single `structure_items` entry is built for the uploaded file.
- **Key expressions or variables used:**
  - `{{ $binary.data.fileExtension }}`
  - `{{ $binary.data.fileName }}`
  - `{{ $('Client lookup').item.json.id }}`
  - `{{ $('Get DATEV user').item.json.id }}`
  - `{{ $json.id }}` for `document_file_id`
  - `{{ new Date().getFullYear() }}`
  - `{{ new Date().getMonth() + 1 }}`
  - `{{ new Date().toISOString() }}`
- **Input and output connections:**
  - Input from **Merge**
  - Output to **Send a message**
- **Version-specific requirements:**
  - Uses custom DATEVconnect node version 1.
  - Requires DATEVconnect credentials.
- **Edge cases or potential failure types:**
  - Expression failures if merged data does not include expected `binary.data` or JSON fields.
  - Invalid or unauthorized client/user IDs.
  - Hard-coded class/domain/folder IDs may not be valid in another DATEV environment.
  - `document_file_id` expects the upload node’s returned file identifier to remain available as `$json.id` after merge.
- **Sub-workflow reference:** None.

---

## Block 5 — Notification and Mailbox Update

### Overview

This block provides optional operational feedback in Slack and then marks the source Outlook email as archived using a category. Both nodes are configured to continue even if they fail, which prevents notification issues from stopping the workflow.

### Nodes Involved

- Send a message
- Update a message

### Node Details

#### 9. Send a message

- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack message to a fixed channel after document creation.
- **Configuration choices:**
  - Authentication: OAuth2
  - Destination: a specific Slack channel ID
  - Error handling: `continueRegularOutput`
  - Message content includes created document description, sender, and email subject
- **Key expressions or variables used:**
  - `{{ $json.description }}`
  - `{{ $('New Outlook emal trigger').item.json.from }}`
  - `{{ $('New Outlook emal trigger').item.json.subject }}`
- **Input and output connections:**
  - Input from **Create a document**
  - Output to **Update a message**
- **Version-specific requirements:**
  - Uses Slack node version 2.4.
  - Requires Slack OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Invalid Slack channel selection.
  - OAuth2 scope or token issues.
  - If `$json.description` is absent in the DATEV create response, the message text may be incomplete.
- **Sub-workflow reference:** None.

#### 10. Update a message

- **Type and technical role:** `n8n-nodes-base.microsoftOutlook`  
  Updates the original Outlook email by applying a category.
- **Configuration choices:**
  - Operation: `update`
  - Message ID is pulled from the original trigger node.
  - Adds category: `REQUEST DMS ARCHIVE`
  - Error handling: `continueRegularOutput`
- **Key expressions or variables used:**
  - `={{ $('New Outlook emal trigger').item.json.id }}`
- **Input and output connections:**
  - Input from **Send a message**
  - No downstream output connected
- **Version-specific requirements:**
  - Uses Microsoft Outlook node version 2.
  - Requires Microsoft Outlook OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Category may not exist or may behave differently depending on Outlook/Microsoft Graph behavior.
  - Message may already be moved, deleted, or inaccessible by the time the update runs.
  - If multiple attachments create multiple execution items, the same message may be updated repeatedly.
- **Sub-workflow reference:** None.

---

## Non-Executable Annotation Nodes

These nodes are documentation elements on the canvas and do not participate in execution, but their contents are important for understanding the design.

### 11. Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Content:** `## Trigger and filter emails with attachments`
- **Context:** Covers the trigger/filter area.
- **Execution impact:** None.

### 12. Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Content:** `## DATEV DMS upload`
- **Context:** Covers the DATEV upload/create area.
- **Execution impact:** None.

### 13. Sticky Note8

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Content:** `## Optional notifications & mailbox update`
- **Context:** Covers Slack and Outlook update nodes.
- **Execution impact:** None.

### 14. Sticky Note10

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Content:**  
  Automatically archive Outlook email attachments to DATEV DMS, with setup guidance:
  1. Connect Outlook OAuth2
  2. Connect DATEVconnect
  3. Optionally connect Slack
  4. Replace demo client lookup
  5. Adjust folder, registry, and metadata
  6. Update Outlook archive action
- **Context:** General overview note spanning the workflow.
- **Execution impact:** None.

### 15. Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Content:** `## Client lookup\nDemo setup selects the first active client`
- **Context:** Covers the client lookup area.
- **Execution impact:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Outlook emal trigger | n8n-nodes-base.microsoftOutlookTrigger | Poll Outlook for new emails and download attachments |  | If we have attachments | ## Trigger and filter emails with attachments |
| If we have attachments | n8n-nodes-base.if | Continue only for emails whose `hasAttachments` flag is true | New Outlook emal trigger | Client lookup | ## Trigger and filter emails with attachments |
| Client lookup | @klardaten/n8n-nodes-datevconnect.masterData | Retrieve a DATEV client record | If we have attachments | Split attachments | ## Client lookup Demo setup selects the first active client |
| Split attachments | n8n-nodes-base.code | Split one email with many binaries into one item per attachment | Client lookup | Upload a document file; Merge; Get DATEV user | ## DATEV DMS upload |
| Get DATEV user | @klardaten/n8n-nodes-datevconnect.identityAndAccessManagement | Retrieve current DATEV user metadata | Split attachments | Merge | ## DATEV DMS upload |
| Upload a document file | @klardaten/n8n-nodes-datevconnect.documentManagement | Upload attachment binary to DATEV as document file | Split attachments | Merge | ## DATEV DMS upload |
| Merge | n8n-nodes-base.merge | Combine user data, attachment metadata, and upload result by position | Get DATEV user; Split attachments; Upload a document file | Create a document | ## DATEV DMS upload |
| Create a document | @klardaten/n8n-nodes-datevconnect.documentManagement | Create DATEV DMS document record linked to uploaded file | Merge | Send a message | ## DATEV DMS upload |
| Send a message | n8n-nodes-base.slack | Post Slack notification after document creation | Create a document | Update a message | ## Optional notifications & mailbox update |
| Update a message | n8n-nodes-base.microsoftOutlook | Mark original Outlook email with archive category | Send a message |  | ## Optional notifications & mailbox update |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas annotation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas annotation |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Canvas annotation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Klardaten Email to DMS`.
   - In workflow settings, set:
     - **Binary mode** to `separate`
     - **Execution order** to `v1`

2. **Add the Outlook trigger**
   - Create a **Microsoft Outlook Trigger** node.
   - Name it exactly: `New Outlook emal trigger`
     - Keep this exact name if you want all expressions to work unchanged.
   - Configure:
     - Polling interval: every minute
     - Download attachments: enabled
     - Attachment prefix: `attachment_`
   - Connect Microsoft Outlook OAuth2 credentials.

3. **Add an IF node**
   - Create an **If** node named `If we have attachments`.
   - Connect it after the Outlook trigger.
   - Configure condition:
     - Left value: `{{ $json.hasAttachments }}`
     - Operation: `is true`
   - Leave only the true output connected.

4. **Add the DATEV client lookup**
   - Create the DATEVconnect **Master Data** node named `Client lookup`.
   - Connect it from the true output of the IF node.
   - Configure:
     - Top/limit: `1`
     - Filter: `status eq active`
   - Connect DATEVconnect credentials.
   - Important: this is only a demo lookup. In production, replace it with your own client matching logic, such as matching sender email, subject, tax ID, client number, or extracted metadata.

5. **Add the attachment splitting node**
   - Create a **Code** node named `Split attachments`.
   - Connect it after `Client lookup`.
   - Use JavaScript that:
     - reads all items from `New Outlook emal trigger`
     - iterates through each binary property
     - emits one item per attachment
     - stores the current attachment in `binary.data`
   - Equivalent logic:
     - For every original email item
     - For every key in `item.binary`
     - Return a new item with copied `json` and `binary.data = binaries[key]`

6. **Add the DATEV current user node**
   - Create the DATEVconnect **Identity and Access Management** node named `Get DATEV user`.
   - Connect it from `Split attachments`.
   - Configure:
     - Resource: `currentUser`

7. **Add the DATEV document file upload node**
   - Create the DATEVconnect **Document Management** node named `Upload a document file`.
   - Connect it from `Split attachments`.
   - Configure:
     - Resource: `documentFile`
     - Operation: `upload`
     - Binary property: `data`

8. **Add the Merge node**
   - Create a **Merge** node named `Merge`.
   - Set:
     - Mode: `combine`
     - Combine by: `position`
     - Number of inputs: `3`
   - Connect:
     - `Get DATEV user` to input 1
     - `Split attachments` to input 2
     - `Upload a document file` to input 3
   - The order matters because the merged item is assumed to contain all three streams aligned per attachment.

9. **Add the DATEV document creation node**
   - Create another DATEVconnect **Document Management** node named `Create a document`.
   - Connect it after `Merge`.
   - Set operation to `create`.
   - Build the document payload so it includes:
     - file extension from `binary.data.fileExtension`
     - description from `binary.data.fileName`
     - class `Dokument` with ID `1`
     - correspondence partner GUID from `Client lookup`
     - user ID from `Get DATEV user`
     - domain `Mandanten` with ID `1`
     - folder ID `2` and name `<Keine Angabe>`
     - current year and current month
     - one `structure_items` entry referencing uploaded file ID and file name
   - Required expressions:
     - Client ID: `{{ $('Client lookup').item.json.id }}`
     - User ID: `{{ $('Get DATEV user').item.json.id }}`
     - File ID: `{{ $json.id }}`
     - File name: `{{ $binary.data.fileName }}`
     - File extension: `{{ $binary.data.fileExtension }}`
   - If your DATEV tenant uses different class/domain/folder identifiers, replace the hard-coded values.

10. **Add the Slack notification node**
    - Create a **Slack** node named `Send a message`.
    - Connect it after `Create a document`.
    - Configure:
      - Authentication: OAuth2
      - Operation: send message to channel
      - Choose the target channel
    - Use message text similar to:
      - `Created a document {{ $json.description }} from client {{ $('New Outlook emal trigger').item.json.from }} in DATEV DMS. Email subject was {{ $('New Outlook emal trigger').item.json.subject }}`
    - In node settings, enable error handling equivalent to:
      - **On Error**: continue regular output
    - Connect Slack OAuth2 credentials.

11. **Add the Outlook update node**
    - Create a **Microsoft Outlook** node named `Update a message`.
    - Connect it after `Send a message`.
    - Configure:
      - Operation: `update`
      - Message ID: `{{ $('New Outlook emal trigger').item.json.id }}`
      - Update field: categories
      - Category value: `REQUEST DMS ARCHIVE`
    - Enable error handling:
      - **On Error**: continue regular output
    - Use the same Microsoft Outlook OAuth2 credentials as the trigger.

12. **Add optional sticky notes**
    - Add a sticky note over the trigger area:
      - `## Trigger and filter emails with attachments`
    - Add a sticky note over the client lookup:
      - `## Client lookup`
      - `Demo setup selects the first active client`
    - Add a sticky note over the DATEV processing area:
      - `## DATEV DMS upload`
    - Add a sticky note over the Slack/Outlook end section:
      - `## Optional notifications & mailbox update`
    - Add a large overview note with the setup guidance from the original workflow.

13. **Connect all nodes in final order**
    - `New Outlook emal trigger` → `If we have attachments`
    - `If we have attachments` (true) → `Client lookup`
    - `Client lookup` → `Split attachments`
    - `Split attachments` → `Upload a document file`
    - `Split attachments` → `Get DATEV user`
    - `Split attachments` → `Merge`
    - `Get DATEV user` → `Merge`
    - `Upload a document file` → `Merge`
    - `Merge` → `Create a document`
    - `Create a document` → `Send a message`
    - `Send a message` → `Update a message`

14. **Set up credentials**
    - **Microsoft Outlook OAuth2 API**
      - Must allow reading trigger mailbox contents and updating messages.
    - **DATEVconnect account**
      - Must allow:
        - master data lookup
        - current user retrieval
        - document file upload
        - document creation
    - **Slack OAuth2 API**
      - Must allow posting to the selected channel.

15. **Test with a sample email**
    - Send an Outlook email with one or more attachments.
    - Confirm:
      - the trigger receives the email
      - the IF node passes
      - one item is created per attachment
      - each attachment uploads to DATEV
      - one DATEV document is created per attachment
      - Slack notification appears
      - Outlook email gets category `REQUEST DMS ARCHIVE`

16. **Production hardening recommendations**
    - Replace the demo client lookup with a real identification method.
    - Consider deduplication if the same email may be processed twice.
    - Move `Get DATEV user` before the split if you want to avoid repeated calls.
    - Consider updating the Outlook message only once after all attachments succeed, rather than once per attachment.
    - Add explicit error branches or alerts for DATEV upload/document creation failures.

### Sub-workflow setup

This workflow does **not** use any sub-workflow or Execute Workflow node. No separate sub-workflow configuration is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically archive Outlook email attachments to DATEV DMS. This workflow watches your Outlook inbox for new emails with attachments and archives those attachments in DATEV DMS. It skips emails without attachments, looks up a related DATEV client, processes each attachment separately, uploads the file to DATEV DMS, creates the matching document entry, optionally sends a Slack notification, and then marks the email as archived in Outlook. | General workflow note |
| Setup steps: Connect your Microsoft Outlook OAuth2 credentials. Connect your DATEVconnect credentials. Optionally connect Slack credentials. Replace the demo client lookup with your own client matching logic. Adjust the folder, registry, and metadata fields in the document creation step. Update the Outlook archive action to match your process. | General workflow note |
| Client lookup demo setup selects the first active client. | Important implementation caveat |
| The workflow title supplied by the user is: `Archive Outlook email attachments to DATEV DMS and notify Slack`. The internal workflow JSON name is: `Klardaten Email to DMS`. | Naming/context note |