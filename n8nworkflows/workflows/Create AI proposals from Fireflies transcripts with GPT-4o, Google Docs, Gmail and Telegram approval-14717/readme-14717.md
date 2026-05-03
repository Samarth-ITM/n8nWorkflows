Create AI proposals from Fireflies transcripts with GPT-4o, Google Docs, Gmail and Telegram approval

https://n8nworkflows.xyz/workflows/create-ai-proposals-from-fireflies-transcripts-with-gpt-4o--google-docs--gmail-and-telegram-approval-14717


# Create AI proposals from Fireflies transcripts with GPT-4o, Google Docs, Gmail and Telegram approval

# Workflow Documentation: Fireflies Meeting Transcript to Auto-Proposal

## 1. Workflow Overview
This workflow automates the transition from a recorded sales meeting to a professional PDF proposal. It monitors Fireflies.ai for new transcripts, uses GPT-4o to determine if a proposal is necessary, generates the content, populates a Google Doc template, and manages a human-in-the-loop approval process via Telegram before sending the final document to the client via Gmail.

### Logical Blocks
- **1.1 Transcript Intake:** Receives the webhook from Fireflies, acknowledges receipt, and fetches the full meeting transcript via GraphQL.
- **1.2 AI Qualification:** Analyzes the transcript to decide if the meeting was a sales/discovery call requiring a proposal and extracts key client data.
- **1.3 Proposal Drafting & Document Generation:** Generates professional proposal text, creates a personalized copy of a Google Doc template, and exports it as a PDF.
- **1.4 Telegram Approval Flow:** Notifies the user of the ready proposal and waits for an "Approve" or "Reject" action.
- **1.5 Email Delivery:** Ensures the client's email is present (prompting the user via Telegram if missing) and emails the PDF.

---

## 2. Block-by-Block Analysis

### 2.1 Transcript Intake
**Overview:** Handles the initial trigger and data retrieval from Fireflies.ai.
- **Nodes Involved:** `When Transcript Ready`, `Respond 200 OK`, `Get Full Transcript`, `If Transcript Exists`, `Notify Transcript Error`.
- **Node Details:**
    - **When Transcript Ready (Webhook):** Listens for POST requests from Fireflies.
    - **Respond 200 OK (Respond to Webhook):** Immediately sends a JSON confirmation to Fireflies to prevent webhook timeouts.
    - **Get Full Transcript (HTTP Request):** Performs a GraphQL POST request to `https://api.fireflies.ai/graphql`. It uses a query to fetch `title`, `participants`, `sentences` (transcript), and `summary`.
    - **If Transcript Exists (If):** Checks if `data.transcript` exists in the API response.
    - **Notify Transcript Error (Telegram):** Sends a warning if the transcript could not be retrieved.

### 2.2 AI Qualification
**Overview:** Determines if the meeting is worth a proposal and extracts entities.
- **Nodes Involved:** `Classify Meeting Type`, `If Proposal Needed`, `Notify No Proposal Needed`.
- **Node Details:**
    - **Classify Meeting Type (OpenAI):** Uses GPT-4o to analyze the transcript. It returns a structured JSON including `needs_proposal` (boolean), `client_name`, `client_email`, and `pain_points`.
    - **If Proposal Needed (If):** Filters based on the `needs_proposal` boolean.
    - **Notify No Proposal Needed (Telegram):** Informs the user why no proposal was generated (e.g., internal meeting).

### 2.3 Proposal Drafting & Document Generation
**Overview:** Transforms AI analysis into a formatted document.
- **Nodes Involved:** `Generate Proposal Content`, `Prepare Proposal Data`, `Copy Proposal Template`, `Fill Template Placeholders`, `Export Doc as PDF`, `Save PDF to Drive`.
- **Node Details:**
    - **Generate Proposal Content (OpenAI):** Uses GPT-4o to write professional sections (Recap, Solution, Deliverables, Timeline) based on the previous analysis.
    - **Prepare Proposal Data (Set):** Maps AI-generated JSON keys to a flat structure for easy reference in later nodes.
    - **Copy Proposal Template (Google Drive):** Copies a master template file.
    - **Fill Template Placeholders (Google Docs):** Replaces `{{PLACEHOLDERS}}` in the copied doc with the actual proposal content.
    - **Export Doc as PDF (HTTP Request):** Calls the Google Drive API to export the Doc as a PDF binary.
    - **Save PDF to Drive (Google Drive):** Uploads the resulting binary as a `.pdf` file.

