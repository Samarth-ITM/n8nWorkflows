Classify invoices and route them to Google Drive folders with easybits and Slack

https://n8nworkflows.xyz/workflows/classify-invoices-and-route-them-to-google-drive-folders-with-easybits-and-slack-14318


# Classify invoices and route them to Google Drive folders with easybits and Slack

# 1. Workflow Overview

This workflow receives an uploaded invoice-like document through an n8n-hosted web form, sends the file to an easybits Extractor pipeline for AI-based classification, and routes the original file into a category-specific Google Drive folder. If the AI classification is low-confidence or does not match any expected category, the file is sent to a review folder and a Slack notification is posted for manual handling.

Typical use cases:
- Sorting incoming invoices by business domain
- Reducing manual document triage
- Flagging uncertain classifications for human review
- Creating a lightweight AI-assisted intake flow for finance or operations teams

Supported classification targets in the current design:
- `medical_invoice`
- `restaurant_invoice`
- `hotel_invoice`
- `trades_invoice`
- `telecom_invoice`

## 1.1 Input Reception

The workflow begins with a hosted form that accepts a file upload. The uploaded binary is then converted into a text representation suitable for API submission.

## 1.2 File Encoding for API Submission

The uploaded file is extracted into a base64 string, then transformed into a full data URI including MIME type so it can be sent to easybits in the expected format.

## 1.3 AI Classification with easybits

The workflow calls the easybits Extractor API using an HTTP Request node authenticated with Bearer token credentials. The API returns structured classification output including document class and confidence score.

## 1.4 Result Normalization and File Reattachment

The relevant AI output fields are mapped into simpler workflow fields, then merged back together with the original uploaded binary so downstream upload nodes have both metadata and file content available.

## 1.5 Confidence-Based Routing

A confidence threshold check determines whether the classification is trusted. High-confidence results proceed to category routing; low-confidence results are sent directly to manual review handling.

## 1.6 Category-Based Storage Routing

A Switch node routes high-confidence documents to one of several Google Drive upload nodes based on the returned document category. If the returned category is outside the supported list, the fallback path sends the file to review.

## 1.7 Review Handling and Alerting

Uncertain or unmatched documents are uploaded to a dedicated Google Drive review folder, then a Slack message is sent with the classification result, confidence score, original filename, and a Drive link.

---

# 2. Block-by-Block Analysis

## Block 1 — Form Intake

### Overview
This block exposes a web form that accepts an uploaded document. It is the workflow’s single entry point and provides the original binary file used throughout the process.

### Nodes Involved
- On form submission
- Sticky Note
- Sticky Note4

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that starts the workflow when a user submits the hosted form.
- **Configuration choices:**
  - Form title is set to **Image Upload**
  - One form field is defined:
    - Type: `file`
    - Label: `image`
- **Key expressions or variables used:**
  - Downstream nodes reference the uploaded binary as:
    - `$('On form submission').first().binary.image`
    - file name: `$('On form submission').first().binary.image.fileName`
    - MIME type: `$('On form submission').first().binary.image.mimeType`
- **Input and output connections:**
  - No input; this is the workflow trigger
  - Outputs to:
    - `Extract from File`
    - `Attach Original File` (second merge input)
- **Version-specific requirements:**
  - Uses node type version `2.5`
  - Requires a recent n8n version with Form Trigger support enabled
- **Edge cases or potential failure types:**
  - User submits no file
  - Uploaded file exceeds form or instance size limits
  - Browser upload interruption
  - MIME type may not be reliably set by client/browser for uncommon files
  - The workflow documentation says PDF, PNG, and JPEG are supported, but the form itself does not enforce extension filtering in this JSON
- **Sub-workflow reference:** None

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the upload form purpose
- **Key content:** Accepts file upload via web form; supports PDF, PNG, JPEG
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Large overview/setup note for the full workflow
- **Key content:** End-to-end explanation, setup steps for easybits, Google Drive, Slack, testing guidance
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## Block 2 — Binary Extraction and Data URI Construction

### Overview
This block converts the uploaded binary file into a base64 string and then wraps it into a valid data URI. The resulting value is what gets sent to the easybits API.

### Nodes Involved
- Extract from File
- Edit Fields
- Sticky Note1
- Sticky Note2

### Node Details

