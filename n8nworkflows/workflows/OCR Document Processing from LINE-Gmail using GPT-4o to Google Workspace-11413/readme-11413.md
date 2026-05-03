OCR Document Processing from LINE/Gmail using GPT-4o to Google Workspace

https://n8nworkflows.xyz/workflows/ocr-document-processing-from-line-gmail-using-gpt-4o-to-google-workspace-11413


# OCR Document Processing from LINE/Gmail using GPT-4o to Google Workspace

### 1. Workflow Overview

The "Document OCR and Intelligent Summarization System" workflow automates the extraction, summarization, and archiving of documents received via LINE messages or Gmail emails. It targets users who frequently receive documents as images or email attachments and wish to maintain an automated archive with full extracted text and summaries stored in Google Sheets and Gmail drafts.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Triggering:** Captures incoming files from LINE webhook events and Gmail inbox emails.
- **1.2 Source Tagging & Normalization:** Tags each incoming file with its source (LINE or EMAIL) and normalizes Gmail attachments to a consistent binary format.
- **1.3 File Upload & OCR Processing:** Uploads received files to Google Drive, downloads the file binary, converts to Base64, and runs OCR via an OpenAI Vision model.
- **1.4 Text Summarization & Output Preparation:** Uses an LLM to summarize extracted text, logs results in Google Sheets, and creates Gmail drafts with OCR text and summaries.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Triggering

**Overview:**  
This block listens for new incoming documents from LINE (via webhook) or Gmail (via IMAP trigger) to start the processing workflow.

**Nodes Involved:**  
- LINE Webhook  
- Gmail IMAP Trigger  
- Workflow Configuration  
- If (conditional check for LINE images)

**Node Details:**

- **LINE Webhook**  
  - Type: Webhook (HTTP POST listener)  
  - Role: Receives incoming LINE message events, specifically image messages in this flow.  
  - Config: HTTP POST at path `line-webhook`  
  - Input: External LINE platform POST requests  
  - Output: JSON event data  
  - Failure: Invalid or missing webhook secret, malformed requests  
  - Version: 2.1  

- **Gmail IMAP Trigger**  
  - Type: Email Trigger  
  - Role: Polls Gmail inbox every minute for new emails with attachments  
  - Config: Poll interval 1 minute, label INBOX, downloads attachments  
  - Input: New email notifications with attachments  
  - Output: Email data with binary attachments  
  - Failure: Authentication errors, rate limits, attachment download failures  
  - Version: 1.3  

- **Workflow Configuration**  
  - Type: Set  
  - Role: Defines reusable parameters such as LINE channel token and Google Drive folder ID  
  - Config: Variables `lineChannelAccessToken`, `googleDriveFolderId` with placeholders to be replaced by user  
  - Input: From LINE Webhook only  
  - Output: Adds config fields to data  
  - Failure: Missing or invalid tokens/IDs will cause downstream failures  
  - Version: 3.4  

- **If**  
  - Type: Conditional  
  - Role: Filters LINE webhook events to process only those where message type is "image"  
  - Config: Checks `$json.body.events[0].message.type == "image"`  
  - Input: From Workflow Configuration node on LINE path  
  - Output: Passes only image messages to next node (HTTP Request)  
  - Failure: Expression errors if JSON path missing, false negatives if message type changed in LINE API  
  - Version: 2.2  

---

#### 2.2 Source Tagging & Normalization

**Overview:**  
Labels each incoming file with its source ("LINE" or "EMAIL"), attaches Google Drive folder ID, and normalizes Gmail attachments to a consistent binary format for unified downstream processing.

**Nodes Involved:**  
- Tag Source - LINE  
- Tag Source - EMAIL  
- Code in JavaScript  
- Merge Triggers

**Node Details:**

- **Tag Source - LINE**  
  - Type: Set  
  - Role: Adds `source = "LINE"` to items from LINE webhook and carries Drive folder ID from config  
  - Config: Fixed string assignment `source: "LINE"`  
  - Input: From HTTP Request (LINE content)  
  - Output: Enriched JSON with source and folder ID  
  - Failure: Missing folder ID in config causes upload failures  
  - Version: 3.4  

