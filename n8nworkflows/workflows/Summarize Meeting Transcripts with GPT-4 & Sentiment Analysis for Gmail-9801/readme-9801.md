Summarize Meeting Transcripts with GPT-4 & Sentiment Analysis for Gmail

https://n8nworkflows.xyz/workflows/summarize-meeting-transcripts-with-gpt-4---sentiment-analysis-for-gmail-9801


# Summarize Meeting Transcripts with GPT-4 & Sentiment Analysis for Gmail

### 1. Workflow Overview

This workflow automates the process of summarizing meeting transcripts stored as files in a specified Google Drive folder. It supports both PDF and plain text files as input formats. Upon detecting a new file, it extracts the text content, sends it to OpenAI's GPT-4.1-mini model to generate a structured meeting summary including key decisions, notes, sentiment analysis, and categorized tasks. The structured output is then normalized, converted into a polished HTML email, and automatically sent to the last modifying user of the file via Gmail.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception & File Detection:** Watches a specific Google Drive folder for new meeting transcript files.
- **1.2 File Type Routing & Text Extraction:** Differentiates between PDF and text files and extracts their content accordingly.
- **1.3 AI Processing:** Sends extracted text to an AI agent (GPT-4.1-mini) to generate a strict JSON structured summary with sentiment and tasks.
- **1.4 Output Validation & Normalization:** Parses and sanitizes AI output, grouping tasks by sentiment and preparing statistics.
- **1.5 HTML Summary Generation:** Converts normalized JSON data into a clean, email-friendly HTML summary.
- **1.6 Email Dispatch:** Sends the generated summary as an HTML email to the file's last modifying user via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & File Detection

- **Overview:**  
  This block monitors a designated Google Drive folder for any newly created files to trigger the workflow.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - If

- **Node Details:**

  - **Google Drive Trigger**  
    - Type: Trigger node (Google Drive)  
    - Configuration: Watches for `fileCreated` events every hour in a specific folder identified by ID `1IsQ3KyyfiexcYPlOiAkzYaH4MHI4VBPZ` (named "GraphRag" in config cache).  
    - Credentials: Uses Google Drive OAuth2 credentials.  
    - Inputs: None (trigger)  
    - Outputs: Emits the new file metadata JSON.  
    - Potential Failures: Authentication issues, folder ID mismatch, API rate limits.

  - **If**  
    - Type: Conditional node  
    - Configuration: Checks if the file extension (`$json.fileExtension`) equals `"pdf"`.  
    - Inputs: From Google Drive Trigger  
    - Outputs: Two branches—true (PDF file) and false (non-PDF file)  
    - Potential Failures: Missing or malformed file extension field.

#### 1.2 File Type Routing & Text Extraction

- **Overview:**  
  Routes files based on type (PDF or text). Downloads the file from Google Drive, then extracts text content using appropriate extraction methods.

- **Nodes Involved:**  
  - Download file (for PDF)  
  - Download file1 (for text)  
  - Extract from File (PDF extraction)  
  - Extract from File1 (Text extraction)  
  - Merge

- **Node Details:**

  - **Download file**  
    - Type: Google Drive node (download operation)  
    - Configuration: Downloads the file by its ID (`$json.id`) from the Google Drive Trigger output for PDF files.  
    - Credentials: Google Drive OAuth2  
    - Inputs: From If node (true branch)  
    - Outputs: Binary file data for PDF extraction  
    - Edge Cases: File access permissions, download errors.

  - **Extract from File**  
    - Type: Extract from File node  
    - Configuration: Extracts text from a PDF file binary content.  
    - Inputs: From Download file  
    - Outputs: Extracted text JSON under `text` field  
    - Edge Cases: PDF parsing errors, corrupted files.

  - **Download file1**  
    - Type: Google Drive node (download operation)  
    - Configuration: Downloads file by ID taken from the parent folder of the new file (`$('Google Drive Trigger').item.json.parents[0]`), intended for text files.  
    - Credentials: Google Drive OAuth2  
    - Inputs: From If node (false branch)  
    - Outputs: Binary file data for text extraction  
    - Edge Cases: Same as Download file.

  - **Extract from File1**  
    - Type: Extract from File node  
    - Configuration: Extracts raw text from the downloaded file (text/plain).  
    - Inputs: From Download file1  
    - Outputs: Extracted text JSON under `text` field  
    - Edge Cases: Text encoding issues.

  - **Merge**  
    - Type: Merge node (combine)  
    - Configuration: Merges outputs from both extraction paths into a single stream for AI processing.  
    - Inputs: Extract from File (PDF path) and Extract from File1 (text path)  
    - Outputs: Single merged output stream  
    - Edge Cases: Timing or synchronization issues if multiple files arrive simultaneously.

