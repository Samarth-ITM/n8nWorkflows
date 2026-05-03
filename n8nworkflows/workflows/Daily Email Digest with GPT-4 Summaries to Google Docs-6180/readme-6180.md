Daily Email Digest with GPT-4 Summaries to Google Docs

https://n8nworkflows.xyz/workflows/daily-email-digest-with-gpt-4-summaries-to-google-docs-6180


# Daily Email Digest with GPT-4 Summaries to Google Docs

### 1. Workflow Overview

This workflow, titled **"Daily Email Digest with GPT-4 Summaries to Google Docs"**, automates the daily retrieval, summarization, and storage of the most recent email in a Gmail inbox. Triggered by a daily schedule, it fetches the newest email, cleans and extracts its content, then uses OpenAI’s GPT-4 to generate a professional summary. Finally, it stores this summary in a Google Docs document for easy review and archival.

**Target Use Cases:**
- Users who want daily summaries of incoming emails without manually reading each message.
- Teams or individuals who archive daily email digests for record-keeping or project management.
- Automation enthusiasts seeking modular AI-powered email processing workflows.

**Logical Blocks:**

- **1.1 Trigger and Email Retrieval:**
  - Schedule Trigger runs daily at 8 AM.
  - Gmail node fetches the most recent email.
  - Limit node ensures only one email is processed.
  - IF node checks whether a new email exists.

- **1.2 Email Content Processing:**
  - Code node "Email" extracts, decodes, and cleans email content (plain text extraction from HTML and Base64).
  - Code node "No Email" generates a fallback message if no email is found.

- **1.3 AI Summarization:**
  - OpenAI GPT-4 node processes the cleaned email content and generates a detailed summary using a rich prompt.

- **1.4 Google Docs Storage:**
  - Google Docs node creates a new document titled “Email Summary” or accesses an existing one.
  - Update node inserts the AI-generated summary into the document.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Email Retrieval

**Overview:**  
This block initiates the workflow on a daily schedule, retrieves the latest email from Gmail, and limits processing to one email. It also branches based on whether an email was found.

**Nodes Involved:**  
- Schedule Trigger  
- Get Gmail  
- Limit  
- If

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts workflow daily at 8 AM  
  - *Config:* Interval set to trigger at 8:00 AM every day  
  - *Connections:* Outputs to Get Gmail  
  - *Edge cases:* Workflow will not run outside the scheduled time; misconfiguration could cause missed triggers.

- **Get Gmail**  
  - *Type:* Gmail node (OAuth2)  
  - *Role:* Fetches emails from Gmail inbox  
  - *Config:* Retrieves all emails but limited later; limit set to 1 in the next node  
  - *Credentials:* Requires valid Gmail OAuth2 credentials  
  - *Connections:* Outputs to Limit node  
  - *Edge cases:* Potential auth failures, API rate limits, or empty inboxes.

- **Limit**  
  - *Type:* Limit node  
  - *Role:* Restricts output to only the most recent email (limit=1)  
  - *Connections:* Outputs to If node  
  - *Edge cases:* If no emails exist, no data passes through.

- **If**  
  - *Type:* If condition node  
  - *Role:* Checks if the most recent email has a non-empty "from" header (i.e., email exists)  
  - *Config:* Condition checks if `$('Get Gmail').item.json.headers.from` is not equal to empty string  
  - *Connections:*  
    - True branch → Email processing node  
    - False branch → No Email fallback node  
  - *Edge cases:* If headers are malformed or missing, condition may fail unexpectedly.

---

#### 1.2 Email Content Processing

**Overview:**  
This block extracts email details and cleans the content for summarization. It handles both the case of a found email and the fallback when none exist.

**Nodes Involved:**  
- Email (Code)  
- No Email (Code)

**Node Details:**

- **Email (Code Node)**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Parses the email payload, decodes Base64 body content, strips HTML tags, and extracts key headers (From, Subject, Date)  
  - *Config:*  
    - Prioritizes `text/plain` MIME part; falls back to `text/html` if plain text unavailable  
    - Decodes Base64 content into UTF-8 string  
    - Uses regex to remove HTML tags and excessive whitespace  
  - *Input:* True path from If node (email exists)  
  - *Output:* JSON with `from`, `subject`, `date`, `body` (cleaned text)  
  - *Edge cases:*  
    - Emails with complex multipart structures may not parse correctly  
    - Malformed Base64 or missing parts may cause decoding errors  
    - Empty or spam emails could produce no useful content.

