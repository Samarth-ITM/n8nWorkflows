Auto-respond to job opportunities with Gmail, LinkedIn, GPT-4.1-mini and Google Sheets

https://n8nworkflows.xyz/workflows/auto-respond-to-job-opportunities-with-gmail--linkedin--gpt-4-1-mini-and-google-sheets-14924


# Auto-respond to job opportunities with Gmail, LinkedIn, GPT-4.1-mini and Google Sheets

### 1. Workflow Overview

The **Personal AI Job Application Auto-Responder** is an automated system designed to manage inbound job opportunities from emails and LinkedIn messages. Its primary purpose is to filter relevant job offers, generate highly personalized responses based on the user's resume and preferences using AI, send the replies, and maintain a tracking log in Google Sheets.

The workflow is logically divided into three functional blocks:
- **1.1 Trigger & Intake:** Handles the reception of messages via webhooks or scheduled polling and standardizes the data format.
- **1.2 Analysis & AI Reply:** Filters out irrelevant messages and uses a Large Language Model (LLM) to draft a professional response.
- **1.3 Send & Track:** Dispatches the finalized message to the recruiter and logs the application details for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Intake
**Overview:** This block serves as the entry point for the workflow, capturing incoming communications and normalizing the data structure for downstream processing.

- **Nodes Involved:** `Webhook - New Job Message`, `Poll New Job Emails`, `Prepare Message Context`.
- **Node Details:**
    - **Webhook - New Job Message**: 
        - *Type:* Webhook.
        - *Role:* Receives real-time POST requests (e.g., from a LinkedIn automation tool).
        - *Path:* `job-application-inbound`.
    - **Poll New Job Emails**: 
        - *Type:* Schedule Trigger.
        - *Role:* Triggers the workflow every 20 minutes to check for new emails via cron expression `*/20 * * * *`.
    - **Prepare Message Context**: 
        - *Type:* Set.
        - *Role:* Standardizes input data into four key variables: `sender`, `subject`, `body`, and `messageId`.
        - *Expressions:* Uses fallback logic (e.g., `{{ $json.from || $json.body?.from }}`) to ensure compatibility between webhook and email triggers.

#### 2.2 Analysis & AI Reply
**Overview:** This block filters the noise to ensure only job-related messages are processed and utilizes AI to craft a custom response based on the user's professional background.

- **Nodes Involved:** `Python - Detect Job Message`, `Filter Relevant Jobs`, `Wait 1 - Rate Limit`, `AI - Generate Tailored Reply`, `OpenAI Chat Model`.
- **Node Details:**
    - **Python - Detect Job Message**: 
        - *Type:* Code (Python).
        - *Role:* Analyzes the message body to determine if it is a job-related opportunity.
    - **Filter Relevant Jobs**: 
        - *Type:* Filter.
        - *Role:* Boolean check on `isJobRelated`. Only "True" values proceed.
    - **Wait 1 - Rate Limit**: 
        - *Type:* Wait.
        - *Role:* Introduces a delay to prevent API rate limiting on the AI provider.
    - **AI - Generate Tailored Reply**: 
        - *Type:* AI Agent (LangChain).
        - *Role:* Generates a professional response under 170 words.
        - *Key Logic:* Injects the user's Resume and Preferences into the prompt alongside the job message.
    - **OpenAI Chat Model**: 
        - *Type:* LLM Chat Model.
        - *Configuration:* Uses `gpt-4.1-mini`.
        - *Failure Type:* Authentication errors if OpenAI API key is expired.

#### 2.3 Send & Track
**Overview:** The final phase handles the delivery of the generated text and updates the administrative record of the application.

- **Nodes Involved:** `JS - Format Reply`, `Wait 2 - Review Buffer`, `Send Tailored Reply`, `Update Google Sheet Tracker`.
- **Node Details:**
    - **JS - Format Reply**: 
        - *Type:* Code (JavaScript).
        - *Role:* Cleans the AI output and assigns a status of "Replied" and the current date.
    - **Wait 2 - Review Buffer**: 
        - *Type:* Wait.
        - *Role:* Provides a temporal buffer before sending the email.
    - **Send Tailored Reply**: 
        - *Type:* HTTP Request.
        - *Role:* Sends the email via the SendGrid API.
        - *Configuration:* POST request with a JSON body containing `personalizations`, `from`, `subject`, and `content`.
    - **Update Google Sheet Tracker**: 
        - *Type:* HTTP Request.
        - *Role:* Appends a new row to a Google Sheet via the Google Sheets API.
        - *Variables:* Logs date, job title, sender, subject, status, and timestamp.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook - New Job Message | Webhook | Trigger (External) | - | Prepare Message Context | 1. Trigger & Intake |
