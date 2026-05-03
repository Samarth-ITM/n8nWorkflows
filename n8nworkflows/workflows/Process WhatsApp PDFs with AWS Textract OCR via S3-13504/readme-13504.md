Process WhatsApp PDFs with AWS Textract OCR via S3

https://n8nworkflows.xyz/workflows/process-whatsapp-pdfs-with-aws-textract-ocr-via-s3-13504


# Process WhatsApp PDFs with AWS Textract OCR via S3

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Process WhatsApp PDFs with AWS Textract OCR via S3  
**Workflow name (in JSON):** Onboardly AI Agent - WhatsApp Document Collection  
**Purpose:** Automatically receive a PDF sent via WhatsApp Cloud API, download the media file, upload it to AWS S3, run AWS Textract asynchronous OCR/analysis (FORMS + TABLES), then extract clean text lines from the Textract result.

### 1.1 WhatsApp Input Reception
Triggered by inbound WhatsApp messages. Fetches media metadata and the downloadable URL for the PDF.

### 1.2 Media Download + S3 Storage
Downloads the PDF binary from WhatsApp media URL and uploads it to a fixed S3 bucket.

### 1.3 Textract Asynchronous Processing
Starts a Textract `StartDocumentAnalysis` job pointing to the S3 object, waits a fixed amount of time, then calls `GetDocumentAnalysis` to retrieve results.

### 1.4 Output Cleanup
Parses Textract blocks, extracts `LINE` text, and returns a single joined text output.

---

## 2. Block-by-Block Analysis

### Block 1 — WhatsApp Input Reception
**Overview:** Listens for inbound WhatsApp messages, then requests the media metadata from Meta Graph API to obtain a URL used to download the document.

**Nodes involved:**
- WhatsApp Trigger
- Get PDF

#### Node: WhatsApp Trigger
- **Type / Role:** `whatsAppTrigger` — webhook trigger for WhatsApp Cloud API updates.
- **Configuration (interpreted):**
  - Listens to `updates: ["messages"]`
  - Also enables message status updates in options (`messageStatusUpdates: ["all"]`), though the flow uses message content.
- **Key data used later:**
  - `$('WhatsApp Trigger').item.json.messages[0].document.id` (media/document id)
  - `$('WhatsApp Trigger').item.json.messages[0].document.filename` (file name)
  - `$('WhatsApp Trigger').item.json.metadata.phone_number_id`
- **Connections:**
  - **Output →** Get PDF
- **Credentials:** `WhatsApp OAuth account` (WhatsApp Trigger API OAuth)
- **Edge cases / failures:**
  - Incoming message is not a document (no `messages[0].document`) → downstream expressions will fail.
  - Webhook verification / subscription misconfig → trigger never fires.
  - Multiple messages in payload → workflow assumes index `[0]`.

#### Node: Get PDF
- **Type / Role:** `httpRequest` — calls Meta Graph API to retrieve media object metadata (including downloadable URL).
- **Configuration (interpreted):**
  - **GET** `https://graph.facebook.com/v24.0/{{document.id}}`
  - Query parameter `phone_number_id={{metadata.phone_number_id}}`
  - Auth via predefined WhatsApp credential type.
- **Key outputs expected:**
  - `url` field used by the next node (`File Download`).
- **Connections:**
  - **Input ←** WhatsApp Trigger
  - **Output →** File Download
- **Credentials:** `WhatsApp account` (predefined credential type `whatsAppApi`)
- **Edge cases / failures:**
  - Missing/expired token → 401/403.
  - Wrong Graph API version/permissions → 400/403.
  - If the document id is invalid or no longer available → 404.
  - If returned payload doesn’t contain `url`, next node fails.

---

### Block 2 — Media Download + S3 Storage
**Overview:** Downloads the actual PDF binary from the WhatsApp media URL and uploads it to S3 using the original WhatsApp filename.

**Nodes involved:**
- File Download
- Upload File to S3
- Download File from S3

#### Node: File Download
- **Type / Role:** `httpRequest` — downloads the PDF file binary from the media `url`.
- **Configuration (interpreted):**
  - URL: `{{$json.url}}` (from Get PDF output)
  - Response format: **file** (binary)
  - Auth: predefined credential type `whatsAppApi` to access media URL.
- **Connections:**
  - **Input ←** Get PDF
  - **Output →** Upload File to S3
- **Credentials:** `WhatsApp account`
- **Edge cases / failures:**
  - Media URLs can be short-lived; if expired → 401/403.
  - Large files may hit timeout limits depending on n8n settings.
  - If response isn’t treated as binary properly, S3 upload may fail.

