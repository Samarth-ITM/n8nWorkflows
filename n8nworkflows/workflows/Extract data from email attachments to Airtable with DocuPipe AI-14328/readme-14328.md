Extract data from email attachments to Airtable with DocuPipe AI

https://n8nworkflows.xyz/workflows/extract-data-from-email-attachments-to-airtable-with-docupipe-ai-14328


# Extract data from email attachments to Airtable with DocuPipe AI

# 1. Workflow Overview

This workflow automates document intake from email and pushes structured extraction results into Airtable using DocuPipe AI.

It is split into two connected scenarios:

- **Scenario 1: Upload**
  - Watches an IMAP inbox for new emails with attachments
  - Filters emails by subject keywords
  - Preserves email metadata
  - Uploads the first attachment to DocuPipe for asynchronous extraction

- **Scenario 2: Process & Save**
  - Waits for a DocuPipe completion event via webhook trigger
  - Fetches the extraction result from DocuPipe
  - Flattens/serializes structured data into Airtable-friendly fields
  - Adds metadata
  - Creates a new Airtable record

## 1.1 Input Reception and Email Filtering

This block listens for incoming emails with attachments and keeps only messages whose subject contains specific business-relevant keywords such as “invoice”, “receipt”, or “contract”.

## 1.2 Email Metadata Preservation and DocuPipe Submission

This block extracts selected email metadata fields and forwards the attachment to DocuPipe using a chosen extraction schema. The extraction is asynchronous.

## 1.3 DocuPipe Completion Trigger

This block starts the second scenario when DocuPipe notifies n8n that processing has completed successfully.

## 1.4 Result Retrieval and Airtable Formatting

This block retrieves the extracted result from DocuPipe and transforms nested data structures into flat values suitable for Airtable fields.

## 1.5 Metadata Enrichment and Airtable Record Creation

This final block appends document-level metadata and writes the final item into Airtable.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Email Filtering

### Overview
This block monitors an IMAP inbox and filters messages to process only those likely to contain target business documents. It reduces unnecessary DocuPipe usage by matching subject keywords before upload.

### Nodes Involved
- New Email with Attachment
- Filter by Subject

### Node Details

#### New Email with Attachment
- **Type and technical role:** `n8n-nodes-base.emailReadImap`  
  IMAP trigger/polling node that reads emails from a mailbox and downloads attachments.
- **Configuration choices:**
  - Mailbox: `INBOX`
  - Attachment download enabled
- **Key expressions or variables used:**  
  None in configuration, but downstream nodes use fields such as:
  - `$json.subject`
  - `$json.from`
  - `$json.date`
  - binary attachment data such as `attachment_0`
- **Input and output connections:**
  - No input; this is an entry point
  - Output → `Filter by Subject`
- **Version-specific requirements:**
  - Uses node type version `2`
- **Edge cases or potential failure types:**
  - IMAP authentication failure
  - Wrong server/port/TLS settings in credentials
  - Mailbox access denied
  - Messages without attachments may not provide the expected binary property
  - If multiple attachments exist, only the first one is later used
  - Polling/trigger behavior depends on n8n execution mode and IMAP support
- **Sub-workflow reference:**  
  None

#### Filter by Subject
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router that checks whether the email subject contains at least one target keyword.
- **Configuration choices:**
  - Case-insensitive matching
  - Strict type validation
  - `OR` combinator across conditions
  - Subject must contain one of:
    - `invoice`
    - `receipt`
    - `contract`
- **Key expressions or variables used:**
  - `={{ $json.subject }}`
- **Input and output connections:**
  - Input ← `New Email with Attachment`
  - True output → `Extract Email Info`
  - False output is unused
- **Version-specific requirements:**
  - Uses type version `2.2`
  - Conditions configured with the newer condition model (`version: 2`)
- **Edge cases or potential failure types:**
  - Missing or null subject may cause no match
  - Useful documents with unmatched wording will be ignored
  - Subject-only filtering can lead to false positives or false negatives
- **Sub-workflow reference:**  
  None

---

## 2.2 Email Metadata Preservation and DocuPipe Submission