#### Extract from File
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Converts binary file content into a regular JSON property.
- **Configuration choices:**
  - Operation: `binaryToPropery`
  - Source binary field: `image`
  - Output becomes a JSON field named `data`
- **Key expressions or variables used:**
  - Reads from binary property `image`
  - Produces `$json.data`
- **Input and output connections:**
  - Input from `On form submission`
  - Output to `Edit Fields`
- **Version-specific requirements:**
  - Uses node version `1.1`
  - Exact operation label may vary slightly across n8n releases
- **Edge cases or potential failure types:**
  - Missing binary field `image`
  - Corrupted file data
  - Large file causing memory pressure during conversion
- **Sub-workflow reference:** None

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Rewrites the extracted base64 string into a data URI string.
- **Configuration choices:**
  - Creates one field:
    - `data` as string
  - Value is built dynamically from MIME type and extracted base64 payload
- **Key expressions or variables used:**
  - `=data:{{ $('On form submission').first().binary.image.mimeType }};base64,{{ $json.data }}`
- **Input and output connections:**
  - Input from `Extract from File`
  - Output to `easybits Extractor (Classification)`
- **Version-specific requirements:**
  - Uses Set node version `3.4`
- **Edge cases or potential failure types:**
  - If MIME type is missing, the generated data URI may be malformed
  - If `$json.data` is empty, the API request will still be sent but likely fail or classify incorrectly
  - If the uploaded file is not actually supported by easybits, the API may reject it
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Explains binary-to-base64 conversion
- **Key content:** Converts uploaded binary file into base64 stored in `data`
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Explains MIME-aware data URI assembly
- **Key content:** Prepends MIME type dynamically to base64 content
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## Block 3 — easybits Classification

### Overview
This block sends the prepared file payload to easybits for classification. The workflow expects the API response to include a nested `data` object with `document_class` and `confidence_score`.

### Nodes Involved
- easybits Extractor (Classification)
- Sticky Note5

### Node Details

#### easybits Extractor (Classification)
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the easybits Extractor API pipeline endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID`
  - Authentication: predefined credential type
  - Credential type: `httpBearerAuth`
  - Request body format: JSON
  - Body:
    - `files` is an array containing the data URI string from the previous node
- **Key expressions or variables used:**
  - Request body includes:
    - `{{ $json.data }}`
- **Input and output connections:**
  - Input from `Edit Fields`
  - Output to `Parse Result`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.3`
  - Requires a Bearer Auth credential configured in n8n
- **Edge cases or potential failure types:**
  - Invalid or missing Bearer token
  - Pipeline ID placeholder not replaced
  - API timeout or network failure
  - Unexpected response shape from easybits
  - Request body too large for API limits
  - 4xx/5xx errors if the pipeline is misconfigured
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes the API submission step
- **Key content:** POSTs data URI to easybits Extractor API
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## Block 4 — Result Parsing and Original File Merge

### Overview
This block extracts the AI response fields needed for routing, then combines them with the original uploaded binary. This is necessary because the HTTP/API path only carries JSON, while Google Drive upload nodes require the original binary file.

### Nodes Involved
- Parse Result
- Attach Original File
- Sticky Note3
- Sticky Note6

### Node Details

#### Parse Result
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes the easybits response into simpler top-level fields.
- **Configuration choices:**
  - Creates:
    - `document_type` from `data.document_class`
    - `confidence_score` from `data.confidence_score`
- **Key expressions or variables used:**
  - `={{ $json.data.document_class }}`
  - `={{ $json.data.confidence_score }}`
- **Input and output connections:**
  - Input from `easybits Extractor (Classification)`
  - Output to `Attach Original File`
- **Version-specific requirements:**
  - Uses Set node version `3.4`
- **Edge cases or potential failure types:**
  - If API response does not contain `data.document_class`, `document_type` becomes empty/undefined
  - If confidence is returned as text rather than a number, type coercion issues may occur
  - There is a notable configuration oddity: the field name appears as `=document_type` in the JSON. In the live workflow, it is intended to produce `document_type`. If imported exactly and n8n preserves the leading `=`, downstream expressions like `$json.document_type` would fail. This should be verified after import.
- **Sub-workflow reference:** None

