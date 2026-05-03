Parse Resumes from Gmail to Google Sheets with Azure OpenAI GPT-4o

https://n8nworkflows.xyz/workflows/parse-resumes-from-gmail-to-google-sheets-with-azure-openai-gpt-4o-8481


# Parse Resumes from Gmail to Google Sheets with Azure OpenAI GPT-4o

### 1. Workflow Overview

This workflow automates the process of parsing resumes received via Gmail and appending extracted structured data into a Google Sheets spreadsheet. It integrates Gmail for email reception, Google Drive for temporary file storage, Azure OpenAI GPT-4o for AI-powered resume parsing, and Google Sheets for data storage. The logical flow is divided into the following blocks:

- **1.1 Input Reception:** Watches Gmail inbox for incoming emails (assumed to be resumes), retrieves attachments.
- **1.2 File Handling:** Uploads attachments to Google Drive, downloads them again to prepare for extraction.
- **1.3 Content Extraction:** Extracts text content from the downloaded files.
- **1.4 AI Processing:** Uses Azure OpenAI GPT-4o through an AI Agent to parse resume content into structured data.
- **1.5 Data Handling:** Processes AI output and appends the parsed data into Google Sheets.

This modular design ensures clean separation of responsibilities and scalability.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Monitors Gmail inbox for incoming emails, specifically targeting those with resumes as attachments. Retrieves email attachments for further processing.
- **Nodes Involved:**  
  - Gmail Trigger  
  - Get the attahcment

- **Node Details:**

  - **Gmail Trigger**  
    - *Type:* Trigger node for Gmail  
    - *Role:* Watches Gmail inbox for new emails, triggers workflow on new message arrival  
    - *Configuration:* Default Gmail inbox monitoring, no filters specified (assumed to trigger on all incoming emails)  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Email metadata and message ID to next node  
    - *Failure cases:* Authentication errors, Gmail API rate limits, missing permissions  
    - *Version:* 1  

  - **Get the attahcment**  
    - *Type:* Gmail node  
    - *Role:* Retrieves attachments from the triggered email  
    - *Configuration:* Uses message ID from Gmail Trigger, fetches all attachments  
    - *Inputs:* Email metadata from Gmail Trigger  
    - *Outputs:* Attachment binary data and metadata  
    - *Failure cases:* Attachment missing, large attachments causing timeouts, Gmail API errors  
    - *Version:* 2.1

#### 2.2 File Handling

- **Overview:** Manages attachments by uploading them to Google Drive for temporary storage, then downloads them again to prepare for content extraction.
- **Nodes Involved:**  
  - Google Drive Upload  
  - Download file

- **Node Details:**

  - **Google Drive Upload**  
    - *Type:* Google Drive node  
    - *Role:* Uploads the email attachment to Google Drive  
    - *Configuration:* Uses binary data from "Get the attahcment", uploads to a specified folder or root (folder not specified in JSON)  
    - *Inputs:* Attachment data from Get the attahcment  
    - *Outputs:* Google Drive file metadata including file ID  
    - *Failure cases:* Google Drive API errors, insufficient permissions, quota exceeded  
    - *Version:* 1  

  - **Download file**  
    - *Type:* Google Drive node  
    - *Role:* Downloads the uploaded file from Google Drive to provide file content for extraction  
    - *Configuration:* Uses file ID from Google Drive Upload  
    - *Inputs:* File metadata from Google Drive Upload  
    - *Outputs:* Binary file data for extraction  
    - *Failure cases:* File not found, permission denied, API errors  
    - *Version:* 3

#### 2.3 Content Extraction

- **Overview:** Extracts text or other relevant content from the downloaded file to prepare it for AI parsing.
- **Nodes Involved:**  
  - Extract from File

