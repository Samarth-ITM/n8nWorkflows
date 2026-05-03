Fast-track expense reimbursements with UploadToURL, Mindee OCR, Expensify, and Slack

https://n8nworkflows.xyz/workflows/fast-track-expense-reimbursements-with-uploadtourl--mindee-ocr--expensify--and-slack-13636


# Fast-track expense reimbursements with UploadToURL, Mindee OCR, Expensify, and Slack

# Workflow Reference: Fast-track Expense Reimbursements

This document provides a technical breakdown of an automated expense reimbursement pipeline. The workflow leverages **Mindee OCR** for data extraction, **UploadToURL** for file hosting, **Expensify** for accounting, and **Slack** for human-in-the-loop approvals.

---

### 1. Workflow Overview

The workflow automates the transition from a raw receipt image to a structured expense entry. It categorizes logic into four functional phases:

*   **1.1 Input & Hosting:** Captures employee data and receipt files via an n8n Form, validates inputs, and hosts the file on a CDN.
*   **1.2 AI Data Extraction:** Uses Mindee OCR to extract merchant name, total, date, and tax, while calculating a custom confidence score.
*   **1.3 Intelligent Routing:** Routes expenses based on a threshold (e.g., $50) and OCR confidence. Low-risk items are auto-approved; high-risk items trigger a Slack approval request.
*   **1.4 Finalization & Audit:** Records the transaction in Expensify and Google Sheets, and notifies the employee via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Upload
**Overview:** Captures receipt data and ensures the file is accessible via a public URL for downstream APIs.
*   **Nodes:** 
    *   *Form Trigger - Submit Receipt*: Collects Name, Email, Manager Email, Category, and File.
    *   *Validate & Build Expense Record*: Sanitizes input, validates email formats, checks file extensions (JPG, PNG, PDF, HEIC), and sets the auto-approval threshold from an environment variable.
    *   *Has Remote URL?*: Determines if the file was provided as a link or a binary upload.
    *   *Upload to URL (Remote/Binary)*: Community node that hosts the file.
    *   *Extract CDN URL*: Normalizes the hosting response into a standard HTTPS CDN link.

#### 2.2 OCR Processing & Scoring
**Overview:** Processes the image through AI and evaluates the reliability of the extracted data.
*   **Nodes:**
    *   *Mindee - Extract Receipt Data*: Calls the Mindee v5 API. It sends the hosted receipt URL and receives structured JSON.
    *   *Parse OCR & Score Confidence*: A logic node that calculates a weighted confidence score: **Total (40%), Merchant (30%), Date (20%), Tax (10%)**. It flags missing fields and decides if the record is eligible for auto-approval.

#### 2.3 Decision Logic & Approval
**Overview:** Branches the workflow based on the "Confidence Gate."
*   **Nodes:**
    *   *Confidence Gate — Auto or Manager?*: An IF node checking the `approvalRoute` variable.
    *   *Mark Auto-Approved*: Sets status to `approved` and calculates a reimbursement ETA (3 days for <$25, 5 days otherwise).
    *   *Slack - Manager Approval Request*: Sends an interactive message to the manager with a "View Receipt" link and Approve/Reject buttons.
    *   *Mark Pending Manager Review*: Updates status to `pending_manager`. *Note: A secondary webhook workflow is required to handle the Slack button interaction.*

#### 2.4 Accounting & Notification
**Overview:** Finalizes the expense in external systems and informs the user.
*   **Nodes:**
    *   *Expensify - Create Expense Entry*: Uses the Expensify Integration Server API to create a record. It converts amounts to cents and attaches the CDN URL as the receipt proof.
    *   *Merge Approval + Expensify Result*: Combines the original data with the new Expensify Report ID.
    *   *Sheets - Append Audit Row*: Logs all data (including OCR confidence and raw merchant data) into a master Google Sheet.
    *   *Gmail - Employee Confirmation*: Sends a branded HTML email to the employee with the status and ETA.
    *   *Respond to Form*: Provides immediate feedback on the form submission page.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Form Trigger** | formTrigger | Entry Point | None | Validate... | 1 — Form intake & upload |
