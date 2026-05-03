Daily Applicant Digest by Role with Gemini AI Extraction for Hiring Managers

https://n8nworkflows.xyz/workflows/daily-applicant-digest-by-role-with-gemini-ai-extraction-for-hiring-managers-7699


# Daily Applicant Digest by Role with Gemini AI Extraction for Hiring Managers

### 1. Workflow Overview

This workflow automates the daily processing and distribution of applicant information extracted from emails labeled as job applications. It targets hiring managers who require a consolidated digest of new applicants by role, enriched with structured data extracted using Google Gemini AI.

The workflow is logically divided into these blocks:

- **1.1 Daily Trigger:** Scheduled initiation at 6 AM daily to start the process.
- **1.2 Fetch Applicant Emails:** Retrieves unread emails labeled as applicants received within the last 24 hours from a Gmail account.
- **1.3 Mark Emails as Read:** Marks processed emails as read to avoid reprocessing.
- **1.4 Extract Applicant Details:** Uses Google Gemini AI to parse applicant information from email snippets, outputting structured JSON data.
- **1.5 Assign Manager Emails:** Maps each applicant’s role to a specific hiring manager’s email address or assigns a fallback if no match.
- **1.6 Group & Build HTML Tables:** Groups applicants by hiring manager and by role, creating formatted HTML tables summarizing applicant details.
- **1.7 Send Digest to Managers:** Sends the generated HTML digest email to each manager listing their relevant applicants.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger

- **Overview:**  
  Initiates the workflow automatically every day at 6 AM (server/local time).

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Triggers daily at 6:00 AM  
    - Inputs: None (start node)  
    - Outputs: Connects to "fetch_applicant_emails" node  
    - Potential failure modes: None typical; watch for system clock or timezone misconfiguration.

#### 2.2 Fetch Applicant Emails

- **Overview:**  
  Queries Gmail for unread emails labeled "applicant" received within the last 24 hours.

- **Nodes Involved:**  
  - fetch_applicant_emails  
  - readAll_applicant_emails

- **Node Details:**  
  - **fetch_applicant_emails**  
    - Type: Gmail node  
    - Operation: Get all messages matching filter `label: applicant newer_than:1d is:unread`  
    - Credentials: Gmail OAuth2 (configured)  
    - Output: List of email metadata (including snippet and message IDs)  
    - Possible failures: Authentication errors, Gmail API rate limits, query errors.

  - **readAll_applicant_emails**  
    - Type: Gmail node  
    - Operation: Mark each fetched email as read (using message ID from previous node)  
    - Credentials: Same Gmail OAuth2  
    - Inputs: From fetch_applicant_emails  
    - Outputs: None (end of this branch)  
    - Potential issues: Marking emails read may fail if message ID is invalid or Gmail API errors.

#### 2.3 Extract Applicant Details

- **Overview:**  
  Sends each email snippet to Google Gemini AI to extract structured applicant information in JSON format.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Extract Applicant Details

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: Google Gemini AI chat model node (LangChain integration)  
    - Model: `models/gemini-1.5-flash`  
    - Purpose: Provides AI language model processing for downstream extraction  
    - Credentials: Google PaLM API credentials configured  
    - Inputs: Connected logically to Extract Applicant Details node (AI language model input)  
    - Potential failures: API quota exceeded, network, or invalid model name.

  - **Extract Applicant Details**  
    - Type: Chain LLM (LangChain) node  
    - Configuration: Uses a defined prompt to parse applicant email snippets into JSON keys: name, email, phone, role, years_of_experience, top_skills, location, notice_period, summary  
    - Input: Email snippet text from fetch_applicant_emails  
    - Output: Structured JSON  
    - Potential failure: AI response malformed, JSON parse errors, misinterpretation of input text.

#### 2.4 Assign Manager Emails

- **Overview:**  
  Parses AI output JSON, normalizes the role, and maps it to the appropriate hiring manager’s email address, with a fallback email for unmatched roles.

- **Nodes Involved:**  
  - Assign Manager Emails (Code node)

- **Node Details:**  
  - **Assign Manager Emails**  
    - Type: Code node (JavaScript)  
    - Function:  
      - Strips markdown formatting from AI JSON output (removes triple backticks and "json" tags)  
      - Parses JSON safely, logging errors if parsing fails  
      - Normalizes role string (trim, lowercase)  
      - Maps normalized role to manager email via a predefined dictionary  
      - If no match, assigns a fallback manager email  
    - Input: AI-extracted JSON string field `text` from previous node  
    - Output: Enriched JSON with parsed applicant fields and `managerEmail`  
    - Edge cases: JSON parse failures, missing role field, undefined mapping for role, malformed input.

#### 2.5 Group & Build HTML Tables

