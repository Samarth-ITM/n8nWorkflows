Validate customs clearance documents with Claude AI, Google Drive and Slack

https://n8nworkflows.xyz/workflows/validate-customs-clearance-documents-with-claude-ai--google-drive-and-slack-13691


# Validate customs clearance documents with Claude AI, Google Drive and Slack

# AI Customs Clearance Document Checker: Technical Reference

### 1. Workflow Overview
The **AI Customs Clearance Document Checker** is an automated system designed to validate international shipping documentation before dispatch. By leveraging Claude AI and structured data extraction, the workflow identifies inconsistencies across multiple documents (e.g., Invoice vs. Packing List), verifies HS Code accuracy, and checks compliance against destination-specific regulations.

The workflow is organized into four functional blocks:
1.  **Shipment Intake & Document Retrieval:** Captures shipment metadata and downloads files from cloud storage.
2.  **Document Classification & Consistency Check:** Extracts text, identifies document types, and cross-references values (weights, amounts) between files.
3.  **Claude AI Compliance Validation:** Performs a deep legal and regulatory review using LLMs.
4.  **Risk Routing & Automated Action:** Distributes reports via Slack, Email, and Jira, and logs the audit trail in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Shipment Intake & Document Retrieval
**Overview:** This block acts as the entry point, normalizing data from different sources and preparing a list of document objects to be processed.
*   **Nodes Involved:** `Receive Shipment Documents`, `Watch Shipment Docs Folder`, `Register Shipment Case`.
*   **Node Details:**
    *   **Receive Shipment Documents (Webhook):** Listens for POST requests containing shipment metadata and Drive IDs.
    *   **Watch Shipment Docs Folder (Google Drive Trigger):** Polling trigger for new files in a specific intake folder.
    *   **Register Shipment Case (Code):** Normalizes inputs into a consistent JSON structure. It generates a `shipmentId` (if missing) and maps each document into an array for iterative processing.

#### 2.2 Block 2: Text Extraction & Initial Analysis
**Overview:** Converts binary files into machine-readable text and uses heuristic logic to classify document types and extract key fields.
*   **Nodes Involved:** `Fetch Document from Drive`, `Extract Text from Document`, `Classify Type & Extract Key Fields`, `Cross-Document Consistency Engine`.
*   **Node Details:**
    *   **Fetch Document from Drive (Google Drive):** Downloads the file content using the `driveFileId`.
    *   **Extract Text from Document (Extract from File):** Uses OCR/Text extraction to pull raw content from PDFs.
    *   **Classify Type & Extract Key Fields (Code):** Uses RegEx patterns to identify document types (e.g., COMMERCIAL_INVOICE, MSDS) and extracts variables like `grossWeight`, `hsCode`, and `declaredValue`.
    *   **Cross-Document Consistency Engine (Code):** A critical logic gate that compares extracted values across all documents in the shipment. It flags a `HIGH` severity mismatch if, for example, the invoice value differs from the shipment header.

#### 2.3 Block 3: Claude AI Compliance Validation
**Overview:** The core intelligence layer where document text is analyzed against international trade laws.
*   **Nodes Involved:** `AI Customs Compliance Validator`, `Claude AI Model`, `Parse AI Compliance Results`.
*   **Node Details:**
    *   **AI Customs Compliance Validator (Agent):** Orchestrates the prompt. It provides Claude with shipment context, extracted fields, and destination-specific rules (AU, US, EU, UK).
    *   **Claude AI Model (Anthropic):** Specifically utilizes `claude-sonnet-4-20250514` with a low temperature (0.05) to ensure deterministic, structured JSON output.
    *   **Parse AI Compliance Results (Code):** Sanitizes the AI output, handling potential markdown formatting or parsing errors to ensure the downstream nodes receive valid JSON.

#### 2.4 Block 4: Risk Routing & Multi-Channel Reporting
**Overview:** Aggregates findings and executes business logic based on the calculated risk level.
*   **Nodes Involved:** `Aggregate Shipment Validation Report`, `Route by Shipment Risk Level`, `Alert Logistics Team on Slack`, `Email Validation Report to Exporter`, `Update Shipment Tracker in Sheets`, `Create Jira Compliance Issue`, `Build Final Compliance Response`, `Return Compliance Result to Caller`.
*   **Node Details:**
    *   **Aggregate Shipment Validation Report (Code):** Merges individual document results into a single shipment-level summary, calculating `overallStatus` (CLEAR, HOLD, or REJECT).
    *   **Route by Shipment Risk Level (Switch):** Directs the flow based on the `overallStatus`.
    *   **Alert Logistics Team on Slack (HTTP Request):** Sends formatted blocks to `#customs-critical` or `#customs-review` channels.
    *   **Create Jira Compliance Issue (HTTP Request):** Opens tickets for REJECT or HOLD statuses in the `CUSTOMS` project.
    *   **Email Validation Report (Email Send):** Generates a high-quality HTML report for the exporter with specific remediation steps.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Shipment Documents | Webhook | Webhook Entry | None | Register Shipment Case | 1. Shipment Intake & Document Retrieval |