### 2.4 Telegram Approval Flow
**Overview:** Implements a "Human-in-the-loop" gate.
- **Nodes Involved:** `Send Approval Request`, `Wait for Approval`, `If Approved`, `Notify Proposal Rejected`.
- **Node Details:**
    - **Send Approval Request (Telegram):** Sends a summary of the proposal and a link to the Google Doc.
    - **Wait for Approval (Wait):** Pauses the workflow execution until a webhook response is received (from Telegram buttons).
    - **If Approved (If):** Checks if the action is `approve`.
    - **Notify Proposal Rejected (Telegram):** Alerts the user that the proposal was rejected and provides the doc link for manual edits.

### 2.5 Email Delivery
**Overview:** Final validation and delivery of the proposal.
- **Nodes Involved:** `If Client Email Exists`, `Ask for Client Email`, `Wait for Email Reply`, `Prepare Email Data`, `Download PDF from Drive`, `Send Proposal Email`, `Confirm Email Sent`.
- **Node Details:**
    - **If Client Email Exists (If):** Validates if `client_email` was captured during the AI phase.
    - **Ask for Client Email (Telegram):** If missing, requests the email address from the user.
    - **Wait for Email Reply (Wait):** Pauses for the user's text response.
    - **Prepare Email Data (Set):** Consolidates the final email address and PDF ID.
    - **Download PDF from Drive (Google Drive):** Retrieves the PDF binary for attachment.
    - **Send Proposal Email (Gmail):** Sends the PDF to the client with a professional personalized body.
    - **Confirm Email Sent (Telegram):** Final notification to the user that the process is complete.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Transcript Ready | Webhook | Trigger | - | Respond 200 OK | Transcript Intake |
| Respond 200 OK | Respond to Webhook | API Ack | When Transcript Ready | Get Full Transcript | Transcript Intake |
| Get Full Transcript | HTTP Request | Data Fetch | Respond 200 OK | If Transcript Exists | Transcript Intake |
| If Transcript Exists | If | Validation | Get Full Transcript | Classify Meeting Type, Notify Transcript Error | Transcript Intake |
| Classify Meeting Type | OpenAI | Qualification | If Transcript Exists | If Proposal Needed | AI Qualification |
| If Proposal Needed | If | Logic Gate | Classify Meeting Type | Generate Proposal Content, Notify No Proposal Needed | AI Qualification |
| Generate Proposal Content | OpenAI | Content Creation | If Proposal Needed | Prepare Proposal Data | Proposal Drafting... |
| Prepare Proposal Data | Set | Data Mapping | Generate Proposal Content | Copy Proposal Template | Proposal Drafting... |
| Copy Proposal Template | Google Drive | Doc Creation | Prepare Proposal Data | Fill Template Placeholders | Proposal Drafting... / Google Doc Template Required |
| Fill Template Placeholders | Google Docs | Text Injection | Copy Proposal Template | Export Doc as PDF | Proposal Drafting... |
| Export Doc as PDF | HTTP Request | Conversion | Fill Template Placeholders | Save PDF to Drive | Proposal Drafting... |
| Save PDF to Drive | Google Drive | File Storage | Export Doc as PDF | Send Approval Request | Proposal Drafting... |
| Send Approval Request | Telegram | Notification | Save PDF to Drive | Wait for Approval | Telegram Approval |
| Wait for Approval | Wait | State Pause | Send Approval Request | If Approved | Telegram Approval |
| If Approved | If | User Decision | Wait for Approval | If Client Email Exists, Notify Proposal Rejected | Telegram Approval |
| If Client Email Exists | If | Validation | If Approved | Prepare Email Data, Ask for Client Email | Telegram Approval |
| Ask for Client Email | Telegram | User Input | If Client Email Exists | Wait for Email Reply | Telegram Approval |
| Wait for Email Reply | Wait | State Pause | Ask for Client Email | Prepare Email Data | Telegram Approval |
| Prepare Email Data | Set | Final Mapping | If Client Email Exists, Wait for Email Reply | Download PDF from Drive | Email Delivery |
| Download PDF from Drive | Google Drive | File Retrieval | Prepare Email Data | Send Proposal Email | Email Delivery |
| Send Proposal Email | Gmail | Delivery | Download PDF from Drive | Confirm Email Sent | Email Delivery |
| Confirm Email Sent | Telegram | Notification | Send Proposal Email | - | Email Delivery |
| Notify Proposal Rejected | Telegram | Notification | If Approved | - | Telegram Approval |
| Notify No Proposal Needed | Telegram | Notification | If Proposal Needed | - | AI Qualification |
| Notify Transcript Error | Telegram | Notification | If Transcript Exists | - | Transcript Intake |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Intake Setup
1. Create a **Webhook** node (`POST`) with the path `fireflies-transcript`.
2. Connect it to a **Respond to Webhook** node returning a 200 JSON status.
3. Add an **HTTP Request** node:
    - Method: `POST`
    - URL: `https://api.fireflies.ai/graphql`
    - Auth: Header Auth (Fireflies API Key).
    - Body: GraphQL query as defined in the workflow to fetch transcript details.
