Summarize Google Docs & PDFs with GPT-4 and Send to Slack or Email

https://n8nworkflows.xyz/workflows/summarize-google-docs---pdfs-with-gpt-4-and-send-to-slack-or-email-8645


# Summarize Google Docs & PDFs with GPT-4 and Send to Slack or Email

### 1. Workflow Overview

This workflow automates the summarization of Google Docs and PDF documents using GPT-4 and then sends the generated summary to Slack and/or via email. It is designed for teams or individuals who want to automatically monitor a specific Google Drive folder, detect new or updated documents, extract their content, generate concise summaries with AI, and notify stakeholders promptly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Type Detection:** Watches a specific Google Drive folder for new or updated files and determines whether the file is a Google Doc (or similar) or a PDF.
- **1.2 Document Content Retrieval:** Depending on the file type, either retrieves the text content from Google Docs or downloads and extracts text from PDFs.
- **1.3 AI-Powered Summarization:** Sends the extracted text to an OpenAI GPT-4 based summarizer agent configured to produce structured summaries.
- **1.4 Notification Delivery:** Sends the generated summary to a Slack channel and/or via email to designated recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Type Detection

- **Overview:**  
  This block listens for new or updated files in a specified Google Drive folder every minute. It then checks the MIME type of each file to route Google Docs or other text files differently from PDFs.

- **Nodes Involved:**  
  - File Created  
  - Check Document Type  
  - Sticky Note (contextual comment)

- **Node Details:**

  - **File Created**  
    - Type: Google Drive Trigger  
    - Role: Watches a specific Google Drive folder for new or updated files (polls every minute).  
    - Config: Set to trigger on updates in a specific folder by folder ID. Uses Google Drive OAuth2 credentials.  
    - Input/Output: No input; outputs metadata of changed files including MIME type and ID.  
    - Edge Cases: Possible API rate limits, authentication failure, or delay in file availability after trigger.  
    - Sticky Note Content: "📑 Auto trigger and checking file type"

  - **Check Document Type**  
    - Type: If node (conditional branching)  
    - Role: Checks if the MIME type contains "application/vnd.google-apps." which identifies Google Docs or similar files.  
    - Config: Condition checks for MIME type string containment.  
    - Input: Output from File Created node.  
    - Output: Two branches—True (Google Docs / text files) and False (PDF files).  
    - Edge Cases: Files with unusual or missing MIME types may not be routed correctly.  
    - Sticky Note Content: Part of "📑 Auto trigger and checking file type"

  - **Sticky Note (da8dfa4a)**  
    - Content: "## 📑 Auto trigger and checking file type"  
    - Role: Describes this block’s purpose.

---

#### 2.2 Document Content Retrieval

- **Overview:**  
  This block handles two parallel routes based on file type: Google Docs are retrieved via API and their content is extracted; PDFs are downloaded and their text content extracted using a dedicated node.

- **Nodes Involved:**  
  - Get a document  
  - Set Field  
  - Download PDF  
  - Convert to Text  
  - Sticky Notes (Doc/TXT route and PDF route)

- **Node Details:**

  - **Get a document**  
    - Type: Google Docs node  
    - Role: Retrieves the full content of a Google Doc given its document URL (uses file ID).  
    - Config: Document URL is dynamically set from the "Check Document Type" node’s output. Uses Google Docs OAuth2 credentials.  
    - Input: True branch from Check Document Type.  
    - Output: JSON with Google Doc content.  
    - Edge Cases: Document access permission errors, invalid URL, or API quota exceeded.  
    - Sticky Note Content: Part of "📑 Docs, TXT file route"

  - **Set Field**  
    - Type: Set node  
    - Role: Normalizes the document content by extracting the "content" field and assigning it to a uniform "text" field for downstream AI summarizer.  
    - Config: Sets a new field `text` with value from the document content.  
    - Input: Output from Get a document.  
    - Output: JSON with field `text`.  
    - Edge Cases: Missing or empty content field can cause summarization to fail.  
    - Sticky Note Content: Part of "📑 Docs, TXT file route"

  - **Download PDF**  
    - Type: Google Drive node  
    - Role: Downloads the PDF file binary data given its file ID.  
    - Config: File ID parameter dynamically obtained from Check Document Type node’s output. Uses Google Drive OAuth2 credentials.  
    - Input: False branch from Check Document Type.  
    - Output: Binary data of the PDF file saved under binary property `data`.  
    - Edge Cases: Download failures due to permissions, file not found, or API limits.  
    - Sticky Note Content: Part of "📑 PDF file route"

  - **Convert to Text**  
    - Type: Extract From File node  
    - Role: Extracts text content from the downloaded PDF binary data.  
    - Config: Operation set to "pdf". Input is binary property `data`.  
    - Input: Output from Download PDF.  
    - Output: Extracted text in JSON format under `content` or similar.  
    - Edge Cases: PDFs that are scanned images or encrypted may cause extraction failure or produce poor text.  
    - Sticky Note Content: Part of "📑 PDF file route"

  - **Sticky Note (d6411536)**  
    - Content: "## 📑 Docs, TXT file route"  
    - Describes the nodes handling Google Docs and text files.

  - **Sticky Note (ba29ca4f)**  
    - Content: "## 📑 PDF file route"  
    - Describes the nodes handling PDFs.

