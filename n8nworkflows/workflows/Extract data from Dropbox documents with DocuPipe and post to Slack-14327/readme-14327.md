Extract data from Dropbox documents with DocuPipe and post to Slack

https://n8nworkflows.xyz/workflows/extract-data-from-dropbox-documents-with-docupipe-and-post-to-slack-14327


# Extract data from Dropbox documents with DocuPipe and post to Slack

# 1. Workflow Overview

This workflow automates document extraction from Dropbox using DocuPipe and sends the extracted results to Slack.

It is designed for teams that receive documents such as invoices, receipts, or forms in Dropbox and want structured data extraction plus immediate visibility in Slack once processing is complete.

The workflow is split into two independent but related execution paths:

## 1.1 Scenario 1 – Detect New Files and Send Them to DocuPipe

This block runs on a schedule every minute, lists files in a Dropbox folder, filters for files added very recently, downloads each matching file, and uploads it to DocuPipe for extraction.

This part is asynchronous: it does not wait for extraction results. Instead, DocuPipe later notifies n8n through a webhook.

## 1.2 Scenario 2 – Receive Completion Event and Notify Slack

This block starts when DocuPipe sends a webhook indicating that extraction has completed successfully. The workflow fetches the extraction result, formats the fields into a human-readable Slack message, and posts that message to a selected Slack channel.

## 1.3 Supporting Documentation Layer

The workflow also includes sticky notes that describe its purpose, setup requirements, and each scenario’s behavior. These notes are important for maintainability and should be preserved if the workflow is recreated.

---

# 2. Block-by-Block Analysis

## 2.1 Documentation and Setup Context

**Overview:**  
This block consists of sticky notes used to document the workflow’s purpose, scenarios, setup steps, and requirements. They do not affect execution, but they provide critical operational context.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation node
- **Configuration choices:** Contains the general workflow description, user profile, setup steps, required services, and dependency on the DocuPipe community node
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** No runtime failure; risk is informational loss if omitted during recreation
- **Sub-workflow reference:** None

### Node: Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation node
- **Configuration choices:** Describes Scenario 1: polling Dropbox, filtering new files, downloading, and uploading to DocuPipe asynchronously
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

### Node: Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation node
- **Configuration choices:** Describes Scenario 2: webhook reception, result retrieval, formatting, and Slack delivery
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None at runtime
- **Sub-workflow reference:** None

---

## 2.2 Scenario 1 – Poll Dropbox and Upload New Files to DocuPipe

**Overview:**  
This block checks Dropbox every minute for newly modified files, filters out anything that is not a file or not recent enough, downloads eligible files, and sends them to DocuPipe for extraction.

**Nodes Involved:**  
- Check Every Minute
- List Dropbox Folder
- Filter New Files
- Download File from Dropbox
- Upload File & Extract Data

### Node: Check Every Minute
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry-point trigger
- **Configuration choices:** Runs every 1 minute using an interval-based schedule
- **Key expressions or variables used:** None
- **Input and output connections:** No input; outputs to **List Dropbox Folder**
- **Version-specific requirements:** Type version 1.2
- **Edge cases or potential failure types:**
  - Workflow not active, so scheduled executions never occur
  - High polling frequency can increase API calls to Dropbox
  - If Dropbox listing is slow, runs may overlap in busy environments
- **Sub-workflow reference:** None

### Node: List Dropbox Folder
- **Type and technical role:** `n8n-nodes-base.dropbox`; folder listing operation
- **Configuration choices:**
  - Resource: `folder`
  - Operation: `list`
  - Path: currently empty and must be configured
  - Uses Dropbox OAuth2 credentials
- **Key expressions or variables used:** None
- **Input and output connections:** Input from **Check Every Minute**; output to **Filter New Files**
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:**
  - Empty path may default incorrectly or fail depending on environment; should be explicitly set
  - Authentication or token expiration issues
  - Folder not found
  - Permission issues on the selected Dropbox path
  - Large folders may return many items and increase execution load
- **Sub-workflow reference:** None

### Node: Filter New Files
- **Type and technical role:** `n8n-nodes-base.if`; conditional branching and filtering
- **Configuration choices:**
  - Uses AND logic with strict type validation
  - Condition 1: `{{$json['.tag']}}` must equal `file`
  - Condition 2: `{{$json.server_modified}}` must be after `{{$now.minus({minutes: 2}).toISO()}}`
  - This effectively keeps only Dropbox items that are files and were modified within the last 2 minutes