4. Add an **If** node to check if `data.transcript` exists. Connect the `false` output to a **Telegram** node for error alerts.

### Step 2: AI Intelligence
1. Create an **OpenAI** node (Model: `gpt-4o`, Temperature: `0.2`). Use a system prompt to classify the meeting into "Proposal Needed" or "Not Needed" and return JSON.
2. Add an **If** node to check if `needs_proposal == true`.
3. Connect the `false` path to a **Telegram** node notifying "No Proposal Needed".

### Step 3: Document Generation
1. Create a second **OpenAI** node (Model: `gpt-4o`, Temperature: `0.4`). System prompt: Write a detailed proposal based on the analysis. Output must be JSON.
2. Add a **Set** node to organize the AI output into clean variables (e.g., `CLIENT_NAME`, `TIMELINE`).
3. Create a **Google Drive** node: Operation `Copy` $\rightarrow$ select your master template Doc.
4. Create a **Google Docs** node: Operation `Update` $\rightarrow$ use `replaceAll` to replace `{{PLACEHOLDERS}}` with the variables from the Set node.
5. Create an **HTTP Request** node:
    - URL: `https://www.googleapis.com/drive/v3/files/{{docId}}/export?mimeType=application/pdf`.
    - Auth: Google OAuth2.
    - Response Format: `File`.
6. Create a **Google Drive** node: Operation `Upload` to save the PDF binary.

### Step 4: Approval & Delivery
1. Create a **Telegram** node sending the proposal summary and doc link.
2. Add a **Wait** node (Resume: `Webhook`) to pause for user action.
3. Add an **If** node to check if the webhook query `action == "approve"`.
4. Connect `false` to a **Telegram** notification for "Rejected".
5. Add another **If** node to check if `client_email` is present.
    - If missing: **Telegram** "Ask for Email" $\rightarrow$ **Wait** node $\rightarrow$ proceed to mapping.
6. Add a final **Set** node to prepare the email recipient and PDF ID.
7. Add a **Google Drive** node: Operation `Download` using the PDF ID.
8. Add a **Gmail** node: Send email to `final_client_email` with the binary attachment.
9. End with a **Telegram** node confirming "Proposal Sent".

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Template Requirement** | You must create a Google Doc with exactly these tags: `{{CLIENT_NAME}}`, `{{CLIENT_COMPANY}}`, `{{MEETING_DATE}}`, `{{MEETING_RECAP}}`, `{{UNDERSTANDING_OF_NEEDS}}`, `{{PROPOSED_SOLUTION}}`, `{{DELIVERABLES}}`, `{{TIMELINE}}`, `{{INVESTMENT}}`, `{{NEXT_STEPS}}`. |
| **Authentication** | Requires: Fireflies API Key, OpenAI API Key, Google OAuth2 (Drive/Docs/Gmail), and Telegram Bot Token. |
| **AI Model** | Optimized for `gpt-4o` for high-reasoning extraction and professional writing. |