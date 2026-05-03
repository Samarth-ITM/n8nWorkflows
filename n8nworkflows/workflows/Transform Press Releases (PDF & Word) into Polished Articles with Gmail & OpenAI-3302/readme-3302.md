Transform Press Releases (PDF & Word) into Polished Articles with Gmail & OpenAI

https://n8nworkflows.xyz/workflows/transform-press-releases--pdf---word--into-polished-articles-with-gmail---openai-3302


# Transform Press Releases (PDF & Word) into Polished Articles with Gmail & OpenAI

### 1. Workflow Overview

This workflow automates the transformation of press releases received via Gmail into polished, publication-ready articles using AI models. It targets editors and journalists who manage incoming press releases from various sources (governments, companies, NGOs, individuals). The workflow extracts text from email bodies or attachments (PDF or Word), processes the content through AI to generate structured articles, and performs a self-assessment comparing the original input with the AI-generated output. Finally, it replies to the original sender with the article draft, self-assessment, and original content for review.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Filtering:** Triggered by incoming Gmail messages, filters attachments to keep only PDFs and Word documents.
- **1.2 Attachment Presence and Type Decision:** Checks if attachments exist and routes processing based on attachment type (PDF, Word, or none).
- **1.3 Content Extraction:** Extracts text content from PDFs or Word documents, or uses inline email text if no attachments.
- **1.4 AI Article Generation:** Uses AI agents to convert extracted or inline text into a polished article following editorial guidelines.
- **1.5 AI Self-Assessment:** AI compares original input and generated article to evaluate accuracy and quality.
- **1.6 Reply Email Composition and Sending:** Sends a reply email to the original sender containing the article, self-assessment, and original content.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

- **Overview:**  
  This block listens for incoming emails from authorized senders, downloads attachments, and filters them to retain only PDF and Word files.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Code: delete all but pdf and word  
  - has attachment? (If node)

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Configuration:  
      - Polls Gmail every 1 minute  
      - Filters emails from sender domain `*@somedia.ch`  
      - Downloads attachments with prefix `attachment_`  
    - Inputs: None (trigger)  
    - Outputs: Email data including attachments  
    - Potential Failures: Authentication errors, Gmail API rate limits, network issues

  - **Code: delete all but pdf and word**  
    - Type: Function (Code) node  
    - Configuration:  
      - Iterates over all binary attachments  
      - Keeps only those with MIME types:  
        - `application/pdf`  
        - `application/msword`  
        - `application/vnd.openxmlformats-officedocument.wordprocessingml.document`  
      - Outputs filtered attachments as new items  
    - Inputs: Gmail Trigger output  
    - Outputs: Filtered attachments only  
    - Edge Cases: No attachments, unsupported MIME types, malformed binary data

  - **has attachment?**  
    - Type: If node  
    - Configuration:  
      - Checks if any binary data exists (attachment count > 0)  
    - Inputs: Filtered attachments from Code node  
    - Outputs:  
      - True branch: attachments exist  
      - False branch: no attachments  
    - Edge Cases: Empty attachments array, binary data missing

---

#### 2.2 Attachment Type Decision

- **Overview:**  
  Determines if the attachment is a PDF or Word document to select the appropriate extraction method.

- **Nodes Involved:**  
  - PDF or WORD? (If node)

- **Node Details:**

  - **PDF or WORD?**  
    - Type: If node  
    - Configuration:  
      - Checks if the attachment MIME type contains "pdf" (case-insensitive)  
      - True branch: PDF processing  
      - False branch: Word processing  
    - Inputs: Filtered attachments from has attachment? node  
    - Outputs: Routes to PDF extraction or Word extraction  
    - Edge Cases: MIME type missing or malformed, unsupported file types slipping through

---

#### 2.3 Content Extraction

- **Overview:**  
  Extracts text content from attachments or uses inline email text if no attachments.