- **Tag Source - EMAIL**  
  - Type: Set  
  - Role: Adds `source = "EMAIL"` and Drive folder ID to Gmail email items  
  - Config: Fixed string `source: "EMAIL"`, folder ID from user config  
  - Input: From Gmail IMAP Trigger via Code in JavaScript  
  - Output: Enriched JSON  
  - Failure: Misconfigured folder ID or missing attachments impact downstream steps  
  - Version: 3.4  

- **Code in JavaScript**  
  - Type: Code  
  - Role: Normalizes email attachments binary data to a standard key `data` for compatibility with LINE binary format  
  - Config: Copies first binary attachment to `binary.data`  
  - Input: From Tag Source - EMAIL  
  - Output: Modified item with normalized binary data key  
  - Failure: No binary data in email attachments leads to empty processing downstream  
  - Version: 2  

- **Merge Triggers**  
  - Type: Merge  
  - Role: Combines LINE and EMAIL branches into a single stream for unified processing  
  - Config: Default merge settings (no special combinator)  
  - Input: From Tag Source - LINE (index 0) and Code in JavaScript (index 1)  
  - Output: Single stream of items with uniform structure  
  - Failure: Data format mismatches between branches could cause errors downstream  
  - Version: 3.2  

---

#### 2.3 File Upload & OCR Processing

**Overview:**  
Uploads the file to Google Drive, downloads the binary content, converts to Base64, and uses OpenAI Vision model to extract raw text. A delay is added to avoid API rate limits.

**Nodes Involved:**  
- Upload to Google Drive  
- Convert to Base64  
- HTTP Request2 (Google Drive file download)  
- Analyze image (OpenAI Vision OCR)  
- Wait

**Node Details:**

- **Upload to Google Drive**  
  - Type: Google Drive  
  - Role: Saves the incoming file (image or PDF) to a specified Drive folder  
  - Config: Uses Drive ID "My Drive", folder ID from the tagged source data, file name from binary data or defaults to "document"  
  - Input: From Merge Triggers  
  - Output: JSON including new Google Drive file ID  
  - Failure: Invalid folder ID, insufficient permissions, file size limits  
  - Version: 3  

- **Convert to Base64**  
  - Type: Code  
  - Role: Passes through JSON metadata and binary data unmodified (placeholder for possible Base64 conversion)  
  - Config: JavaScript maintains JSON and binary data intact  
  - Input: From Upload to Google Drive  
  - Output: Same data forwarded  
  - Failure: None expected; code is a passthrough  
  - Version: 2  

- **HTTP Request2**  
  - Type: HTTP Request  
  - Role: Downloads the file content from Google Drive using the file ID, to get the raw binary for OCR  
  - Config: URL constructed dynamically using Drive file ID, uses Google Drive OAuth2 authentication  
  - Input: From Convert to Base64  
  - Output: Binary content of the file  
  - Failure: Expired token, permission denied, network errors  
  - Version: 4.2  

- **Analyze image**  
  - Type: OpenAI Vision (Langchain)  
  - Role: Runs OCR on the Base64-encoded image/PDF to extract all textual content  
  - Config: Model `gpt-4o-mini`, input type Base64, instruction to transcribe all content preserving line breaks  
  - Input: From HTTP Request2 (binary converted to Base64)  
  - Output: JSON with extracted text content  
  - Failure: API quota exceeded, invalid input format, timeout, incomplete extraction  
  - Version: 1.8  

- **Wait**  
  - Type: Wait  
  - Role: Delays 60 seconds before next step to avoid API rate limiting or quota issues  
  - Config: 60 seconds delay  
  - Input: From Analyze image  
  - Output: Delayed pass-through  
  - Failure: None expected  
  - Version: 1.1  

---

#### 2.4 Text Summarization & Output Preparation

**Overview:**  
Takes the OCR text, generates a concise summary using an LLM, logs the summary in Google Sheets, and creates a Gmail draft email with the OCR text, summary, and a link to the uploaded file.

**Nodes Involved:**  
- Message a model (OpenAI chat)  
- Append row in sheet (Google Sheets)  
- Create Gmail Draft

**Node Details:**

