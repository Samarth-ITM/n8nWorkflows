Generate Professional Document Drafts from PDFs with Google Drive, GPT-4 & Email Notifications

https://n8nworkflows.xyz/workflows/generate-professional-document-drafts-from-pdfs-with-google-drive--gpt-4---email-notifications-5441


# Generate Professional Document Drafts from PDFs with Google Drive, GPT-4 & Email Notifications

### 1. Workflow Overview

This workflow automates the generation of professional document drafts from PDF files uploaded to a specific Google Drive folder. It is designed for scenarios where structured extraction and summarization of PDF content are required, such as legal, business, or technical document processing. The workflow integrates Google Drive for file triggering and storage, uses OpenAI GPT-4 models for intelligent extraction and drafting, Google Docs for document creation and update, and Gmail for notification emails.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & File Download:** Detects new PDFs in a designated Google Drive folder and downloads them.
- **1.2 Text Extraction & Aggregation:** Extracts text content from PDF binaries and aggregates it.
- **1.3 Information Extraction:** Uses a LangChain-powered OpenAI information extractor to parse key data points from aggregated text.
- **1.4 Draft Generation:** Employs a GPT-4 based drafting agent to create a formal document summary from extracted information.
- **1.5 Google Docs Creation & Update:** Creates a Google Doc in the same Drive folder and updates it with the generated draft content.
- **1.6 Summary Email Notification:** Generates a concise email summary of the draft and sends a notification with a link to the Google Doc.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & File Download

**Overview:**  
Monitors a specific Google Drive folder for new PDF files and downloads them for further processing.

**Nodes Involved:**  
- Google Drive Trigger  
- Download Google Drive  

**Node Details:**

- **Google Drive Trigger**  
  - Type: Trigger node for Google Drive events  
  - Config: Watches a specific folder (`AI_Folder`) by folder ID, triggers every minute on new file creation  
  - Input: None (triggered by Google Drive event)  
  - Output: File metadata including file ID  
  - Edge Cases: Folder permission errors, API rate limits, delayed triggers if polling frequency is too low  
  - Credentials: Google Drive OAuth2  

- **Download Google Drive**  
  - Type: Operation node to download file content from Google Drive  
  - Config: Downloads file using dynamic file ID from trigger output  
  - Input: File ID from trigger node  
  - Output: Binary file content for downstream extraction  
  - Edge Cases: File not found, download timeout, permission denied  
  - Credentials: Google Drive OAuth2  

---

#### 1.2 Text Extraction & Aggregation

**Overview:**  
Extracts textual content from the PDF binary data and combines texts from multiple files into a single aggregated string for analysis.

**Nodes Involved:**  
- Extract from File  
- Aggregate Extract Text  

**Node Details:**

- **Extract from File**  
  - Type: File content extraction node  
  - Config: Extracts text from PDF binary data provided in the node's binary input property (`data`)  
  - Input: Binary PDF file from "Download Google Drive"  
  - Output: Extracted text as JSON property (`extracted_text` or `text`)  
  - Edge Cases: Corrupted PDFs, unsupported PDF formats, extraction failures  
  - Version: v1  

- **Aggregate Extract Text** (Code node)  
  - Type: JavaScript code node for data processing  
  - Config: Concatenates extracted text from all input items with delimiters (`\n---\n`), returns combined text string or error message if no text  
  - Key Expressions: Accesses `extracted_text` or `text` fields dynamically  
  - Input: Multiple extracted text items  
  - Output: Single JSON object with `combined_text` property for downstream use  
  - Edge Cases: Empty or missing text fields, handling multiple PDFs together  
  - Version: v2  

---

#### 1.3 Information Extraction

**Overview:**  
Applies an AI-powered information extractor to parse relevant structured data from the aggregated text, such as names, dates, financials, and references.

**Nodes Involved:**  
- Information Extractor  
- OpenAI Chat Model (linked as AI language model for extraction)  

**Node Details:**

- **Information Extractor**  
  - Type: LangChain node specialized for extracting detailed information from text  
  - Config: Uses a system prompt directing the AI to extract entities like names, dates, amounts, events, responsibilities, legal references, and outcomes. Output is organized into paragraphs/bullet points by category.  
  - Key Inputs: Combined PDF text (`{{ $json.combined_text }}`)  
  - Attributes Defined: Date, Letter Head Name, Address, Date of Loss (with types where applicable)  
  - Input: Aggregated text from "Aggregate Extract Text"  
  - Output: Structured extraction results suitable for drafting  
  - Edge Cases: Missing or ambiguous data, incomplete content, AI extraction inaccuracies  
  - Version: v1  
  - AI Model: Uses GPT-4 (node "OpenAI Chat Model") as backend  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI integration for language modeling  
  - Config: Uses GPT-4 model for information extraction tasks  
  - Input: Feeds text to "Information Extractor" as AI language model  
  - Edge Cases: API rate limits, auth errors, model response delays  
  - Credentials: OpenAI API key  

