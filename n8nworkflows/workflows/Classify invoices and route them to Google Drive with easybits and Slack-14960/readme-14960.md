Classify invoices and route them to Google Drive with easybits and Slack

https://n8nworkflows.xyz/workflows/classify-invoices-and-route-them-to-google-drive-with-easybits-and-slack-14960


# Classify invoices and route them to Google Drive with easybits and Slack

# Invoice Classification and Routing Workflow Reference

This document provides a detailed technical analysis of the n8n workflow designed to classify uploaded invoices using the easybits Extractor and route them to specific Google Drive folders based on the classification and confidence score.

---

### 1. Workflow Overview

The primary purpose of this workflow is to automate the sorting of business invoices. It accepts a document via a web form, utilizes AI (via easybits) to identify the invoice type and confidence level, and then distributes the file to the appropriate cloud storage folder. If the AI is uncertain, the workflow flags the document for human review via Slack.

**Logical Blocks:**
- **1.1 Input Reception:** Captures the file via an n8n Form trigger.
- **1.2 AI Processing:** Sends the file to easybits for classification and confidence scoring.
- **1.3 Data Integration:** Merges the AI results back with the original binary file.
- **1.4 Quality Gate:** Evaluates the confidence score to determine if the result is trustworthy.
- **1.5 Routing & Storage:** Routes the file to specific Google Drive folders based on the identified category.
- **1.6 Exception Handling:** Handles low-confidence results by uploading to a review folder and alerting a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Handles the initial entry point where users upload invoice documents.
- **Nodes Involved:** `On form submission`
- **Node Details:**
    - **On form submission (Form Trigger):** 
        - **Technical Role:** Workflow Trigger.
        - **Configuration:** Set up with a form titled "Image Upload" containing one file field named `image`.
        - **Input/Output:** Triggers the workflow; outputs binary data.
        - **Edge Cases:** Unsupported file formats or empty uploads.

#### 2.2 AI Processing
- **Overview:** Extracts classification data from the uploaded document.
- **Nodes Involved:** `easybits Extractor for Classification`, `Parse Result`
- **Node Details:**
    - **easybits Extractor for Classification (Community Node):**
        - **Technical Role:** AI Extraction API call.
        - **Configuration:** Uses a specific Pipeline ID and API Key. It is expected to return `document_class` and `confidence_score`.
        - **Input:** Binary file from the form trigger.
        - **Failure Types:** API timeouts, invalid API keys, or unsupported file types (non-PDF/PNG/JPEG).
    - **Parse Result (Set):**
        - **Technical Role:** Data transformation.
        - **Configuration:** Maps `{{ $json.data.document_class }}` to `document_type` and `{{ $json.data.confidence_score }}` to `confidence_score`.
        - **Input:** easybits response.
        - **Output:** A cleaned JSON object for routing.

#### 2.3 Data Integration
- **Overview:** Re-associates the AI-extracted metadata with the original binary file.
- **Nodes Involved:** `Attach Original File`
- **Node Details:**
    - **Attach Original File (Merge):**
        - **Technical Role:** Data join.
        - **Configuration:** Mode set to "Combine" using "Combine by Position".
        - **Input:** Connects the original `On form submission` node (Input 1) and the `Parse Result` node (Input 0).
        - **Logic:** Ensures the subsequent Google Drive nodes have access to both the file binary and the classification labels.

#### 2.4 Quality Gate
- **Overview:** Determines if the classification is reliable enough for automatic routing.
- **Nodes Involved:** `Confidence Check`
- **Node Details:**
    - **Confidence Check (If):**
        - **Technical Role:** Boolean logic gate.
        - **Configuration:** Checks if `confidence_score` is greater than `0.5`.
        - **Output:** True $\rightarrow$ Category Router; False $\rightarrow$ Upload to Review Folder.

#### 2.5 Routing & Storage
- **Overview:** Distributes the file to the correct destination based on its type.
- **Nodes Involved:** `Category Router`, `Upload to Medical Folder`, `Upload to Restaurant Folder`, `Upload to Hotel Folder`, `Upload to Trades Folder`, `Upload to Telecom Folder`
- **Node Details:**
    - **Category Router (Switch):**
        - **Technical Role:** Multi-path router.
        - **Configuration:** Evaluates `document_type` against five strings: `medical_invoice`, `restaurant_invoice`, `hotel_invoice`, `trades_invoice`, and `telecom_invoice`.
        - **Fallback:** Any unrecognized type is sent to the "extra" output (Review Folder).
    - **Google Drive Upload Nodes:**
        - **Technical Role:** File storage.
        - **Configuration:** Each node targets a specific `folderId`. Uses the expression `{{ $('On form submission').first().binary.image.fileName }}` to preserve the original filename.
        - **Input:** Binary data from the form.