---

#### 2.3 AI-Powered Summarization

- **Overview:**  
  This block takes the extracted text from either route and sends it to an OpenAI GPT-4 chat model to generate a structured document summary with key points and sentiment analysis.

- **Nodes Involved:**  
  - Doc Summarizer (Langchain Agent node)  
  - OpenAI Chat Model (Language Model node)  
  - Sticky Note (Summarizing docs)

- **Node Details:**

  - **Doc Summarizer**  
    - Type: Langchain Agent node  
    - Role: Acts as an AI agent that summarizes the document text according to a detailed system prompt with strict output formatting rules.  
    - Config:  
      - Text input: `Summarize the following document in less than 300 characters: {{ $json.text }}`  
      - System message instructs the agent to produce title, summary, key points, action items, language code, sentiment, truncation flag, and summary length.  
      - Prompt type: "define" (structured prompt)  
    - Input: Takes uniform `text` field from previous Set Field or Convert to Text node.  
    - Output: JSON with structured summary fields under `output`.  
    - Edge Cases: Very short or empty input text can cause poor or empty summaries; API rate limits or quota errors possible; prompt misconfiguration leads to bad summaries.  
    - Sticky Note Content: "## 📑 Summarizing docs"

  - **OpenAI Chat Model**  
    - Type: Langchain LM Chat OpenAI node  
    - Role: Provides the underlying GPT-4 language model instance for the Doc Summarizer agent node.  
    - Config: Model set to "gpt-4.1-mini"; uses OpenAI API credentials.  
    - Input: None (invoked by Doc Summarizer agent).  
    - Output: Language model responses to the Doc Summarizer.  
    - Edge Cases: Authentication failures, API downtime, or unsupported model errors.  
    - Sticky Note Content: Part of "## 📑 Summarizing docs"

  - **Sticky Note (cd769f6e)**  
    - Content: "## 📑 Summarizing docs"  
    - Describes the purpose of the summarization block.

---

#### 2.4 Notification Delivery

- **Overview:**  
  This block sends the AI-generated document summary as a message to a Slack channel and as an email to a specified recipient.

- **Nodes Involved:**  
  - Send message to Slack Channel  
  - Email the Summary  
  - Sticky Note (Notifier)