---

#### 1.4 Draft Generation

**Overview:**  
Generates a professionally structured draft summary document from the extracted information using an advanced GPT-4 based drafting agent.

**Nodes Involved:**  
- Drafting Agent  

**Node Details:**

- **Drafting Agent**  
  - Type: LangChain OpenAI node for text generation  
  - Config: Uses GPT-4.1-mini-2025-04-14 model to create formal, well-written summaries from extracted PDF information. The prompt instructs to rephrase and synthesize rather than copy verbatim, aiming for clarity and professionalism.  
  - Input: Extracted information text (`{{ $('Aggregate Extract Text').item.json.combined_text }}`)  
  - Output: Draft document content as text in the first choice message  
  - Edge Cases: Long input truncation, model hallucination, API failures  
  - Credentials: OpenAI API key  

---

#### 1.5 Google Docs Creation & Update

**Overview:**  
Creates a new Google Document in the designated folder, then updates it with the draft content generated by the AI agent.

**Nodes Involved:**  
- CREATE GOOGLE DOC  
- UPDATE Google Docs  

**Node Details:**

- **CREATE GOOGLE DOC**  
  - Type: Google Docs node  
  - Config: Creates a new document titled "Draft Document" in the same Google Drive folder being monitored  
  - Input: Trigger from "Drafting Agent" output  
  - Output: Document metadata including document ID  
  - Edge Cases: Folder permission issues, document creation failure  
  - Credentials: Google Docs OAuth2  

- **UPDATE Google Docs**  
  - Type: Google Docs node  
  - Config: Updates the newly created document by inserting the draft text from "Drafting Agent"'s output message  
  - Parameters: Insert action with text content dynamically taken from draft agent's first choice message  
  - Input: Document ID from "CREATE GOOGLE DOC" and draft content from "Drafting Agent"  
  - Output: Updated document status  
  - Edge Cases: Document locked or deleted during update, API errors  
  - Credentials: Google Docs OAuth2  

---

#### 1.6 Summary Email Notification

**Overview:**  
Generates a concise summary email of key points from the draft and sends it with a link to the Google Doc.

**Nodes Involved:**  
- Email Summary Agent  
- Edit Email Message  
- Send Email  

**Node Details:**

- **Email Summary Agent**  
  - Type: LangChain OpenAI node specialized for concise summarization  
  - Config: Uses GPT-4o-mini to extract 3–5 key bullet points focusing on relevant claims, coverage, losses, or responsible parties from aggregated text  
  - Input: Aggregated text from "Aggregate Extract Text"  
  - Output: Summary text starting with "Summary of Drafted Document:" and bulleted points  
  - Edge Cases: Summary missing key points, model hallucination  
  - Credentials: OpenAI API key  

- **Edit Email Message**  
  - Type: Set node for preparing email content  
  - Config: Constructs a URL string pointing to the updated Google Doc using the document ID from "CREATE GOOGLE DOC" node output  
  - Output: JSON property `DraftURL` containing the Google Docs edit link  
  - Edge Cases: Invalid or missing document ID  

- **Send Email**  
  - Type: Gmail node for sending email  
  - Config:  
    - Recipient: Fixed to "mgullo24@gmail.com" (can be parameterized)  
    - Subject: "Draft Agent"  
    - Body: Includes greeting, summary from "Email Summary Agent", the Google Doc link from "Edit Email Message", and a note about the automated nature of the message  
  - Input: Summary and draft URL  
  - Edge Cases: SMTP errors, authentication failures, email deliverability issues  
  - Credentials: Gmail OAuth2  

---

### 3. Summary Table

