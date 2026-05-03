Generate Professional Zoom Meeting Summaries with GPT-4O and Google Docs

https://n8nworkflows.xyz/workflows/generate-professional-zoom-meeting-summaries-with-gpt-4o-and-google-docs-8009


# Generate Professional Zoom Meeting Summaries with GPT-4O and Google Docs

### 1. Workflow Overview

This n8n workflow automates the generation of professional Zoom meeting summaries by processing incoming Zoom meeting emails, extracting and formatting their content using AI (GPT-4O and GPT-4.1 models), and integrating the results with Google Docs and email delivery. It targets use cases where users want consistent, structured summaries of Zoom meetings delivered as emails and stored as Google Docs without manual effort.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Preprocessing**  
  Receives new Zoom meeting summary emails via Gmail trigger, marks them as read, and converts email HTML content into markdown for further processing.

- **1.2 Data Extraction & AI Summary Generation**  
  Cleans and prepares email data fields, then sends the transcript to an AI agent (GPT-4O) which generates a structured meeting summary in both HTML (for emails) and plain text (for Google Docs).

- **1.3 Formatting for Google Docs**  
  Uses GPT-4.1 to convert the markdown summary into a sanitized JSON object containing a safe document title and Google Docs–friendly plain-text content.

- **1.4 Google Docs Integration**  
  Creates a new Google Doc with the generated title, then updates its content with the formatted summary.

- **1.5 Email Delivery & Finalization**  
  Sends the formatted HTML summary via Gmail and marks the processed email as read to avoid reprocessing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

- **Overview:**  
  This block listens for incoming Zoom meeting summary emails matching specific subject filters, converts the email HTML body to markdown, and prepares data fields for AI processing.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Markdown (HTML to Markdown converter)  
  - Mark a message as read  
  - Edit Fields1 (set node to clean/assign fields)

- **Node Details:**  

  1. **Gmail Trigger**  
     - Type: Trigger node for Gmail  
     - Configuration: Filters for unread emails with subject containing "Meeting assets", "Zoom", or "Meeting Assets". Polls every minute.  
     - Inputs: Incoming emails from user's Gmail inbox  
     - Outputs: Email JSON including headers, HTML content, and metadata  
     - Edge cases: No matching email → workflow idle; Gmail auth errors; rate limiting or API quota exceeded  

  2. **Markdown (HTML to Markdown converter)**  
     - Type: Markdown converter  
     - Configuration: Converts email HTML body to markdown using "-" for bullet markers, preserves embedded images  
     - Inputs: HTML content from Gmail Trigger  
     - Outputs: Markdown text for AI consumption  
     - Edge cases: Malformed HTML → imperfect markdown; large emails → longer processing  

  3. **Mark a message as read**  
     - Type: Gmail operation node  
     - Configuration: Marks the processed email message as read based on message ID  
     - Inputs: Email message ID from Gmail Trigger  
     - Outputs: Confirmation of marking read  
     - Edge cases: Auth errors; message already marked read by another client → no error  

  4. **Edit Fields1 (Set node)**  
     - Type: Data preparation node  
     - Configuration: Assigns key metadata fields (subject, from, date, SPF, delivered-to, auth results) and extracts main email body text for AI input  
     - Inputs: Markdown output, original email JSON  
     - Outputs: JSON with cleaned fields ready for AI generation  
     - Edge cases: Missing headers or fields → fields may be empty or undefined  

#### 2.2 Data Extraction & AI Summary Generation

- **Overview:**  
  This block sends the cleaned transcript text to a GPT-4O-based AI agent configured as a meeting summarizer. The AI must generate a structured, professional summary in two formats (HTML for email and plain text for Google Docs) and then call the Send Email tool.

- **Nodes Involved:**  
  - Generate Summary (LangChain AI Agent with GPT-4O)  
  - Apply Formatting (Markdown to plain text)  