#### 2.6 Exception Handling
- **Overview:** Manages documents that failed the confidence check or the category match.
- **Nodes Involved:** `Upload to Review Folder`, `Review Message`
- **Node Details:**
    - **Upload to Review Folder (Google Drive):**
        - **Technical Role:** Storage for manual review.
        - **Configuration:** Uploads to a dedicated "Needs Review" folder.
    - **Review Message (Slack):**
        - **Technical Role:** Notification.
        - **Configuration:** Sends a formatted message containing the category, confidence score, filename, and the Google Drive `webViewLink` of the uploaded file.
        - **Input:** Output from the Review Folder upload.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On form submission | Form Trigger | Input Reception | None | easybits Extractor, Attach Original File | Accepts a file upload via a web form. Supports PDF, PNG, and JPEG. |
| easybits Extractor... | Community Node | AI Classification | On form submission | Parse Result | POSTs the data URI to the easybits Extractor API pipeline for processing. |
| Parse Result | Set | Data Parsing | easybits Extractor | Attach Original File | Extracts document_type and confidence_score from the easybits API response. |
| Attach Original File | Merge | Binary/JSON Join | On form submission, Parse Result | Confidence Check | Merges classification result (JSON) with the original binary file. |
| Confidence Check | If | Quality Gate | Attach Original File | Category Router, Upload to Review Folder | Routes based on the confidence_score: > 0.5 (Route) or $\le$ 0.5 (Review). |
| Category Router | Switch | Logic Routing | Confidence Check | Various GDrive Nodes | Routes the invoice to the correct Google Drive folder based on document_type. |
| Upload to Medical Folder | Google Drive | File Storage | Category Router | None | Uploads the invoice to the Medical folder in Google Drive. |
| Upload to Restaurant Folder | Google Drive | File Storage | Category Router | None | Uploads the invoice to the Restaurant folder in Google Drive. |
| Upload to Hotel Folder | Google Drive | File Storage | Category Router | None | Uploads the invoice to the Hotel folder in Google Drive. |
| Upload to Trades Folder | Google Drive | File Storage | Category Router | None | Uploads the invoice to the Trades folder in Google Drive. |
| Upload to Telecom Folder | Google Drive | File Storage | Category Router | None | Uploads the invoice to the Telecom folder in Google Drive. |
| Upload to Review Folder | Google Drive | File Storage | Confidence Check, Category Router | Review Message | Uploads unclassified or low-confidence invoices to the Needs Review folder. |
| Review Message | Slack | Notification | Upload to Review Folder | None | Sends a message to #n8n-invoice-review with document details and Drive link. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: External Setup
1. **easybits:** Create a pipeline at `extractor.easybits.tech`. Define two fields: `document_class` (categorical) and `confidence_score` (numeric 0.0–1.0). Copy the Pipeline ID and API Key.
2. **Google Drive:** Create 6 folders (Medical, Restaurant, Hotel, Trades, Telecom, Needs Review). Note their Folder IDs.
3. **Slack:** Create a Slack App, grant `chat:write` permissions, and invite the bot to a specific review channel.

#### Step 2: Node Implementation
1. **Trigger:** Add a **Form Trigger**. Set title to "Image Upload" and add a file field named `image`.
2. **AI Extraction:** Add the **easybits Extractor** community node. Configure credentials using the Pipeline ID and API Key. Connect it to the Form Trigger.
3. **Parsing:** Add a **Set** node. Create two string/number assignments:
    - `document_type` $\rightarrow$ `{{ $json.data.document_class }}`
    - `confidence_score` $\rightarrow$ `{{ $json.data.confidence_score }}`
4. **Integration:** Add a **Merge** node. Set mode to "Combine" $\rightarrow$ "Combine by Position".
    - Input 1: `On form submission`
    - Input 2: `Parse Result`
5. **Quality Control:** Add an **If** node. Condition: `confidence_score` (Number) $\rightarrow$ Greater than $\rightarrow$ `0.5`.
6. **Classification Routing:** Add a **Switch** node connected to the `True` path of the If node.
    - Set 5 rules based on `{{ $json.document_type }}` equaling the specific labels (medical, restaurant, hotel, trades, telecom).
    - Set "Fallback" to the `extra` output.
7. **Storage:** Create 6 **Google Drive** nodes (one for each category + one for review).
    - Set "File Content" to `image`.
    - Set "File Name" to `{{ $('On form submission').first().binary.image.fileName }}`.
    - Assign the respective Folder IDs.
8. **Alerting:** Add a **Slack** node connected to the "Upload to Review Folder" node.
    - Configure the channel ID and a text message using expressions for `document_type`, `confidence_score`, and `{{ $json.webViewLink }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| easybits Extractor Pipeline Setup | [extractor.easybits.tech](https://extractor.easybits.tech) |
| Slack App Configuration | [api.slack.com/apps](https://api.slack.com/apps) |
| Google Cloud Console (OAuth2) | [console.cloud.google.com](https://console.cloud.google.com) |
| Supported Formats | PDF, PNG, JPEG |