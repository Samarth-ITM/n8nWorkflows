Generate overdue lead follow-up Gmail drafts with Google Sheets and Gemini

https://n8nworkflows.xyz/workflows/generate-overdue-lead-follow-up-gmail-drafts-with-google-sheets-and-gemini-14578


# Generate overdue lead follow-up Gmail drafts with Google Sheets and Gemini

# Workflow Analysis: Generate Overdue Lead Follow-Up Gmail Drafts

## 1. Workflow Overview
This workflow automates the identification of sales leads who have not been contacted for five or more days and generates a personalized follow-up email based on a meeting transcript stored in Google Drive. Instead of sending emails automatically, the workflow creates **Gmail Drafts** for a human sales representative to review, ensuring a high-touch, personalized approach.

The logic is organized into four primary functional blocks:
- **1.1 Lead Identification & Filtering:** Scheduled retrieval of leads from Google Sheets and filtering based on status and last contact date.
- **1.2 Document Retrieval & Processing:** Downloading transcripts from Google Drive, validating file types, and extracting raw text.
- **1.3 AI Email Generation:** Utilizing Google Gemini to synthesize transcript data into a concise, personalized email body.
- **1.4 Finalization & Notification:** Creating the Gmail draft, updating the tracking sheet to prevent duplicate processing, and alerting the team via Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Lead Identification & Filtering
**Overview:** This block triggers the workflow and isolates specific leads that meet the "overdue" criteria.
- **Nodes Involved:** `Daily 9AM Trigger`, `Read Leads Sheet`, `Filter - Active Leads Only`, `Filter - 5+ Days No Contact`, `Check - Transcript Exists?`.
- **Node Details:**
    - **Daily 9AM Trigger:** Schedule trigger executing Monday-Friday at 09:00.
    - **Read Leads Sheet:** Google Sheets node reading a specific document. It retrieves all rows from the leads database.
    - **Filter - Active Leads Only:** IF node that only allows leads where `Current status` equals "New".
    - **Filter - 5+ Days No Contact:** Filter node using a JavaScript expression to calculate the difference between today and the `Last Contact` date. Only leads with $\ge 5$ days of inactivity pass.
    - **Check - Transcript Exists?:** IF node checking if the `Meeting Transcript` column is not empty.
- **Edge Cases:** If the `Last Contact` date is incorrectly formatted, the expression returns -1 and the lead is skipped.

### 2.2 Document Retrieval & Processing
**Overview:** This block handles the ingestion of the meeting transcript from an external URL and converts various file formats into usable text.
- **Nodes Involved:** `HTTP - Download File from Google Drive`, `Check - Supported File Type`, `Route - Is TXT?`, `Extract Text: txt`, `Extract Text: Pdf`, `Alert - Download Failed`, `Alert - Unsupported File Type`, `Alert - Missing Transcript`.
- **Node Details:**
    - **HTTP - Download File from Google Drive:** Uses a regex-style expression to transform a standard Google Drive share link into a direct download URL.
    - **Check - Supported File Type:** IF node validating if the downloaded file extension is `txt` or `pdf`.
    - **Route - Is TXT?:** IF node routing the binary data to the appropriate extractor.
    - **Extract Text (txt/Pdf):** Extracts raw text from the binary file.
    - **Slack Alert Nodes:** Three distinct nodes to notify the `followups-reminders` and `failing-followups` channels if transcripts are missing, downloads fail, or files are unsupported.
- **Edge Cases:** If the Google Drive file is not shared "publicly" or with the service account, the HTTP node will fail and trigger the "Download Failed" alert.

### 2.3 AI Email Generation
**Overview:** This block cleanses the data and uses a Large Language Model (LLM) to write the email.
- **Nodes Involved:** `Transcript Data From Files`, `Basic LLM Chain`, `Google Gemini Chat Model`.
- **Node Details:**
    - **Transcript Data From Files:** Code node that normalizes input from various extractors (PDF or TXT) and maps lead data (Name, Company, Row ID) into a single object for the AI.
    - **Basic LLM Chain:** A LangChain node with a strict prompt. It instructs the AI to act as a sales assistant, use ONLY provided details, extract pain points/next steps, and limit the length to 6 sentences.
    - **Google Gemini Chat Model:** The underlying LLM providing the intelligence for the chain.
- **Failure Types:** AI may occasionally hallucinate if the transcript is ambiguous, though the prompt explicitly forbids this.

### 2.4 Finalization & Notification
**Overview:** This block closes the loop by creating the draft and updating the source of truth.
- **Nodes Involved:** `Gmail - Create Draft`, `Update FollowUp Status with Dates`, `Notify - Draft Ready`.
- **Node Details:**
    - **Gmail - Create Draft:** Creates a draft in the user's Gmail account. The subject is set to "Following up - [Company]".
    - **Update FollowUp Status with Dates:** Google Sheets node that updates the `Current status` to "Draft Created" and adds a timestamp to `FollowUp Draft Creation date`. It uses `row_number` as the unique identifier.
    - **Notify - Draft Ready:** Final Slack notification informing the sales rep that their draft is ready for review.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily 9AM Trigger | Schedule Trigger | Workflow Entry | - | Read Leads Sheet | 1. Read & Filter Leads |