- **Node Details:**  

  1. **Generate Summary**  
     - Type: LangChain AI Agent node (OpenAI GPT-4O)  
     - Configuration: System prompt defines a strict execution sequence: output two content blocks ([EMAIL_HTML] and [GOOGLE_DOCS_TEXT]), then call the Send Email tool with extracted recipients and formatted content.  
     - Inputs: Meeting transcript text (from Edit Fields1)  
     - Outputs: Formatted summary content (HTML + plain text)  
     - Edge cases: Partial or unclear transcript → AI outputs placeholders "[Not Provided]"; missing attendees → single row placeholder; no action items → note "No specific action items were identified"  
     - Notes: Ensures professional tone, bold decisions, clear formatting  

  2. **Apply Formatting (Markdown)**  
     - Type: Markdown converter  
     - Configuration: Converts AI-generated markdown output into plain text for Google Docs preparation  
     - Inputs: AI output from Generate Summary  
     - Outputs: Clean plain text for next block  
     - Edge cases: Malformed markdown from AI → best-effort clean conversion  

#### 2.3 Formatting for Google Docs

- **Overview:**  
  Uses GPT-4.1 to transform the markdown summary into a sanitized JSON object with two keys: a safe document title and Google Docs–ready plain-text content.

- **Nodes Involved:**  
  - DocFormat (OpenAI GPT-4.1)

- **Node Details:**  

  1. **DocFormat**  
     - Type: OpenAI node (GPT-4.1)  
     - Configuration:  
       - System prompt instructs extraction of subject for title, sanitization of title for filename safety, and conversion of markdown to plain text with specific formatting and placeholders for missing data.  
       - Produces JSON output with exactly keys "title" and "content".  
     - Inputs: Markdown text from Apply Formatting  
     - Outputs: JSON object with sanitized title and Google Docs content  
     - Edge cases: Malformed markdown → best-effort parsing; missing subject → fallback titles; oversized content → not truncated but formatted; must produce valid JSON only  
     - Version: Requires GPT-4.1 for reliable formatting  

#### 2.4 Google Docs Integration

- **Overview:**  
  Creates a new Google Docs document with the sanitized title, then updates it with the formatted meeting summary content.

- **Nodes Involved:**  
  - Create a document (Google Docs node)  
  - Update a document (Google Docs node)  

- **Node Details:**  

  1. **Create a document**  
     - Type: Google Docs node  
     - Configuration: Creates document with title from DocFormat’s "title" key, placed in default Google Drive folder  
     - Inputs: JSON output from DocFormat (title)  
     - Outputs: Document metadata including document URL/ID  
     - Credentials: Google Docs OAuth2 account required  
     - Edge cases: Auth errors; folder permission issues  

  2. **Update a document**  
     - Type: Google Docs node  
     - Configuration: Inserts the content from DocFormat’s "content" key into the created document  
     - Inputs: Document URL/ID from Create a document; content text from DocFormat  
     - Outputs: Confirmation of update  
     - Credentials: Same as above  
     - Edge cases: Auth errors; document locked or deleted  

#### 2.5 Email Delivery & Finalization

- **Overview:**  
  Sends the HTML summary email to recipients extracted from the transcript, then marks the processed email as read and completes the workflow.

- **Nodes Involved:**  
  - Send Email (Gmail Tool)  
  - No Operation, do nothing (end nodes)

- **Node Details:**  

  1. **Send Email**  
     - Type: Gmail Tool node  
     - Configuration:  
       - Sends to email addresses extracted by AI or uses “[Not Provided]” fallback  
       - Subject formatted as "Meeting Summary: {Meeting Title} – {YYYY-MM-DD}"  
       - HTML body from AI-generated HTML content (between [EMAIL_HTML] markers)  
       - Plain text body from AI-generated Google Docs text (between [GOOGLE_DOCS_TEXT] markers)  
     - Inputs: AI-generated content from Generate Summary node  
     - Credentials: Gmail OAuth2 account required  
     - Edge cases: Auth errors; invalid email addresses; sending limits  

  2. **No Operation, do nothing**  
     - Type: NoOp node  
     - Purpose: Marks logical workflow end points for success or fallback handling  
     - Inputs: From Google Docs update and Gmail mark-as-read nodes respectively  