| **Validate & Build...** | code | Data Validation | Form Trigger | Has Remote URL? | 1 — Form intake & upload |
| **Has Remote URL?** | if | Routing | Validate... | Upload to URL (x2) | 1 — Form intake & upload |
| **Upload to URL** | uploadToUrl | File Hosting | Has Remote URL? | Extract CDN URL | 1 — Form intake & upload |
| **Extract CDN URL** | code | Data Normalization | Upload to URL | Mindee OCR | 1 — Form intake & upload |
| **Mindee OCR** | httpRequest | AI Extraction | Extract CDN URL | Parse OCR... | 2 — Mindee OCR & confidence gate |
| **Parse OCR & Score** | code | Logic/Scoring | Mindee OCR | Confidence Gate | 2 — Mindee OCR & confidence gate |
| **Confidence Gate** | if | Branching | Parse OCR... | Mark Auto / Slack | 2 — Mindee OCR & confidence gate |
| **Mark Auto-Approved** | code | Status Update | Confidence Gate | Expensify | 3 — Dual-path approval |
| **Slack Approval** | slack | Human Interaction | Confidence Gate | Mark Pending | 3 — Dual-path approval |
| **Mark Pending...** | code | Status Update | Slack Approval | Expensify | 3 — Dual-path approval |
| **Expensify Entry** | httpRequest | Accounting | Mark Auto / Mark Pending | Merge Results | 3 — Dual-path approval |
| **Merge Results** | code | Data Merging | Expensify Entry | Sheets | 4 — Audit log & confirmation |
| **Sheets Audit** | googleSheets | Logging | Merge Results | Gmail | 4 — Audit log & confirmation |
| **Gmail Confirmation**| gmail | Notification | Sheets Audit | Respond to Form | 4 — Audit log & confirmation |
| **Respond to Form** | respondToWebhook| UI Response | Gmail | None | 4 — Audit log & confirmation |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Install the `n8n-nodes-uploadtourl` community node.
    *   Define the following n8n variables:
        *   `AUTO_APPROVE_THRESHOLD`: e.g., `50`
        *   `EXPENSIFY_POLICY_ID`: Your Expensify policy ID.
        *   `GSHEET_SPREADSHEET_ID`: The ID of your audit Google Sheet.

2.  **Trigger Configuration:**
    *   Create an **n8n Form Trigger**. Add fields for Name, Email, Manager Email, Category (Dropdown), Purpose, and File.

3.  **Data Processing Logic:**
    *   Add a **Code Node** to validate the email and file extension.
    *   Add an **IF Node** to check if the file is a URL or a binary upload.
    *   Add **UploadToURL Nodes** for both branches to get a public CDN link.

4.  **OCR Integration:**
    *   Add an **HTTP Request Node** for Mindee. Use `POST` to `https://api.mindee.net/v1/products/mindee/expense_receipts/v5/predict`. 
    *   Set Header `Authorization: Token [Your_Mindee_API_Key]`.
    *   Use a **Code Node** to parse the Mindee output. Implement the weighting logic: `(totalConf * 0.4) + (merchantConf * 0.3) + (dateConf * 0.2) + (taxConf * 0.1)`.

5.  **Branching & Approvals:**
    *   Add an **IF Node** to check if `confidenceScore >= 0.85` AND `total <= AUTO_APPROVE_THRESHOLD`.
    *   **True Path:** Use a Code node to set `approvalStatus = 'approved'`.
    *   **False Path:** Use the **Slack Node** to send a block message with interactive buttons to the `managerEmail`. Follow it with a Code node setting `approvalStatus = 'pending_manager'`.

6.  **Accounting & Logging:**
    *   Add an **HTTP Request Node** for Expensify using `form-urlencoded` format. Send the `requestJobDescription` JSON containing the receipt URL and amount (in cents).
    *   Add a **Google Sheets Node** to append a row to your audit log.
    *   Add a **Gmail Node** to send the final HTML summary to the employee.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Community Node Required** | This workflow requires `n8n-nodes-uploadtourl` to be installed via the "Community Nodes" settings. |
| **Expensify Cent Calculation** | Expensify API requires amounts in cents. The workflow uses `Math.round(total * 100)`. |
| **Interactive Slack Handlers** | To make the Slack "Approve" buttons work, a separate workflow with a Slack Trigger (Interaction) is required to listen for the button clicks. |
| **Mindee API Documentation** | [Mindee Expense Receipts V5](https://developers.mindee.com/docs/expense-receipt-ocr) |