- **Nodes Involved:**  
  - Extrahiere aus PDF1 (Extract from File)  
  - Google Drive  
  - HTTP Request2 (Copy file to Google Docs)  
  - HTTP Request3 (Export Google Doc as plain text)  
  - AI Article Writer 1 (PDF path)  
  - AI Article Writer 2 (Word path)  
  - AI Article Writer 3 (No attachment path)

- **Node Details:**

  - **Extrahiere aus PDF1**  
    - Type: Extract from File node  
    - Configuration:  
      - Operation: PDF text extraction  
    - Inputs: PDF attachments from PDF or WORD? node  
    - Outputs: Extracted text from PDF  
    - Edge Cases: Corrupt PDFs, extraction failures, large files causing timeouts

  - **Google Drive**  
    - Type: Google Drive node  
    - Configuration:  
      - Uploads the Word attachment to Google Drive root folder  
      - File name dynamically set from attachment filename  
    - Inputs: Word attachments from PDF or WORD? node (False branch)  
    - Outputs: File metadata including file ID  
    - Edge Cases: Google Drive API errors, insufficient permissions, quota limits

  - **HTTP Request2**  
    - Type: HTTP Request node  
    - Configuration:  
      - POST request to Google Drive API to copy uploaded file as Google Docs format  
      - Uses OAuth2 credentials for Google Docs API  
    - Inputs: Google Drive file metadata  
    - Outputs: Google Docs file metadata with new file ID  
    - Edge Cases: API errors, invalid file ID, auth token expiration

  - **HTTP Request3**  
    - Type: HTTP Request node  
    - Configuration:  
      - GET request to export Google Docs file as plain text  
      - Uses OAuth2 credentials for Google Docs API  
    - Inputs: Google Docs file ID from HTTP Request2  
    - Outputs: Plain text content of Word document  
    - Edge Cases: API errors, export failures, large document size

  - **AI Article Writer 1**  
    - Type: LangChain Agent node (Anthropic Claude model)  
    - Configuration:  
      - Converts extracted PDF text plus email text into a polished article  
      - Editorial instructions include Swiss German spelling, neutral style, inverted pyramid structure, gender-neutral wording, no invented info  
      - Output: publication-ready article text  
    - Inputs: Extracted PDF text + email text  
    - Outputs: Article text  
    - Edge Cases: AI model timeouts, prompt errors, incomplete generation

  - **AI Article Writer 2**  
    - Type: LangChain Agent node (Anthropic Claude model)  
    - Configuration:  
      - Similar editorial instructions as AI Article Writer 1  
      - Processes Word document text plus email text  
    - Inputs: Extracted Word text + email text  
    - Outputs: Article text  
    - Edge Cases: Same as AI Article Writer 1

  - **AI Article Writer 3**  
    - Type: LangChain Agent node (Anthropic Claude model)  
    - Configuration:  
      - Processes inline email text only (no attachments)  
      - Same editorial instructions as above  
    - Inputs: Email text only  
    - Outputs: Article text  
    - Edge Cases: Insufficient input text, AI generation errors

---

#### 2.4 AI Self-Assessment

- **Overview:**  
  AI compares the original input with the generated article to evaluate completeness, accuracy, and quality.

- **Nodes Involved:**  
  - OpenAI self-assesment (for Word path)  
  - OpenAI self-assesment2 (for PDF path)  
  - OpenAI self assesment (for no attachment path)