### Overview
This block keeps selected email metadata and submits the attachment to DocuPipe using a configured schema. It prepares the document for asynchronous structured extraction.

### Nodes Involved
- Extract Email Info
- Upload Attachment & Extract Data

### Node Details

#### Extract Email Info
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds normalized email metadata fields while keeping existing incoming data.
- **Configuration choices:**
  - Includes all existing input fields
  - Adds:
    - `emailFrom` from sender
    - `emailSubject` from email subject
    - `emailDate` from email date
- **Key expressions or variables used:**
  - `={{ $json.from }}`
  - `={{ $json.subject }}`
  - `={{ $json.date }}`
- **Input and output connections:**
  - Input ← `Filter by Subject`
  - Output → `Upload Attachment & Extract Data`
- **Version-specific requirements:**
  - Uses type version `3.4`
- **Edge cases or potential failure types:**
  - Missing email metadata fields may produce empty values
  - Preserved metadata is not directly carried into Scenario 2 unless DocuPipe or external correlation supports it
- **Sub-workflow reference:**  
  None

#### Upload Attachment & Extract Data
- **Type and technical role:** `n8n-nodes-docupipe.docuPipe`  
  Community DocuPipe node used to upload a binary file and start extraction.
- **Configuration choices:**
  - Resource: `extraction`
  - Operation: `uploadAndExtract`
  - Input mode: `binary`
  - Binary property: `attachment_0`
  - Schema selected from DocuPipe schema list
- **Key expressions or variables used:**
  - Binary property name is static: `attachment_0`
- **Input and output connections:**
  - Input ← `Extract Email Info`
  - No downstream connection in this workflow branch
- **Version-specific requirements:**
  - Uses DocuPipe community node version `1`
  - Requires installation of `n8n-nodes-docupipe`
- **Edge cases or potential failure types:**
  - Missing community node installation
  - Invalid or missing DocuPipe API credentials
  - Empty schema selection
  - Missing binary property if attachment naming differs or attachment is absent
  - If email has multiple attachments, only the first (`attachment_0`) is uploaded
  - Unsupported file type or API-side validation failure
  - Large attachment upload errors or timeout
- **Sub-workflow reference:**  
  None

---

## 2.3 DocuPipe Completion Trigger

### Overview
This block waits for DocuPipe to notify n8n when extraction has completed successfully. It acts as the entry point for the second scenario.

### Nodes Involved
- Extraction Complete

### Node Details

#### Extraction Complete
- **Type and technical role:** `n8n-nodes-docupipe.docuPipeTrigger`  
  Webhook-based trigger that starts execution when a DocuPipe event is emitted.
- **Configuration choices:**
  - Event: `standardization.processed.success`
  - Uses a generated webhook ID
- **Key expressions or variables used:**  
  None directly in configuration. Downstream node expects:
  - `$json.standardizationId`
- **Input and output connections:**
  - No input; this is a second entry point
  - Output → `Get Extracted Data`
- **Version-specific requirements:**
  - Uses DocuPipe trigger node version `1`
  - Requires active workflow and accessible webhook endpoint
- **Edge cases or potential failure types:**
  - Workflow inactive, so webhook not available
  - Public webhook URL inaccessible from DocuPipe
  - Credential mismatch between upload and trigger environments
  - Event payload missing `standardizationId`
  - Trigger only listens for successful processing; failures are not handled here
- **Sub-workflow reference:**  
  None

---

## 2.4 Result Retrieval and Airtable Formatting

### Overview
This block retrieves the finished extraction result and transforms it into a flat structure suitable for Airtable. Arrays become readable multiline text and nested objects are serialized.

### Nodes Involved
- Get Extracted Data
- Process for Airtable

### Node Details

#### Get Extracted Data
- **Type and technical role:** `n8n-nodes-docupipe.docuPipe`  
  Fetches the extraction result using the standardization ID received from the webhook trigger.
- **Configuration choices:**
  - Resource: `extraction`
  - Operation: `getResult`
  - Standardization ID comes from trigger payload
- **Key expressions or variables used:**
  - `={{ $json.standardizationId }}`