| Node Name            | Node Type                               | Functional Role                          | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                            |
|----------------------|---------------------------------------|----------------------------------------|------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger  | n8n-nodes-base.googleDriveTrigger     | Trigger on new PDFs in Drive folder    | —                      | Download Google Drive      | This step retrieves the PDF from the designated Google Drive folder and extracts key information from it using an OpenAI agent. The extraction process can be customized to target specific data within the PDF's binary content, ensuring the right information is included in the final draft. Currently, it captures all details that appear relevant for generating a complete draft. |
| Download Google Drive | n8n-nodes-base.googleDrive             | Downloads PDF binary                   | Google Drive Trigger    | Extract from File          |                                                                                                                        |
| Extract from File     | n8n-nodes-base.extractFromFile         | Extracts text from PDF binaries        | Download Google Drive   | Aggregate Extract Text     |                                                                                                                        |
| Aggregate Extract Text| n8n-nodes-base.code                    | Aggregates extracted text from PDFs    | Extract from File       | Information Extractor      |                                                                                                                        |
| OpenAI Chat Model     | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides AI model for information extraction | Aggregate Extract Text | Information Extractor      |                                                                                                                        |
| Information Extractor | @n8n/n8n-nodes-langchain.informationExtractor | Extracts structured information from text | Aggregate Extract Text  | Drafting Agent             |                                                                                                                        |
| Drafting Agent        | @n8n/n8n-nodes-langchain.openAi       | Generates formal draft summaries       | Information Extractor   | CREATE GOOGLE DOC          | This part of the workflow uses an OpenAI agent to generate a draft based on the extracted binary data from the PDF. It then creates a Google Document within the same Google Drive folder and updates it with the generated content. Finally, the workflow captures the document's URL and sends a summary email containing a link to the completed draft. |
| CREATE GOOGLE DOC     | n8n-nodes-base.googleDocs              | Creates new Google Doc for draft       | Drafting Agent          | UPDATE Google Docs         |                                                                                                                        |
| UPDATE Google Docs    | n8n-nodes-base.googleDocs              | Inserts draft content into Google Doc  | CREATE GOOGLE DOC       | Email Summary Agent        |                                                                                                                        |
| Email Summary Agent   | @n8n/n8n-nodes-langchain.openAi       | Summarizes draft into bullet points    | UPDATE Google Docs      | Edit Email Message         |                                                                                                                        |
| Edit Email Message    | n8n-nodes-base.set                     | Constructs email content with doc URL  | Email Summary Agent     | Send Email                |                                                                                                                        |
| Send Email            | n8n-nodes-base.gmail                   | Sends notification email with summary  | Edit Email Message      | —                         |                                                                                                                        |
| Sticky Note           | n8n-nodes-base.stickyNote              | Informational comment                   | —                      | —                         | This step retrieves the PDF from the designated Google Drive folder and extracts key information from it using an OpenAI agent. The extraction process can be customized to target specific data within the PDF's binary content, ensuring the right information is included in the final draft. Currently, it captures all details that appear relevant for generating a complete draft. |
| Sticky Note1          | n8n-nodes-base.stickyNote              | Informational comment                   | —                      | —                         | This part of the workflow uses an OpenAI agent to generate a draft based on the extracted binary data from the PDF. It then creates a Google Document within the same Google Drive folder and updates it with the generated content. Finally, the workflow captures the document's URL and sends a summary email containing a link to the completed draft. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Set event to "File Created"  
   - Set to watch a specific folder by folder ID (use the folder ID where PDFs are uploaded)  
   - Set polling interval to every minute  
   - Connect Google Drive OAuth2 credentials  

2. **Create Download Google Drive Node**  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: Use expression `{{$json["id"]}}` from trigger node output  
   - Connect output of Google Drive Trigger to input of this node  
   - Use same Google Drive OAuth2 credentials  

3. **Create Extract from File Node**  
   - Type: Extract from File  
   - Operation: PDF text extraction  
   - Binary Property Name: Set to `data` (to receive the downloaded PDF file)  
   - Connect output of Download Google Drive node to this node  

4. **Create Aggregate Extract Text Node (Code)**  
   - Type: Code  
   - Paste the provided JavaScript code to concatenate extracted text from multiple PDFs:  
     ```js
     let combinedText = '';

     for (const item of items) {
       const text = item.json.extracted_text || item.json.text || '';
       if (typeof text === 'string' && text.trim()) {
         combinedText += text.trim() + '\n---\n';
       }
     }

     if (combinedText === '') {
       return [{
         json: {
           data: 'No substantial text could be extracted from the provided PDFs.'
         }
       }];
     }

     return [{
       json: {
         combined_text: combinedText
       }
     }];
     ```
   - Connect output of Extract from File node to this node  

5. **Create OpenAI Chat Model Node for Extraction**  
   - Type: LangChain OpenAI Chat Model  
   - Model: Select `gpt-4`  
   - Leave options default  
   - Connect output of Aggregate Extract Text node to this node as AI language model input (ai_languageModel connection)  
   - Use OpenAI API credentials  