- **Message a model**  
  - Type: OpenAI Chat (Langchain)  
  - Role: Summarizes the OCR extracted text into a concise, human-readable summary  
  - Config: Model `gpt-4o-mini`, single message with content from OCR result JSON  
  - Input: From Wait  
  - Output: JSON with summary text  
  - Failure: API rate limits, invalid prompt, model unavailability  
  - Version: 1.8  

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Logs each summary and metadata (e.g., timestamp) as a new row in a designated Google Sheet  
  - Config: Append operation on "Sheet1" of a specified Google Sheets document ID, writes summary to column "要約内容"  
  - Input: From Message a model  
  - Output: Confirmation of append operation  
  - Failure: Incorrect sheet ID, permission denied, rate limits  
  - Version: 4.7  

- **Create Gmail Draft**  
  - Type: Gmail (Create Draft)  
  - Role: Creates a draft email containing the original OCR text, summary, and file URL for user review or forwarding  
  - Config: Subject includes file name, body includes OCR text and placeholders for summary and file URL  
  - Input: From Append row in sheet  
  - Output: Gmail draft message  
  - Failure: Authentication errors, invalid email format  
  - Version: 2.1  

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                        | Input Node(s)                       | Output Node(s)                   | Sticky Note                                                                                                   |
|-------------------------|---------------------------|-------------------------------------|-----------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| LINE Webhook            | Webhook                   | Receives LINE messages via webhook | External LINE POST request        | Workflow Configuration           | ## Triggers & configuration - LINE Webhook and Gmail IMAP Trigger start the workflow when new data arrives.   |
| Gmail IMAP Trigger      | Email Trigger             | Receives new Gmail emails           | Gmail inbox                      | Tag Source - EMAIL               | ## Triggers & configuration - LINE Webhook and Gmail IMAP Trigger start the workflow when new data arrives.   |
| Workflow Configuration  | Set                       | Stores tokens and folder IDs        | LINE Webhook                    | If                              | ## Triggers & configuration - LINE Webhook and Gmail IMAP Trigger start the workflow when new data arrives.   |
| If                      | Conditional               | Filters LINE images only             | Workflow Configuration           | HTTP Request                    | ## Triggers & configuration - LINE Webhook and Gmail IMAP Trigger start the workflow when new data arrives.   |
| HTTP Request            | HTTP Request              | Downloads LINE message content      | If                              | Tag Source - LINE               | ## Source tagging & merge - labels each item with source and folder ID for unified downstream processing.     |
| Tag Source - LINE       | Set                       | Adds source = "LINE" and folder ID | HTTP Request                    | Merge Triggers                 | ## Source tagging & merge - labels each item with source and folder ID for unified downstream processing.     |
| Tag Source - EMAIL      | Set                       | Adds source = "EMAIL" and folder ID | Gmail IMAP Trigger              | Code in JavaScript             | ## Source tagging & merge - labels each item with source and folder ID for unified downstream processing.     |
| Code in JavaScript      | Code                      | Normalizes Gmail attachments binary | Tag Source - EMAIL              | Merge Triggers                 | ## Source tagging & merge - labels each item with source and folder ID for unified downstream processing.     |
| Merge Triggers          | Merge                     | Combines LINE and EMAIL branches    | Tag Source - LINE, Code in JS    | Upload to Google Drive          | ## Source tagging & merge - labels each item with source and folder ID for unified downstream processing.     |
| Upload to Google Drive  | Google Drive              | Uploads files to Drive folder       | Merge Triggers                  | Convert to Base64               | ## File upload & OCR - uploads files, downloads binary, converts, and runs OCR via OpenAI Vision model.        |
| Convert to Base64       | Code                      | Passes JSON and binary data         | Upload to Google Drive          | HTTP Request2                  | ## File upload & OCR - uploads files, downloads binary, converts, and runs OCR via OpenAI Vision model.        |
| HTTP Request2           | HTTP Request              | Downloads file content from Drive   | Convert to Base64               | Analyze image                  | ## File upload & OCR - uploads files, downloads binary, converts, and runs OCR via OpenAI Vision model.        |
| Analyze image           | OpenAI Vision (Langchain) | Extracts text via OCR from image    | HTTP Request2                  | Wait                          | ## File upload & OCR - uploads files, downloads binary, converts, and runs OCR via OpenAI Vision model.        |
| Wait                    | Wait                      | Adds delay to avoid API rate limits | Analyze image                  | Message a model                | ## File upload & OCR - uploads files, downloads binary, converts, and runs OCR via OpenAI Vision model.        |
| Message a model         | OpenAI Chat (Langchain)   | Summarizes OCR extracted text       | Wait                          | Append row in sheet           | ## Summaries & outputs - generates summary, logs in Sheets, creates Gmail draft with results.                  |
| Append row in sheet     | Google Sheets             | Logs summaries to Google Sheets     | Message a model                | Create Gmail Draft            | ## Summaries & outputs - generates summary, logs in Sheets, creates Gmail draft with results.                  |
| Create Gmail Draft      | Gmail                     | Creates draft email with results    | Append row in sheet            |                                 | ## Summaries & outputs - generates summary, logs in Sheets, creates Gmail draft with results.                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create LINE Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `line-webhook`  
   - Purpose: Capture incoming LINE messages (must configure LINE bot to send events here).