#### Node: Upload File to S3
- **Type / Role:** `awsS3` — uploads the downloaded file to S3.
- **Configuration (interpreted):**
  - Operation: `upload`
  - Bucket: `onboardly-ai` (hard-coded)
  - File name: `$('WhatsApp Trigger').item.json.messages[0].document.filename`
  - Uses incoming binary from File Download as the upload content.
- **Connections:**
  - **Input ←** File Download
  - **Output →** Download File from S3
- **Credentials:** `AWS account`
- **Edge cases / failures:**
  - Bucket doesn’t exist / wrong region policy → upload error.
  - IAM permissions missing (`s3:PutObject`) → 403.
  - Duplicate filenames can overwrite objects (unless S3 node is configured to prevent overwrites; here it’s not).
  - If WhatsApp filename contains path-like characters, key naming may behave unexpectedly.

#### Node: Download File from S3
- **Type / Role:** `awsS3` — downloads the just-uploaded object (likely to obtain/confirm `Bucket` and `Key` fields in output).
- **Configuration (interpreted):**
  - Bucket name from data: `{{$json.Bucket}}`
  - File key from data: `{{$json.Key}}`
- **Connections:**
  - **Input ←** Upload File to S3
  - **Output →** AWS StartDocumentAnalysis
- **Credentials:** `AWS account`
- **Edge cases / failures:**
  - If previous node output does not contain `Bucket`/`Key`, expressions fail.
  - Missing IAM permissions (`s3:GetObject`) → 403.
  - Object not found (eventual consistency is rare for S3 but still possible in some edge conditions).

---

### Block 3 — Textract Asynchronous Processing
**Overview:** Starts an asynchronous Textract analysis job against the S3 object, waits a fixed 30 seconds, then fetches the job results.

**Nodes involved:**
- AWS StartDocumentAnalysis
- Wait for Processing Time
- AWS GetDocumentAnalysis

#### Node: AWS StartDocumentAnalysis
- **Type / Role:** `httpRequest` — calls Textract API `StartDocumentAnalysis`.
- **Configuration (interpreted):**
  - POST `https://textract.us-east-1.amazonaws.com/`
  - Headers:
    - `X-Amz-Target: Textract.StartDocumentAnalysis`
    - `Content-Type: application/x-amz-json-1.1`
  - JSON body uses S3 references:
    - Bucket: `{{$json.Bucket}}`
    - Name/Key: `{{$json.Key}}`
  - FeatureTypes: `["FORMS", "TABLES"]`
  - Authentication: AWS predefined credential type (SigV4 signing via n8n AWS creds).
- **Connections:**
  - **Input ←** Download File from S3
  - **Output →** Wait for Processing Time
- **Credentials:** `AWS account`
- **Version specifics:** HTTP Request node v4.3 with predefined AWS credential type.
- **Edge cases / failures:**
  - Textract requires the file to be in the same AWS account/accessible; permissions needed:
    - Textract service role access to S3 object (often via IAM policy on bucket/object).
  - Wrong region: endpoint is hard-coded to `us-east-1`; bucket may be elsewhere (Textract supports multi-region but you must call the matching endpoint).
  - Unsupported PDF properties (encrypted PDFs, corrupted files) → job failure.
  - Large PDFs can take >30 seconds, making the later fetch incomplete.

#### Node: Wait for Processing Time
- **Type / Role:** `wait` — pauses workflow for a fixed time.
- **Configuration (interpreted):**
  - Amount: `30` (seconds by default in Wait node unless otherwise specified; node config shows only amount)
- **Connections:**
  - **Input ←** AWS StartDocumentAnalysis
  - **Output →** AWS GetDocumentAnalysis
- **Edge cases / failures:**
  - Fixed wait is fragile: Textract job may still be `IN_PROGRESS` after 30 seconds.
  - If your n8n instance has execution time limits, long waits may be constrained.

#### Node: AWS GetDocumentAnalysis
- **Type / Role:** `httpRequest` — calls Textract API `GetDocumentAnalysis` to retrieve results.
- **Configuration (interpreted):**
  - POST `https://textract.us-east-1.amazonaws.com/`
  - Headers:
    - `X-Amz-Target: Textract.GetDocumentAnalysis`
    - `Content-Type: application/x-amz-json-1.1`
  - JSON body:
    - `JobId: {{ $json.data.parseJson().JobId }}`
      - This assumes the incoming item has a `data` field that is a JSON string or object with `.JobId`.
- **Connections:**
  - **Input ←** Wait for Processing Time
  - **Output →** Extract Text
- **Credentials:** `AWS account`
- **Edge cases / failures:**
  - **Expression risk:** `$json.data.parseJson()` depends on `data` being a string and that `parseJson()` is available; if `data` is already an object, this may fail unless n8n coerces it.
  - Textract may paginate results (`NextToken`) for multi-page documents; this workflow does not loop over `NextToken`, so output may be partial.
  - Job may be `FAILED` or still `IN_PROGRESS`; handling is not implemented.