#### Attach Original File
- **Type and technical role:** `n8n-nodes-base.merge`  
  Merges the parsed classification result with the original form submission item.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
  - Input 1 receives parsed JSON
  - Input 2 receives original form submission including binary file
- **Key expressions or variables used:** None directly in configuration
- **Input and output connections:**
  - Input 1 from `Parse Result`
  - Input 2 from `On form submission`
  - Output to `Confidence Check`
- **Version-specific requirements:**
  - Uses Merge node version `3.2`
- **Edge cases or potential failure types:**
  - Combine-by-position assumes both branches emit items in the same order and quantity
  - If one branch fails or emits zero items, merge output may be incomplete or absent
  - If future modifications introduce multiple files/items, positional merge must be revalidated carefully
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Explains result extraction
- **Key content:** Extracts `document_type` and `confidence_score`
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Explains the merge rationale
- **Key content:** Combines classification JSON and original binary using Combine by Position
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## Block 5 — Confidence Gating

### Overview
This block decides whether the AI result is trusted enough to route automatically. Documents with confidence above 0.5 continue to category routing; all others go straight to review.

### Nodes Involved
- Confidence Check
- Sticky Note7

### Node Details

#### Confidence Check
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates a numeric threshold on the returned confidence score.
- **Configuration choices:**
  - Condition:
    - `confidence_score > 0.5`
- **Key expressions or variables used:**
  - `={{ $json.confidence_score }}`
- **Input and output connections:**
  - Input from `Attach Original File`
  - True output to `Category Router`
  - False output to `Upload to Review Folder`
- **Version-specific requirements:**
  - Uses IF node version `2.3`
- **Edge cases or potential failure types:**
  - Missing `confidence_score`
  - Non-numeric confidence value
  - Threshold may be too permissive or too strict depending on the easybits model behavior
  - Values equal to exactly `0.5` go to review, not auto-routing
- **Sub-workflow reference:** None

#### Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents threshold behavior
- **Key content:** `> 0.5` routes to category router; `≤ 0.5` routes to review
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## Block 6 — Category Routing to Google Drive

### Overview
This block routes high-confidence documents into category-specific Google Drive folders. If the category returned by easybits is not one of the expected labels, the switch fallback sends the file to the review folder.

### Nodes Involved
- Category Router
- Upload to Medical Folder
- Upload to Restaurant Folder
- Upload to Hotel Folder
- Upload to Trades Folder
- Upload to Telecom Folder
- Upload to Review Folder
- Sticky Note8
- Sticky Note9
- Sticky Note10

### Node Details

#### Category Router
- **Type and technical role:** `n8n-nodes-base.switch`  
  Dispatches each item to one of several outputs based on the `document_type` string.
- **Configuration choices:**
  - Strict string equality checks for:
    - `medical_invoice`
    - `restaurant_invoice`
    - `hotel_invoice`
    - `trades_invoice`
    - `telecom_invoice`
  - Fallback output enabled as `extra`
- **Key expressions or variables used:**
  - `={{ $json.document_type }}`
- **Input and output connections:**
  - Input from `Confidence Check` true branch
  - Outputs to:
    - `Upload to Medical Folder`
    - `Upload to Restaurant Folder`
    - `Upload to Hotel Folder`
    - `Upload to Trades Folder`
    - `Upload to Telecom Folder`
    - fallback to `Upload to Review Folder`
- **Version-specific requirements:**
  - Uses Switch node version `3.4`
  - Conditions use versioned rule format with strict validation
- **Edge cases or potential failure types:**
  - Category label mismatch due to casing, spelling, spacing, or prompt drift
  - Missing `document_type`
  - If `Parse Result` accidentally outputs a field named `=document_type` instead of `document_type`, all items would fall through to review
- **Sub-workflow reference:** None

#### Upload to Medical Folder
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads binary file to the Medical folder in Google Drive.
- **Configuration choices:**
  - File name comes from original upload filename
  - Drive: `My Drive`
  - Folder ID placeholder: `YOUR_MEDICAL_FOLDER_ID`
  - Binary input field: `image`
- **Key expressions or variables used:**
  - `={{ $('On form submission').first().binary.image.fileName }}`
- **Input and output connections:**
  - Input from `Category Router`
  - No downstream node
- **Version-specific requirements:**
  - Uses Google Drive node version `3`
  - Requires Google Drive OAuth2 credential
