Summarize the New Documents from Google Drive and Save Summary in Google Sheet

https://n8nworkflows.xyz/workflows/summarize-the-new-documents-from-google-drive-and-save-summary-in-google-sheet-2754


# Summarize the New Documents from Google Drive and Save Summary in Google Sheet

### 1. Workflow Overview

This workflow automates the process of summarizing newly added Google Docs files in a specific Google Drive folder and storing the summaries in a Google Sheet. It is designed for users who want to efficiently manage and review document content without manual intervention.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Detects new Google Docs files added to a specified Google Drive folder.
- **1.2 Document Content Extraction:** Retrieves the full text content from the detected Google Doc.
- **1.3 AI Summarization:** Uses an AI model (OpenAI GPT-4o-mini) to generate a concise summary of the document content.
- **1.4 Data Storage:** Appends the summary along with document metadata into a designated Google Sheet for organized record-keeping.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new file is created in a specific Google Drive folder. It filters for Google Docs files and passes the file metadata downstream.

- **Nodes Involved:**  
  - Google Drive (Trigger)

- **Node Details:**  
  - **Google Drive (Trigger)**  
    - Type: Trigger node for Google Drive events  
    - Configuration:  
      - Event: `fileCreated`  
      - Polling frequency: every minute  
      - Folder to watch: Configured via URL (specific folder ID)  
      - Trigger on: Specific folder only  
    - Inputs: None (trigger node)  
    - Outputs: Emits metadata of the newly created file, including file ID and user info  
    - Credentials: Google Drive OAuth2  
    - Edge Cases / Failures:  
      - Authentication errors if OAuth token expires  
      - Folder ID misconfiguration leading to no triggers  
      - Rate limits on polling frequency  
    - Notes: Sticky note labeled "## Get Latest File" visually groups this node  

#### 1.2 Document Content Extraction

- **Overview:**  
  Retrieves the full textual content of the Google Doc file identified by the trigger node.

- **Nodes Involved:**  
  - Google Docs

- **Node Details:**  
  - **Google Docs**  
    - Type: Google Docs node for document operations  
    - Configuration:  
      - Operation: `get` (fetch document content)  
      - Document URL: Dynamically set to the file ID from the Google Drive trigger (`={{ $json.id }}`)  
    - Inputs: Receives file metadata from Google Drive trigger  
    - Outputs: Outputs the document content as JSON  
    - Credentials: Google Docs OAuth2  
    - Edge Cases / Failures:  
      - Invalid or missing document ID  
      - Permission denied if OAuth scope insufficient  
      - API rate limits or timeouts  
    - Notes: Sticky note labeled "## Get Document Content" groups this node  

#### 1.3 AI Summarization

- **Overview:**  
  Processes the extracted document content through an AI model to generate a concise summary.

- **Nodes Involved:**  
  - Generate Summary AI (OpenAI GPT-4o-mini)  
  - Wikipedia (LangChain Wikipedia tool)  
  - Calculator (LangChain Calculator tool)

- **Node Details:**  
  - **Generate Summary AI**  
    - Type: LangChain OpenAI node  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - Prompt: `"Summarise the below content\n {{ $json.content }}"` — dynamically injects document content  
      - Messages: Single message with summarization instruction  
    - Inputs: Receives document content from Google Docs node  
    - Outputs: AI-generated summary text  
    - Credentials: OpenAI API key (configured in n8n credentials)  
    - Edge Cases / Failures:  
      - API quota exceeded or authentication failure  
      - Prompt injection or formatting errors  
      - Timeout or incomplete responses  
    - Notes: Sticky note labeled "## AI Summarization" groups this node  
  - **Wikipedia** and **Calculator** nodes are connected as AI tools but not actively used in this workflow’s main path; they may serve as auxiliary tools for extended AI capabilities or future enhancements.

#### 1.4 Data Storage

- **Overview:**  
  Appends the summarized content along with document metadata (name, email of last modifier) into a Google Sheet for record-keeping.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Google Sheets**  
    - Type: Google Sheets node for data appending  
    - Configuration:  
      - Operation: `append` (add new row)  
      - Document ID: Configured via URL (target Google Sheet)  
      - Sheet Name: `Sheet1` (gid=0)  
      - Columns mapped:  
        - Name: `={{ $('Google Drive ').item.json.lastModifyingUser.displayName }}`  
        - Email: `={{ $('Google Drive ').item.json.lastModifyingUser.emailAddress }}`  
        - Summarise Content data: `={{ $json.message.content }}` (summary from AI node)  
    - Inputs: Receives AI summary and Google Drive metadata  
    - Outputs: Confirmation of row append operation  
    - Credentials: Google Sheets OAuth2  
    - Edge Cases / Failures:  
      - Incorrect Sheet ID or permissions  
      - Schema mismatch or missing columns  
      - API rate limits or quota issues  
    - Notes: Sticky note labeled "## Store Summary in Sheet" groups this node  

---

### 3. Summary Table