- **Node Details:**

  - **Send message to Slack Channel**  
    - Type: Slack node  
    - Role: Posts the summary text to a specific Slack channel using OAuth2 authentication.  
    - Config:  
      - Text content mapped dynamically from `output` field of Doc Summarizer.  
      - Channel fixed to "general" (channel ID: C096E5P119V).  
      - Uses Slack OAuth2 credentials.  
    - Input: Output from Doc Summarizer.  
    - Output: Slack API response confirming message delivery.  
    - Edge Cases: Slack API rate limits, invalid channel ID, or OAuth token expiry.  
    - Sticky Note Content: "## 📑 Notifier"

  - **Email the Summary**  
    - Type: Gmail node  
    - Role: Sends the summary text as an email to a hardcoded recipient using Gmail OAuth2 credentials.  
    - Config:  
      - Recipient: krishna@triggerall.com  
      - Subject: "Doc Summary"  
      - Email body: summary text from Doc Summarizer’s output.  
      - Email type: plain text, no attribution appended.  
    - Input: Output from Doc Summarizer.  
    - Output: Email sending confirmation.  
    - Edge Cases: Gmail API quota, authentication failure, or invalid recipient.  
    - Sticky Note Content: Part of "## 📑 Notifier"

  - **Sticky Note (8515df5f)**  
    - Content: "## 📑 Notifier"  
    - Describes the notification block purpose.

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                       | Input Node(s)           | Output Node(s)                         | Sticky Note                                              |
|----------------------------|-----------------------------------|------------------------------------|------------------------|--------------------------------------|----------------------------------------------------------|
| Sticky Note1               | Sticky Note                       | Setup guide and instructions        | -                      | -                                    | # 🛠️ Setup Guide... (full detailed setup instructions)    |
| File Created               | Google Drive Trigger              | Watches folder for new/updated files| -                      | Check Document Type                   | 📑 Auto trigger and checking file type                    |
| Check Document Type        | If node                          | Routes files by MIME type           | File Created           | Get a document (True), Download PDF (False) | 📑 Auto trigger and checking file type                    |
| Get a document             | Google Docs                      | Retrieves Google Docs content       | Check Document Type (True) | Set Field                            | 📑 Docs, TXT file route                                   |
| Set Field                  | Set node                        | Normalizes text field for AI input  | Get a document          | Doc Summarizer                       | 📑 Docs, TXT file route                                   |
| Download PDF               | Google Drive                    | Downloads PDF binary data           | Check Document Type (False) | Convert to Text                     | 📑 PDF file route                                        |
| Convert to Text            | Extract From File               | Extracts text from PDF              | Download PDF            | Doc Summarizer                       | 📑 PDF file route                                        |
| OpenAI Chat Model          | Langchain LM Chat OpenAI        | Provides GPT-4 model for summarizer | Doc Summarizer (ai_languageModel) | Doc Summarizer (agent)             | 📑 Summarizing docs                                      |
| Doc Summarizer             | Langchain Agent                 | AI document summarization agent    | Set Field / Convert to Text | Send message to Slack Channel, Email the Summary | 📑 Summarizing docs                                      |
| Send message to Slack Channel | Slack                         | Posts summary to Slack channel     | Doc Summarizer          | -                                    | 📑 Notifier                                              |
| Email the Summary          | Gmail                          | Sends summary via email            | Doc Summarizer          | -                                    | 📑 Notifier                                              |
| Sticky Note                | Sticky Note                    | Notes for auto trigger block       | -                      | -                                    | 📑 Auto trigger and checking file type                    |
| Sticky Note3               | Sticky Note                    | Notes for Docs/TXT route            | -                      | -                                    | 📑 Docs, TXT file route                                   |
| Sticky Note4               | Sticky Note                    | Notes for summarizing docs          | -                      | -                                    | 📑 Summarizing docs                                      |
| Sticky Note5               | Sticky Note                    | Notes for notifier block            | -                      | -                                    | 📑 Notifier                                              |
| Sticky Note6               | Sticky Note                    | Notes for PDF route                 | -                      | -                                    | 📑 PDF file route                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Folder**  
   - Create a folder in Google Drive where files (Google Docs or PDFs) will be added for summarization.

2. **Add Google Drive Trigger Node ("File Created")**  
   - Node Type: Google Drive Trigger  
   - Credentials: Set up Google Drive OAuth2 credentials (client ID, secret).  
   - Parameters:  
     - Event: "watchFolderUpdated"  
     - Poll interval: every minute  
     - Folder to watch: select the folder created above by ID or URL.

3. **Add If Node ("Check Document Type")**  
   - Node Type: If  
   - Input: Connect output of "File Created".  
   - Condition:  
     - Check if `mimeType` contains `application/vnd.google-apps.` (case sensitive).  
   - Output branches:  
     - True branch: Google Docs/Text files  
     - False branch: PDFs

4. **Google Docs/Text Files Route**

   4.1. **Add Google Docs Node ("Get a document")**  
        - Credentials: Google Docs OAuth2 credentials.  
        - Parameters:  
          - Operation: Get  
          - Document URL: `={{ $('Check Document Type').item.json.id }}` (dynamically set from file ID).  
        - Connect True branch of If node to this node.

   4.2. **Add Set Node ("Set Field")**  
        - Purpose: Extract content field and assign to `text` field for AI input.  
        - Parameters:  
          - Add field "text" with value: `={{ $json.content }}`  
        - Connect output of "Get a document" to this node.

