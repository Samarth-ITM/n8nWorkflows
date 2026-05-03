Summarize Google Drive Documents with Mistral AI and Send via Gmail

https://n8nworkflows.xyz/workflows/summarize-google-drive-documents-with-mistral-ai-and-send-via-gmail-3109


# Summarize Google Drive Documents with Mistral AI and Send via Gmail

### 1. Workflow Overview

This workflow automates the process of summarizing documents stored in Google Drive using Mistral AI and sending the summarized content via Gmail. It is designed for professionals who require quick, concise insights from lengthy documents without manual reading. The workflow supports PDF and DOCX files, processes them through an AI summarization chain, and delivers a styled email summary with a timestamp in the Lagos timezone.

**Logical Blocks:**

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 File Retrieval:** Download the specified document from Google Drive.
- **1.3 AI Summarization:** Process the document content using Mistral AI via a summarization chain.
- **1.4 Email Delivery:** Send the formatted summary to a Gmail inbox with custom styling and timestamp.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Initiates the workflow manually, enabling easy testing and controlled execution.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Triggers the next node (Google Drive file download).  
    - Version-specific: None  
    - Edge Cases: None, as it requires manual activation.  
    - Sub-workflow: None

#### 1.2 File Retrieval

- **Overview:**  
  Downloads a specified file from Google Drive, supporting PDF and DOCX formats, to provide the content for summarization.

- **Nodes Involved:**  
  - Download uploaded File from Google Drive

- **Node Details:**

  - **Download uploaded File from Google Drive**  
    - Type: Google Drive Node (Operation: download)  
    - Role: Fetches the file binary data from Google Drive using a file ID.  
    - Configuration:  
      - `fileId`: Set to a specific file ("1d0njBA2ZM0zYyJOEbUeFwQmHSYIO7IM2") representing the document to summarize.  
      - Operation: "download" to retrieve the file content.  
      - Options: Default, no additional options set.  
    - Credentials: Uses OAuth2 credentials for Google Drive access.  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: Provides binary data of the file to the summarization chain.  
    - Version-specific: Uses version 3 of Google Drive node.  
    - Edge Cases:  
      - File not found or deleted → node failure or empty output.  
      - Permission errors due to invalid or expired OAuth2 credentials.  
      - Unsupported file formats may cause downstream errors.  
    - Sub-workflow: None

#### 1.3 AI Summarization

- **Overview:**  
  Processes the downloaded file by chunking its content and generating a concise summary using Mistral AI.

- **Nodes Involved:**  
  - Mistral Cloud Chat Model  
  - Summarization Chain to summarize a file

- **Node Details:**

  - **Mistral Cloud Chat Model**  
    - Type: Language Model Node (Mistral Cloud)  
    - Role: Provides AI language model capabilities to generate summaries.  
    - Configuration: Default options; no custom prompt or parameters specified here.  
    - Credentials: Uses Mistral Cloud API credentials.  
    - Inputs: Receives input from the summarization chain node as an AI language model provider.  
    - Outputs: Returns AI-generated text responses to the summarization chain.  
    - Version-specific: Version 1 of the Mistral Cloud node.  
    - Edge Cases:  
      - API authentication failure.  
      - Rate limiting or quota exceeded.  
      - Network timeouts or latency.  
    - Sub-workflow: None

  - **Summarization Chain to summarize a file**  
    - Type: LangChain Summarization Chain Node  
    - Role: Splits the file content into chunks and processes each chunk through the AI model to produce a summary.  
    - Configuration:  
      - `chunkSize`: 800 characters per chunk.  
      - `chunkOverlap`: 0 (no overlap between chunks).  
      - `operationMode`: "nodeInputBinary" indicating it processes binary file input.  
      - `binaryDataKey`: "data" (the key where the binary file is stored).  
    - Inputs: Receives binary file data from Google Drive node and AI model from Mistral Cloud Chat Model node.  
    - Outputs: Produces a summarized text output under the JSON path `$json['response']['text']`.  
    - Version-specific: Version 2 of the summarization chain node.  
    - Edge Cases:  
      - Failure to parse or chunk the binary file (corrupted or unsupported format).  
      - AI model errors or empty responses.  
      - Large files exceeding chunk size limits.  
    - Sub-workflow: None

#### 1.4 Email Delivery

- **Overview:**  
  Sends the AI-generated summary as a styled HTML email to a specified Gmail address, including a timestamp formatted for the Lagos timezone.

- **Nodes Involved:**  
  - Send Summarized text to Gmail

- **Node Details:**

  - **Send Summarized text to Gmail**  
    - Type: Gmail Node (Send Email)  
    - Role: Delivers the formatted summary email to the user’s inbox.  
    - Configuration:  
      - `sendTo`: "swot.ai25@gmail.com" (recipient email address).  
      - `subject`: "Here is Your Summarized Response".  
      - `message`: HTML formatted content including:  
        - Header with icon and title.  
        - Summary text injected dynamically from the summarization chain output, replacing newlines with `<br>`.  
        - Current date/time formatted in 'en-GB' locale and 'Africa/Lagos' timezone.  
      - `options`:  
        - `senderName`: "Swot.AI" (custom sender display name).  
        - `appendAttribution`: false (no automatic attribution appended).  
    - Credentials: Uses Gmail OAuth2 credentials.  
    - Inputs: Receives summarized text from the summarization chain node.  
    - Outputs: None (final node).  
    - Version-specific: Version 2.1 of Gmail node.  
    - Edge Cases:  
      - Authentication failure or expired Gmail OAuth2 token.  
      - Email sending limits or quota exceeded.  
      - HTML rendering issues if summary text contains unexpected characters.  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                 | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                      |