- **Edge cases or potential failure types:**
  - Invalid folder ID
  - Missing Google credential
  - File upload permission issues
  - Duplicate filenames in Drive may create ambiguity depending on Drive behavior
  - Binary field `image` must still exist after merge
- **Sub-workflow reference:** None

#### Upload to Restaurant Folder
- Same functional pattern as Medical node, with folder ID placeholder `YOUR_RESTAURANT_FOLDER_ID`
- **Input:** `Category Router`
- **Output:** none
- **Edge cases:** same as above

#### Upload to Hotel Folder
- Same functional pattern as Medical node, with folder ID placeholder `YOUR_HOTEL_FOLDER_ID`
- **Input:** `Category Router`
- **Output:** none
- **Edge cases:** same as above

#### Upload to Trades Folder
- Same functional pattern as Medical node, with folder ID placeholder `YOUR_TRADES_FOLDER_ID`
- **Input:** `Category Router`
- **Output:** none
- **Edge cases:** same as above

#### Upload to Telecom Folder
- Same functional pattern as Medical node, with folder ID placeholder `YOUR_TELECOM_FOLDER_ID`
- **Input:** `Category Router`
- **Output:** none
- **Edge cases:** same as above

#### Upload to Review Folder
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the file into the review folder for low-confidence or unmatched cases.
- **Configuration choices:**
  - File name comes from original upload filename
  - Drive: `My Drive`
  - Folder ID placeholder: `YOUR_NEEDS_REVIEW_FOLDER_ID`
  - Binary input field: `image`
- **Key expressions or variables used:**
  - `={{ $('On form submission').first().binary.image.fileName }}`
- **Input and output connections:**
  - Input from:
    - `Confidence Check` false branch
    - `Category Router` fallback output
  - Output to `Review Message`
- **Version-specific requirements:**
  - Uses Google Drive node version `3`
  - Requires Google Drive OAuth2 credential
- **Edge cases or potential failure types:**
  - Same Google Drive upload risks as other upload nodes
  - If upload fails, Slack notification is not sent because Slack depends on successful upload output
- **Sub-workflow reference:** None

#### Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents category-based switch logic
- **Key content:** Lists five supported categories and fallback to review
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes Google Drive upload block
- **Key content:** Says uploads to Medical folder, but visually this sticky note covers the broader folder upload area; replicate contextually in the summary table for all covered upload nodes
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note10
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes review upload handling
- **Key content:** Uploads unclassified or low-confidence invoices to Needs Review
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## Block 7 — Review Notification

### Overview
This block sends a Slack message after a review-case file has been uploaded to Google Drive. The Slack message includes the classification result, confidence score, filename, and the Google Drive `webViewLink` returned by the upload node.

### Nodes Involved
- Review Message
- Sticky Note11

### Node Details

#### Review Message
- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a message into a Slack channel for manual review.
- **Configuration choices:**
  - Sends a text message
  - Target selection mode: channel
  - Channel ID placeholder: `YOUR_SLACK_CHANNEL_ID`
  - Message body includes:
    - document category
    - confidence
    - filename
    - Google Drive link from the upload response
- **Key expressions or variables used:**
  - `{{ $('Parse Result').item.json.document_type }}`
  - `{{ $('Parse Result').item.json.confidence_score }}`
  - `{{ $('On form submission').first().binary.image.fileName }}`
  - `{{ $json.webViewLink }}`
- **Input and output connections:**
  - Input from `Upload to Review Folder`
  - No downstream node
- **Version-specific requirements:**
  - Uses Slack node version `2.4`
  - Requires Slack API credential with permission to post messages
- **Edge cases or potential failure types:**
  - Invalid Slack credential or missing `chat:write`
  - Bot not invited to the target channel
  - `webViewLink` missing if Google Drive response format changes
  - If `Parse Result` did not generate expected fields, Slack message contents may be blank
- **Sub-workflow reference:** None