#### 1.3 AI Processing

- **Overview:**  
  Sends the extracted meeting text to an AI agent running OpenAI GPT-4.1-mini to generate a structured JSON meeting summary that includes key decisions, notes, sentiment classification, and categorized tasks.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model (langchain integration)

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Configuration:  
      - Input Text: Uses extracted text from merged node (`$json.text`).  
      - System Message: Defines the assistant role as a rigorous meeting assistant tasked with converting meeting text into a strict JSON project plan with keys: summary, decisions (string array), notes (string array), meeting_sentiment (enum), and tasks (array of objects with description, owner, deadline, sentiment).  
      - Constraints: No prose or markdown, empty lists or `"TBD"` for unknowns.  
    - Inputs: From Merge node  
    - Outputs: JSON string output under `output` key  
    - Edge Cases: Model timeouts, malformed AI responses, API quota limits.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Configuration: Uses GPT-4.1-mini model (selected via credentials) as the underlying language model for the AI Agent node.  
    - Inputs: AI Agent's LLM integration  
    - Outputs: Language model responses  
    - Credentials: OpenAI API key  
    - Edge Cases: API authentication errors, model unavailability.

#### 1.4 Output Validation & Normalization

- **Overview:**  
  Parses the AI-generated JSON output safely, normalizes data types, groups tasks by sentiment categories, and prepares summary statistics to ensure reliable downstream processing.

- **Nodes Involved:**  
  - Data Validation (Code node)

- **Node Details:**

  - **Data Validation**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Robust parser that handles multiple possible AI output formats (e.g., raw string, nested in `choices[0].message.content`, or already parsed JSON).  
      - Parses JSON text strictly; throws errors on invalid JSON.  
      - Ensures all expected keys exist with default fallback values.  
      - Groups tasks into `positive`, `neutral`, and `negative` arrays.  
      - Calculates totals for each sentiment category.  
    - Inputs: From AI Agent node  
    - Outputs: Normalized JSON object with keys: summary, decisions, notes, meeting_sentiment, tasks_grouped, totals  
    - Edge Cases: Malformed AI output, missing keys, invalid sentiment values.

#### 1.5 HTML Summary Generation

- **Overview:**  
  Converts the normalized structured data into a clean, minimal HTML email format, including inline CSS optimized for Gmail, and sections for summary, decisions, notes, and tasks grouped by sentiment.

- **Nodes Involved:**  
  - Data Preparation (Code node)

- **Node Details:**

  - **Data Preparation**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Escapes HTML special characters to prevent injection.  
      - Builds HTML lists for decisions, notes, and tasks under positive, neutral, and negative headings.  
      - Includes counts for each task sentiment category.  
      - Produces a minimal and compatible HTML document with inline CSS styling and semantic sections.  
      - Adds a footer line crediting n8n as the sender.  
      - Outputs the HTML string as `email_html` along with original data.  
    - Inputs: From Data Validation node  
    - Outputs: JSON with `email_html` and structured data  
    - Edge Cases: Missing or empty arrays, HTML injection if escaping fails.

#### 1.6 Email Dispatch

- **Overview:**  
  Sends the generated HTML meeting summary via Gmail to the email address of the user who last modified the uploaded file.

- **Nodes Involved:**  
  - Send a message (Gmail node)

- **Node Details:**

  - **Send a message**  
    - Type: Gmail node (send email)  
    - Configuration:  
      - Recipient: Email address of last modifying user of the Google Drive file (`$('Google Drive Trigger').item.json.lastModifyingUser.emailAddress`).  
      - Subject: Includes meeting sentiment tone, e.g., `Meeting Summary — positive tone`.  
      - Message Body: Uses the generated HTML from `email_html`.  
      - Email type: HTML  
    - Credentials: Gmail OAuth2 credentials  
    - Inputs: From Data Preparation node  
    - Outputs: Email send confirmation  
    - Edge Cases: Authentication errors, invalid recipient email, Gmail API quota limits.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                     | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                                     |