- **Key expressions or variables used:**
  - `{{$json['.tag']}}`
  - `{{$json.server_modified}}`
  - `{{$now.minus({minutes: 2}).toISO()}}`
- **Input and output connections:** Input from **List Dropbox Folder**; true output to **Download File from Dropbox**
- **Version-specific requirements:** Type version 2.2; condition syntax depends on newer IF node structure
- **Edge cases or potential failure types:**
  - Time-window mismatch may cause missed or duplicate processing if the schedule drifts
  - `server_modified` is based on Dropbox metadata, not necessarily upload time
  - Files updated twice may be reprocessed
  - Timezone and ISO parsing issues are unlikely but possible if source values are malformed
  - Non-file items such as folders are intentionally excluded
- **Sub-workflow reference:** None

### Node: Download File from Dropbox
- **Type and technical role:** `n8n-nodes-base.dropbox`; file download operation
- **Configuration choices:**
  - Resource: `file`
  - Operation: `download`
  - Path is dynamic: `{{$json.path_display}}`
  - Downloads binary data into the standard binary property used by the Dropbox node
- **Key expressions or variables used:**
  - `{{$json.path_display}}`
- **Input and output connections:** Input from **Filter New Files**; output to **Upload File & Extract Data**
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:**
  - File deleted or moved between listing and download
  - Insufficient permissions
  - Unsupported path value if metadata is incomplete
  - Large files may increase memory usage or hit node/platform limits
- **Sub-workflow reference:** None

### Node: Upload File & Extract Data
- **Type and technical role:** `n8n-nodes-docupipe.docuPipe`; DocuPipe community node for document upload and extraction
- **Configuration choices:**
  - Resource: `extraction`
  - Operation: `uploadAndExtract`
  - Input mode: `binary`
  - Binary property name: `data`
  - Schema ID selected from list and currently unset in the JSON, so user must choose one
  - Uses DocuPipe API credentials
- **Key expressions or variables used:** None in the current configuration
- **Input and output connections:** Input from **Download File from Dropbox**; no downstream node in this scenario
- **Version-specific requirements:**
  - Type version 1
  - Requires installation of the `n8n-nodes-docupipe` community package
- **Edge cases or potential failure types:**
  - Missing DocuPipe community node
  - Missing or invalid API key
  - Schema ID not selected
  - Binary property mismatch if downloaded file is not available as `data`
  - Unsupported file format or DocuPipe API validation failure
  - Extraction starts asynchronously; users may mistakenly expect immediate results in the same execution
- **Sub-workflow reference:** None

---

## 2.3 Scenario 2 – Receive DocuPipe Completion and Send Slack Notification

**Overview:**  
This block listens for successful DocuPipe extraction events, retrieves the detailed extraction result, converts it into a readable message, and posts the output to Slack.

**Nodes Involved:**  
- Extraction Complete
- Get Extracted Data
- Format Slack Message
- Set Channel & Message
- Post Results to Slack

### Node: Extraction Complete
- **Type and technical role:** `n8n-nodes-docupipe.docuPipeTrigger`; webhook-based trigger for DocuPipe events
- **Configuration choices:**
  - Event subscribed: `standardization.processed.success`
  - Uses a DocuPipe webhook ID internally
  - Starts the second scenario when DocuPipe confirms a successful extraction
- **Key expressions or variables used:** None directly
- **Input and output connections:** No input; outputs to **Get Extracted Data**
- **Version-specific requirements:**
  - Type version 1
  - Requires DocuPipe community node
  - Workflow must be active and webhook must be reachable from DocuPipe
- **Edge cases or potential failure types:**
  - Webhook not registered or not reachable
  - DocuPipe event delivery failure
  - Only success events are handled; failures and partial results are ignored
  - Test vs production webhook URL confusion during setup
- **Sub-workflow reference:** None

### Node: Get Extracted Data
- **Type and technical role:** `n8n-nodes-docupipe.docuPipe`; fetches extraction result by ID
- **Configuration choices:**
  - Resource: `extraction`
  - Operation: `getResult`
  - Standardization ID expression: `{{$json.standardizationId}}`
- **Key expressions or variables used:**
  - `{{$json.standardizationId}}`
- **Input and output connections:** Input from **Extraction Complete**; output to **Format Slack Message**
- **Version-specific requirements:**
  - Type version 1
  - Requires DocuPipe community node