| Node Name          | Node Type                      | Functional Role               | Input Node(s)     | Output Node(s)       | Sticky Note                          |
|--------------------|--------------------------------|------------------------------|-------------------|----------------------|------------------------------------|
| Google Drive       | Google Drive Trigger           | Detect new Google Doc files  | None              | Google Docs          | Received the doc                   |
| Google Docs        | Google Docs Node               | Extract document content     | Google Drive      | Generate Summary AI   | ## Get Document Content            |
| Generate Summary AI| LangChain OpenAI Node          | Generate summary via AI      | Google Docs       | Google Sheets        | ## AI Summarization                |
| Wikipedia          | LangChain Wikipedia Tool       | Auxiliary AI tool            | Generate Summary AI| Generate Summary AI   |                                    |
| Calculator         | LangChain Calculator Tool      | Auxiliary AI tool            | Generate Summary AI| Generate Summary AI   |                                    |
| Google Sheets      | Google Sheets Node             | Append summary to sheet      | Generate Summary AI| None                 | ## Store Summary in Sheet          |
| Sticky Note        | Sticky Note                   | Visual grouping and notes    | None              | None                 | ## Get Latest File                 |
| Sticky Note1       | Sticky Note                   | Visual grouping and notes    | None              | None                 | ## Get Document Content            |
| Sticky Note2       | Sticky Note                   | Visual grouping and notes    | None              | None                 | ## AI Summarization                |
| Sticky Note3       | Sticky Note                   | Visual grouping and notes    | None              | None                 | ## Store Summary in Sheet          |
| Sticky Note4       | Sticky Note                   | Visual header                | None              | None                 | # Google Doc Summarizer to Google Sheets |
| Sticky Note5       | Sticky Note                   | Workflow description         | None              | None                 | ## Description                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure event: `fileCreated`  
   - Set polling interval: every 1 minute  
   - Set folder to watch: specify folder ID via URL mode  
   - Connect Google Drive OAuth2 credentials  
   - Position node as the workflow start  

2. **Add Google Docs Node**  
   - Type: Google Docs  
   - Operation: `get` (retrieve document content)  
   - Document URL: set expression to `={{ $json.id }}` from Google Drive trigger output  
   - Connect Google Docs OAuth2 credentials  
   - Connect Google Drive trigger node output to this node input  

3. **Add OpenAI Node for Summarization**  
   - Type: LangChain OpenAI node  
   - Model: Select `gpt-4o-mini` or equivalent GPT-4 variant  
   - Messages: Use prompt `"Summarise the below content\n {{ $json.content }}"`  
   - Connect OpenAI API credentials  
   - Connect Google Docs node output to this node input  

4. **Add Google Sheets Node**  
   - Type: Google Sheets  
   - Operation: `append` (add new row)  
   - Document ID: set target Google Sheet URL or ID  
   - Sheet Name: `Sheet1` (gid=0)  
   - Map columns:  
     - Name: `={{ $('Google Drive ').item.json.lastModifyingUser.displayName }}`  
     - Email: `={{ $('Google Drive ').item.json.lastModifyingUser.emailAddress }}`  
     - Summarise Content data: `={{ $json.message.content }}` (summary from AI node)  
   - Connect Google Sheets OAuth2 credentials  
   - Connect OpenAI node output to this node input  

5. **(Optional) Add Auxiliary AI Tool Nodes**  
   - Add Wikipedia and Calculator LangChain nodes if extended AI capabilities are desired  
   - Connect them as AI tools to the OpenAI node if needed  

6. **Add Sticky Notes for Documentation**  
   - Add sticky notes at appropriate positions with content:  
     - "# Google Doc Summarizer to Google Sheets" (header)  
     - "## Get Latest File" near Google Drive trigger  
     - "## Get Document Content" near Google Docs node  
     - "## AI Summarization" near OpenAI node  
     - "## Store Summary in Sheet" near Google Sheets node  
     - Full workflow description as a large sticky note  

7. **Test the Workflow**  
   - Upload a new Google Doc file to the watched folder  
   - Verify the workflow triggers, extracts content, generates summary, and appends data to the sheet  
   - Monitor for errors and adjust credentials or permissions as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow is created by AI developers at WeblineIndia to automate document summarization.       | Workflow description and purpose                             |
| For custom AI solutions and automation services, visit WeblineIndia’s AI development page.           | https://www.weblineindia.com/ai-development.html             |
| Ensure OAuth2 credentials for Google Drive, Google Docs, Google Sheets, and OpenAI are properly set. | Credential setup instructions                                |
| The AI model used is GPT-4o-mini via LangChain integration in n8n.                                  | AI summarization node configuration                          |
| Polling frequency is set to every minute; adjust based on API quota and use case.                   | Google Drive trigger configuration                           |

---

This document provides a complete and detailed reference to understand, reproduce, and maintain the "Summarize the New Documents from Google Drive and Save Summary in Google Sheet" workflow in n8n.