5. **PDF Files Route**

   5.1. **Add Google Drive Node ("Download PDF")**  
        - Credentials: Google Drive OAuth2 credentials.  
        - Parameters:  
          - Operation: Download  
          - File ID: `={{ $json.id }}` (from If node output).  
          - Binary Property Name: `data`  
        - Connect False branch of If node to this node.

   5.2. **Add Extract From File Node ("Convert to Text")**  
        - Parameters:  
          - Operation: pdf  
          - Input binary property: `data` (from "Download PDF")  
        - Connect output of "Download PDF" node to this node.

6. **Add Langchain LM Chat OpenAI Node ("OpenAI Chat Model")**  
   - Credentials: OpenAI API key.  
   - Parameters:  
     - Model: gpt-4.1-mini (or preferred GPT-4 variant).  
   - No direct input connections (used as the underlying model for agent).

7. **Add Langchain Agent Node ("Doc Summarizer")**  
   - Parameters:  
     - Text: `=Summarize the following document in less than 300 characters:{{ $json.text }}` (dynamic from previous nodes).  
     - System Message:  
       ```
       You are a concise, factual document summarizer. 
       - Title: short title (<= 60 chars)
       - Summary: concise summary (3–5 sentences; <= 300 characters)
       - Key Points: array of up to 5 short bullet points (each <= 140 chars)
       - Action Items: array of up to 3 suggested actions (may be empty)
       - Language: 2-letter ISO code of detected language (e.g., "en")
       - Sentiment: "positive", "neutral", or "negative"
       - Truncated: boolean (true if input was trimmed / chunked)
       - Summary Length Chars: integer (actual character length of summary)
       - Max Tokens: 200-300
       
       Rules:
       1. Keep the summary factual; do not invent facts or dates. If unsure, use "likely" or omit the fact.
       2. If the input contains clear dates, people, numbers or deadlines, include them in key_points.
       5. Always follow the character / length constraints above.
       ```
     - Prompt Type: Define  
   - Input: Connect output of "Set Field" (Docs/Text route) and "Convert to Text" (PDF route) to this node’s main input.  
   - Connect OpenAI Chat Model node as the `ai_languageModel` for this agent node.

8. **Add Slack Node ("Send message to Slack Channel")**  
   - Credentials: Slack OAuth2 credentials.  
   - Parameters:  
     - Channel: Select your target Slack channel (e.g., "general").  
     - Text: `={{ $json.output }}` (summary text from Doc Summarizer).  
   - Connect output of "Doc Summarizer" node to this node.

9. **Add Gmail Node ("Email the Summary")**  
   - Credentials: Gmail OAuth2 credentials.  
   - Parameters:  
     - Send To: e.g., "krishna@triggerall.com" (set your recipient).  
     - Subject: "Doc Summary"  
     - Message: `={{ $json.output }}` (summary from Doc Summarizer).  
     - Email Type: Text  
     - Options: Disable appending attribution.  
   - Connect output of "Doc Summarizer" node to this node.

10. **Activate and Test**  
    - Save the workflow.  
    - Add test documents (Google Docs and PDFs) to the monitored Google Drive folder.  
    - Verify that summaries are generated and notifications are sent to Slack and email as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Setup guide with detailed step instructions and author credit to Krishna Sharma. Includes setup of Google Drive folder, OAuth credentials, node configuration, and testing instructions.                                                                           | Embedded as Sticky Note in workflow; author: Krishna Sharma                                                  |
| Google OAuth2 API credentials setup documentation: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service                                                                                                                                | Credential setup reference                                                                                     |
| OpenAI API integration guide: https://docs.n8n.io/integrations/builtin/credentials/openAi                                                                                                                                                                          | Credential setup reference                                                                                     |
| Slack OAuth2 credential and API usage instructions available in n8n docs.                                                                                                                                                                                         | Credential setup reference                                                                                     |
| Use of Langchain Agent and Langchain LM Chat OpenAI nodes for structured prompt-based AI summarization.                                                                                                                                                            | n8n Langchain nodes documentation                                                                              |
| Potential improvements: Add dynamic email recipient input; handle scanned PDFs with OCR; add error handling nodes for API failures; add chunking for very large documents.                                                                                         | Suggestions for enhancement                                                                                     |

---

**Disclaimer:** The provided text is derived solely from an n8n automated workflow. All data and operations comply with current content policies and contain no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.