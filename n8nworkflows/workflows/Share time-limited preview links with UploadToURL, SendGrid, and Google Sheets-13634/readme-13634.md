Share time-limited preview links with UploadToURL, SendGrid, and Google Sheets

https://n8nworkflows.xyz/workflows/share-time-limited-preview-links-with-uploadtourl--sendgrid--and-google-sheets-13634


# Share time-limited preview links with UploadToURL, SendGrid, and Google Sheets

# Reference Document: Share Time-Limited Preview Links Workflow

This document provides a technical breakdown of an n8n workflow designed to create "burner links" for sensitive file sharing. It automates file hosting, secure link delivery, activity logging, and automatic expiration.

---

### 1. Workflow Overview

The **Share Time-Limited Preview Links** workflow addresses the security risks of permanent email attachments. By hosting files temporarily and automating their "expiration" notice, it provides a controlled environment for sharing agency drafts or sensitive documents.

**Logical Blocks:**
*   **1.1 Input & Validation:** Receives file data (Binary or URL) via Webhook and validates parameters.
*   **1.2 Asset Hosting:** Uses the `UploadToURL` community node to generate a public CDN link.
*   **1.3 Record Generation:** Calculates expiry timestamps and pre-renders HTML email templates.
*   **1.4 Logging & Delivery:** Records the active link in Google Sheets and sends the branded preview email via SendGrid.
*   **1.5 Lifecycle Management:** Pauses execution for the specified duration, then updates the status to "Expired" and notifies both the client and the agency.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Validation
This block ensures the incoming data is formatted correctly and safe to process.
*   **Nodes Involved:** `Webhook - Create Burner Link`, `Validate Payload`.
*   **Node Details:**
    *   **Webhook:** Listens for POST requests. Returns a response via the `Respond to Webhook` node later in the flow to avoid timing out the caller.
    *   **Validate Payload (Code Node):** 
        *   Checks for required fields (`recipientEmail`, `fileUrl` or `filename`).
        *   Clamps expiry duration between 1 and 168 hours (7 days).
        *   Sanitizes strings and validates file extensions (PDF, JPG, PNG, DOCX, etc.).
    *   **Edge Cases:** Throws an error if the email format is invalid or if the file extension is not supported.

#### 2.2 Asset Hosting
Determines the source of the file and uploads it to a public CDN.
*   **Nodes Involved:** `Has Remote URL?`, `Upload to URL - Remote`, `Upload to URL - Binary`, `Extract CDN URL`.
*   **Node Details:**
    *   **Has Remote URL? (If Node):** Routes the workflow based on whether a `fileUrl` was provided or if the file is a binary upload.
    *   **Upload to URL (Community Node):** Specifically uses the `n8n-nodes-uploadtourl` node.
    *   **Extract CDN URL (Code Node):** Normalizes the response from the upload provider to ensure a valid HTTPS URL and captures file metadata (size, ID).

#### 2.3 Record Generation & Logging
Prepares the database entry and all communication assets before delivery.
*   **Nodes Involved:** `Generate Link Record & Email HTML`, `Sheets - Log Active Link`.
*   **Node Details:**
    *   **Generate Link Record (Code Node):** 
        *   Creates a unique hex token (e.g., `TIME-RANDOM`).
        *   Generates three distinct HTML strings: the **Preview Email**, the **Client Expiry Notice**, and the **Agency Summary**.
        *   Calculates `waitMs` for the downstream Wait node.
    *   **Sheets - Log Active Link:** Appends a new row to a Google Sheet with status `active`. Using `append` ensures a record exists even if the following email node fails.

#### 2.4 Delivery & Response
Communicates the link to the recipient and closes the initial HTTP request.
*   **Nodes Involved:** `SendGrid - Send Preview Email`, `Respond to Webhook`.
*   **Node Details:**
    *   **SendGrid:** Sends the `previewEmailHtml` to the client.
    *   **Respond to Webhook:** Returns the CDN URL and token to the API caller immediately so the client/app doesn't wait for the expiration period to end.

