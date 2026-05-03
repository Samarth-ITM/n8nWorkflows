Process vendor invoices with UploadToURL, AWS Textract, and Google Sheets

https://n8nworkflows.xyz/workflows/process-vendor-invoices-with-uploadtourl--aws-textract--and-google-sheets-13615


# Process vendor invoices with UploadToURL, AWS Textract, and Google Sheets

# Reference Document: Process Vendor Invoices with UploadToURL, AWS Textract, and Google Sheets

This document provides a technical breakdown of an automated Accounts Payable pipeline. The workflow captures invoice data (via binary upload or URL), hosts it on a CDN, extracts key information using AWS AI, synchronizes the data with Google Sheets (handling duplicates), and notifies stakeholders via Slack.

---

### 1. Workflow Overview

The workflow is designed to eliminate manual data entry for PDF or image-based invoices. It is structured into four functional logic blocks:

*   **1.1 Intake & Validation:** Receives the file, validates the format, and generates a public CDN link using the `UploadToURL` community node.
*   **1.2 AI OCR Processing:** Uses AWS Textract’s `AnalyzeExpense` API to identify vendor names, dates, and amounts. It includes logic to handle partial data extractions.
*   **1.3 Data Synchronization:** Performs a lookup in Google Sheets to prevent duplicates. It intelligently updates existing rows if a previous entry was "incomplete."
*   **1.4 Notification & Response:** Sends a formatted summary to a Slack finance channel and returns a structured JSON response to the initial caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Section 1 — Intake & Upload
**Overview:** This block handles the entry point. It accepts data via Webhook and ensures the file is hosted on a public URL accessible by AWS Textract.

*   **Nodes Involved:**
    *   `Webhook - Receive Invoice`
    *   `Validate Payload`
    *   `Has Remote URL?`
    *   `Upload to URL - Remote`
    *   `Upload to URL - Binary`
    *   `Extract CDN URL`
*   **Node Details:**
    *   **Webhook:** Set to `POST`. Expects either a `fileUrl` or a binary file.
    *   **Validate Payload (Code):** Normalizes the input. It checks for allowed extensions (`pdf`, `jpg`, `jpeg`, `png`, `tiff`) and creates a sanitized, timestamped filename.
    *   **Has Remote URL? (If):** Diverts logic based on whether the file is already online or needs a binary upload.
    *   **Upload to URL (Community Node):** Authenticates with the UploadToURL service to host the file. This is crucial for providing AWS Textract with a steady URL.
    *   **Extract CDN URL (Code):** Unifies the response from the upload nodes into a single `cdnUrl` variable and ensures the protocol is HTTPS.

#### 2.2 Section 2 — OCR Extraction
**Overview:** This block interfaces with AWS to perform intelligent document analysis.

*   **Nodes Involved:**
    *   `AWS Textract - Analyse Expense`
    *   `Parse & Validate Extracted Data`
*   **Node Details:**
    *   **AWS Textract (HTTP Request):** Performs a POST request to `textract.us-east-1.amazonaws.com`. It uses the `AnalyzeExpense` target. Note: Requires SigV4 signing or a Header-based auth if proxied. It references the hosted CDN file.
    *   **Parse & Validate Extracted Data (Code):** Extracts fields like `VENDOR_NAME`, `INVOICE_NUMBER`, and `AMOUNT_DUE`. It calculates an `extractionStatus` (complete/incomplete) based on whether essential fields were found. It also extracts the first 10 line items.

#### 2.3 Section 3 — Duplicate Check & Sheets Write
**Overview:** Manages the database logic within Google Sheets to maintain data integrity.

*   **Nodes Involved:**
    *   `Sheets - Search for Duplicate`
    *   `Resolve Write Action`
    *   `Route Write Action`
    *   `Sheets - Append New Invoice`
    *   `Sheets - Update Incomplete Row`
    *   `Mark as Duplicate`
    *   `Merge Sheets Result`
*   **Node Details:**
    *   **Sheets - Search for Duplicate:** Looks up the `Invoice No` in the spreadsheet.
    *   **Resolve Write Action (Code):** Logic determines if the entry is a new record (`append`), an update to a previously failed extraction (`update`), or a redundant copy (`skip`).
    *   **Sheets Nodes (Append/Update):** Performs the respective action. The "Update" path specifically targets rows previously marked as "incomplete."
    *   **Mark as Duplicate (Code):** Prepares metadata for duplicates so they can still be reported in the notification phase without being re-written to the Sheet.

#### 2.4 Section 4 — Notification & Response
**Overview:** Closes the loop by notifying the team and responding to the webhook caller.

*   **Nodes Involved:**
    *   `Slack - Notify Finance`
    *   `Build Final Response`
    *   `Respond to Webhook`