- **Input and output connections:**
  - Input ← `Extraction Complete`
  - Output → `Process for Airtable`
- **Version-specific requirements:**
  - Uses DocuPipe node version `1`
- **Edge cases or potential failure types:**
  - Missing `standardizationId`
  - Result not yet available despite success event timing issues
  - API authorization failure
  - API-side transient errors
- **Sub-workflow reference:**  
  None

#### Process for Airtable
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that flattens extraction output for Airtable ingestion.
- **Configuration choices:**
  - Reads the first input item
  - Uses `data.result || {}`
  - Iterates over all result fields
  - Transformation behavior:
    - Arrays → multiline text
    - Array objects → `key: value` pairs joined by commas, then joined by newlines
    - Nested objects → JSON string
    - Scalars → unchanged
- **Key expressions or variables used:**
  - `$input.first().json`
  - `data.result`
- **Input and output connections:**
  - Input ← `Get Extracted Data`
  - Output → `Add Metadata`
- **Version-specific requirements:**
  - Uses code node version `2`
  - Assumes JavaScript execution support in the n8n environment
- **Edge cases or potential failure types:**
  - If no input item exists, `$input.first()` may fail
  - If result fields contain deeply nested or large arrays, output may exceed Airtable field expectations
  - Airtable field types may still reject some values if schema is incompatible
  - Serialization can reduce structured queryability in Airtable
- **Sub-workflow reference:**  
  None

---

## 2.5 Metadata Enrichment and Airtable Record Creation

### Overview
This block adds final metadata fields and inserts the transformed data into Airtable as a new record. It is the persistence layer of the workflow.

### Nodes Involved
- Add Metadata
- Create Record in Airtable

### Node Details

#### Add Metadata
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds document-level metadata while retaining all transformed extraction fields.
- **Configuration choices:**
  - Includes all incoming fields
  - Adds:
    - `Document Name` from DocuPipe result payload
    - `Extracted At` with current execution timestamp