- **Node Details:**

  - **OpenAI self-assesment**  
    - Type: LangChain OpenAI node (GPT-4o-mini)  
    - Configuration:  
      - Prompt instructs AI to compare input text and output article  
      - Rates from 1 (poor) to 5 (excellent) on completeness, accuracy, added info, and provides textual justification  
      - Input includes Word-extracted text + email text and generated article  
    - Inputs: AI Article Writer 2 output + HTTP Request3 data + email text  
    - Outputs: Self-assessment text and rating  
    - Edge Cases: API rate limits, prompt failures, incomplete input data

  - **OpenAI self-assesment2**  
    - Type: LangChain OpenAI node (GPT-4o-mini)  
    - Configuration:  
      - Same prompt as above  
      - Input includes PDF-extracted text + email text and generated article  
    - Inputs: AI Article Writer 1 output + PDF extracted text + email text  
    - Outputs: Self-assessment text and rating  
    - Edge Cases: Same as above

  - **OpenAI self assesment**  
    - Type: LangChain OpenAI node (GPT-4o-mini)  
    - Configuration:  
      - Same prompt as above  
      - Input includes email text only and generated article (no attachment path)  
    - Inputs: AI Article Writer 3 output + email text  
    - Outputs: Self-assessment text and rating  
    - Edge Cases: Same as above

---

#### 2.5 Reply Email Composition and Sending

- **Overview:**  
  Sends a reply email to the original sender containing the AI-generated article, self-assessment, original email text, and extracted attachment content if applicable.

- **Nodes Involved:**  
  - reply to sender (pdf)  
  - reply to sender (word)  
  - reply to sender (no attachment)

- **Node Details:**

  - **reply to sender (pdf)**  
    - Type: Gmail node (Send operation)  
    - Configuration:  
      - Replies to original email message ID  
      - To: original sender email  
      - Subject: "RE: [original subject]"  
      - Body:  
        - Edited article (from AI Article Writer 1)  
        - Self-assessment rating and text (from OpenAI self-assesment2)  
        - Original email text  
        - Extracted PDF text  
    - Inputs: OpenAI self-assesment2 output, AI Article Writer 1 output, Gmail Trigger data, PDF extracted text  
    - Edge Cases: Gmail API errors, message ID missing, large email body

  - **reply to sender (word)**  
    - Type: Gmail node (Send operation)  
    - Configuration:  
      - Similar to PDF reply but includes extracted Word text (from HTTP Request3)  
      - Uses AI Article Writer 2 and OpenAI self-assesment outputs  
    - Inputs: OpenAI self-assesment output, AI Article Writer 2 output, Gmail Trigger data, Word extracted text  
    - Edge Cases: Same as PDF reply

  - **reply to sender (no attachment)**  
    - Type: Gmail node (Send operation)  
    - Configuration:  
      - Replies with article generated from inline email text only (AI Article Writer 3)  
      - Includes self-assessment (OpenAI self assesment)  
      - Includes original email text only (no attachment content)  
    - Inputs: OpenAI self assesment output, AI Article Writer 3 output, Gmail Trigger data  
    - Edge Cases: Same as above

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                     |
|-------------------------------|----------------------------------|----------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Gmail Trigger                 | Gmail Trigger                    | Receive incoming emails and attachments| None                          | Code: delete all but pdf and word |                                                                                                |
| Code: delete all but pdf and word | Code (Function)                  | Filter attachments to keep only PDFs and Word files | Gmail Trigger                | has attachment?                |                                                                                                |
| has attachment?               | If                              | Check if any filtered attachments exist| Code: delete all but pdf and word | PDF or WORD?, AI Article Writer 3 |                                                                                                |
| PDF or WORD?                 | If                              | Determine if attachment is PDF or Word | has attachment?               | Extrahiere aus PDF1, Google Drive |                                                                                                |
| Extrahiere aus PDF1           | Extract from File (PDF)          | Extract text from PDF attachments      | PDF or WORD? (PDF branch)     | AI Article Writer 1            |                                                                                                |
| Google Drive                 | Google Drive                    | Upload Word attachment to Google Drive | PDF or WORD? (Word branch)    | HTTP Request2                 |                                                                                                |
| HTTP Request2                | HTTP Request                   | Copy uploaded Word file as Google Doc  | Google Drive                  | HTTP Request3                 |                                                                                                |
| HTTP Request3                | HTTP Request                   | Export Google Doc as plain text         | HTTP Request2                 | AI Article Writer 2           |                                                                                                |
| AI Article Writer 1          | LangChain Agent (Anthropic Claude) | Generate article from PDF text + email | Extrahiere aus PDF1           | OpenAI self-assesment2        |                                                                                                |
| AI Article Writer 2          | LangChain Agent (Anthropic Claude) | Generate article from Word text + email | HTTP Request3                 | OpenAI self-assesment         |                                                                                                |
| AI Article Writer 3          | LangChain Agent (Anthropic Claude) | Generate article from inline email text | has attachment? (False branch) | OpenAI self assesment         |                                                                                                |
| OpenAI self-assesment        | LangChain OpenAI (GPT-4o-mini)  | Self-assessment comparing input and output (Word path) | AI Article Writer 2           | reply to sender (word)        |                                                                                                |
| OpenAI self-assesment2       | LangChain OpenAI (GPT-4o-mini)  | Self-assessment comparing input and output (PDF path) | AI Article Writer 1           | reply to sender (pdf)         |                                                                                                |
| OpenAI self assesment        | LangChain OpenAI (GPT-4o-mini)  | Self-assessment comparing input and output (No attachment) | AI Article Writer 3           | reply to sender (no attachment) |                                                                                                |
| reply to sender (pdf)        | Gmail                          | Send reply email with article, self-assessment, original email, and PDF text | OpenAI self-assesment2        | None                         |                                                                                                |
| reply to sender (word)       | Gmail                          | Send reply email with article, self-assessment, original email, and Word text | OpenAI self-assesment         | None                         |                                                                                                |
| reply to sender (no attachment) | Gmail                          | Send reply email with article, self-assessment, and original email (no attachment) | OpenAI self assesment         | None                         |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Credentials: Connect Gmail OAuth2 account  
   - Parameters:  
     - Poll every 1 minute  
     - Filter sender: `*@somedia.ch`  
     - Enable "Download Attachments" with prefix `attachment_`  
     - Output: Full email data including attachments