- **Edge cases or potential failure types:**
  - Missing `standardizationId` in webhook payload
  - Result not yet available despite success event timing
  - Authentication failures
  - API rate limit or transient DocuPipe service errors
- **Sub-workflow reference:** None

### Node: Format Slack Message
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript formatter
- **Configuration choices:**
  - Reads the first input item only via `$input.first().json`
  - Extracts:
    - `data.result || {}`
    - `data.documentName || 'Document'`
  - Builds a Slack-formatted message:
    - Adds a success title
    - Adds document name
    - Iterates over all fields in `result`
    - Handles arrays, objects, and scalar values differently
  - Returns:
    - `formattedMessage`
    - `documentName`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `data.result`
  - `data.documentName`
  - Loop over `Object.entries(result)`
- **Input and output connections:** Input from **Get Extracted Data**; output to **Set Channel & Message**
- **Version-specific requirements:** Type version 2
- **Edge cases or potential failure types:**
  - If multiple items arrive, only the first is processed
  - Nested objects are stringified as JSON and may be hard to read in Slack
  - Very large extraction results may create oversized Slack messages
  - Null or unusual values may produce visually awkward output
  - Arrays of objects are flattened by joining object values, which may lose field names
- **Sub-workflow reference:** None

### Node: Set Channel & Message
- **Type and technical role:** `n8n-nodes-base.set`; prepares Slack payload fields
- **Configuration choices:**
  - Creates `text` from `{{$json.formattedMessage}}`
  - Creates `subject` as `DocuPipe: {{$json.documentName}} extracted`
  - `subject` is prepared but not used by the Slack node in this workflow
- **Key expressions or variables used:**
  - `{{$json.formattedMessage}}`
  - `DocuPipe: {{$json.documentName}} extracted`
- **Input and output connections:** Input from **Format Slack Message**; output to **Post Results to Slack**
- **Version-specific requirements:** Type version 3.4
- **Edge cases or potential failure types:**
  - If `formattedMessage` is missing, Slack text will be empty
  - `subject` may mislead maintainers into thinking it is used downstream
- **Sub-workflow reference:** None

### Node: Post Results to Slack
- **Type and technical role:** `n8n-nodes-base.slack`; posts a message to a Slack channel
- **Configuration choices:**
  - Uses the `text` field from input: `{{$json.text}}`
  - Sends to a selected channel using Slack OAuth2 credentials
  - Channel ID is selected from list and currently unset in the JSON, so it must be configured
- **Key expressions or variables used:**
  - `{{$json.text}}`