---

### 3. Summary Table

| Node Name            | Node Type                            | Functional Role                        | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                          |
|----------------------|------------------------------------|-------------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Gmail Trigger        | gmailTrigger                       | Entry point, monitors Zoom emails   | —                     | Markdown, Mark a message as read | 📧 EMAIL TRIGGER: Monitors inbox for Zoom meeting summaries with subject filters                    |
| Markdown             | markdown                          | Converts email HTML to Markdown     | Gmail Trigger         | Edit Fields1             | 📝 MARKDOWN CONVERTER: Converts HTML content to markdown with bullet "-" and preserves images      |
| Mark a message as read| gmail                            | Marks email as read after processing| Gmail Trigger         | No Operation, do nothing1 |                                                                                                    |
| Edit Fields1         | set                              | Cleans/extracts key email fields    | Markdown              | Generate Summary         |                                                                                                    |
| Generate Summary     | langchain.agent (GPT-4O)          | AI summarization & email content gen| Edit Fields1          | Apply Formatting         | 🤖 GENERATE SUMMARY: AI creates structured email-safe HTML and Google Docs text with professional formatting |
| Apply Formatting     | markdown                         | Converts AI markdown output to plain text| Generate Summary    | DocFormat                |                                                                                                    |
| DocFormat            | openAi (GPT-4.1)                  | Formats markdown to JSON for Google Docs| Apply Formatting    | Create a document         | 📄 DOCFORMAT: Extracts subject, sanitizes title, formats content for Google Docs                   |
| Create a document    | googleDocs                       | Creates new Google Doc with title   | DocFormat             | Update a document         | 📁 CREATE A DOCUMENT: Uses sanitized title, creates doc in default folder                          |
| Update a document    | googleDocs                       | Inserts formatted content into doc  | Create a document      | No Operation, do nothing  |                                                                                                    |
| Send Email           | gmailTool                       | Sends summary email                  | Generate Summary       | —                        |                                                                                                    |
| No Operation, do nothing | noOp                         | Marks workflow end                   | Update a document      | —                        |                                                                                                    |
| No Operation, do nothing1| noOp                         | Marks workflow end                   | Mark a message as read | —                        |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Configure OAuth2 Gmail credentials  
   - Set filter query: subject contains "Meeting assets", "Zoom", or "Meeting Assets"  
   - Polling: every minute  
   - Output: Emits new Zoom meeting summary emails  

2. **Add Markdown node:**  
   - Type: Markdown converter  
   - Input: Gmail Trigger node output (email HTML content)  
   - Parameters: Convert HTML to markdown, bulletMarker: "-"  
   - Preserve data images  

3. **Add "Mark a message as read" Gmail node:**  
   - Input: Gmail Trigger (message ID)  
   - Operation: Mark message as read  
   - Credentials: Gmail OAuth2  

4. **Add Set node ("Edit Fields1"):**  
   - Input: Markdown output  
   - Assign fields: subject, from, date, SPF, delivered-to, auth results, data (email body text or plain text fallback)  
   - Value extraction via expressions on `$json`  

5. **Add LangChain AI Agent node ("Generate Summary"):**  
   - Model: GPT-4O  
   - Input: Text from Edit Fields1 data field  
   - System prompt: Detailed meeting summarizer instructions with output format including [EMAIL_HTML] and [GOOGLE_DOCS_TEXT] blocks, plus email tool call instructions  
   - Output: Generates structured HTML and plain text summaries, and triggers Send Email tool  

6. **Add Markdown node ("Apply Formatting"):**  
   - Input: Output from Generate Summary (markdown text)  
   - Convert markdown to plain text for Google Docs formatting  