- **Node Details:**

  - **Extract from File**  
    - *Type:* Extract from File node  
    - *Role:* Processes binary file input and extracts text content (e.g., from PDF, DOCX)  
    - *Configuration:* Default extraction settings (file type autodetection assumed)  
    - *Inputs:* Binary file data from Download file  
    - *Outputs:* Extracted text content in JSON or text format  
    - *Failure cases:* Unsupported file format, corrupted files, extraction timeout  
    - *Version:* 1

#### 2.4 AI Processing

- **Overview:** Sends extracted resume text to an AI agent configured with Azure OpenAI GPT-4o for semantic parsing, extracting structured resume data.
- **Nodes Involved:**  
  - AI Agent  
  - Azure OpenAI Chat Model

- **Node Details:**

  - **Azure OpenAI Chat Model**  
    - *Type:* Azure OpenAI Chat Model node  
    - *Role:* Provides GPT-4o language model endpoint for AI Agent  
    - *Configuration:* Azure OpenAI credentials configured; model set to GPT-4o  
    - *Inputs:* Prompt and context from AI Agent  
    - *Outputs:* Language model generated response  
    - *Failure cases:* Credential errors, API throttling, network issues  
    - *Version:* 1  

  - **AI Agent**  
    - *Type:* Langchain AI Agent node  
    - *Role:* Orchestrates AI prompt, manages conversation flow with Azure OpenAI GPT-4o to parse resume content  
    - *Configuration:* Uses Azure OpenAI Chat Model as language model; prompt templates likely set for resume parsing  
    - *Inputs:* Extracted resume text from Extract from File  
    - *Outputs:* Parsed resume data in structured JSON format  
    - *Failure cases:* Model response errors, prompt construction failures  
    - *Version:* 2.1

#### 2.5 Data Handling

- **Overview:** Processes the structured data output by the AI Agent and appends it as rows into a designated Google Sheets spreadsheet.
- **Nodes Involved:**  
  - Code  
  - Google Sheets Append

- **Node Details:**

  - **Code**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Transforms AI output, formats fields as needed before appending to Google Sheets  
    - *Configuration:* Contains custom JavaScript to map parsed JSON to sheet columns  
    - *Inputs:* Parsed resume data from AI Agent  
    - *Outputs:* Formatted data for Google Sheets  
    - *Failure cases:* Script errors, data structure mismatches  
    - *Version:* 2  

  - **Google Sheets Append**  
    - *Type:* Google Sheets node  
    - *Role:* Appends a new row with parsed resume data to a specified Google Sheets sheet  
    - *Configuration:* Spreadsheet ID and sheet name configured; data mapped from Code output  
    - *Inputs:* Formatted data from Code node  
    - *Outputs:* Confirmation of row append operation  
    - *Failure cases:* Permission errors, quota exceeded, invalid sheet ID  
    - *Version:* 3

---

### 3. Summary Table

| Node Name           | Node Type                                | Functional Role                      | Input Node(s)          | Output Node(s)           | Sticky Note                     |
|---------------------|----------------------------------------|------------------------------------|------------------------|--------------------------|--------------------------------|
| Gmail Trigger       | n8n-nodes-base.gmailTrigger             | Watches Gmail inbox for new emails | None                   | Get the attahcment        |                                |
| Get the attahcment  | n8n-nodes-base.gmail                    | Retrieves email attachments        | Gmail Trigger          | Google Drive Upload       |                                |
| Google Drive Upload | n8n-nodes-base.googleDrive              | Uploads attachment to Google Drive | Get the attahcment      | Download file             |                                |
| Download file       | n8n-nodes-base.googleDrive              | Downloads the uploaded file        | Google Drive Upload     | Extract from File         |                                |
| Extract from File   | n8n-nodes-base.extractFromFile          | Extracts text from file             | Download file           | AI Agent                  |                                |
| AI Agent            | @n8n/n8n-nodes-langchain.agent          | Parses resume text with AI          | Extract from File       | Code                      |                                |
| Azure OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Provides GPT-4o language model     | AI Agent (ai_languageModel) | AI Agent                |                                |
| Code                | n8n-nodes-base.code                     | Transforms AI output for Sheets    | AI Agent                | Google Sheets Append      |                                |
| Google Sheets Append| n8n-nodes-base.googleSheets             | Appends parsed data to spreadsheet | Code                    | None                      |                                |
| Sticky Note         | n8n-nodes-base.stickyNote                | Notes/Comments (multiple nodes)    | None                    | None                      | Various empty notes present     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Purpose: Monitor incoming emails  
   - Credentials: Connect your Gmail account with appropriate API access  
   - Parameters: Use default inbox monitoring, no filters (or set filters as needed for resumes)  