- **Key expressions or variables used:**
  - `={{ $('Get Extracted Data').item.json.documentName }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**
  - Input ← `Process for Airtable`
  - Output → `Create Record in Airtable`
- **Version-specific requirements:**
  - Uses type version `3.4`
  - Cross-node item reference assumes `Get Extracted Data` has a matching current item context
- **Edge cases or potential failure types:**
  - If `documentName` is missing, metadata field may be empty
  - Cross-node reference can break in more complex multi-item scenarios
  - Airtable field names must match exactly, including capitalization and spaces if mapped automatically
- **Sub-workflow reference:**  
  None

#### Create Record in Airtable
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Creates a new row in a selected Airtable base/table using incoming fields.
- **Configuration choices:**
  - Operation: `create`
  - Base selected from Airtable application list
  - Table selected from table list
  - Column mapping mode: automatic mapping from input data
- **Key expressions or variables used:**  
  No custom expressions shown in configuration.
- **Input and output connections:**
  - Input ← `Add Metadata`
  - No downstream output
- **Version-specific requirements:**
  - Uses Airtable node version `2.1`
- **Edge cases or potential failure types:**
  - Invalid Airtable token or insufficient permissions
  - Base/table not selected
  - Auto-mapping only works when Airtable field names match incoming JSON keys
  - Type mismatch, especially for number/date/select fields
  - Long serialized text may exceed Airtable limits
  - Required Airtable fields not present
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Email with Attachment | IMAP Email Read | Watches inbox and downloads email attachments |  | Filter by Subject | ## Scenario 1 - Upload Email Attachments to DocuPipe New emails with attachments are detected via IMAP, filtered by subject keywords (invoice, receipt, etc.), and email metadata is extracted before uploading to DocuPipe. Works with any IMAP email provider - Outlook, Yahoo, or custom domains. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Filter by Subject | If | Keeps only emails whose subject contains target keywords | New Email with Attachment | Extract Email Info | ## Scenario 1 - Upload Email Attachments to DocuPipe New emails with attachments are detected via IMAP, filtered by subject keywords (invoice, receipt, etc.), and email metadata is extracted before uploading to DocuPipe. Works with any IMAP email provider - Outlook, Yahoo, or custom domains. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Extract Email Info | Set | Adds normalized email metadata fields | Filter by Subject | Upload Attachment & Extract Data | ## Scenario 1 - Upload Email Attachments to DocuPipe New emails with attachments are detected via IMAP, filtered by subject keywords (invoice, receipt, etc.), and email metadata is extracted before uploading to DocuPipe. Works with any IMAP email provider - Outlook, Yahoo, or custom domains. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Upload Attachment & Extract Data | DocuPipe | Uploads the first email attachment and starts extraction | Extract Email Info |  | ## Scenario 1 - Upload Email Attachments to DocuPipe New emails with attachments are detected via IMAP, filtered by subject keywords (invoice, receipt, etc.), and email metadata is extracted before uploading to DocuPipe. Works with any IMAP email provider - Outlook, Yahoo, or custom domains. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Extraction Complete | DocuPipe Trigger | Receives extraction-complete webhook event |  | Get Extracted Data | ## Scenario 2 - Process Extraction Results & Save to Airtable When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into Airtable-compatible format (nested objects and arrays are serialized), enriched with metadata, and saved as a new record in your Airtable base. |
| Get Extracted Data | DocuPipe | Fetches extracted result from DocuPipe using standardization ID | Extraction Complete | Process for Airtable | ## Scenario 2 - Process Extraction Results & Save to Airtable When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into Airtable-compatible format (nested objects and arrays are serialized), enriched with metadata, and saved as a new record in your Airtable base. |
| Process for Airtable | Code | Flattens extraction output into Airtable-friendly values | Get Extracted Data | Add Metadata | ## Scenario 2 - Process Extraction Results & Save to Airtable When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into Airtable-compatible format (nested objects and arrays are serialized), enriched with metadata, and saved as a new record in your Airtable base. |
| Add Metadata | Set | Adds document name and extraction timestamp | Process for Airtable | Create Record in Airtable | ## Scenario 2 - Process Extraction Results & Save to Airtable When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into Airtable-compatible format (nested objects and arrays are serialized), enriched with metadata, and saved as a new record in your Airtable base. |
| Create Record in Airtable | Airtable | Creates a new Airtable record from processed data | Add Metadata |  | ## Scenario 2 - Process Extraction Results & Save to Airtable When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into Airtable-compatible format (nested objects and arrays are serialized), enriched with metadata, and saved as a new record in your Airtable base. |
| Sticky Note | Sticky Note | Workspace documentation and setup instructions |  |  | ## Email to DocuPipe to Airtable Automatically extract structured data from email attachments using [DocuPipe](https://docupipe.ai) AI and save results to Airtable. Works with any IMAP email provider (Outlook, Yahoo, custom domains). ### Who is this for? Teams that receive documents via email and want structured data automatically extracted and saved to an Airtable base for tracking, review, or further automation. ### How it works This template contains two connected flows: **Scenario 1 - Upload:** Watches your inbox for new emails with attachments, filters by subject keywords (e.g. invoice, receipt), extracts email metadata, and uploads the attachment to DocuPipe. **Scenario 2 - Process & Save:** When DocuPipe finishes extracting, the webhook fires, results are fetched, processed into Airtable-compatible format with metadata, and saved as a new record. ### How to set up 1. Connect your **IMAP email** account (server, port, username, password) 2. Customize the **subject filter** keywords in the Filter by Subject node 3. Sign up at [docupipe.ai](https://docupipe.ai), then get your **DocuPipe API key** at [app.docupipe.ai/settings/general](https://app.docupipe.ai/settings/general) 4. Select an extraction **schema** in the Upload node 5. Connect your **Airtable** account, select your base and table 6. Ensure your Airtable column names match the schema field names 7. Activate this workflow ### Requirements - A [DocuPipe](https://docupipe.ai) account with an API key - An email account with IMAP access - An Airtable account with a table configured for your data - An extraction schema configured in DocuPipe **Note:** Requires the [DocuPipe community node](https://www.npmjs.com/package/n8n-nodes-docupipe). Install via Settings > Community Nodes. |
| Sticky Note1 | Sticky Note | Visual documentation for Scenario 1 |  |  | ## Scenario 1 - Upload Email Attachments to DocuPipe New emails with attachments are detected via IMAP, filtered by subject keywords (invoice, receipt, etc.), and email metadata is extracted before uploading to DocuPipe. Works with any IMAP email provider - Outlook, Yahoo, or custom domains. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Sticky Note2 | Sticky Note | Visual documentation for Scenario 2 |  |  | ## Scenario 2 - Process Extraction Results & Save to Airtable When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into Airtable-compatible format (nested objects and arrays are serialized), enriched with metadata, and saved as a new record in your Airtable base. |

---

# 4. Reproducing the Workflow from Scratch

1. **Install the DocuPipe community node**
   - In n8n, go to **Settings > Community Nodes**
   - Install: `n8n-nodes-docupipe`
   - Confirm your n8n instance allows community nodes

2. **Prepare external services**
   - Create or access a **DocuPipe** account
   - Create an extraction **schema** in DocuPipe
   - Copy your **DocuPipe API key** from:
     - `https://app.docupipe.ai/settings/general`
   - Prepare an **IMAP-enabled email account**
   - Prepare an **Airtable base and table**
   - Ensure Airtable field names match the extraction output keys you expect, plus:
     - `Document Name`
     - `Extracted At`