- **Overview:**  
  Groups applicants by manager email and then by role, building a comprehensive HTML table summarizing all applicants for that manager.

- **Nodes Involved:**  
  - Group & Build HTML Tables (Code node)

- **Node Details:**  
  - **Group & Build HTML Tables**  
    - Type: Code node (JavaScript)  
    - Logic:  
      - Groups data by `managerEmail`, then by applicant `role`  
      - Constructs HTML tables with columns: Name, Email (mailto link), Phone, Years of Experience, Top Skills, Location, Notice Period, Summary  
      - Escapes HTML entities to prevent injection or rendering issues  
      - Outputs one JSON item per manager with fields `managerEmail` and HTML content string in `html`  
    - Input: Enriched applicant JSON with managerEmail from previous node  
    - Output: Array of manager-specific objects ready for email sending  
    - Edge cases: Missing fields, arrays in top_skills, HTML escaping errors.

#### 2.6 Send Digest to Managers

- **Overview:**  
  Sends the generated HTML digest email to each hiring manager with a subject reflecting the current date.

- **Nodes Involved:**  
  - Send Digest to Managers (Gmail node)

- **Node Details:**  
  - **Send Digest to Managers**  
    - Type: Gmail node  
    - Operation: Send email  
    - Parameters:  
      - To: dynamic `{{$json.managerEmail}}`  
      - Subject: "Today's Applicants Digest – MM-DD" (date formatted dynamically)  
      - Message body: HTML content from previous node `{{$json.html}}`  
    - Credentials: Gmail OAuth2 configured  
    - Edge cases: Email sending failures, invalid email addresses, Gmail quota limits.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                          | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                   |
|-------------------------|---------------------------------|----------------------------------------|-------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                 | Initiates workflow daily at 6 AM       | None                    | fetch_applicant_emails      | ## Send daily applicant digest by role from Gmail to hiring managers with Google Gemini       |
| fetch_applicant_emails  | Gmail Node                      | Fetch unread applicant emails from Gmail | Schedule Trigger        | Extract Applicant Details, readAll_applicant_emails |                                                                                                |
| readAll_applicant_emails| Gmail Node                      | Marks fetched applicant emails as read | fetch_applicant_emails  | None                       |                                                                                                |
| Google Gemini Chat Model| Google Gemini AI Chat Model     | Provides AI LLM for applicant data extraction | None (AI model input)   | Extract Applicant Details (AI input) |                                                                                                |
| Extract Applicant Details | Chain LLM (LangChain)           | Extracts structured applicant info from email snippets | fetch_applicant_emails, Google Gemini Chat Model | Assign Manager Emails       |                                                                                                |
| Assign Manager Emails   | Code Node                      | Parses JSON output, maps role to manager email | Extract Applicant Details | Group & Build HTML Tables      |                                                                                                |
| Group & Build HTML Tables | Code Node                      | Groups applicants by manager and role, builds HTML tables | Assign Manager Emails    | Send Digest to Managers      |                                                                                                |
| Send Digest to Managers | Gmail Node                      | Sends daily applicant digest email to each manager | Group & Build HTML Tables | None                       |                                                                                                |
| Sticky Note             | Sticky Note                    | Workflow description                    | None                    | None                       | ## Send daily applicant digest by role from Gmail to hiring managers with Google Gemini       |
| Sticky Note1            | Sticky Note                    | Detailed workflow overview              | None                    | None                       | ## Workflow Overview: Send daily applicant digest by role from Gmail to hiring managers with Google Gemini <br> **Purpose:** Automatically fetches new job application emails labeled applicants, extracts structured applicant details using OpenAI, groups candidates by role and manager, then sends a daily HTML summary email to each hiring manager.<br><br>Daily Applicant Digest Workflow – Node Overview<br><br>**1. Daily Trigger (6PM IST)**<br>Starts the workflow every day at 18:00 (Asia/Kolkata timezone).<br>**2. Fetch Applicant Emails**<br>Retrieves all new application emails labeled applicants from the last 24 hours.<br>**3. Read All Emails**<br>Read each email’s labeled applicants which we retrieves<br>**4. Extract Applicant Details**<br>Uses OpenAI to extract and structure applicant info (name, email, role, skills, etc.) in JSON format.<br>**5. Assign Manager Emails**<br>Maps each applicant’s role to a hiring manager’s email address.<br>Uses a fallback email if the role does not match any manager.<br>**6. Group & Build HTML Tables**<br>Groups applicants by manager and role.<br>Builds a clear, formatted HTML table for each group, summarizing all applicants.<br>**7. Send Digest to Managers**<br>Sends one HTML summary email per manager, listing all relevant new applicants for the day. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 6:00 AM (server time or adjust timezone accordingly).