- **No Email (Code Node)**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Provides static JSON response indicating no new emails found  
  - *Config:* Outputs fixed JSON with placeholders:  
    - `from: "N/A"`  
    - `subject: "No Email Found"`  
    - `date`: current ISO timestamp  
    - `body`: message stating no emails to summarize  
  - *Input:* False path from If node (no email)  
  - *Output:* JSON mimicking email structure for downstream processing  
  - *Edge cases:* Minimal, but ensures downstream nodes always receive input.

---

#### 1.3 AI Summarization

**Overview:**  
This block uses OpenAI’s GPT-4 model to generate a professional, concise summary of the email content, including sender info, subject, date, and key points.

**Nodes Involved:**  
- Summary Email OpenAI

**Node Details:**

- **Summary Email OpenAI**  
  - *Type:* OpenAI (Langchain) node  
  - *Role:* Sends prompt and email content to GPT-4 for summarization  
  - *Model:* GPT-4.1 (latest GPT-4 variant)  
  - *Credentials:* Requires valid OpenAI API key  
  - *Prompt:*  
    - Detailed instructions to summarize sender, date, subject, main purpose, key points, requests, and context  
    - Avoids repetition and filler  
    - Injects dynamic data from the original email headers and body using expressions referencing the "Get Gmail" node  
  - *Input:* JSON with cleaned email data from Email or No Email node  
  - *Output:* JSON containing GPT-4 generated summary text inside `choices[0].message.content`  
  - *Edge cases:*  
    - API rate limits or network errors  
    - Prompt injection or malformed input could degrade summary quality  
    - Missing or incomplete email fields may cause less informative summaries.

---

#### 1.4 Google Docs Storage

**Overview:**  
This block creates or accesses a Google Doc titled "Email Summary" and inserts the AI-generated summary into the document for archival and easy access.

**Nodes Involved:**  
- Create Google Docs  
- Update Google Docs

**Node Details:**

- **Create Google Docs**  
  - *Type:* Google Docs node  
  - *Role:* Creates a new Google Docs document titled "Email Summary" in a specified folder  
  - *Config:*  
    - Title set to "Email Summary"  
    - Folder ID configured to a specific Google Drive folder (must exist and be accessible)  
  - *Credentials:* Google OAuth2 credentials with Docs and Drive scopes required  
  - *Input:* Receives data from Summary Email OpenAI node (though document creation does not depend on content)  
  - *Output:* Outputs document metadata including document ID for updating  
  - *Edge cases:*  
    - Folder ID invalid or inaccessible  
    - Insufficient permissions or expired tokens

- **Update Google Docs**  
  - *Type:* Google Docs node  
  - *Role:* Inserts the AI-generated summary text into the created document  
  - *Config:*  
    - Operation: Update with "insert" action  
    - Text to insert is dynamically set from the OpenAI node output (`choices[0].message.content`)  
    - Document URL or ID is taken from the Create Google Docs node output  
  - *Credentials:* Same Google OAuth2 as above  
  - *Input:* Document metadata from Create Google Docs node  
  - *Output:* Final updated document data  
  - *Edge cases:*  
    - Insert action could fail if document is locked or permissions change  
    - Large summaries might exceed Google Docs API limits (rare)  
    - Network or quota issues.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                         | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                   |
|-----------------------|----------------------------------|---------------------------------------|-----------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note           | Sticky Note                      | Informational label                    |                       |                         | Telegram Email Summary Workflow                                                                              |
| Sticky Note1          | Sticky Note                      | Workflow description and usage details|                       |                         | Email Summary Workflow: This n8n workflow automatically summarizes the latest email in your inbox and saves the summary to a Google Doc. It's designed as a simple, modular starting point that users can easily expand or customize... |
| Schedule Trigger      | Schedule Trigger                 | Triggers workflow daily at 8 AM       |                       | Get Gmail               |                                                                                                              |
| Get Gmail             | Gmail                           | Retrieves latest emails from Gmail    | Schedule Trigger       | Limit                   |                                                                                                              |
| Limit                 | Limit                           | Limits output to 1 email               | Get Gmail              | If                      |                                                                                                              |
| If                    | If                              | Checks if email exists by 'from' header| Limit                  | Email (true), No Email (false) |                                                                                                              |
| Email                 | Code (JavaScript)               | Extracts and cleans email content     | If (true)              | Summary Email OpenAI     |                                                                                                              |
| No Email              | Code (JavaScript)               | Provides fallback message when no email| If (false)             | Summary Email OpenAI     |                                                                                                              |
| Summary Email OpenAI  | OpenAI (Langchain)              | Generates AI summary of email         | Email, No Email        | Create Google Docs       |                                                                                                              |
| Create Google Docs    | Google Docs                     | Creates new Google Doc titled "Email Summary" | Summary Email OpenAI   | Update Google Docs       |                                                                                                              |
| Update Google Docs    | Google Docs                     | Inserts AI summary text into document | Create Google Docs     |                         |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**.

