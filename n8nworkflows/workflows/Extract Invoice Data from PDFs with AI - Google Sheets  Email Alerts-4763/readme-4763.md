Extract Invoice Data from PDFs with AI - Google Sheets  Email Alerts

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-pdfs-with-ai---google-sheets--email-alerts-4763


# Extract Invoice Data from PDFs with AI - Google Sheets  Email Alerts

### 1. Workflow Overview

This workflow automates the extraction of invoice data from newly created PDF files in Google Drive, processes the extracted information using AI models, updates a Google Sheets spreadsheet with the data, and sends email alerts accordingly. It is designed for organizations handling invoices digitally who want to streamline data entry, storage, and notification processes.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and File Acquisition**: Detects new PDF files in Google Drive and downloads them for processing.
- **1.2 Data Extraction from PDF**: Extracts textual content from the downloaded PDF files.
- **1.3 AI Processing and Information Extraction**: Uses AI language models to extract structured invoice data from the raw text.
- **1.4 Data Storage and Update**: Updates Google Sheets and a Postgres vector store with the extracted data.
- **1.5 Email Notification**: Sends email alerts based on processed data or workflow outcomes.
- **1.6 Workflow Control and Fallback**: Includes nodes to handle no operations or fallback scenarios.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and File Acquisition

- **Overview**: This block detects newly created files in Google Drive and downloads them for further processing.
- **Nodes Involved**:
  - Google Drive Trigger - file creation
  - Download Binary - file

- **Node Details**:

  - **Google Drive Trigger - file creation**
    - **Type**: Trigger node; monitors Google Drive for new file creation events.
    - **Configuration**: Defaults to triggering on any new file creation in specified Google Drive folders or root.
    - **Key Expressions/Variables**: None explicitly configured; relies on trigger output.
    - **Connections**: Outputs to "Download Binary - file".
    - **Version Requirements**: Compatible with n8n Google Drive API implementation.
    - **Potential Failures**: Authentication errors with Google OAuth2, permission issues, or quota limits.
  
  - **Download Binary - file**
    - **Type**: Google Drive node to download binary content of the triggered file.
    - **Configuration**: Configured to download the file identified by the trigger node.
    - **Key Expressions/Variables**: Uses file ID from trigger.
    - **Connections**: Outputs to "Extract from PDF" and "Update DB - spreadsheet".
    - **Potential Failures**: Network timeouts, file not found errors, permission errors.

#### 1.2 Data Extraction from PDF

- **Overview**: Extracts raw text content from the downloaded PDF files to prepare for AI-based information extraction.
- **Nodes Involved**:
  - Extract from PDF

- **Node Details**:

  - **Extract from PDF**
    - **Type**: File processing node that extracts text from PDF binary data.
    - **Configuration**: Default settings extracting full textual content.
    - **Key Expressions/Variables**: Consumes binary data from "Download Binary - file".
    - **Connections**: Outputs to "Information Extractor".
    - **Potential Failures**: Corrupted PDF files, unsupported PDF formats, extraction errors.

#### 1.3 AI Processing and Information Extraction

- **Overview**: Uses AI language models to analyze extracted text and obtain structured invoice information.
- **Nodes Involved**:
  - Information Extractor
  - Ollama Model
  - Postgres PGVector Store
  - Create Email Agent