|---------------------|--------------------------------|-----------------------------------|------------------------|-----------------------|----------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger | Google Drive Trigger (Trigger) | Detect new files in Drive folder  | —                      | If                    | ## Google Drive Trigger + File Routing<br>Watch New Files (Meeting Notes Folder). Routes files by MIME type.    |
| If                  | If Condition                   | Check if file is PDF              | Google Drive Trigger    | Download file, Download file1 | ## Google Drive Trigger + File Routing<br>Routes files by MIME type: PDF or TXT for extraction paths.          |
| Download file       | Google Drive (Download)         | Download PDF file                 | If (true branch)        | Extract from File      | ## Google Drive Trigger + File Routing<br>Downloads PDF files for text extraction.                             |
| Extract from File   | Extract from File               | Extract text from PDF             | Download file           | Merge                 | ## Google Drive Trigger + File Routing<br>Extracts text from PDF files.                                        |
| Download file1      | Google Drive (Download)         | Download text file                | If (false branch)       | Extract from File1     | ## Google Drive Trigger + File Routing<br>Downloads text files for extraction.                                 |
| Extract from File1  | Extract from File               | Extract text from TXT             | Download file1          | Merge                 | ## Google Drive Trigger + File Routing<br>Extracts text from plain text files.                                 |
| Merge               | Merge                         | Merge text extraction outputs     | Extract from File, Extract from File1 | AI Agent              |                                                                                                                |
| AI Agent            | LangChain AI Agent             | Generate structured meeting summary with sentiment and tasks | Merge                  | Data Validation       | ## AI Summarization<br>Uses OpenAI GPT-4o-mini to create JSON meeting summaries with key insights and tasks.   |
| OpenAI Chat Model   | LangChain OpenAI Model          | Underlying LLM for AI agent       | AI Agent (LLM input)    | AI Agent (LLM output)  | ## AI Summarization<br>Backend model GPT-4o-mini for text generation.                                          |
| Data Validation     | Code Node                     | Parse and normalize AI JSON output | AI Agent                | Data Preparation       | ## Validate & Structure Output<br>Robust parser and normalizer to ensure reliable, structured data.            |
| Data Preparation    | Code Node                     | Generate HTML email content        | Data Validation         | Send a message         | ## Generate HTML Summary<br>Creates a clean, Gmail-compatible HTML report from normalized data.                |
| Send a message      | Gmail Node                    | Send summary email                 | Data Preparation        | —                     | ## Send AI-Generated Report<br>Sends HTML email to file's last modifying user with sentiment in subject line.  |
| Sticky Note         | Sticky Note                   | Documentation & instructions      | —                      | —                     | ## 🧩 AI Meeting Summary & Email Automation<br>Workflow overview and setup instructions.                       |
| Sticky Note1        | Sticky Note                   | Documentation for file routing    | —                      | —                     | ## Google Drive Trigger + File Routing<br>Explains file detection and routing logic.                           |
| Sticky Note2        | Sticky Note                   | Documentation for AI summarization | —                      | —                     | ## AI Summarization<br>Details on AI agent's role and output format.                                          |
| Sticky Note3        | Sticky Note                   | Documentation for validation      | —                      | —                     | ## Validate & Structure Output<br>Explains JSON normalization and task grouping.                               |
| Sticky Note4        | Sticky Note                   | Documentation for HTML generation | —                      | —                     | ## Generate HTML Summary<br>Describes email report construction and output variable.                          |
| Sticky Note5        | Sticky Note                   | Documentation for sending email   | —                      | —                     | ## Send AI-Generated Report<br>Details on email sending node and message formatting.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Parameters:  
     - Event: `fileCreated`  
     - Poll Interval: Every hour  
     - Trigger on: Specific folder with ID `"1IsQ3KyyfiexcYPlOiAkzYaH4MHI4VBPZ"`  
   - Credential: Google Drive OAuth2 account  
   - Connect no input (trigger node).

2. **Add If node**  
   - Type: If  
   - Parameters: Check condition: `$json.fileExtension` equals `"pdf"` (case sensitive)  
   - Connect input from Google Drive Trigger node.

3. **Add Download file node (PDF path)**  
   - Type: Google Drive node  
   - Operation: Download  
   - File ID: `={{ $json.id }}` (from Google Drive Trigger output)  
   - Credential: Same Google Drive OAuth2  
   - Connect input from If node's true branch.

4. **Add Extract from File node (PDF extraction)**  
   - Type: Extract from File  
   - Operation: PDF  
   - Connect input from Download file node.