---

### Block 4 — Output Cleanup
**Overview:** Converts Textract’s block-based output into plain text by taking all LINE blocks and joining them with newlines.

**Nodes involved:**
- Extract Text

#### Node: Extract Text
- **Type / Role:** `code` — transforms Textract JSON into a single `text` string.
- **Configuration (interpreted):**
  - Reads all incoming items (`$input.all()`).
  - For each item:
    - Takes `item.json.data`
    - If string → `JSON.parse(raw)` else uses as-is
    - Extracts `textract.Blocks`
    - Filters blocks where `BlockType === 'LINE'` and has `Text`
    - Joins all lines with `\n`
  - Outputs: `{ text: "..." }`
- **Connections:**
  - **Input ←** AWS GetDocumentAnalysis
  - **Output →** none (workflow ends here)
- **Edge cases / failures:**
  - If incoming structure is not `{ data: ... }` but already the Textract object at root, `raw` will be `undefined` and output becomes empty.
  - If `data` is not valid JSON string when parsed → throws.
  - Ordering: Textract LINE blocks are not guaranteed to be perfectly “reading order” without additional geometry-based sorting; this workflow does not sort.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| WhatsApp Trigger | whatsAppTrigger | Entry point: receive inbound WhatsApp messages | — | Get PDF | ## Whatsapp Input |
| Get PDF | httpRequest | Fetch media metadata (Graph API) to obtain media URL | WhatsApp Trigger | File Download | ## Whatsapp Input |
| File Download | httpRequest | Download PDF binary from media URL | Get PDF | Upload File to S3 | ## AWS S3 Handling |
| Upload File to S3 | awsS3 | Upload binary to S3 bucket | File Download | Download File from S3 | ## AWS S3 Handling |
| Download File from S3 | awsS3 | Retrieve object (and/or normalize Bucket/Key for next step) | Upload File to S3 | AWS StartDocumentAnalysis | ## AWS Textract Processing |
| AWS StartDocumentAnalysis | httpRequest | Start Textract async analysis job (FORMS/TABLES) | Download File from S3 | Wait for Processing Time | ## AWS Textract Processing |
| Wait for Processing Time | wait | Pause to allow Textract processing | AWS StartDocumentAnalysis | AWS GetDocumentAnalysis | ## AWS Textract Processing |
| AWS GetDocumentAnalysis | httpRequest | Fetch Textract analysis results by JobId | Wait for Processing Time | Extract Text | ## AWS Textract Processing |
| Extract Text | code | Extract LINE text from Textract blocks and join | AWS GetDocumentAnalysis | — | ## Output Cleanup |
| Sticky Note | stickyNote | Documentation / high-level description | — | — | ## 🟨 **MAIN STICKY — WhatsApp PDF OCR Workflow**  ; **How it Works** ; This workflow automatically extracts text from PDF documents sent via WhatsApp. When a message containing a PDF arrives, the file is downloaded from the WhatsApp media URL and uploaded to your AWS S3 bucket. A Textract analysis job is then started to perform OCR on the stored PDF. After a short wait, the workflow retrieves the results from Textract and converts them into clean, ordered text. The extracted output is returned for further processing such as AI analysis, storage, or automation. ; **Setup Steps** ; 1. Connect your WhatsApp integration (e.g., webhook from WhatsApp Cloud API). ; 2. Add AWS credentials with permission for S3 upload and Textract analysis. ; 3. Set your S3 bucket name and Textract region in the AWS nodes. ; 4. Ensure HTTP Request nodes are authenticated for WhatsApp media access. ; 5. Activate the workflow and send a PDF via WhatsApp to test. ; **Good to Know** ; * Supports both scanned and digital PDFs. ; * Textract pricing is per page; processing time varies by file size. ; * S3 acts as the required temporary storage layer. |
| Sticky Note1 | stickyNote | Section label | — | — | ## Whatsapp Input |
| Sticky Note2 | stickyNote | Section label | — | — | ## AWS S3 Handling |
| Sticky Note3 | stickyNote | Section label | — | — | ## AWS Textract Processing |
| Sticky Note4 | stickyNote | Section label | — | — | ## Output Cleanup |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it (e.g.) **“Process WhatsApp PDFs with AWS Textract OCR via S3”**.
- Ensure workflow settings execution order is default (`v1` is fine).

2) **Add credentials**
- **WhatsApp Trigger credential (OAuth):** create/attach a WhatsApp Cloud API OAuth credential for the Trigger node.
- **WhatsApp API credential (for HTTP requests):** create/attach the predefined `whatsAppApi` credential used by HTTP Request nodes to call Graph API and download media.
- **AWS credential:** add AWS access key/secret (or role-based) with permissions for:
  - `s3:PutObject`, `s3:GetObject` on your bucket
  - Textract actions: `textract:StartDocumentAnalysis`, `textract:GetDocumentAnalysis`
  - Plus required S3 access for Textract to read the object (bucket policy/IAM).