2. **Create Gmail IMAP Trigger Node**  
   - Type: Gmail Trigger  
   - Poll Interval: Every 1 minute  
   - Label Filter: INBOX  
   - Download Attachments: Enabled  
   - Purpose: Detect new Gmail emails with attachments.

3. **Create Workflow Configuration Node (Set)**  
   - Add string parameters:  
     - `lineChannelAccessToken` = "YOUR_LINE_CHANNEL_ACCESS_TOKEN"  
     - `googleDriveFolderId` = "YOUR_GOOGLE_DRIVE_FOLDER_ID"  
   - Purpose: Store credentials and folder IDs for reuse.

4. **Connect LINE Webhook → Workflow Configuration Node.**

5. **Create If Node**  
   - Condition: Check if `$json.body.events[0].message.type` equals `"image"`  
   - Purpose: Filter LINE events to only process images.

6. **Connect Workflow Configuration → If Node.**

7. **Create HTTP Request Node ("HTTP Request")**  
   - URL: `https://api-data.line.me/v2/bot/message/{{ $json.body.events[0].message.id }}/content`  
   - Authentication: Use LINE API token via HTTP Header Auth  
   - Purpose: Download image content from LINE messages.

8. **Connect If (true output) → HTTP Request Node.**

9. **Create Set Node ("Tag Source - LINE")**  
   - Assign `source` = "LINE"  
   - Assign `googleDriveFolderId` from Workflow Configuration  
   - Purpose: Tag LINE files with source and folder info.

10. **Connect HTTP Request → Tag Source - LINE.**

11. **Connect Gmail IMAP Trigger → Set Node ("Tag Source - EMAIL")**  
    - Assign `source` = "EMAIL"  
    - Assign `googleDriveFolderId` with same Drive folder ID as LINE if desired.

12. **Create Code Node ("Code in JavaScript")**  
    - JavaScript to normalize Gmail attachment binary keys to `binary.data` for uniformity:  
      ```javascript
      const item = $input.first();
      if (item.binary) {
        const keys = Object.keys(item.binary);
        if (keys.length > 0) {
          const firstKey = keys[0];
          item.binary.data = item.binary[firstKey];
        }
      }
      return item;
      ```
    - Purpose: Normalize Gmail attachments binary format.

13. **Connect Tag Source - EMAIL → Code in JavaScript Node.**

14. **Create Merge Node ("Merge Triggers")**  
    - Default merge mode  
    - Inputs:  
      - Input 1 from Tag Source - LINE  
      - Input 2 from Code in JavaScript

15. **Connect Tag Source - LINE → Merge Triggers (input 1).**  
    Connect Code in JavaScript → Merge Triggers (input 2).

16. **Create Google Drive Node ("Upload to Google Drive")**  
    - Operation: Upload file  
    - Drive: "My Drive"  
    - Folder ID: Use `googleDriveFolderId` from merged item data  
    - File Name: From binary data fileName or default `"document"`  
    - Purpose: Save incoming document to Google Drive.

17. **Connect Merge Triggers → Upload to Google Drive.**

18. **Create Code Node ("Convert to Base64")**  
    - JavaScript passthrough, preserving JSON and binary data intact (placeholder for Base64 conversion if needed).  
    - Purpose: Maintain data for next steps.

19. **Connect Upload to Google Drive → Convert to Base64.**