#### 2.5 Lifecycle Management (Wait & Expire)
Handles the "burn" logic after the set duration.
*   **Nodes Involved:** `Wait - Until Link Expires`, `Sheets - Mark Link Expired`, `SendGrid - Notify Client Expired`, `SendGrid - Agency Summary Email`.
*   **Node Details:**
    *   **Wait Node:** Pauses execution using the `waitMs` variable. 
    *   **Sheets - Mark Link Expired:** Updates the existing row (matched by Token) to status `expired`.
    *   **SendGrid (x2):** Sends the final notifications. One informs the client the link is gone; the other provides the agency with a full lifecycle audit.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - Create Burner Link | Webhook | Entry Point | None | Validate Payload | 1 — Upload & link generation |
| Validate Payload | Code | Data Sanitization | Webhook | Has Remote URL? | 1 — Upload & link generation |
| Has Remote URL? | If | Routing | Validate Payload | Upload to URL (Remote/Binary) | 1 — Upload & link generation |
| Upload to URL - Remote | UploadToURL | File Hosting | Has Remote URL? | Extract CDN URL | 1 — Upload & link generation |
| Upload to URL - Binary | UploadToURL | File Hosting | Has Remote URL? | Extract CDN URL | 1 — Upload & link generation |
| Extract CDN URL | Code | URL Normalization | Upload to URL | Generate Link Record | 1 — Upload & link generation |
| Generate Link Record & Email HTML | Code | Data Preparation | Extract CDN URL | Sheets - Log Active Link | 1 — Upload & link generation |
| Sheets - Log Active Link | Google Sheets | DB Logging | Generate Link Record | SendGrid - Send Preview | 2 — Log & send |
| SendGrid - Send Preview Email | SendGrid | Client Communication | Sheets - Log Active | Respond to Webhook | 2 — Log & send |
| Respond to Webhook | Respond to Webhook | API Response | SendGrid - Send Preview | Wait - Until Link Expires | |
| Wait - Until Link Expires | Wait | Delay Logic | Respond to Webhook | Sheets - Mark Link Expired | ⚙️ Wait node note |
| Sheets - Mark Link Expired | Google Sheets | DB Update | Wait | SendGrid (Client/Agency) | 3 — Wait & expire |
| SendGrid - Notify Client Expired | SendGrid | Expiry Notice | Sheets - Mark Link | None | 3 — Wait & expire |
| SendGrid - Agency Summary Email | SendGrid | Audit Log | Sheets - Mark Link | None | 3 — Wait & expire |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Install the `n8n-nodes-uploadtourl` community node.
    *   Create a Google Sheet named `BurnerLinks` with headers: `Token`, `Sender`, `Status`, `CDN URL`, `Sent At`, `File Name`, `Expires At`, `Expiry Hours`, `Project Name`, `Recipient Name`, `File Size Bytes`, `Recipient Email`, `Expired At`.
    *   Set global variables: `GSHEET_SPREADSHEET_ID` and `DEFAULT_EXPIRY_HOURS`.

2.  **Input Configuration:**
    *   Create a **Webhook** node (POST) with path `burner-link`.
    *   Add a **Code** node ("Validate Payload") to parse the body, validate the email regex, and clamp the expiry hours between 1 and 168.

3.  **Upload Logic:**
    *   Add an **If** node to check if `fileUrl` exists.
    *   Connect both branches to the **UploadToURL** node (operation: `uploadFile`).
    *   Add a **Code** node ("Extract CDN URL") to find the URL in the varying response structures of the upload node.

4.  **Template Generation:**
    *   Add a **Code** node to generate a unique token using `Math.random()` and `Date.now()`.
    *   In the same node, write the HTML templates for the three emails using template literals, incorporating the `cdnUrl` and `expiresAt` variables.

5.  **Logging & Initial Send:**
    *   Add a **Google Sheets** node (Append) to log the initial "active" state.
    *   Add a **SendGrid** node to send the first HTML email.
    *   Add a **Respond to Webhook** node (Response Code 201) to acknowledge the request.

6.  **Automation of Expiry:**
    *   Add a **Wait** node set to "milliseconds" using the expression `{{ $json.waitMs }}`.
    *   Add a **Google Sheets** node (Update) to change the status to "expired" based on the Token.
    *   Add two **SendGrid** nodes to send the final notification emails.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Community Node Installation | Search for `n8n-nodes-uploadtourl` in n8n settings. |
| Execution Persistence | Ensure "Save executions" is enabled on self-hosted instances for the Wait node to survive restarts. |
| Supported Extensions | pdf, jpg, jpeg, png, docx, pptx, mp4, zip. |