5. **Add Download file1 node (text path)**  
   - Type: Google Drive node  
   - Operation: Download  
   - File ID: `={{ $('Google Drive Trigger').item.json.parents[0] }}` (parent folder's first ID)  
   - Credential: Same Google Drive OAuth2  
   - Connect input from If node's false branch.

6. **Add Extract from File1 node (text extraction)**  
   - Type: Extract from File  
   - Operation: Text  
   - Connect input from Download file1 node.

7. **Add Merge node**  
   - Type: Merge  
   - Parameters: default (merge inputs into single output)  
   - Connect inputs from both Extract from File (PDF) and Extract from File1 (text) nodes.

8. **Add AI Agent node**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text input: `={{ $json.text }}` (text extracted)  
     - System Message:  
       ```
       You are a helpful You are a rigorous meeting assistant. Convert the provided meeting text into a project plan.
       Return ONLY strict JSON with these keys and types:
       - summary: string (<= 40 words)
       - decisions: string[]
       - notes: string[]
       - meeting_sentiment: one of {"positive","neutral","negative"}
       - tasks: array of objects {
          description: string,
          owner: string | "TBD",
          deadline: ISO date string | "TBD",
          sentiment: one of {"positive","neutral","negative"}
       }
       Constraints:
       - No prose, no markdown, no code fences.
       - If unknown, use empty list or "TBD".
       - Infer dates only if clearly stated; otherwise "TBD".
       assistant
       ```  
     - Prompt Type: define  
   - Connect input from Merge node.

9. **Add OpenAI Chat Model node**  
   - Type: LangChain OpenAI Chat Model  
   - Parameters:  
     - Model: `gpt-4.1-mini` (from list)  
   - Credential: OpenAI API key (with free credits or valid subscription)  
   - Connect input from AI Agent node's LLM connection.

10. **Connect AI Agent node's output to Data Validation (Code) node**  
    - Create a Code node named "Data Validation"  
    - Paste the provided robust JSON parsing JavaScript code (see section 2.4)  
    - Connect input from AI Agent node.

11. **Add Data Preparation (Code) node**  
    - Paste the provided HTML generation JavaScript code (see section 2.5)  
    - Connect input from Data Validation node.

12. **Add Send a message (Gmail) node**  
    - Type: Gmail node  
    - Parameters:  
      - Send To: `={{ $('Google Drive Trigger').item.json.lastModifyingUser.emailAddress }}`  
      - Subject: `=Meeting Summary — {{$json.meeting_sentiment}} tone`  
      - Message: `={{ $json.email_html }}`  
      - Email format: HTML  
    - Credential: Gmail OAuth2 account  
    - Connect input from Data Preparation node.

13. **Add Sticky Notes for documentation** (optional but recommended for clarity)  
    - Add notes describing each block as per sticky notes content in section 3.

14. **Save and activate the workflow.**  
15. **Test by dropping a PDF or TXT meeting transcript in the watched Google Drive folder.**  
16. **Verify an HTML summary email is sent to the last modifying user of the file.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 🧩 AI Meeting Summary & Email Automation: Automatically turn meeting files in Google Drive into structured summaries and send emails via Gmail.                                                                                                                                                                                                                                | Workflow overview sticky note                                                                                                                               |
| Google Drive Trigger + File Routing: Supports PDF and TXT meeting transcripts by routing files to appropriate extractors.                                                                                                                                                                                                                                                     | Sticky note describing file type routing                                                                                                                    |
| AI Summarization: Uses OpenAI GPT-4o-mini to generate structured JSON meeting summaries with sentiment and tasks to automate meeting insights extraction.                                                                                                                                                                                                                     | Sticky note describing AI agent role                                                                                                                        |
| Validate & Structure Output: Robust JSON parsing and normalization code to ensure reliable processing and task grouping by sentiment.                                                                                                                                                                                                                                         | Sticky note describing data validation code node                                                                                                           |
| Generate HTML Summary: Produces a clean, minimal HTML email report with inline CSS optimized for Gmail, including summary, decisions, notes, and tasks grouped by sentiment.                                                                                                                                                                                                   | Sticky note describing HTML generation code node                                                                                                           |
| Send AI-Generated Report: Automatically emails the meeting summary to the last modifying user, with sentiment shown in the subject line.                                                                                                                                                                                                                                      | Sticky note describing Gmail sending node                                                                                                                  |
| Setup instructions: Connect Google Drive, OpenAI, and Gmail credentials; set Drive trigger folder; paste system prompt into AI node; configure Gmail message field to `{{$json.email_html}}`; drop file to trigger.                                                                                                                                                               | Part of the main overview sticky note                                                                                                                      |
| Perfect for team syncs, standups, sprint reviews, or client calls — automates tedious manual meeting summary tasks.                                                                                                                                                                                                                                                           | From main overview sticky note                                                                                                                             |
| n8n official docs for Google Drive: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googledrive/                                                                                                                                                                                                                                                                | Official n8n node documentation for Google Drive nodes                                                                                                    |
| n8n official docs for Gmail node: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.gmail/                                                                                                                                                                                                                                                                          | Official n8n node documentation for Gmail node                                                                                                            |
| OpenAI API documentation: https://platform.openai.com/docs/api-reference/chat/create                                                                                                                                                                                                                                                                                            | Reference for Chat Completion API used via LangChain                                                                                                      |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a workflow automation and integration tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly accessible.