2. **Add Code Node to Filter Attachments**  
   - Type: Function (Code) node  
   - Paste JavaScript code to keep only PDF and Word attachments by MIME type:  
     - `application/pdf`  
     - `application/msword`  
     - `application/vnd.openxmlformats-officedocument.wordprocessingml.document`  
   - Connect Gmail Trigger output to this node

3. **Add If Node to Check for Attachments**  
   - Type: If node  
   - Condition: Check if number of binary attachments > 0  
   - Connect Code node output to this node

4. **Add If Node to Determine Attachment Type**  
   - Type: If node  
   - Condition: Check if attachment MIME type contains "pdf" (case-insensitive)  
   - Connect True branch of has attachment? node to this node

5. **PDF Path: Extract Text from PDF**  
   - Add Extract from File node  
   - Operation: PDF  
   - Connect True branch of PDF or WORD? node to this node

6. **Word Path: Upload to Google Drive**  
   - Add Google Drive node  
   - Operation: Upload file  
   - Folder: Root or desired folder  
   - File name: Use attachment filename dynamically  
   - Connect False branch of PDF or WORD? node to this node

7. **Word Path: Copy File as Google Doc**  
   - Add HTTP Request node  
   - Method: POST  
   - URL: `https://www.googleapis.com/drive/v3/files/{{$json["id"]}}/copy`  
   - Body: JSON `{ "mimeType": "application/vnd.google-apps.document" }`  
   - Headers: Authorization Bearer token from Google Drive OAuth2 credentials  
   - Connect Google Drive node output to this node

8. **Word Path: Export Google Doc as Plain Text**  
   - Add HTTP Request node  
   - Method: GET  
   - URL: `https://www.googleapis.com/drive/v3/files/{{$json["id"]}}/export?mimeType=text/plain`  
   - Headers: Authorization Bearer token  
   - Connect HTTP Request (copy) node output to this node