| Watch Shipment Docs Folder | Google Drive Trigger | Polling Entry | None | Register Shipment Case | 1. Shipment Intake & Document Retrieval |
| Register Shipment Case | Code | Data Normalization | Webhook / Drive Trigger | Fetch Document from Drive | 1. Shipment Intake & Document Retrieval |
| Fetch Document from Drive | Google Drive | File Download | Register Shipment Case | Extract Text from Document | 1. Shipment Intake & Document Retrieval |
| Extract Text from Document | Extract from File | OCR / PDF Parsing | Fetch Document from Drive | Classify Type & Extract Key Fields | 1. Shipment Intake & Document Retrieval |
| Classify Type & Extract Key Fields | Code | Heuristic Analysis | Extract Text from Document | Cross-Document Consistency Engine | 2. Document Classification & Consistency Check |
| Cross-Document Consistency Engine | Code | Cross-Referencing | Classify Type | AI Customs Compliance Validator | 2. Document Classification & Consistency Check |
| AI Customs Compliance Validator | AI Agent | Compliance Audit | Consistency Engine | Parse AI Compliance Results | 3. Claude AI Customs Compliance Validation |
| Claude AI Model | Anthropic Chat Model | LLM Provider | None | AI Customs Compliance Validator | 3. Claude AI Customs Compliance Validation |
| Parse AI Compliance Results | Code | JSON Sanitization | AI Validator | Aggregate Shipment Validation Report | 3. Claude AI Customs Compliance Validation |
| Aggregate Shipment Validation Report | Code | Global Summary | Parse AI Results | Route by Shipment Risk Level | 4. Risk Routing · Notifications · Tracker Update |
| Route by Shipment Risk Level | Switch | Logic Branching | Aggregate Report | Slack, Email, Jira, Sheets | 4. Risk Routing · Notifications · Tracker Update |
| Alert Logistics Team on Slack | HTTP Request | Team Alerting | Risk Routing | Build Final Compliance Response | 4. Risk Routing · Notifications · Tracker Update |
| Email Validation Report to Exporter | Email Send | External Reporting | Risk Routing | Build Final Compliance Response | 4. Risk Routing · Notifications · Tracker Update |
| Update Shipment Tracker in Sheets | Google Sheets | Audit Logging | Risk Routing | Build Final Compliance Response | 4. Risk Routing · Notifications · Tracker Update |
| Create Jira Compliance Issue | HTTP Request | Ticket Creation | Risk Routing | Build Final Compliance Response | 4. Risk Routing · Notifications · Tracker Update |
| Build Final Compliance Response | Code | Response Prep | Actions (Slack/Email/Jira/Sheets) | Return Compliance Result | 4. Risk Routing · Notifications · Tracker Update |
| Return Compliance Result to Caller | Webhook Response | API Sync | Build Final Response | None | 4. Risk Routing · Notifications · Tracker Update |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Webhook** node (POST) for external TMS integrations.
    *   Create a **Google Drive Trigger** node set to "File Created" in a specific folder.
2.  **Normalization Logic:**
    *   Add a **Code** node. Use JavaScript to map incoming data to a standard object containing `shipmentId`, `destinationCountry`, and an array of `documents` (with names and Drive IDs).
3.  **File Processing Loop:**
    *   Add a **Google Drive** node (Download operation) using the ID from the previous step.
    *   Add an **Extract from File** node (PDF/Text mode).
4.  **Extraction & Consistency:**
    *   Add a **Code** node. Implement RegEx to find document types and specific fields (Invoice No, Weight, HS Code).
    *   Add a second **Code** node to compare these fields across all items in the incoming array.
5.  **AI Integration:**
    *   Add an **AI Agent** node. Set the prompt to act as a "Licensed Customs Broker".
    *   Connect an **Anthropic Chat Model** node (Claude 3.5 Sonnet).
    *   Add a **Code** node to parse the AI's string response back into a JSON object.
6.  **Aggregation & Routing:**
    *   Add a **Code** node to determine the "Overall Status" (Reject if any document is rejected or if mandatory docs like "Packing List" are missing).
    *   Add a **Switch** node with outputs for `CLEAR`, `HOLD`, and `REJECT`.
7.  **Output Channels:**
    *   **Slack:** HTTP Request to `chat.postMessage` using Block Kit for visual reports.
    *   **Jira:** HTTP Request to the Jira REST API to create issues for `HOLD/REJECT`.
    *   **Email:** Email node (SMTP or SendGrid) with an HTML template.
    *   **Sheets:** Google Sheets node (Append) for long-term audit logs.
8.  **Completion:**
    *   Add a **Respond to Webhook** node to return the structured validation report to the caller.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Custom AI automation design services | [Contact OneClick IT Solution](https://www.oneclickitsolution.com/contact-us/) |
| High-severity mismatch flags | Triggered automatically on `declaredValue`, `hsCode`, or `currency` discrepancies. |
| Supported Documents | Invoice, Packing List, B/L, COO, MSDS, Fumigation Cert, etc. |