2. **Add a Schedule Trigger node**:  
   - Set it to trigger daily at 8:00 AM (hour = 8, minute = 0).  
   - Connect its output to the next node.

3. **Add a Gmail node**:  
   - Operation: `getAll` emails.  
   - Credentials: Configure with valid Gmail OAuth2 credentials.  
   - Filters: Leave empty to fetch all emails.  
   - Connect Schedule Trigger output to this node.

4. **Add a Limit node**:  
   - Set limit to 1 to process only the most recent email.  
   - Connect Gmail node output to this node.

5. **Add an If node**:  
   - Condition: Check if `$('Get Gmail').item.json.headers.from` is not equal to empty string (`""`).  
   - Use strict type validation and case sensitivity as per default.  
   - Connect Limit node output to this If node.

6. **Add a Code node named "Email"** (for true branch):  
   - Paste the JavaScript code that:  
     - Extracts headers ("From", "Subject", "Date")  
     - Decodes Base64 email body (prefer plain text part, fallback to HTML)  
     - Removes HTML tags and whitespace from body content  
   - Connect the true output of If node to this Email node.

7. **Add a Code node named "No Email"** (for false branch):  
   - Paste JavaScript code that returns a single JSON object with:  
     - `from: "N/A"`  
     - `subject: "No Email Found"`  
     - `date`: current timestamp (ISO string)  
     - `body`: message stating no new emails found  
   - Connect the false output of If node to this No Email node.

8. **Add an OpenAI (Langchain) node named "Summary Email OpenAI"**:  
   - Model: Select GPT-4.1 or latest GPT-4 variant.  
   - Credentials: Set OpenAI API key credentials.  
   - Messages: Use the detailed prompt instructing the AI to summarize sender, date, subject, main points, requests, and context.  
   - Use expressions to inject email data from previous nodes, e.g.:  
     - Sender: `{{$json.from}}` or from Get Gmail headers  
     - Date: `{{$json.date}}`  
     - Subject: `{{$json.subject}}`  
     - Email body: `{{$json.body}}`  
   - Connect both Email and No Email nodes' outputs to this node.

9. **Add Google Docs node "Create Google Docs"**:  
   - Operation: Create document  
   - Title: "Email Summary"  
   - Folder ID: Specify your target Google Drive folder ID  
   - Credentials: Google OAuth2 with Docs and Drive scopes  
   - Connect OpenAI node output to this node.

10. **Add Google Docs node "Update Google Docs"**:  
    - Operation: Update document  
    - Action: Insert text  
    - Text: Use expression to insert the summary text from OpenAI output, e.g.:  
      `={{$('Summary Email OpenAI').item.json.choices[0].message.content}}`  
    - Document URL or Document ID: Use output from Create Google Docs node (`{{$json.id}}`)  
    - Credentials: Same Google OAuth2 credentials as above  
    - Connect Create Google Docs node output to this node.

11. **Optionally add Sticky Note nodes** for documentation inside the workflow editor (not required for functionality).

12. **Activate the workflow** and test by running manually or waiting for scheduled trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow uses the latest GPT-4.1 model for improved summarization quality.                                                 | OpenAI GPT-4 documentation: https://platform.openai.com/docs/models/gpt-4                               |
| Gmail OAuth2 credentials must be configured with proper scopes for reading emails.                                             | OAuth2 setup instructions: https://developers.google.com/identity/protocols/oauth2                       |
| Google Docs OAuth2 credentials require permissions for both Google Docs and Drive APIs to create and update documents.        | Google APIs guides: https://developers.google.com/docs/api/quickstart/js                                 |
| The email cleaning step in the "Email" node relies on typical email multipart structures; complex emails might require tweaks.| https://tools.ietf.org/html/rfc2046 multipart email standards                                           |
| Empty inboxes trigger a fallback message to ensure downstream nodes always receive input, avoiding workflow failures.          |                                                                                                          |
| This workflow is modular and can be extended to process multiple emails, different schedules, or alternative output targets.  |                                                                                                          |
| Sticky notes in the workflow provide helpful explanations and modular documentation for users modifying or expanding it.     |                                                                                                          |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow designed with strict adherence to content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.