9. **AI Article Writer Nodes (3 total)**  
   - Add three LangChain Agent nodes (Anthropic Claude model) for each path:  
     - PDF path: Input PDF extracted text + email text  
     - Word path: Input Word extracted text + email text  
     - No attachment path: Input email text only  
   - Configure editorial prompt with instructions for Swiss German spelling, neutral style, inverted pyramid, no invented info, structured output including data extraction and word count  
   - Connect:  
     - PDF extracted text node to AI Article Writer 1  
     - Word extracted text node to AI Article Writer 2  
     - No attachment branch from has attachment? node to AI Article Writer 3

10. **AI Self-Assessment Nodes (3 total)**  
    - Add three LangChain OpenAI nodes (GPT-4o-mini) for self-assessment per path  
    - Prompt: Compare input text and generated article, rate 1-5, provide justification  
    - Inputs:  
      - PDF path: PDF extracted text + email text + AI Article Writer 1 output  
      - Word path: Word extracted text + email text + AI Article Writer 2 output  
      - No attachment path: email text + AI Article Writer 3 output  
    - Connect each AI Article Writer node output to corresponding self-assessment node

11. **Reply Email Nodes (3 total)**  
    - Add three Gmail nodes to send replies per path  
    - Operation: Reply  
    - To: original sender email from Gmail Trigger  
    - Subject: "RE: [original email subject]"  
    - Body templates:  
      - PDF: Include edited article, self-assessment, original email text, extracted PDF text  
      - Word: Include edited article, self-assessment, original email text, extracted Word text  
      - No attachment: Include edited article, self-assessment, original email text only  
    - Connect each self-assessment node output to corresponding reply node

12. **Connect all nodes according to logical flow:**  
    - Gmail Trigger → Code filter → has attachment?  
    - has attachment? True → PDF or WORD?  
    - PDF or WORD? True → PDF extraction → AI Article Writer 1 → OpenAI self-assesment2 → reply to sender (pdf)  
    - PDF or WORD? False → Google Drive upload → HTTP Request copy → HTTP Request export → AI Article Writer 2 → OpenAI self-assesment → reply to sender (word)  
    - has attachment? False → AI Article Writer 3 → OpenAI self assesment → reply to sender (no attachment)

13. **Credentials Setup:**  
    - Gmail OAuth2 credentials for Gmail Trigger and reply nodes  
    - Google Drive OAuth2 credentials for Google Drive and HTTP Request nodes  
    - Anthropic API credentials for LangChain Agent nodes  
    - OpenAI API credentials for LangChain OpenAI nodes

14. **Test the workflow with sample emails containing:**  
    - No attachments (inline text only)  
    - PDF attachments  
    - Word attachments

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow assists editors/journalists by automating press release transformation into draft articles.          | Workflow description                                                                                |
| Editorial instructions enforce Swiss German spelling, neutral tone, inverted pyramid structure, gender-neutral wording. | Editorial style guide embedded in AI prompts                                                      |
| AI self-assessment provides a feedback loop to ensure output quality and accuracy.                             | Self-assessment prompt details                                                                     |
| Gmail Trigger filters emails from `*@somedia.ch` domain only.                                                  | Gmail Trigger node configuration                                                                   |
| Google Drive is used as an intermediary to convert Word documents to plain text via Google Docs API.          | Google Drive and HTTP Request nodes                                                                |
| Anthropic Claude model used for article generation; OpenAI GPT-4o-mini used for self-assessment.               | Node credentials and model choices                                                                 |
| For more info on n8n Gmail integration: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ | Official n8n documentation                                                                          |
| For Google Drive API details: https://developers.google.com/drive/api/v3/reference/files/copy and export       | Google Drive API documentation                                                                     |
| For Anthropic Claude model info: https://www.anthropic.com/                                                     | Anthropic official site                                                                             |
| For OpenAI GPT-4o-mini model info: https://platform.openai.com/docs/models/gpt-4o-mini                         | OpenAI official documentation                                                                      |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and modify the workflow, anticipate potential errors, and integrate it effectively into editorial automation pipelines.