- **Input and output connections:** Input from **Set Channel & Message**; no downstream output
- **Version-specific requirements:** Type version 2.2
- **Edge cases or potential failure types:**
  - Slack credential or permission issues
  - Bot not invited to the target channel
  - Channel ID not selected
  - Message length or formatting limitations in Slack
  - If the workspace restricts app posting, delivery may fail
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | General workflow documentation and setup instructions |  |  | ## Dropbox to DocuPipe to Slack<br><br>Automatically extract structured data from files uploaded to Dropbox using [DocuPipe](https://docupipe.ai) AI and post formatted results to Slack.<br><br>### Who is this for?<br>Teams that collect documents in Dropbox (invoices, receipts, forms) and want instant Slack notifications with the extracted data - great for invoice processing, receipt tracking, or any document workflow where visibility matters.<br><br>### How it works<br>This template contains two connected flows:<br><br>**Scenario 1 - Upload:** Polls a Dropbox folder every minute for new files, filters for recently added documents, downloads them, and uploads to DocuPipe for extraction.<br><br>**Scenario 2 - Notify:** When DocuPipe finishes extracting, the webhook fires, results are fetched, formatted into a readable Slack message with field labels and values, and posted to your channel.<br><br>### How to set up<br>1. Connect your **Dropbox** account<br>2. Set the **folder path** to watch in the List Dropbox Folder node<br>3. Sign up at [docupipe.ai](https://docupipe.ai), then get your **DocuPipe API key** at [app.docupipe.ai/settings/general](https://app.docupipe.ai/settings/general)<br>4. Select an extraction **schema** in the Upload node<br>5. Connect your **Slack** account and select the target channel<br>6. Activate this workflow<br><br>### Requirements<br>- A [DocuPipe](https://docupipe.ai) account with an API key<br>- A Dropbox account<br>- A Slack workspace with bot permissions<br>- An extraction schema configured in DocuPipe<br><br>**Note:** Requires the [DocuPipe community node](https://www.npmjs.com/package/n8n-nodes-docupipe). Install via Settings > Community Nodes. |
| Sticky Note1 | stickyNote | Documentation for Scenario 1 |  |  | ## Scenario 1 - Detect New Files & Upload to DocuPipe<br>Polls a Dropbox folder every minute for files. A filter checks for recently added files so only new documents are processed. Each new file is downloaded and uploaded to DocuPipe for AI-powered extraction. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Sticky Note2 | stickyNote | Documentation for Scenario 2 |  |  | ## Scenario 2 - Process Extraction Results & Post to Slack<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, formatted into a readable Slack message with labeled fields, and posted to your chosen channel. |
| Check Every Minute | scheduleTrigger | Scheduled polling trigger for Dropbox |  | List Dropbox Folder | ## Scenario 1 - Detect New Files & Upload to DocuPipe<br>Polls a Dropbox folder every minute for files. A filter checks for recently added files so only new documents are processed. Each new file is downloaded and uploaded to DocuPipe for AI-powered extraction. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| List Dropbox Folder | dropbox | Lists files/folders in the target Dropbox path | Check Every Minute | Filter New Files | ## Scenario 1 - Detect New Files & Upload to DocuPipe<br>Polls a Dropbox folder every minute for files. A filter checks for recently added files so only new documents are processed. Each new file is downloaded and uploaded to DocuPipe for AI-powered extraction. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Filter New Files | if | Filters for recent Dropbox files only | List Dropbox Folder | Download File from Dropbox | ## Scenario 1 - Detect New Files & Upload to DocuPipe<br>Polls a Dropbox folder every minute for files. A filter checks for recently added files so only new documents are processed. Each new file is downloaded and uploaded to DocuPipe for AI-powered extraction. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Download File from Dropbox | dropbox | Downloads the selected Dropbox file as binary | Filter New Files | Upload File & Extract Data | ## Scenario 1 - Detect New Files & Upload to DocuPipe<br>Polls a Dropbox folder every minute for files. A filter checks for recently added files so only new documents are processed. Each new file is downloaded and uploaded to DocuPipe for AI-powered extraction. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Upload File & Extract Data | n8n-nodes-docupipe.docuPipe | Uploads document to DocuPipe and starts extraction | Download File from Dropbox |  | ## Scenario 1 - Detect New Files & Upload to DocuPipe<br>Polls a Dropbox folder every minute for files. A filter checks for recently added files so only new documents are processed. Each new file is downloaded and uploaded to DocuPipe for AI-powered extraction. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Extraction Complete | n8n-nodes-docupipe.docuPipeTrigger | Receives DocuPipe success webhook event |  | Get Extracted Data | ## Scenario 2 - Process Extraction Results & Post to Slack<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, formatted into a readable Slack message with labeled fields, and posted to your chosen channel. |
| Get Extracted Data | n8n-nodes-docupipe.docuPipe | Retrieves extraction result from DocuPipe | Extraction Complete | Format Slack Message | ## Scenario 2 - Process Extraction Results & Post to Slack<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, formatted into a readable Slack message with labeled fields, and posted to your chosen channel. |
| Format Slack Message | code | Converts extraction result into Slack-friendly text | Get Extracted Data | Set Channel & Message | ## Scenario 2 - Process Extraction Results & Post to Slack<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, formatted into a readable Slack message with labeled fields, and posted to your chosen channel. |
| Set Channel & Message | set | Prepares message fields for Slack | Format Slack Message | Post Results to Slack | ## Scenario 2 - Process Extraction Results & Post to Slack<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, formatted into a readable Slack message with labeled fields, and posted to your chosen channel. |
| Post Results to Slack | slack | Sends the final message to Slack | Set Channel & Message |  | ## Scenario 2 - Process Extraction Results & Post to Slack<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, formatted into a readable Slack message with labeled fields and values, and posted to your chosen channel. |

---

# 4. Reproducing the Workflow from Scratch

1. **Install the DocuPipe community node**
   - In n8n, go to **Settings > Community Nodes**.
   - Install the package: `n8n-nodes-docupipe`.
   - Confirm your n8n instance allows community nodes.

