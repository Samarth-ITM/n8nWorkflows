Validate property documents with Claude (Anthropic), Google Drive and Slack

https://n8nworkflows.xyz/workflows/validate-property-documents-with-claude--anthropic---google-drive-and-slack-13688


# Validate property documents with Claude (Anthropic), Google Drive and Slack

This document provides a comprehensive technical reference for the **Automated Property Document Validator** workflow in n8n.

---

### 1. Workflow Overview

This workflow automates the ingestion, analysis, and validation of property-related legal documents (e.g., Contracts of Sale, Title Searches, Certificates). It leverages Anthropic's Claude AI to perform jurisdiction-specific compliance checks, identifies missing mandatory documents, and routes the results to relevant stakeholders through email, Slack, and Google Sheets.

#### Logical Blocks:
*   **1.1 Intake & Registration:** Receives data via Webhook or Google Drive and normalizes metadata into a standardized "Case."
*   **1.2 Document Processing:** Downloads files, extracts text, and performs initial keyword-based classification.
*   **1.3 AI Compliance Analysis:** Uses a LangChain-powered agent with Claude AI to evaluate document text against legal requirements.
*   **1.4 Result Aggregation:** Consolidates findings from multiple documents and compares them against a mandatory checklist.
*   **1.5 Routing & Notification:** Executes conditional actions based on the compliance status (PASS/FAIL/REVIEW).
*   **1.6 Response & Audit:** Returns a structured JSON response to the caller and logs the transaction in an audit trail.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Registration
This block handles the entry points and ensures every submission has a unique Case ID.
*   **Nodes:** `Receive Document Submission` (Webhook), `Watch Drive Intake Folder` (Google Drive Trigger), `Register Validation Case` (Code).
*   **Node Details:**
    *   **Register Validation Case:** A Code node that merges input from either the Webhook or Drive. It generates a `caseId` (if missing), calculates a `runId`, and splits the input so that each document is processed as an individual item in the subsequent nodes.
    *   **Edge Cases:** Handles missing document arrays by treating a single file trigger as a list of one.

#### 2.2 Retrieval, Extraction & Classification
Converts binary files into structured text and identifies the type of document.
*   **Nodes:** `Download Document from Drive`, `Extract Document Text`, `Classify & Prepare Document` (Code).
*   **Node Details:**
    *   **Download Document from Drive:** Uses the `driveFileId` parsed in the registration step.
    *   **Extract Document Text:** Specifically configured for PDF extraction to retrieve the raw text layer.
    *   **Classify & Prepare Document:** A Code node that uses keyword matching (e.g., "Section 32", "Folio Identifier") to determine the document type. It also uses RegEx to find potential expiry dates and prepares the text for AI analysis by limiting it to the first 12,000 characters.

#### 2.3 AI Legal Compliance Analysis
The "brain" of the workflow where the legal review occurs.
*   **Nodes:** `AI Legal Compliance Check` (AI Agent), `Claude AI Model` (Anthropic Chat Model), `Parse AI Validation Result` (Code).
*   **Node Details:**
    *   **AI Legal Compliance Check:** A LangChain Agent using a system prompt that defines specific validation rules for AU jurisdictions (NSW, VIC, QLD, etc.). It is instructed to return strictly JSON.
    *   **Claude AI Model:** Uses `claude-sonnet-4-20250514` with a temperature of 0.05 for high precision and low creativity.
    *   **Parse AI Validation Result:** Cleans the AI's string output, removes markdown formatting, and converts it into a valid JSON object. It includes a fallback mechanism for "INCOMPLETE_DATA" if parsing fails.

#### 2.4 Result Aggregation & Logic
Consolidates the per-document AI results into a single Case report.
*   **Nodes:** `Aggregate All Document Results` (Code), `Route by Compliance Status` (Switch).
*   **Node Details:**
    *   **Aggregate All Document Results:** Compares the submitted documents against a hardcoded `MANDATORY_DOCS` list (based on state/jurisdiction). It calculates the `overallStatus` (PASS/FAIL/REQUIRES_REVIEW) and compiles a global list of remediation steps.
    *   **Route by Compliance Status:** Directs the workflow flow based on the `overallStatus`.

#### 2.5 Notifications & Auditing
Handles external communications and data persistence.
*   **Nodes:** `Email Validation Report to Submitter`, `Alert Legal Team on Slack`, `Write Compliance Audit Record`, `Mark Case Auto-Approved`.
*   **Node Details:**
    *   **Email Validation Report:** Sends a dynamic HTML email containing a summary of passed/failed checks and remediation steps.
    *   **Alert Legal Team on Slack:** Uses a POST request to the Slack API to send structured "Blocks." It routes messages to `#legal-critical` if the status is FAIL.
    *   **Write Compliance Audit Record:** Appends all case data to a Google Sheet for long-term tracking.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Document Submission | Webhook | Intake (API) | None | Register Validation Case | ## 1. Intake, Registration & File Retrieval |