3) **Create the trigger**
- Add node: **WhatsApp Trigger**
  - Updates: `messages`
  - Options: enable message status updates if desired (`all`)
  - Select the WhatsApp OAuth trigger credential.
- This is the entry node.

4) **Fetch media metadata (Graph API)**
- Add node: **HTTP Request** named **Get PDF**
  - Method: GET
  - URL: `https://graph.facebook.com/v24.0/{{$('WhatsApp Trigger').item.json.messages[0].document.id}}`
  - Query parameter:
    - `phone_number_id = {{$('WhatsApp Trigger').item.json.metadata.phone_number_id}}`
  - Authentication: predefined credential type `whatsAppApi`
- Connect: **WhatsApp Trigger → Get PDF**

5) **Download the actual PDF**
- Add node: **HTTP Request** named **File Download**
  - URL: `{{$json.url}}`
  - Response: set **Response Format = File** (binary)
  - Authentication: predefined credential type `whatsAppApi`
- Connect: **Get PDF → File Download**

6) **Upload the PDF to S3**
- Add node: **AWS S3** named **Upload File to S3**
  - Operation: Upload
  - Bucket: set to your bucket (workflow uses `onboardly-ai`)
  - File name expression:
    - `{{$('WhatsApp Trigger').item.json.messages[0].document.filename}}`
  - Select AWS credentials
- Connect: **File Download → Upload File to S3**

7) **(Optional but present in this workflow) Download from S3**
- Add node: **AWS S3** named **Download File from S3**
  - Operation: Download (default)
  - Bucket name: `{{$json.Bucket}}`
  - File key: `{{$json.Key}}`
- Connect: **Upload File to S3 → Download File from S3**

8) **Start Textract analysis**
- Add node: **HTTP Request** named **AWS StartDocumentAnalysis**
  - Method: POST
  - URL: `https://textract.us-east-1.amazonaws.com/`
  - Authentication: predefined credential type **AWS**
  - Headers:
    - `X-Amz-Target: Textract.StartDocumentAnalysis`
    - `Content-Type: application/x-amz-json-1.1`
  - Body: JSON
    - DocumentLocation.S3Object.Bucket = `{{$json.Bucket}}`
    - DocumentLocation.S3Object.Name = `{{$json.Key}}`
    - FeatureTypes = `["FORMS","TABLES"]`
- Connect: **Download File from S3 → AWS StartDocumentAnalysis**

9) **Wait for Textract to finish**
- Add node: **Wait** named **Wait for Processing Time**
  - Amount: `30` (seconds)
- Connect: **AWS StartDocumentAnalysis → Wait for Processing Time**

10) **Get Textract results**
- Add node: **HTTP Request** named **AWS GetDocumentAnalysis**
  - Method: POST
  - URL: `https://textract.us-east-1.amazonaws.com/`
  - Authentication: predefined credential type **AWS**
  - Headers:
    - `X-Amz-Target: Textract.GetDocumentAnalysis`
    - `Content-Type: application/x-amz-json-1.1`
  - Body: JSON
    - `JobId = {{ $json.data.parseJson().JobId }}`
- Connect: **Wait for Processing Time → AWS GetDocumentAnalysis**

11) **Extract readable text**
- Add node: **Code** named **Extract Text**
  - Paste logic equivalent to:
    - Parse `$json.data` (string or object)
    - Get `Blocks`
    - Filter `BlockType === 'LINE'`
    - Join `.Text` with newline into `json.text`
- Connect: **AWS GetDocumentAnalysis → Extract Text**

12) **(Optional) Add sticky notes / labels**
- Add sticky notes with headings:
  - “Whatsapp Input”
  - “AWS S3 Handling”
  - “AWS Textract Processing”
  - “Output Cleanup”
  - And the main note content if you want embedded documentation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **MAIN STICKY — WhatsApp PDF OCR Workflow**: Explains flow (download WhatsApp PDF → upload to S3 → Textract job → retrieve → clean text), setup steps, and “Good to Know” items (supports scanned/digital PDFs, pricing per page, S3 as storage layer). | Internal workflow note (no external link provided) |
| Fixed 30-second wait may be insufficient for large PDFs; consider polling `GetDocumentAnalysis` until `JobStatus == "SUCCEEDED"` and handling `NextToken` pagination. | Reliability consideration for Textract async jobs |
| Textract endpoint is hard-coded to `us-east-1`; ensure your chosen region matches your deployment and any compliance constraints. | AWS regional configuration consideration |