| Read Leads Sheet | Google Sheets | Data Retrieval | Daily 9AM Trigger | Filter - Active Leads Only | 1. Read & Filter Leads |
| Filter - Active Leads Only | IF | Status Filtering | Read Leads Sheet | Filter - 5+ Days No Contact | 1. Read & Filter Leads |
| Filter - 5+ Days No Contact | Filter | Recency Filtering | Filter - Active Leads Only | Check - Transcript Exists? | 1. Read & Filter Leads |
| Check - Transcript Exists? | IF | Validation | Filter - 5+ Days No Contact | HTTP - Download... / Alert - Missing... | 1. Read & Filter Leads |
| HTTP - Download... | HTTP Request | File Retrieval | Check - Transcript Exists? | Check - Supported File Type / Alert - Download Failed | 2. Download, Route & Extract... |
| Check - Supported File Type | IF | Format Validation | HTTP - Download... | Route - Is TXT? / Alert - Unsupported... | 2. Download, Route & Extract... |
| Route - Is TXT? | IF | Format Routing | Check - Supported File Type | Extract Text: txt / Extract Text: Pdf | 2. Download, Route & Extract... |
| Extract Text: txt | Extract from File | Text Extraction | Route - Is TXT? | Transcript Data From Files | 2. Download, Route & Extract... |
| Extract Text: Pdf | Extract from File | Text Extraction | Route - Is TXT? | Transcript Data From Files | 2. Download, Route & Extract... |
| Alert - Missing Transcript | Slack | Notification | Check - Transcript Exists? | - | Error Handling |
| Alert - Download Failed | Slack | Notification | HTTP - Download... | - | Error Handling |
| Alert - Unsupported File Type | Slack | Notification | Check - Supported File Type | - | Error Handling |
| Transcript Data From Files | Code | Data Normalization | Extract Text: txt/Pdf | Basic LLM Chain | 2. Download, Route & Extract... |
| Basic LLM Chain | Chain LLM | Content Generation | Transcript Data From Files | Gmail - Create Draft | 3. AI Email Generation |
| Google Gemini Chat Model | Chat Model | AI Engine | - | Basic LLM Chain | 3. AI Email Generation |
| Gmail - Create Draft | Gmail | Draft Creation | Basic LLM Chain | Update FollowUp Status... | 3. AI Email Generation |
| Update FollowUp Status... | Google Sheets | Status Update | Gmail - Create Draft | Notify - Draft Ready | 4. Deduplication |
| Notify - Draft Ready | Slack | Notification | Update FollowUp Status... | - | 4. Deduplication |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger and Data Input
1. Create a **Schedule Trigger** set to a cron expression `0 9 * * 1-5` (9 AM, Mon-Fri).
2. Add a **Google Sheets** node (`Read Leads Sheet`). Configure it to read from your specific Spreadsheet ID and Sheet name.
3. Add an **IF** node (`Filter - Active Leads Only`) to check if `Current status` equals "New".
4. Add a **Filter** node (`Filter - 5+ Days No Contact`). Use this expression for the value:
   `{{ $json['Last Contact'] && !isNaN(new Date($json['Last Contact'])) ? Math.floor((new Date() - new Date($json['Last Contact'])) / (1000 * 60 * 60 * 24)) : -1 }}`. Set the operator to "Greater than or equal to" `5`.
5. Add an **IF** node (`Check - Transcript Exists?`) to ensure the `Meeting Transcript` column is not empty.

### Step 2: File Handling & Extraction
6. Add an **HTTP Request** node. Set the URL to: `{{ 'https://drive.google.com/uc?export=download&id=' + $json['Meeting Transcript'].split('/d/')[1].split('/')[0] }}`. Set the Response Format to "File".
7. Add an **IF** node (`Check - Supported File Type`) to check if `$binary.data.fileExtension` equals `txt` OR `pdf`.
8. Add a second **IF** node (`Route - Is TXT?`) to route specifically to a **Text Extract** node if the extension is `txt`, otherwise route to a **PDF Extract** node.
9. Add a **Code** node (`Transcript Data From Files`) to consolidate the extracted text and lead metadata into a clean JSON object.

### Step 3: AI and Emailing
10. Add a **Basic LLM Chain** node. Input the specific prompt provided in the workflow (Sales Assistant persona, 6-sentence limit, no placeholders).
11. Connect the **Google Gemini Chat Model** to the LLM Chain and configure Gemini API credentials.
12. Add a **Gmail** node (`Create Draft`). Set the resource to `draft`. Map the AI's output text to the body and the Lead's email to the recipient.

### Step 4: Closing and Alerts
13. Add a **Google Sheets** node (`Update FollowUp Status`). Set the operation to "Update", use `row_number` as the matching column, and set `Current status` to "Draft Created".
14. Add a **Slack** node to notify the team when the draft is successfully created.
15. **Error Paths:** Connect the "False" outputs of the validation nodes (Missing transcript, Download failed, Unsupported file) to separate **Slack** notification nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Replace `YOUR_SHEET_ID` | Required for Google Sheets connection |
| Supported Formats: TXT, PDF | File extraction limits |
| Gmail Drafts are used instead of direct emails | Human-in-the-loop safety measure |
| Use Gemini (PaLM) API credentials | Necessary for the AI Chain to function |