20. **Create HTTP Request Node ("HTTP Request2")**  
    - Method: GET  
    - URL: `https://www.googleapis.com/drive/v3/files/{{ $json.id }}?alt=media`  
    - Authentication: Google Drive OAuth2  
    - Purpose: Download file binary content from Google Drive for OCR.

21. **Connect Convert to Base64 → HTTP Request2.**

22. **Create OpenAI Vision Node ("Analyze image")**  
    - Operation: Analyze image (OCR)  
    - Model: `gpt-4o-mini`  
    - Input type: Base64 (binary converted as needed)  
    - Prompt: "Transcribe all content written in this image exactly, preserving all line breaks."  
    - Purpose: Extract text content from image/PDF.

23. **Connect HTTP Request2 → Analyze image.**

24. **Create Wait Node**  
    - Duration: 60 seconds  
    - Purpose: Avoid API rate limits and ensure stable processing.

25. **Connect Analyze image → Wait.**

26. **Create OpenAI Chat Node ("Message a model")**  
    - Model: `gpt-4o-mini`  
    - Message content: OCR text from previous node JSON  
    - Purpose: Generate a concise summary of the OCR text.

27. **Connect Wait → Message a model.**

28. **Create Google Sheets Node ("Append row in sheet")**  
    - Operation: Append  
    - Sheet name: "Sheet1"  
    - Document ID: Your Google Sheets ID  
    - Map column "要約内容" to summary text field from Message a model output  
    - Purpose: Log summaries and metadata in Google Sheets.

29. **Connect Message a model → Append row in sheet.**

30. **Create Gmail Node ("Create Gmail Draft")**  
    - Operation: Create draft  
    - Subject: `"Document Processing Results - {{file name}}"`  
    - Message: Include OCR original text, summary, and file URL placeholders  
    - Purpose: Prepare a reviewable email draft with document processing results.

31. **Connect Append row in sheet → Create Gmail Draft.**

32. **Configure all credentials:**  
    - LINE API token (HTTP Request node)  
    - Gmail OAuth2 (Gmail IMAP Trigger and Create Gmail Draft)  
    - Google Drive OAuth2 (Upload, HTTP Request2)  
    - Google Sheets OAuth2 (Append row)  
    - OpenAI API key (OpenAI Vision and Chat nodes)

33. **Replace placeholders:**  
    - `YOUR_LINE_CHANNEL_ACCESS_TOKEN` in Workflow Configuration  
    - `YOUR_GOOGLE_DRIVE_FOLDER_ID` in Workflow Configuration and source tagging nodes  
    - `YOUR_GOOGLE_SHEETS_DOCUMENT_ID` in Google Sheets node

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is designed for users receiving many documents as photos or email attachments who want automated OCR, summarization, and archival.             | Overview sticky note in the workflow ("Sticky Note - Overview")                                          |
| Setup requires adding credentials for LINE, Gmail, Google Drive, Google Sheets, and OpenAI in your n8n instance before activation.                             | Setup notes in "Sticky Note - Overview"                                                                 |
| Triggers use LINE Webhook and Gmail IMAP Trigger nodes to start processing on new incoming data.                                                               | "Sticky Note - Triggers & configuration"                                                                |
| Source tagging standardizes data from LINE and Gmail for consistent downstream processing.                                                                      | "Sticky Note - Source tagging & merge"                                                                   |
| File upload and OCR uses Google Drive storage and OpenAI Vision model for high-quality text extraction.                                                        | "Sticky Note - File upload & OCR"                                                                        |
| Summaries are generated by OpenAI chat model, results logged to Google Sheets, and Gmail drafts created for easy review and sharing.                          | "Sticky Note - Summaries & outputs"                                                                      |
| You may need to adjust API rate limits or add more sophisticated error handling for production use, especially around API quotas and network reliability.        | General best practice note                                                                                 |
| OpenAI GPT-4o-mini model is used for both OCR and summarization steps, providing a unified AI solution.                                                        | Model IDs referenced in Analyze image and Message a model nodes                                           |
| For LINE API content download, ensure the channel access token is current and has required permissions.                                                        | HTTP Request node configuration details                                                                  |

---

**Disclaimer:** The provided content comes exclusively from an n8n automated workflow. The processing strictly respects content policies and contains no illegal or protected elements. All data handled is legal and publicly accessible.