3. **Create the first entry node: IMAP email reader**
   - Add node: **Email Read IMAP**
   - Name it: `New Email with Attachment`
   - Set:
     - Mailbox: `INBOX`
     - Options → enable **Download Attachments**
   - Create/select IMAP credentials:
     - Host/server
     - Port
     - Username
     - Password
     - TLS/SSL settings as required by your provider

4. **Add the subject filter**
   - Add node: **If**
   - Name it: `Filter by Subject`
   - Connect `New Email with Attachment` → `Filter by Subject`
   - Configure conditions:
     - Combinator: **OR**
     - Case sensitive: **false**
     - Add three string contains rules using the email subject:
       - Left value: `{{ $json.subject }}`
       - Contains: `invoice`
       - Left value: `{{ $json.subject }}`
       - Contains: `receipt`
       - Left value: `{{ $json.subject }}`
       - Contains: `contract`

5. **Add the email metadata mapping node**
   - Add node: **Set**
   - Name it: `Extract Email Info`
   - Connect the **true** output of `Filter by Subject` → `Extract Email Info`
   - Set **Include Input Fields** / keep all existing fields
   - Add fields:
     - `emailFrom` = `{{ $json.from }}`
     - `emailSubject` = `{{ $json.subject }}`
     - `emailDate` = `{{ $json.date }}`

6. **Add the DocuPipe upload node**
   - Add node: **DocuPipe**
   - Name it: `Upload Attachment & Extract Data`
   - Connect `Extract Email Info` → `Upload Attachment & Extract Data`
   - Configure:
     - Resource: `extraction`
     - Operation: `uploadAndExtract`
     - Input mode: `binary`
     - Binary property name: `attachment_0`
     - Schema: select your DocuPipe extraction schema
   - Create/select DocuPipe API credentials

7. **Create the second entry node: DocuPipe completion trigger**
   - Add node: **DocuPipe Trigger**
   - Name it: `Extraction Complete`
   - Configure:
     - Event: `standardization.processed.success`
   - Use the same or appropriate DocuPipe API credentials
   - Save the workflow so the webhook can be registered properly

8. **Add the result retrieval node**
   - Add node: **DocuPipe**
   - Name it: `Get Extracted Data`
   - Connect `Extraction Complete` → `Get Extracted Data`
   - Configure:
     - Resource: `extraction`
     - Operation: `getResult`
     - Standardization ID: `{{ $json.standardizationId }}`