2. **Create a new workflow**
   - Name it: **Extract data from Dropbox documents with DocuPipe and post to Slack**.

3. **Add the general documentation sticky note**
   - Create a **Sticky Note** node.
   - Place it on the left side of the canvas.
   - Paste the overview/setup text describing:
     - Dropbox to DocuPipe to Slack
     - Setup steps
     - Requirements
     - Links:
       - `https://docupipe.ai`
       - `https://app.docupipe.ai/settings/general`
       - `https://www.npmjs.com/package/n8n-nodes-docupipe`

4. **Add a second sticky note for Scenario 1**
   - Create another **Sticky Note**.
   - Title it conceptually as Scenario 1.
   - Describe that the flow polls Dropbox every minute, filters recent files, downloads them, and uploads them to DocuPipe asynchronously.

5. **Add a third sticky note for Scenario 2**
   - Create another **Sticky Note**.
   - Describe that DocuPipe webhook completion triggers retrieval of extracted data, formatting, and Slack posting.

## Build Scenario 1

6. **Create the schedule trigger**
   - Add **Schedule Trigger**.
   - Name it **Check Every Minute**.
   - Configure the rule as:
     - Interval
     - Every **1 minute**

7. **Create the Dropbox folder listing node**
   - Add a **Dropbox** node.
   - Name it **List Dropbox Folder**.
   - Configure:
     - **Resource:** Folder
     - **Operation:** List
     - **Path:** set the Dropbox folder you want to monitor, for example `/incoming-docs`
   - Create or select **Dropbox OAuth2** credentials.

8. **Connect the trigger to the listing node**
   - Connect **Check Every Minute** → **List Dropbox Folder**.

9. **Create the filter node**
   - Add an **If** node.
   - Name it **Filter New Files**.
   - Use **AND** combinator.
   - Add condition 1:
     - Left value: `={{ $json['.tag'] }}`
     - Operator: equals
     - Right value: `file`
   - Add condition 2:
     - Left value: `={{ $json.server_modified }}`
     - Operator: after
     - Right value: `={{ $now.minus({minutes: 2}).toISO() }}`
   - Keep strict validation enabled if available in your n8n version.

10. **Connect the Dropbox list node to the filter**
    - Connect **List Dropbox Folder** → **Filter New Files**.

11. **Create the Dropbox download node**
    - Add another **Dropbox** node.
    - Name it **Download File from Dropbox**.
    - Configure:
      - **Resource:** File
      - **Operation:** Download
      - **Path:** `={{ $json.path_display }}`
    - Reuse the same **Dropbox OAuth2** credentials.

12. **Connect the true branch of the filter**
    - Connect **Filter New Files** true output → **Download File from Dropbox**.
    - Leave the false output unconnected unless you want logging or audit handling.

13. **Create the DocuPipe upload node**
    - Add a **DocuPipe** node.
    - Name it **Upload File & Extract Data**.
    - Configure:
      - **Resource:** Extraction
      - **Operation:** Upload and Extract
      - **Input Mode:** Binary
      - **Binary Property Name:** `data`
      - **Schema ID:** choose your DocuPipe extraction schema
    - Create or select **DocuPipe API** credentials using your API key from:
      - `https://app.docupipe.ai/settings/general`

14. **Connect download to DocuPipe upload**
    - Connect **Download File from Dropbox** → **Upload File & Extract Data**.

15. **Verify binary property compatibility**
    - Run a test with a sample file.
    - Confirm the Dropbox node outputs the file in binary property `data`.
    - If your n8n/Dropbox behavior differs, adjust the binary property name in the DocuPipe node accordingly.

## Build Scenario 2

16. **Create the DocuPipe trigger**
    - Add a **DocuPipe Trigger** node.
    - Name it **Extraction Complete**.
    - Configure:
      - **Event:** `standardization.processed.success`
    - Use the same **DocuPipe API** credentials.
    - Save the workflow so the webhook can be registered properly.

17. **Create the DocuPipe result retrieval node**
    - Add a **DocuPipe** node.
    - Name it **Get Extracted Data**.
    - Configure:
      - **Resource:** Extraction
      - **Operation:** Get Result
      - **Standardization ID:** `={{ $json.standardizationId }}`
    - Use the same **DocuPipe API** credentials.

18. **Connect the DocuPipe trigger**
    - Connect **Extraction Complete** → **Get Extracted Data**.