| Watch Drive Intake Folder | Google Drive Trigger | Intake (Cloud) | None | Register Validation Case | ## 1. Intake, Registration & File Retrieval |
| Register Validation Case | Code | Data Normalization | Webhook / Drive | Download Document | ## 1. Intake, Registration & File Retrieval |
| Download Document from Drive | Google Drive | File Retrieval | Register Case | Extract Document Text | ## 2. Document Extraction & Type Classification |
| Extract Document Text | Extract From File | OCR/Text Extraction | Download Document | Classify & Prepare | ## 2. Document Extraction & Type Classification |
| Classify & Prepare Document | Code | Pre-processing | Extract Text | AI Compliance Check | ## 2. Document Extraction & Type Classification |
| AI Legal Compliance Check | AI Agent | Legal Logic | Classify & Prepare | Parse AI Result | ## 3. Claude AI Legal Compliance Analysis |
| Claude AI Model | Anthropic Chat | LLM Provider | None | AI Compliance Check | ## 3. Claude AI Legal Compliance Analysis |
| Parse AI Validation Result | Code | Data Formatting | AI Compliance Check | Aggregate Results | ## 3. Claude AI Legal Compliance Analysis |
| Aggregate All Document Results | Code | Case Logic | Parse AI Result | Route Status | ## 4. Aggregation... |
| Route by Compliance Status | Switch | Branching | Aggregate Results | Email, Slack, Sheets | ## 4. Aggregation... |
| Email Validation Report | Email Send | External Notify | Route Status | Final Response | ## 4. Aggregation... |
| Alert Legal Team on Slack | HTTP Request | Internal Notify | Route Status | Final Response | ## 4. Aggregation... |
| Write Compliance Audit Record | Google Sheets | Data Logging | Route Status | Final Response | ## 4. Aggregation... |
| Mark Case Auto-Approved | Set | Status Update | Route Status | None | ## 4. Aggregation... |
| Build Final Validation Response | Code | Result Formatting | Email, Slack, Sheets | Return to Caller | ## 4. Aggregation... |
| Return Validation Result | Respond to Webhook | API Response | Build Final Response | None | ## 4. Aggregation... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Triggers:** Create a **Webhook** node (POST) and a **Google Drive Trigger** node. Set the Drive trigger to watch a specific folder for "File Created" events.
2.  **Case Normalization:** Add a **Code** node. Use it to ensure that whether the data comes from a webhook (array of files) or a single file trigger, it results in an array of items where each item contains both `case` metadata and `document` specific data.
3.  **File Handling:**
    *   Add a **Google Drive** node set to "Download". Link the File ID to the expression from the Code node.
    *   Add an **Extract from File** node. Set operation to "PDF" (or auto-detect).
4.  **Classification:** Add a **Code** node to perform string matching on the extracted text to identify document types (e.g., "CONTRACT_OF_SALE") and extract dates via RegEx.
5.  **AI Integration:**
    *   Add an **AI Agent** node. Select "Define Below" for the prompt.
    *   Connect an **Anthropic Chat Model** node to the Agent. Set the model to `claude-sonnet` and temperature to `0.05`.
    *   *Prompt Setup:* The system prompt must define the expected JSON schema and the legal rules for each document type (Contracts, Titles, etc.).
6.  **Data Cleaning:** Add a **Code** node after the AI Agent to parse the JSON response string and handle potential AI formatting errors (markdown backticks).
7.  **Aggregation:** Add a **Code** node that uses the `.all()` method to wait for all document items to finish. Compare the list of detected types against a "Mandatory Checklist" (e.g., NSW Sale requires a Contract and a Title).
8.  **Routing:** Add a **Switch** node checking the `overallStatus`.
9.  **Outputs:**
    *   **Email:** Use the **Email Send** node with HTML to format a report.
    *   **Slack:** Use the **HTTP Request** node to POST to Slack's `chat.postMessage` endpoint.
    *   **Sheets:** Use the **Google Sheets** node (Append) to log the Case ID and status.
    *   **Webhook Response:** Add a **Respond to Webhook** node to close the initial POST request with the final validation JSON.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Contact for custom AI property automation | [Contact OneClick IT Solution](https://www.oneclickitsolution.com/contact-us/) |
| Jurisdiction Support | Configured for Australian states (NSW, VIC, QLD, SA, WA). |
| Document Limitations | AI analysis is limited to the first 12,000 characters to manage token limits. |
| Compliance Disclaimer | Automated validation is not a substitute for professional legal advice. |