|----------------------------------|----------------------------------|--------------------------------|-------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger                   | Starts workflow manually       | None                          | Download uploaded File from Google Drive |                                                                                                 |
| Download uploaded File from Google Drive | Google Drive Node (download)     | Downloads file from Google Drive | When clicking ‘Test workflow’ | Summarization Chain to summarize a file |                                                                                                 |
| Summarization Chain to summarize a file | LangChain Summarization Chain    | Processes file and summarizes  | Download uploaded File from Google Drive, Mistral Cloud Chat Model (AI model) | Send Summarized text to Gmail          |                                                                                                 |
| Mistral Cloud Chat Model          | AI Language Model (Mistral Cloud) | Provides AI model for summarization | Summarization Chain to summarize a file (as AI model) | Summarization Chain to summarize a file |                                                                                                 |
| Send Summarized text to Gmail     | Gmail Node                      | Sends formatted summary email  | Summarization Chain to summarize a file | None                           |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - No special configuration needed.

2. **Add Google Drive Node to Download File**  
   - Add a **Google Drive** node named "Download uploaded File from Google Drive".  
   - Set operation to **download**.  
   - Configure `fileId` with the target file ID (e.g., "1d0njBA2ZM0zYyJOEbUeFwQmHSYIO7IM2").  
   - Connect credentials: Use a valid **Google Drive OAuth2** credential with read access.  
   - Connect output of Manual Trigger node to this node.

3. **Add Mistral Cloud Chat Model Node**  
   - Add a **Mistral Cloud Chat Model** node named "Mistral Cloud Chat Model".  
   - Use default options.  
   - Connect credentials: Use **Mistral Cloud API** credentials.  
   - No direct input connection; it will be linked as AI model provider in the summarization chain.

4. **Add Summarization Chain Node**  
   - Add a **LangChain Summarization Chain** node named "Summarization Chain to summarize a file".  
   - Set `operationMode` to **nodeInputBinary**.  
   - Set `binaryDataKey` to `"data"` (matches the binary key from Google Drive node).  
   - Set `chunkSize` to 800 characters and `chunkOverlap` to 0.  
   - Connect input from **Download uploaded File from Google Drive** node.  
   - In the AI model setting, select the **Mistral Cloud Chat Model** node as the language model provider.

5. **Add Gmail Node to Send Email**  
   - Add a **Gmail** node named "Send Summarized text to Gmail".  
   - Set `sendTo` to the recipient email (e.g., "swot.ai25@gmail.com").  
   - Set `subject` to "Here is Your Summarized Response".  
   - Set `message` to the following HTML template (use expression editor to insert dynamic content):  
     ```html
     <h1 style="color: #4CAF50;">📌 Quick Summary of Your Document! ✨</h1>
     <p>
     <h2>📝 Summary:</h2>
     <p>
     {{ $json["response"]["text"].replace("\n", "<br>") }}
     <p>
     <h3>📅 Date Processed: </h3>
     {{ new Date().toLocaleString('en-GB', { timeZone: 'Africa/Lagos' }) }}
     ```
   - Set options:  
     - `senderName`: "Swot.AI"  
     - `appendAttribution`: false  
   - Connect credentials: Use valid **Gmail OAuth2** credentials.  
   - Connect input from **Summarization Chain to summarize a file** node.

6. **Connect Nodes in Sequence:**  
   - Manual Trigger → Google Drive Download → Summarization Chain → Gmail Send  
   - Mistral Cloud Chat Model node is linked as AI model provider to the Summarization Chain node.

7. **Test Workflow:**  
   - Trigger manually.  
   - Confirm file downloads, summarization completes, and email is sent with formatted content.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow supports dynamic timezone formatting; change `'Africa/Lagos'` in the Gmail node to your preferred timezone. | Timezone customization tip.                                                                    |
| The email output uses HTML styling with icons and headers for clarity and professional appearance. | Enhances readability of summary emails.                                                        |
| The workflow can be extended to send summaries via WhatsApp or Slack by adding respective nodes after summarization. | Future expansion possibilities.                                                                |
| For alternative AI models, swap the Mistral Cloud Chat Model node with an OpenAI node and adjust summarization chain accordingly. | Customization tip for different AI providers.                                                  |
| Error handling is recommended for production use, such as checking file existence and API credential validity. | Suggested improvement for robustness.                                                         |
| Workflow credits: Developed for productivity in legal, content creation, and business analysis contexts. | Use case context.                                                                              |

---

This documentation provides a complete, structured reference for understanding, reproducing, and extending the "Summarize Google Drive Documents with Mistral AI and Send via Gmail" workflow.