2. **Create a Gmail Node to Fetch Applicant Emails:**  
   - Type: Gmail  
   - Operation: Get All  
   - Filter Query: `label: applicant newer_than:1d is:unread`  
   - Credentials: Configure Gmail OAuth2 with appropriate Gmail account  
   - Connect input from Schedule Trigger node.

3. **Create a Gmail Node to Mark Emails as Read:**  
   - Type: Gmail  
   - Operation: Mark As Read  
   - Parameter Message ID: `={{ $json.id }}` (from fetch_applicant_emails node)  
   - Credentials: Same Gmail OAuth2  
   - Connect input from fetch_applicant_emails node.

4. **Create a Google Gemini Chat Model Node:**  
   - Type: Google Gemini AI Chat Model (via LangChain integration)  
   - Model Name: `models/gemini-1.5-flash`  
   - Credentials: Google PaLM API credentials configured  
   - No direct input; this node acts as AI model provider for the next node.

5. **Create a Chain LLM Node to Extract Applicant Details:**  
   - Type: Chain LLM (LangChain)  
   - Prompt Type: Define prompt with the following text (replace `$json.snippet` dynamically):  
     ```
     You are an assistant that extracts job applicant information from emails.

     Extract the following fields and return ONLY valid JSON with these keys:
     - name
     - email
     - phone
     - role
     - years_of_experience
     - top_skills
     - location
     - notice_period
     - summary

     Applicant email:
     """
     {{$json.snippet }}
     """
     ```
   - Connect AI languageModel input from Google Gemini Chat Model node  
   - Connect main input from fetch_applicant_emails node.

6. **Create a Code Node to Assign Manager Emails:**  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript logic that:  
     - Strips markdown formatting from AI JSON output  
     - Parses JSON safely  
     - Normalizes role string  
     - Maps role to manager email or fallback  
   - Connect input from Extract Applicant Details node.

7. **Create a Code Node to Group & Build HTML Tables:**  
   - Type: Code (JavaScript)  
   - Paste provided JavaScript logic to group applicants by manager and role and build HTML tables  
   - Connect input from Assign Manager Emails node.

8. **Create a Gmail Node to Send Digest Emails:**  
   - Type: Gmail  
   - Operation: Send  
   - To: `={{$json.managerEmail}}`  
   - Subject: `="Today's Applicants Digest – " + new Date().toLocaleDateString("en-US",{month:"2-digit",day:"2-digit"})`  
   - Message: `={{$json.html}}` (HTML format)  
   - Credentials: Same Gmail OAuth2  
   - Connect input from Group & Build HTML Tables node.

9. **Add Sticky Notes (Optional):**  
   - Add descriptive sticky notes summarizing the workflow purpose and node overview for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| The workflow uses Google Gemini (PaLM API) for AI-based extraction of applicant details, which requires Google Cloud account with PaLM API enabled and proper credentials.                                                                                                                                                                                   | Google PaLM API documentation: https://developers.generativeai.google          |
| The Gmail OAuth2 credentials must have access to the mailbox with label "applicant" and permission to read and send emails. Ensure Gmail API quotas and limits are respected to avoid failures.                                                                                                                                                              | Gmail API quotas: https://developers.google.com/gmail/api/guides/quotas          |
| The fallback manager email is used if an applicant role does not match any predefined role in the mapping. Add additional roles as needed in the Assign Manager Emails code node to scale.                                                                                                                                                                  | Modify roleToManagerEmail object in code node                                    |
| The workflow assumes emails are labeled and filtered correctly in Gmail, and that applicant emails contain sufficient data in the snippet for the AI model to extract structured information. Incomplete or poorly formatted emails may lead to extraction errors.                                                                                         | Consider improving email labeling or preprocessing if extraction errors occur.   |
| The code nodes escape HTML to prevent injection vulnerabilities in the email digest. However, verify that all fields are sanitized for your specific data context before production deployment.                                                                                                                                                           | General best practice for email HTML content                                   |
| Timezone considerations: The Schedule Trigger node uses server time. Adjust trigger hour or convert timezones as needed for local business hours.                                                                                                                                                                                                            | n8n Schedule Trigger docs: https://docs.n8n.io/nodes/n8n-nodes-base.scheduletrigger/ |
| Sticky notes in the workflow provide a valuable summary and overview for maintainers and should be kept updated with any workflow changes.                                                                                                                                                                                                                   | n8n Sticky Note node documentation: https://docs.n8n.io/nodes/nodes-basics/stickynote/ |

---

**Disclaimer:** The text above is extracted exclusively from an automated workflow built with n8n, respecting all applicable content policies and containing no illegal, offensive, or protected material. All data processed is legal and publicly available.