7. **Add OpenAI node ("DocFormat"):**  
   - Model: GPT-4.1  
   - Input: Plain text from Apply Formatting  
   - System prompt: Extract subject, sanitize title, produce JSON with keys "title" and "content" formatted for Google Docs  
   - Enable JSON output  

8. **Add Google Docs node ("Create a document"):**  
   - Operation: Create  
   - Title: Use expression from DocFormat's "title" JSON key  
   - Folder: Default or specify Google Drive folder  
   - Credentials: Google Docs OAuth2  

9. **Add Google Docs node ("Update a document"):**  
   - Operation: Update  
   - Document URL: Use the document URL from Create a document output  
   - Action: Insert text from DocFormat's "content" JSON key  
   - Credentials: Same as above  

10. **Add Gmail Tool node ("Send Email"):**  
    - Configure OAuth2 credentials for Gmail  
    - SendTo: Extract recipients from AI output or fallback to "[Not Provided]"  
    - Subject: "Meeting Summary: {Meeting Title} – {YYYY-MM-DD}" from AI output  
    - HTML Body: Extract content between [EMAIL_HTML] markers from AI output  
    - Text Body: Extract content between [GOOGLE_DOCS_TEXT] markers from AI output  

11. **Add No Operation nodes to mark workflow termination points:**  
    - Connect Update a document output to one NoOp node  
    - Connect Mark a message as read to another NoOp node  

12. **Connect nodes according to data flow:**  
    - Gmail Trigger → Markdown → Edit Fields1 → Generate Summary → Apply Formatting → DocFormat → Create a document → Update a document → NoOp  
    - Gmail Trigger → Mark a message as read → NoOp  
    - Generate Summary (AI tool output) → Send Email  

13. **Test the workflow:**  
    - Ensure Gmail and Google Docs OAuth2 credentials are correctly set and authorized  
    - Send test Zoom meeting summary email matching filter criteria  
    - Verify structured email summary sent and Google Doc created and updated correctly  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Workflow monitors Gmail inbox specifically for Zoom meeting asset emails with subjects like "Meeting assets", "Zoom", "Meeting Assets". Sender filter can be added for no-reply@zoom.us for more precise triggering.                               | Sticky Note on Gmail Trigger node                                                                                               |
| Markdown conversion preserves bullet points as "-" and embedded data images to improve AI processing accuracy.                                                                                                                                 | Sticky Note on Markdown node                                                                                                     |
| AI summarization uses GPT-4O with a detailed system prompt enforcing a strict output format and sequence, including generating both HTML and plain text content blocks before sending email.                                                     | Sticky Note on Generate Summary node                                                                                            |
| Google Docs formatting uses GPT-4.1 model to sanitize titles for filenames and produce plain-text content compatible with Google Docs, outputting strict JSON without extra characters or code fences.                                           | Sticky Note on DocFormat node                                                                                                   |
| Google Docs nodes create and update documents in the default folder; folder can be changed as per user preference. Google Docs OAuth2 credentials are required.                                                                                  | Sticky Note on Create a document node                                                                                           |
| Email sending is integrated via Gmail Tool node called by the AI agent after generating content, ensuring summaries are sent only after full content generation.                                                                                | Described in Generate Summary system prompt                                                                                    |
| AI-generated summaries include placeholders "[Not Provided]" for missing data, handle missing attendees or action items gracefully, and bold decisions in output HTML for clarity.                                                             | Generate Summary system prompt details                                                                                        |
| The workflow marks processed emails as read to prevent reprocessing; uses No Operation nodes to mark logical ends.                                                                                                                             | Workflow design details                                                                                                         |
| Project credits: n8n automation with OpenAI GPT-4 and Google Docs integration for professional meeting note workflows.                                                                                                                        |                                                                                                                                |
| Video or blog links are not included in the workflow but could be added by users for training or documentation.                                                                                                                                 |                                                                                                                                |

---

This completes the detailed, self-contained technical reference document for the "Generate Professional Zoom Meeting Summaries with GPT-4O and Google Docs" n8n workflow.