- **Node Details**:

  - **Information Extractor**
    - **Type**: LangChain Information Extractor node designed to parse text into structured data.
    - **Configuration**: Default configuration; likely uses AI model input internally.
    - **Key Expressions/Variables**: Receives PDF-extracted text; outputs structured invoice data.
    - **Connections**: Output to "Update DB - spreadsheet".
    - **Potential Failures**: AI model errors, unexpected input format, rate limits.

  - **Ollama Model**
    - **Type**: LangChain Language Model node using Ollama's AI model.
    - **Configuration**: Default or custom prompt model for language understanding tasks.
    - **Key Expressions/Variables**: Connected as language model provider for "Information Extractor".
    - **Connections**: AI language model input to "Information Extractor".
    - **Potential Failures**: Model unavailability, API issues, prompt errors.

  - **Postgres PGVector Store**
    - **Type**: LangChain vector store node using Postgres with PGVector extension.
    - **Configuration**: Stores or retrieves vector embeddings for AI tools.
    - **Key Expressions/Variables**: Interfaces with "Create Email Agent" as AI tool.
    - **Connections**: AI tool output to "Create Email Agent".
    - **Potential Failures**: Database connection errors, query failures, data consistency issues.

  - **Create Email Agent**
    - **Type**: LangChain OpenAI node configured to generate email content or agents.
    - **Configuration**: Uses OpenAI credentials; integrates with vector store.
    - **Key Expressions/Variables**: Receives AI tool input from "Postgres PGVector Store"; sends output to "Send Email".
    - **Connections**: Outputs to "Send Email".
    - **Potential Failures**: API key issues, rate limiting, text generation errors.

#### 1.4 Data Storage and Update

- **Overview**: Updates Google Sheets with extracted invoice data for record-keeping and analytics.
- **Nodes Involved**:
  - Update DB - spreadsheet

- **Node Details**:

  - **Update DB - spreadsheet**
    - **Type**: Google Sheets node for writing data.
    - **Configuration**: Appends or updates rows in a specific spreadsheet and sheet.
    - **Key Expressions/Variables**: Receives structured data from "Information Extractor" or "Download Binary - file".
    - **Connections**: Outputs to "Create Email Agent".
    - **Potential Failures**: Authentication issues, sheet access permissions, quota limits.

#### 1.5 Email Notification

- **Overview**: Sends notification emails based on the workflow results, such as confirmation or alerts.
- **Nodes Involved**:
  - Send Email
  - No Operation, do nothing

- **Node Details**:

  - **Send Email**
    - **Type**: Gmail node to send emails.
    - **Configuration**: Uses OAuth2 credentials for Gmail; email content generated by "Create Email Agent".
    - **Key Expressions/Variables**: Email body, recipient, subject derived from AI-generated data.
    - **Connections**: Outputs to "No Operation, do nothing".
    - **Potential Failures**: Authentication errors, message size limits, rate limiting.

  - **No Operation, do nothing**
    - **Type**: NoOp node; effectively ends email sending path.
    - **Configuration**: Default; no action performed.
    - **Connections**: None.
    - **Role**: Used as a terminal node in the email sending branch.

---

### 3. Summary Table

| Node Name                    | Node Type                                   | Functional Role                          | Input Node(s)                 | Output Node(s)              | Sticky Note                        |
|------------------------------|---------------------------------------------|----------------------------------------|------------------------------|-----------------------------|----------------------------------|
| Google Drive Trigger - file creation | Google Drive Trigger                     | Detect new PDF files                    |                              | Download Binary - file       |                                  |
| Download Binary - file         | Google Drive                               | Download triggered PDF file             | Google Drive Trigger          | Extract from PDF, Update DB - spreadsheet |                                  |
| Extract from PDF              | Extract from File                           | Extract text content from PDF           | Download Binary - file        | Information Extractor        |                                  |
| Ollama Model                 | LangChain Language Model                    | Provide AI language model for extraction |                              | Information Extractor (ai_languageModel) |                                  |
| Information Extractor        | LangChain Information Extractor             | Extract structured invoice data         | Extract from PDF, Ollama Model (ai_languageModel) | Update DB - spreadsheet        |                                  |
| Update DB - spreadsheet       | Google Sheets                              | Update invoice data in spreadsheet      | Download Binary - file, Information Extractor | Create Email Agent           |                                  |
| Postgres PGVector Store       | LangChain Vector Store (Postgres PGVector) | Manage vector embeddings for AI tools   |                              | Create Email Agent (ai_tool) |                                  |
| Create Email Agent            | LangChain OpenAI                           | Generate email content                   | Update DB - spreadsheet, Postgres PGVector Store | Send Email                 |                                  |
| Send Email                   | Gmail Node                                 | Send notification email                  | Create Email Agent            | No Operation, do nothing     |                                  |
| No Operation, do nothing      | NoOp                                      | Terminal node after sending email       | Send Email                   |                             |                                  |
| Sticky Note                  | Sticky Note                                | Documentation note                       |                              |                             |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger node**
   - Type: Google Drive Trigger (file creation)
   - Purpose: Detect new PDF files uploaded to Google Drive.
   - Configure OAuth2 credentials for Google Drive.
   - Leave default settings to trigger on any new file creation.