2. **Add Get the attahcment node (Gmail node)**  
   - Connect input to Gmail Trigger output  
   - Configure to fetch all attachments from the incoming email  
   - Use message ID from Gmail Trigger as input  

3. **Add Google Drive Upload node**  
   - Connect input to Get the attahcment output  
   - Configure Google Drive credentials  
   - Map attachment binary data as file content  
   - Set target folder (optional) or default root folder for upload  

4. **Add Download file node (Google Drive)**  
   - Connect input to Google Drive Upload output  
   - Use the uploaded file ID from previous node to download file content  
   - Configure credentials for Google Drive access  

5. **Add Extract from File node**  
   - Connect input to Download file output  
   - Configure to extract text from common resume formats (PDF, DOCX, etc.)  
   - Use default extraction settings or customize based on expected file types  

6. **Add AI Agent node (@n8n/n8n-nodes-langchain.agent)**  
   - Connect input to Extract from File output  
   - Configure to use an AI language model  
   - Ensure Azure OpenAI Chat Model is set as language model node  

7. **Add Azure OpenAI Chat Model node (@n8n/n8n-nodes-langchain.lmChatAzureOpenAi)**  
   - Connect to AI Agent node via `ai_languageModel` connection  
   - Configure Azure OpenAI credentials (API key, region)  
   - Set model to GPT-4o or desired model version  

8. **Add Code node**  
   - Connect input to AI Agent output  
   - Write JavaScript to parse AI response and transform it into an array/object matching Google Sheets columns  
   - Handle possible missing fields and normalize data  

9. **Add Google Sheets Append node**  
   - Connect input to Code node output  
   - Configure Google Sheets credentials  
   - Specify Spreadsheet ID and target sheet name  
   - Map data fields from Code node for appending as new row  

10. **Check credentials and permissions**  
    - Gmail: full read access to inbox and attachments  
    - Google Drive: read/write for file upload and download  
    - Google Sheets: write access to append rows  
    - Azure OpenAI: valid API keys and quota for GPT-4o usage  

11. **Test the workflow with sample emails containing resumes**  
    - Verify each step completes successfully  
    - Monitor for errors such as missing attachments, unsupported file types, API throttling, or data formatting issues  

---

### 5. General Notes & Resources

| Note Content                                                                          | Context or Link                                        |
|---------------------------------------------------------------------------------------|-------------------------------------------------------|
| Ensure Azure OpenAI GPT-4o is correctly provisioned with appropriate quotas and region | Azure OpenAI documentation                             |
| Google Drive upload is used as a temporary buffer for attachments before extraction   | Google Drive API best practices                         |
| AI Agent leverages Langchain framework nodes for flexible AI prompt handling          | https://js.langchain.com/docs/getting-started/        |
| Consider adding filters in Gmail Trigger to limit processing only to relevant emails | Gmail API filtering and label usage                     |
| Error handling for large attachments and unsupported file types should be implemented | n8n error workflow and retry mechanisms                  |

---

This document fully describes the workflow titled **“Parse Resumes from Gmail to Google Sheets with Azure OpenAI GPT-4o”**, enabling advanced users or AI agents to understand, reproduce, and maintain the automation confidently.