#### Sticky Note11
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Describes Slack alert content
- **Key content:** Sends message to review channel with category, confidence, filename, and Drive link
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Workflow entry point; receives uploaded file from hosted form |  | Extract from File; Attach Original File | ## 📋 Form Upload<br>Accepts a file upload via a **web form**. Supports **PDF, PNG, and JPEG**.<br># 📄 Invoice Classification Workflow\n(powered by easybits)\n\n## What This Workflow Does\nUpload a document (PDF, PNG, JPEG) via a web form and let **easybits Extractor** classify it into one of your defined categories. Based on the classification result and a confidence score, the document is automatically sorted into the correct **Google Drive** folder. Low-confidence or unrecognized documents are flagged for manual review via **Slack**.\n\n## How It Works\n1. User uploads a file through the hosted web form\n2. The binary file is converted to base64 and sent to easybits\n3. easybits returns a **document_type** and **confidence_score**\n4. The classification result is merged with the original file binary\n5. If confidence > 0.5 → routed to the matching Google Drive folder\n6. If confidence ≤ 0.5 or no category match → uploaded to **Needs Review** folder + Slack alert\n\n**Supported categories:**\n`medical_invoice` · `restaurant_invoice` · `hotel_invoice` · `trades_invoice` · `telecom_invoice`\n\n---\n\n## Setup Guide\n\n### 1. Set Up Your easybits Extractor Pipeline\n1. Go to **extractor.easybits.tech** and create a new pipeline\n2. Add two fields to the mapping: **document_class** and **confidence_score**\n3. In each field's description, paste the corresponding classification or confidence prompt that tells the model how to analyze the document\n4. The classification prompt should return exactly one category label – or `null` if uncertain\n5. The confidence prompt should return a decimal number between 0.0 and 1.0\n6. Save & test the pipeline, then copy your **Pipeline ID** and **API Key**\n\n### 2. Set Up Google Drive\n1. Create a folder in Google Drive for each category: **Medical**, **Restaurant**, **Hotel**, **Trades**, **Telecom**, and **Needs Review**\n2. In n8n, go to **Settings → Credentials** and create a **Google Drive OAuth2** credential\n3. This requires a **Client ID** and **Client Secret** from the Google Cloud Console (APIs & Services → Credentials → OAuth 2.0 Client ID)\n4. Make sure the **Google Drive API** is enabled in your Google Cloud project\n5. Open each of the 6 Google Drive upload nodes in this workflow and select the correct target folder\n\n### 3. Set Up Slack\n1. In n8n, go to **Settings → Credentials** and create a **Slack API** credential\n2. You'll need a Slack Bot Token – create a Slack App at **api.slack.com/apps**, add the `chat:write` scope, and install it to your workspace\n3. Create a channel for review notifications (e.g. `#n8n-invoice-review`)\n4. Invite the bot to that channel\n5. Open the **Review Message** node and select the correct channel\n\n### 4. Connect the easybits Node\n1. Open the **easybits Extractor (Classification)** node\n2. Replace the pipeline URL with your own: `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID`\n3. Create a **Bearer Auth** credential using your easybits API Key and assign it to the node\n\n### 5. Activate & Test\n1. Click **Active** in the top-right corner of n8n\n2. Open the form URL and upload a test document\n3. Check the execution log to verify the classification result\n4. Confirm the file lands in the correct Google Drive folder\n5. Test with an unrecognized document to verify the Slack notification fires |
| Extract from File | n8n-nodes-base.extractFromFile | Converts uploaded binary file into base64 JSON field | On form submission | Edit Fields | ## 📄 Extract to Base64<br>Converts the uploaded **binary file** into a base64-encoded string stored in `data`. |
| Edit Fields | n8n-nodes-base.set | Builds data URI from MIME type and base64 payload | Extract from File | easybits Extractor (Classification) | ## 🔗 Build Data URI<br>Dynamically reads the **MIME type** from the uploaded file and prepends it as a base64 data URI. |
| easybits Extractor (Classification) | n8n-nodes-base.httpRequest | Sends file to easybits classification API | Edit Fields | Parse Result | ## 🚀 Send to easybits<br>POSTs the data URI to the **easybits Extractor API** pipeline for processing. |
| Parse Result | n8n-nodes-base.set | Extracts normalized classification fields from API response | easybits Extractor (Classification) | Attach Original File | ## 🔍 Parse Result<br>Extracts **document_type** and **confidence_score** from the easybits API response into separate fields for downstream routing. |
| Attach Original File | n8n-nodes-base.merge | Recombines parsed result with original binary upload | Parse Result; On form submission | Confidence Check | ## 🔗 Attach Original File<br>Merges the classification result (JSON) with the original binary file from the form upload using **Combine by Position**, so downstream nodes have both the data and the file. |
| Confidence Check | n8n-nodes-base.if | Tests whether confidence exceeds routing threshold | Attach Original File | Category Router; Upload to Review Folder | ## ✅ Confidence Check<br>Routes based on the **confidence_score**:<br>- **> 0.5** → Category Router<br>- **≤ 0.5** → Needs Review folder + Slack notification |
| Category Router | n8n-nodes-base.switch | Routes trusted classifications by document type | Confidence Check | Upload to Medical Folder; Upload to Restaurant Folder; Upload to Hotel Folder; Upload to Trades Folder; Upload to Telecom Folder; Upload to Review Folder | ## 🗂️ Category Router<br>Routes the invoice to the correct Google Drive folder based on **document_type**:<br><br>1. `medical_invoice`<br>2. `restaurant_invoice`<br>3. `hotel_invoice`<br>4. `trades_invoice`<br>5. `telecom_invoice`<br>6. Fallback → Needs Review |
| Upload to Medical Folder | n8n-nodes-base.googleDrive | Uploads medical invoices to Google Drive | Category Router |  | ## 📤 Upload to Google Drive Folders<br>Uploads the invoice to the **Medical** folder in Google Drive. |
| Upload to Restaurant Folder | n8n-nodes-base.googleDrive | Uploads restaurant invoices to Google Drive | Category Router |  | ## 📤 Upload to Google Drive Folders<br>Uploads the invoice to the **Medical** folder in Google Drive. |
| Upload to Hotel Folder | n8n-nodes-base.googleDrive | Uploads hotel invoices to Google Drive | Category Router |  | ## 📤 Upload to Google Drive Folders<br>Uploads the invoice to the **Medical** folder in Google Drive. |
| Upload to Trades Folder | n8n-nodes-base.googleDrive | Uploads trades invoices to Google Drive | Category Router |  | ## 📤 Upload to Google Drive Folders<br>Uploads the invoice to the **Medical** folder in Google Drive. |
| Upload to Telecom Folder | n8n-nodes-base.googleDrive | Uploads telecom invoices to Google Drive | Category Router |  | ## 📤 Upload to Google Drive Folders<br>Uploads the invoice to the **Medical** folder in Google Drive. |
| Upload to Review Folder | n8n-nodes-base.googleDrive | Uploads low-confidence or unmatched files to review folder | Confidence Check; Category Router | Review Message | ## ⚠️ Upload to Review Folder<br>Uploads unclassified or low-confidence invoices to the **Needs Review** folder in Google Drive. |
| Review Message | n8n-nodes-base.slack | Sends Slack alert for manual review | Upload to Review Folder |  | ## 💬 Slack Notification<br>Sends a message to `#n8n-invoice-review` with the **document type**, **confidence score**, **file name**, and a direct **Google Drive link** so the team can manually classify the invoice. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for form upload block |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for base64 conversion block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for data URI block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for parsing block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Full workflow overview and setup instructions |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation for easybits API block |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation for merge block |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation for confidence block |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation for category routing block |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation for Drive upload area |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual documentation for review upload block |  |  |  |
| Sticky Note11 | n8n-nodes-base.stickyNote | Visual documentation for Slack notification block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Invoice Classification Workflow (powered by easybits)`.
   - Keep the default execution order unless you have a reason to change it.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Set the form title to `Image Upload`
   - Add one form field:
     - Field type: `File`
     - Field label: `image`
   - This is the workflow entry point.

3. **Add an Extract from File node**
   - Node type: **Extract from File**
   - Operation: convert binary to property
   - Set the source binary field to `image`
   - This should output a JSON property containing the base64 content.

4. **Connect `On form submission` → `Extract from File`**

5. **Add a Set node named `Edit Fields`**
   - Node type: **Set**
   - Add a string field named `data`
   - Set its value to:
     ```text
     =data:{{ $('On form submission').first().binary.image.mimeType }};base64,{{ $json.data }}
     ```
   - This builds a full data URI including MIME type.

6. **Connect `Extract from File` → `Edit Fields`**

7. **Create Bearer authentication for easybits**
   - Go to **Credentials**
   - Create credential type: **HTTP Bearer Auth**
   - Paste your easybits API key as the bearer token
   - Save it with a recognizable name

8. **Add an HTTP Request node named `easybits Extractor (Classification)`**
   - Method: `POST`
   - URL:
     ```text
     https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID
     ```
   - Replace `YOUR_PIPELINE_ID` with your actual pipeline ID
   - Authentication:
     - Use predefined credential type
     - Select your **HTTP Bearer Auth** credential
   - Enable request body
   - Body content type: JSON
   - Set the JSON body to:
     ```json
     {
       "files": [
         "{{ $json.data }}"
       ]
     }
     ```
   - In expression mode, ensure the data URI is inserted dynamically.

9. **Connect `Edit Fields` → `easybits Extractor (Classification)`**

10. **Add a Set node named `Parse Result`**
    - Node type: **Set**
    - Add a string field named `document_type`
      - Value:
        ```text
        ={{ $json.data.document_class }}
        ```
    - Add a number field named `confidence_score`
      - Value:
        ```text
        ={{ $json.data.confidence_score }}
        ```
    - Important: verify the field name is exactly `document_type`, not `=document_type`.

11. **Connect `easybits Extractor (Classification)` → `Parse Result`**

12. **Add a Merge node named `Attach Original File`**
    - Node type: **Merge**
    - Mode: `Combine`
    - Combine by: `Position`
    - Input 1 will receive parsed API output
    - Input 2 will receive original form submission

13. **Connect `Parse Result` → `Attach Original File`**
14. **Connect `On form submission` → `Attach Original File`**
   - Make sure this is the second input branch.

15. **Add an IF node named `Confidence Check`**
    - Node type: **IF**
    - Create one numeric condition:
      - Left value:
        ```text
        ={{ $json.confidence_score }}
        ```
      - Operator: `greater than`
      - Right value: `0.5`
    - True branch = trusted classification
    - False branch = manual review

16. **Connect `Attach Original File` → `Confidence Check`**

17. **Add a Switch node named `Category Router`**
    - Node type: **Switch**
    - Use rule-based routing
    - Add five outputs with string equality checks against `={{ $json.document_type }}`:
      1. equals `medical_invoice`
      2. equals `restaurant_invoice`
      3. equals `hotel_invoice`
      4. equals `trades_invoice`
      5. equals `telecom_invoice`
    - Enable fallback output
    - Fallback should be used for unknown or null categories

18. **Connect the true output of `Confidence Check` → `Category Router`**

19. **Create Google Drive OAuth2 credentials**
    - In **Credentials**, create **Google Drive OAuth2 API**
    - Use your Google Cloud OAuth Client ID and Client Secret
    - Ensure the **Google Drive API** is enabled in Google Cloud
    - Authorize the account that owns or can write to the destination folders

20. **Create Google Drive folders**
    - In Google Drive, create:
      - Medical
      - Restaurant
      - Hotel
      - Trades
      - Telecom
      - Needs Review
    - Copy each folder ID from the folder URL

21. **Add a Google Drive node named `Upload to Medical Folder`**
    - Operation: upload file
    - Drive: `My Drive`
    - Folder ID: your Medical folder ID
    - Binary input field name: `image`
    - File name:
      ```text
      ={{ $('On form submission').first().binary.image.fileName }}
      ```

22. **Add a Google Drive node named `Upload to Restaurant Folder`**
    - Same configuration as above
    - Folder ID: Restaurant folder ID

23. **Add a Google Drive node named `Upload to Hotel Folder`**
    - Same configuration
    - Folder ID: Hotel folder ID

24. **Add a Google Drive node named `Upload to Trades Folder`**
    - Same configuration
    - Folder ID: Trades folder ID

25. **Add a Google Drive node named `Upload to Telecom Folder`**
    - Same configuration
    - Folder ID: Telecom folder ID

26. **Add a Google Drive node named `Upload to Review Folder`**
    - Same configuration pattern
    - Folder ID: Needs Review folder ID
    - Binary input field name: `image`
    - File name:
      ```text
      ={{ $('On form submission').first().binary.image.fileName }}
      ```

27. **Connect `Category Router` outputs**
    - Output 1 → `Upload to Medical Folder`
    - Output 2 → `Upload to Restaurant Folder`
    - Output 3 → `Upload to Hotel Folder`
    - Output 4 → `Upload to Trades Folder`
    - Output 5 → `Upload to Telecom Folder`
    - Fallback output → `Upload to Review Folder`

28. **Connect low-confidence items to review**
    - Connect the false output of `Confidence Check` → `Upload to Review Folder`

29. **Create Slack credentials**
    - In **Credentials**, create **Slack API**
    - Use a Bot Token
    - Ensure the Slack app has at least `chat:write`
    - Install the app into your workspace
    - Invite the bot to the review channel

30. **Add a Slack node named `Review Message`**
    - Operation: send message
    - Select target type: `Channel`
    - Choose or paste your target channel ID
    - Set message text to:
      ```text
      =🔍 *Invoice needs manual classification*
      - Document Category: {{ $('Parse Result').item.json.document_type }}
      - Confidence: {{ $('Parse Result').item.json.confidence_score }}
      - File: {{ $('On form submission').first().binary.image.fileName }}
      - Google Drive: {{ $json.webViewLink }}
      ```

31. **Connect `Upload to Review Folder` → `Review Message`**

32. **Optionally add sticky notes for maintainability**
    - Form Upload
    - Extract to Base64
    - Build Data URI
    - Send to easybits
    - Parse Result
    - Attach Original File
    - Confidence Check
    - Category Router
    - Upload to Review Folder
    - Slack Notification
    - Full setup note for credentials and testing

33. **Set up the easybits pipeline externally**
    - In easybits, create a pipeline
    - Configure fields:
      - `document_class`
      - `confidence_score`
    - Ensure the classification prompt returns one of:
      - `medical_invoice`
      - `restaurant_invoice`
      - `hotel_invoice`
      - `trades_invoice`
      - `telecom_invoice`
      - or `null` when uncertain
    - Ensure confidence prompt returns a decimal between `0.0` and `1.0`

34. **Test the workflow**
    - Open the hosted form URL
    - Upload a known invoice
    - Confirm:
      - easybits returns data in expected structure
      - `document_type` and `confidence_score` are populated
      - file uploads to the proper Google Drive folder
    - Test a low-confidence or unknown document
    - Confirm:
      - file uploads to `Needs Review`
      - Slack message posts successfully

35. **Activate the workflow**
    - Once all credentials and folder IDs are verified, activate the workflow for production use.

### Expected Input / Output Behavior

- **Input**
  - One uploaded file in form field `image`
  - Expected practical file types: PDF, PNG, JPEG

- **Intermediate easybits request**
  - JSON body containing:
    - `files: [data-uri-string]`

- **Expected easybits response shape**
  - A `data` object containing at least:
    - `document_class`
    - `confidence_score`

- **Downstream upload requirement**
  - Binary field `image` must be preserved via the merge node

### Important Constraints

- Replace all placeholders:
  - `YOUR_PIPELINE_ID`
  - `YOUR_MEDICAL_FOLDER_ID`
  - `YOUR_RESTAURANT_FOLDER_ID`
  - `YOUR_HOTEL_FOLDER_ID`
  - `YOUR_TRADES_FOLDER_ID`
  - `YOUR_TELECOM_FOLDER_ID`
  - `YOUR_NEEDS_REVIEW_FOLDER_ID`
  - `YOUR_SLACK_CHANNEL_ID`
- Verify the `Parse Result` node field name is `document_type`
- If you add more categories, you must update:
  - easybits prompting
  - the Switch node rules
  - Google Drive destination nodes

### Sub-workflow Setup

This workflow does **not** use any sub-workflow or Execute Workflow node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| easybits Extractor pipeline builder | `https://extractor.easybits.tech` |
| Slack app creation and bot token management | `https://api.slack.com/apps` |
| Google Cloud Console for OAuth client setup and enabling Drive API | Google Cloud Console |
| Review flow behavior | Low-confidence (`<= 0.5`) and unmatched categories are uploaded to Needs Review and generate a Slack alert |
| Supported categories in current workflow | `medical_invoice`, `restaurant_invoice`, `hotel_invoice`, `trades_invoice`, `telecom_invoice` |
| Important implementation check | Confirm that the parsed field name is `document_type` and not `=document_type` after import |
| Operational dependency | Slack alerts only happen after `Upload to Review Folder` succeeds, because the Slack node depends on that node’s output |