6. **Create Information Extractor Node**  
   - Type: LangChain Information Extractor  
   - Text Input: Use expression to pass combined text: `{{ $json.combined_text }}`  
   - System Prompt Template: "You are an expert extraction algorithm..." (use the provided prompt for extraction)  
   - Define attributes:  
     - Date (type: date)  
     - Letter Head Name  
     - Address  
     - Date of Loss (type: date)  
   - Connect output of Aggregate Extract Text node to this node’s main input  
   - Connect OpenAI Chat Model node as AI language model input  
   - Use same OpenAI API credentials  

7. **Create Drafting Agent Node**  
   - Type: LangChain OpenAI  
   - Model: Select `gpt-4.1-mini-2025-04-14`  
   - Messages: Provide system and user prompts as per workflow, instructing formal draft generation from extracted info  
   - Connect output of Information Extractor node to this node  
   - Use OpenAI API credentials  

8. **Create CREATE GOOGLE DOC Node**  
   - Type: Google Docs  
   - Operation: Create Document  
   - Title: Set as "Draft Document" (can be static or dynamic)  
   - Folder ID: Use the same folder ID as the trigger folder  
   - Connect output of Drafting Agent node to this node  
   - Use Google Docs OAuth2 credentials  

9. **Create UPDATE Google Docs Node**  
   - Type: Google Docs  
   - Operation: Update Document  
   - Document URL: Use expression to get new doc ID from CREATE GOOGLE DOC node, e.g. `={{ $node["CREATE GOOGLE DOC"].json.id }}`  
   - Actions: Insert the draft content text from Drafting Agent node's first choice message content  
   - Connect output of CREATE GOOGLE DOC node to this node  
   - Use Google Docs OAuth2 credentials  

10. **Create Email Summary Agent Node**  
    - Type: LangChain OpenAI  
    - Model: Select `gpt-4o-mini`  
    - Prompt: Ask for 3–5 bullet points summary focusing on claims, coverage, losses, or responsible parties, starting with "Summary of Drafted Document:"  
    - Connect output of UPDATE Google Docs node to this node  
    - Use OpenAI API credentials  

11. **Create Edit Email Message Node**  
    - Type: Set  
    - Add field `DraftURL` with value: `https://docs.google.com/document/d/{{ $node["CREATE GOOGLE DOC"].json.id }}/edit`  
    - Connect output of Email Summary Agent node to this node  

12. **Create Send Email Node**  
    - Type: Gmail  
    - Recipient: Set email address (example: mgullo24@gmail.com)  
    - Subject: "Draft Agent"  
    - Message Body: Include greeting, summary from Email Summary Agent node, Google Docs link from Edit Email Message node, and sign-off note about automation  
    - Connect output of Edit Email Message node to this node  
    - Use Gmail OAuth2 credentials  

13. **Connect all nodes in order:**  
    Google Drive Trigger → Download Google Drive → Extract from File → Aggregate Extract Text → Information Extractor (with OpenAI Chat Model as AI language model) → Drafting Agent → CREATE GOOGLE DOC → UPDATE Google Docs → Email Summary Agent → Edit Email Message → Send Email  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow uses advanced LangChain nodes integrated with OpenAI GPT-4 models to provide high-quality extraction and drafting capabilities.                            | n8n LangChain integration documentation                                                             |
| Google Drive folder ID and credentials must be properly configured and have appropriate permissions to read and write files.                                            | Google Drive API & OAuth2 setup                                                                      |
| Gmail node requires OAuth2 credentials with send email permission; ensure correct scopes are enabled for sending notifications.                                         | Gmail API OAuth2 integration                                                                         |
| The prompts for extraction and drafting can be customized to improve accuracy or target specific document types or industries.                                          | OpenAI prompt engineering best practices                                                            |
| Sticky notes in the workflow provide high-level descriptions of key processing blocks for easier maintenance and onboarding.                                            | Embedded in the workflow editor                                                                      |
| For large PDFs or high volume, consider API rate limits and polling frequency adjustments to avoid throttling or delays.                                               | OpenAI and Google Drive API usage guidelines                                                        |
| The workflow currently targets English language documents and formal drafting styles; modifications may be needed for other languages or informal formats.             | OpenAI language model capabilities                                                                  |
| For troubleshooting, inspect node execution logs, especially for extraction accuracy and API response errors.                                                           | n8n execution logs and error handling                                                                |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.