9. **Add the transformation code node**
   - Add node: **Code**
   - Name it: `Process for Airtable`
   - Connect `Get Extracted Data` → `Process for Airtable`
   - Use JavaScript mode
   - Paste this logic:

   ```javascript
   const data = $input.first().json;
   const result = data.result || {};
   const output = {};

   for (const [key, value] of Object.entries(result)) {
     if (Array.isArray(value)) {
       output[key] = value.map(item => {
         if (typeof item === 'object') {
           return Object.entries(item).map(([k, v]) => `${k}: ${v}`).join(', ');
         }
         return String(item);
       }).join('\n');
     } else if (typeof value === 'object' && value !== null) {
       output[key] = JSON.stringify(value);
     } else {
       output[key] = value;
     }
   }

   return [{ json: output }];
   ```

10. **Add metadata enrichment**
    - Add node: **Set**
    - Name it: `Add Metadata`
    - Connect `Process for Airtable` → `Add Metadata`
    - Keep all incoming fields
    - Add:
      - `Document Name` = `{{ $('Get Extracted Data').item.json.documentName }}`
      - `Extracted At` = `{{ $now.toISO() }}`

11. **Add the Airtable create node**
    - Add node: **Airtable**
    - Name it: `Create Record in Airtable`
    - Connect `Add Metadata` → `Create Record in Airtable`
    - Configure:
      - Operation: `Create`
      - Select your Airtable base
      - Select your Airtable table
      - Column mapping mode: **Auto-map input data**
    - Create/select Airtable credentials using a personal access token or supported token auth

12. **Validate Airtable column compatibility**
    - Ensure your Airtable table contains columns matching:
      - Each extraction field returned by DocuPipe after processing
      - `Document Name`
      - `Extracted At`
    - If extraction fields are nested or array-based, expect them as string/text columns unless you customize the mapping logic

13. **Test Scenario 1**
    - Send an email with at least one attachment to the connected inbox
    - Use a subject containing one of:
      - invoice
      - receipt
      - contract
    - Verify that the IMAP node detects the email and the DocuPipe upload node receives binary `attachment_0`

14. **Test Scenario 2**
    - After DocuPipe finishes extraction, confirm the trigger node receives the webhook event
    - Verify `Get Extracted Data` returns:
      - `documentName`
      - `result`
    - Verify `Process for Airtable` outputs flat fields only
    - Verify Airtable record creation succeeds

15. **Activate the workflow**
    - Activation is required so:
      - IMAP monitoring continues
      - DocuPipe webhook trigger is live
    - Keep the n8n instance publicly reachable for webhook delivery if DocuPipe calls a hosted webhook URL

16. **Optional hardening improvements**
    - Expand subject filters or replace them with sender/domain logic
    - Add error handling for failed DocuPipe extractions
    - Handle more than one attachment by iterating through binary properties
    - Persist email metadata correlation into Airtable if you need sender/subject/date in final records
    - Add deduplication to avoid duplicate record creation

### Credential Configuration Summary

- **IMAP**
  - Server/host
  - Port
  - Username
  - Password
  - SSL/TLS as required

- **DocuPipe API**
  - API key from DocuPipe account settings

- **Airtable**
  - Airtable token with access to the selected base/table

### Sub-workflow Setup
This workflow does **not** invoke any sub-workflow and does **not** contain an Execute Workflow node. It does, however, have **two entry points**:
- `New Email with Attachment`
- `Extraction Complete`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DocuPipe main site | https://docupipe.ai |
| DocuPipe API key settings page | https://app.docupipe.ai/settings/general |
| DocuPipe community node package for n8n | https://www.npmjs.com/package/n8n-nodes-docupipe |
| Workflow purpose: automatically extract structured data from email attachments and save it to Airtable | General workflow context |
| Intended users: teams receiving documents by email and wanting structured records for tracking, review, or automation | General workflow context |
| Works with IMAP providers such as Outlook, Yahoo, and custom domains | General workflow context |
| Airtable column names should match schema field names for successful auto-mapping | Deployment note |
| The workflow relies on asynchronous extraction: upload happens in Scenario 1 and completion is received in Scenario 2 | Architecture note |
| Requires a DocuPipe account, IMAP-capable mailbox, Airtable account, and a configured DocuPipe extraction schema | Requirements |