2. **Add Google Drive node to Download Binary**
   - Type: Google Drive
   - Purpose: Download the newly triggered file.
   - Connect input from Google Drive Trigger.
   - Configure to download the file using file ID from trigger.
   - Use same Google Drive OAuth2 credentials.

3. **Add Extract from File node**
   - Type: Extract from File
   - Purpose: Extract text content from the downloaded PDF.
   - Connect input from Download Binary node.
   - Use default settings to extract all text.

4. **Add Ollama Model node**
   - Type: LangChain Language Model (Ollama)
   - Purpose: Provide language model capabilities for AI processing.
   - Configure any required API or local model settings according to Ollama.
   - No direct input connection; used as AI language model provider.

5. **Add Information Extractor node**
   - Type: LangChain Information Extractor
   - Purpose: Extract structured invoice data from PDF text.
   - Connect main input from Extract from File node.
   - Connect AI languageModel input from Ollama Model node.
   - Default or custom extraction prompts can be configured if desired.

6. **Add Google Sheets node "Update DB - spreadsheet"**
   - Type: Google Sheets
   - Purpose: Update or append extracted invoice data to a spreadsheet.
   - Connect main input from Information Extractor node.
   - Also connect input from Download Binary node if additional metadata is needed.
   - Configure spreadsheet ID, sheet name, and authentication credentials (Google OAuth2).
   - Map extracted data fields to appropriate columns.

7. **Add Postgres PGVector Store node**
   - Type: LangChain Vector Store (Postgres PGVector)
   - Purpose: Manage vector embeddings for AI tools, facilitating semantic search or context.
   - Configure connection to Postgres database with PGVector extension.
   - No direct input connection; used as AI tool provider.

8. **Add Create Email Agent node**
   - Type: LangChain OpenAI
   - Purpose: Generate email content or agents using AI.
   - Configure OpenAI credentials (API key).
   - Connect main input from Update DB - spreadsheet node.
   - Connect ai_tool input from Postgres PGVector Store node.

9. **Add Gmail node "Send Email"**
   - Type: Gmail
   - Purpose: Send notification emails.
   - Connect input from Create Email Agent node.
   - Configure OAuth2 credentials for Gmail.
   - Map generated email content (subject, body, recipient) from AI output.

10. **Add No Operation node "No Operation, do nothing"**
    - Type: NoOp
    - Purpose: Terminal node after sending email.
    - Connect input from Send Email node.

11. **Connect all nodes as per the logical flow:**
    - Google Drive Trigger → Download Binary
    - Download Binary → Extract from PDF and Update DB - spreadsheet
    - Extract from PDF → Information Extractor
    - Ollama Model → Information Extractor (ai_languageModel)
    - Information Extractor → Update DB - spreadsheet
    - Update DB - spreadsheet → Create Email Agent
    - Postgres PGVector Store → Create Email Agent (ai_tool)
    - Create Email Agent → Send Email
    - Send Email → No Operation, do nothing

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                      |
|---------------------------------------------------------------------------------------------------------------------------|------------------------------------|
| The workflow leverages LangChain nodes that require proper API keys and access to AI models (Ollama, OpenAI).              | Official LangChain n8n integration docs |
| Google OAuth2 credentials must have permissions for Drive file monitoring, file download, Sheets editing, and Gmail sending. | Google Cloud Console setup          |
| Postgres database requires PGVector extension installed for vector storage functionality.                                  | https://pgvector.com/documentation/ |
| Email generation is AI-driven; ensure proper prompt tuning to avoid inaccurate or incomplete messages.                      | LangChain OpenAI prompt guidelines  |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and does not contain any illegal, offensive, or protected elements. All processed data is legal and publicly accessible.