*   **Node Details:**
    *   **Slack - Notify Finance:** Sends a rich text message. It uses dynamic emojis: ✅ (Complete), 🟡 (Incomplete), or ⚠️ (Duplicate). It includes a direct link to the hosted invoice.
    *   **Respond to Webhook:** Returns a `201 Created` status with a full JSON summary of the operation, including the Google Sheets row ID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - Receive Invoice | Webhook | Entry Point | (None) | Validate Payload | 1 — Intake & upload |
| Validate Payload | Code | Data Normalization | Webhook | Has Remote URL? | 1 — Intake & upload |
| Has Remote URL? | If | Routing | Validate Payload | Upload to URL (Remote/Binary) | 1 — Intake & upload |
| Upload to URL - Remote | UploadToURL | File Hosting | Has Remote URL? | Extract CDN URL | 1 — Intake & upload |
| Upload to URL - Binary | UploadToURL | File Hosting | Has Remote URL? | Extract CDN URL | 1 — Intake & upload |
| Extract CDN URL | Code | URL Normalization | Upload to URL nodes | AWS Textract | 1 — Intake & upload |
| AWS Textract | HTTP Request | AI OCR | Extract CDN URL | Parse & Validate | 2 — OCR extraction |
| Parse & Validate | Code | Data Extraction | AWS Textract | Sheets Search | 2 — OCR extraction |
| Sheets Search | Google Sheets | Duplicate Check | Parse & Validate | Resolve Write Action | 3 — Duplicate check & Sheets write |
| Resolve Write Action | Code | Decision Logic | Sheets Search | Route Write Action | 3 — Duplicate check & Sheets write |
| Route Write Action | Switch | Conditional Routing | Resolve Write Action | Append, Update, or Mark | 3 — Duplicate check & Sheets write |
| Sheets - Append | Google Sheets | Data Storage | Route Write Action | Merge Sheets Result | 3 — Duplicate check & Sheets write |
| Sheets - Update | Google Sheets | Data Storage | Route Write Action | Merge Sheets Result | 3 — Duplicate check & Sheets write |
| Mark as Duplicate | Code | Status Flagging | Route Write Action | Merge Sheets Result | 3 — Duplicate check & Sheets write |
| Merge Sheets Result | Code | Result Unification | Append, Update, or Mark | Slack | 4 — Notification & response |
| Slack - Notify | Slack | Communication | Merge Sheets Result | Build Final Response | 4 — Notification & response |
| Build Final Response | Code | Response Prep | Slack | Respond to Webhook | 4 — Notification & response |
| Respond to Webhook | Respond to Webhook | HTTP Finalization | Build Final Response | (None) | 4 — Notification & response |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Install the `n8n-nodes-uploadtourl` community node.
    *   Create a Google Sheet with columns: `Invoice No`, `Vendor`, `Amount`, `Currency`, `Due Date`, `Issue Date`, `Status`, `Extraction Status`, `Missing Fields`, `File URL`, `Department`, `Uploaded By`, `Received At`, `Notes`.
2.  **Inlet Configuration:**
    *   Create a **Webhook** node (`POST`, `invoice-processing`).
    *   Add a **Code** node ("Validate Payload") to parse the body/binary and enforce the extension allowlist.
    *   Add an **If** node to check for `fileUrl`. Connect it to two **UploadToURL** nodes (one for binary, one for URL).
    *   Add a **Code** node ("Extract CDN URL") to grab the final link from the upload response.
3.  **AI OCR Setup:**
    *   Create an **HTTP Request** node for AWS Textract. Configure it for `POST` to the Textract endpoint. Set headers for `X-Amz-Target: Textract.AnalyzeExpense`.
    *   Add a **Code** node ("Parse & Validate") to map the AWS `SummaryFields` to n8n JSON objects.
4.  **Database Logic:**
    *   Add a **Google Sheets** node ("Search") to find existing `Invoice No`.
    *   Add a **Code** node ("Resolve Write Action") to compare the search result with current extraction.
    *   Add a **Switch** node to route to **Append**, **Update**, or **Mark as Duplicate**.
5.  **Output Setup:**
    *   Add a **Code** node ("Merge Sheets Result") to unify variables regardless of which database path was taken.
    *   Add a **Slack** node with a block-formatted message including the CDN link.
    *   Add a **Code** node ("Build Final Response") and a **Respond to Webhook** node to return the processed data.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires n8n community node | [n8n-nodes-uploadtourl](https://www.npmjs.com/package/n8n-nodes-uploadtourl) |
| AWS Textract Documentation | [AnalyzeExpense API Reference](https://docs.aws.amazon.com/textract/latest/dg/API_AnalyzeExpense.html) |
| Global Variables Required | `GSHEET_SPREADSHEET_ID`, `GSHEET_SHEET_NAME`, `TEXTRACT_S3_BUCKET` (optional) |