| Poll New Job Emails | Schedule Trigger | Trigger (Timed) | - | Prepare Message Context | 1. Trigger & Intake |
| Prepare Message Context | Set | Data Normalization | Webhook / Poll | Python - Detect Job Message | 1. Trigger & Intake |
| Python - Detect Job Message | Code | Relevance Detection | Prepare Message Context | Filter Relevant Jobs | 1. Trigger & Intake |
| Filter Relevant Jobs | Filter | Data Gating | Python - Detect Job Message | Wait 1 - Rate Limit | 1. Trigger & Intake |
| Wait 1 - Rate Limit | Wait | API Throttling | Filter Relevant Jobs | AI - Generate Tailored Reply | 2. Analysis & AI Reply |
| AI - Generate Tailored Reply | AI Agent | Content Generation | Wait 1 / OpenAI Model | JS - Format Reply | 2. Analysis & AI Reply |
| OpenAI Chat Model | LLM Model | AI Engine | - | AI - Generate Tailored Reply | 2. Analysis & AI Reply |
| JS - Format Reply | Code | Data Formatting | AI - Generate Tailored Reply | Wait 2 - Review Buffer | 2. Analysis & AI Reply |
| Wait 2 - Review Buffer | Wait | Delivery Delay | JS - Format Reply | Send Reply / Update Sheet | 3. Send & Track |
| Send Tailored Reply | HTTP Request | Email Dispatch | Wait 2 - Review Buffer | - | 3. Send & Track |
| Update Google Sheet Tracker | HTTP Request | Application Logging | Wait 2 - Review Buffer | - | 3. Send & Track |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Triggers**
1. Create a **Webhook** node. Set the HTTP method to `POST` and the path to `job-application-inbound`.
2. Create a **Schedule Trigger** node. Set the interval to a cron expression: `*/20 * * * *`.
3. Connect both nodes to a **Set** node named "Prepare Message Context". Configure assignments for `sender`, `subject`, `body`, and `messageId` using expressions to handle both webhook and email inputs.

**Step 2: Filtering Logic**
4. Create a **Code** node (Python) to analyze the `body` for job-related keywords. It should output a boolean `isJobRelated`.
5. Add a **Filter** node. Set the condition to check if `isJobRelated` is `true`.
6. Add a **Wait** node to prevent AI rate limits.

**Step 3: AI Configuration**
7. Create an **AI Agent** node. 
    - Set the Prompt to act as an expert job writer.
    - Include placeholders for `{{ $json.resume }}` and `{{ $json.preferences }}`.
    - Link the agent to an **OpenAI Chat Model** node configured with `gpt-4.1-mini`.
8. Create a **Code** node (JavaScript) to format the AI's `response` into `replyText` and add the `appliedDate`.
9. Add a second **Wait** node as a review buffer.

**Step 4: Execution & Tracking**
10. Create an **HTTP Request** node for SendGrid:
    - Method: `POST`.
    - URL: `https://api.sendgrid.com/v3/mail/send`.
    - Body: JSON containing the recipient, sender email, and `replyText`.
11. Create an **HTTP Request** node for Google Sheets:
    - Method: `POST`.
    - URL: `https://sheets.googleapis.com/v4/spreadsheets/YOUR_SHEET_ID/values/Sheet1!A1:append?valueInputOption=USER_ENTERED`.
    - Body: An array of values: `[appliedDate, extractedJobTitle, sender, subject, status, timestamp]`.

**Step 5: Credentials**
- Configure **OpenAI API** credentials.
- Configure **SendGrid** (or Gmail) and **Google OAuth2** credentials for Sheets.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Professional AI Job Application Auto-Responder | High-level workflow purpose |
| Target Users: Job seekers, Professionals, Career switchers | User Personas |
| Requirements: Gmail/IMAP, Google Sheets, OpenAI/Anthropic/Grok API | System Dependencies |
| Customization: Adjust AI tone, Python keywords, or Sheet columns | Maintenance Tips |