19. **Create the code formatting node**
    - Add a **Code** node.
    - Name it **Format Slack Message**.
    - Set language to JavaScript.
    - Paste this logic:

   ```javascript
   const data = $input.first().json;
   const result = data.result || {};
   const documentName = data.documentName || 'Document';

   let message = `*DocuPipe Extraction Complete* :white_check_mark:\n`;
   message += `*Document:* ${documentName}\n\n`;

   for (const [key, value] of Object.entries(result)) {
     let displayValue;
     if (Array.isArray(value)) {
       displayValue = value.map((item) => {
         if (typeof item === 'object') {
           return Object.values(item).join(' | ');
         }
         return item;
       }).join('\n   ');
       message += `*${key}:*\n   ${displayValue}\n`;
     } else if (typeof value === 'object' && value !== null) {
       displayValue = JSON.stringify(value, null, 2);
       message += `*${key}:* \`${displayValue}\`\n`;
     } else {
       message += `*${key}:* ${value}\n`;
     }
   }

   return [{ json: { formattedMessage: message, documentName } }];
   ```

20. **Connect result retrieval to formatter**
    - Connect **Get Extracted Data** → **Format Slack Message**.

21. **Create the Set node**
    - Add a **Set** node.
    - Name it **Set Channel & Message**.
    - Create these fields:
      - `text` as string: `={{ $json.formattedMessage }}`
      - `subject` as string: `=DocuPipe: {{ $json.documentName }} extracted`
    - Note: `subject` is not consumed later, but reproduce it to match the original workflow.

22. **Connect formatter to Set**
    - Connect **Format Slack Message** → **Set Channel & Message**.

23. **Create the Slack node**
    - Add a **Slack** node.
    - Name it **Post Results to Slack**.
    - Configure:
      - Operation to send a message to a channel
      - **Text:** `={{ $json.text }}`
      - **Destination type/select:** Channel
      - **Channel ID:** choose your target Slack channel
    - Create or select **Slack OAuth2** credentials.

24. **Connect the Set node to Slack**
    - Connect **Set Channel & Message** → **Post Results to Slack**.

## Credential Setup

25. **Configure Dropbox OAuth2**
    - Authenticate the Dropbox account with access to the monitored folder.
    - Ensure the app has read access for listing and downloading files.

26. **Configure DocuPipe API credentials**
    - Use the API key from your DocuPipe account settings.
    - Ensure the selected schema exists and is active.

27. **Configure Slack OAuth2**
    - Authorize a Slack app/bot that can post messages.
    - Ensure the bot is a member of the target channel.

## Activation and Testing

28. **Test Scenario 1 manually**
    - Put a supported file into the monitored Dropbox folder.
    - Execute the first branch manually or wait for the next minute.
    - Confirm the file is listed, passes the filter, downloads, and uploads to DocuPipe.

29. **Test Scenario 2 webhook handling**
    - Activate the workflow.
    - Ensure DocuPipe can reach the n8n webhook URL.
    - After extraction completes, confirm the trigger fires and the result is fetched.

30. **Validate the Slack output**
    - Check that the message contains:
      - Success header
      - Document name
      - Extracted fields
    - If formatting is not ideal for your schema, adjust the Code node.

31. **Activate the workflow**
    - Once both scenarios work, activate the workflow permanently.

## Important reproduction constraints

32. **Set the folder path explicitly**
    - The original JSON leaves the Dropbox list path blank.
    - You must provide the folder path manually.

33. **Select the DocuPipe schema**
    - The schema field is empty in the original JSON.
    - You must choose a valid extraction schema.

34. **Select the Slack channel**
    - The Slack channel ID is empty in the original JSON.
    - You must choose the destination channel.

35. **Keep the workflow active for webhooks**
    - The DocuPipe trigger only works correctly when the workflow is active and the production webhook URL is registered/reachable.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DocuPipe main site | https://docupipe.ai |
| DocuPipe API key settings page | https://app.docupipe.ai/settings/general |
| DocuPipe community node package for n8n | https://www.npmjs.com/package/n8n-nodes-docupipe |
| The workflow uses two connected scenarios: scheduled Dropbox polling and webhook-based result notification | Operational design |
| Extraction is asynchronous; the upload branch does not produce Slack output directly | Architectural behavior |
| The workflow only handles successful DocuPipe completion events | Event scope |
| The Slack message formatter may need customization depending on the extraction schema structure | Maintenance note |