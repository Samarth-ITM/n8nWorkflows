Effortless Email Management with AI-Powered Summarization & Review

https://n8nworkflows.xyz/workflows/effortless-email-management-with-ai-powered-summarization---review-2862


# Effortless Email Management with AI-Powered Summarization & Review

### 1. Workflow Overview

This workflow, titled **"Effortless Email Management with AI-Powered Summarization & Review"**, automates the processing of incoming emails by summarizing their content, generating professional replies using AI enhanced with retrieval-augmented generation (RAG), and managing human approval before sending responses. It is designed for business environments where email volume is high and professional, concise communication is critical.

The workflow is logically divided into two main sections:

- **1.1 Email Handling and Summarization**:  
  Captures incoming emails, converts HTML content to plain text, and generates concise AI summaries to distill the core message.

- **1.2 Response Generation and Approval**:  
  Crafts professional email replies based on summaries and business knowledge retrieved from a vector database, sends drafts for human review via Gmail, classifies feedback as approval or decline, and either sends the final email or loops for edits.

Additionally, there is a **Setup and Maintenance** section supporting vector database management and document ingestion for knowledge base updates.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Handling and Summarization

**Overview:**  
This block listens for new emails, converts their content to Markdown for better AI comprehension, and summarizes the email content into a concise text of up to 100 words.

**Nodes Involved:**  
- Email Trigger (IMAP)  
- Markdown  
- Email Summarization Chain

**Node Details:**

- **Email Trigger (IMAP)**  
  - *Type:* Email Read (IMAP)  
  - *Role:* Listens for new incoming emails on a specified IMAP inbox.  
  - *Configuration:* Uses IMAP credentials for `info@n3witalia.com`. No special options set.  
  - *Input:* None (trigger node).  
  - *Output:* Emits email data including HTML content and metadata.  
  - *Edge Cases:* Authentication failures, connection timeouts, empty or malformed emails.

- **Markdown**  
  - *Type:* Markdown converter  
  - *Role:* Converts the incoming email’s HTML content (`textHtml` field) into plain Markdown text to facilitate AI processing.  
  - *Configuration:* Input is bound to `{{$json.textHtml}}`.  
  - *Input:* Email HTML content from Email Trigger.  
  - *Output:* Plain text version of email content.  
  - *Edge Cases:* Emails without HTML content may produce empty output.

- **Email Summarization Chain**  
  - *Type:* LangChain Summarization Chain  
  - *Role:* Uses AI to generate a concise summary (max 100 words) of the email content in Markdown.  
  - *Configuration:* Summarization prompt instructs to write a concise summary without counting words. Operates on binary data keyed by `data`.  
  - *Input:* Markdown text from Markdown node.  
  - *Output:* Summarized text of the email.  
  - *Edge Cases:* AI model timeouts, incomplete input data, prompt failures.

---

#### 2.2 Response Generation and Approval

**Overview:**  
Generates a professional reply based on the summary and business knowledge, sends the draft for human approval via Gmail, classifies the feedback, and either sends the final email or requests edits.

**Nodes Involved:**  
- Write email  
- Edit Fields  
- Gmail  
- Text Classifier  
- Send Email  
- Email Reviewer

**Node Details:**

- **Write email**  
  - *Type:* LangChain Agent (AI Language Model)  
  - *Role:* Generates a professional email reply based on the summarized content and retrieved business knowledge.  
  - *Configuration:* System message instructs to answer professionally, concisely, max 100 words, only body text. Input text is the summarized email response.  
  - *Input:* Summarized email text from Email Summarization Chain.  
  - *Output:* Draft reply email text.  
  - *Edge Cases:* AI generation errors, exceeding word limits, incomplete inputs.

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Prepares the draft email text for sending by assigning it to a field named `email`.  
  - *Configuration:* Sets `email` to the output of Write email node.  
  - *Input:* Draft email text from Write email.  
  - *Output:* JSON with `email` field for downstream use.  
  - *Edge Cases:* Missing or empty draft text.

- **Gmail**  
  - *Type:* Gmail node  
  - *Role:* Sends the draft email to a designated Gmail address for human review and waits for free-text response feedback.  
  - *Configuration:*  
    - Sends to `info@n3w.it` (human reviewer).  
    - Subject prepended with `[Approval Required]` and original email subject.  
    - Message body includes original email HTML and AI response.  
    - Uses "sendAndWait" operation to pause workflow until reviewer replies.  
  - *Input:* Draft email from Edit Fields and original email data.  
  - *Output:* Human feedback text.  
  - *Edge Cases:* Gmail API rate limits, OAuth token expiration, no response timeout.  
  - *Important:* Must use Gmail due to "send and wait" feature limitation.

- **Text Classifier**  
  - *Type:* LangChain Text Classifier  
  - *Role:* Classifies human feedback as either "Approved" or "Declined" to guide next steps.  
  - *Configuration:*  
    - System prompt instructs classification into two categories with JSON output only.  
    - Input text is the human feedback from Gmail.  
    - Categories:  
      - Approved: Explicit or implicit approval (e.g., "Ok", "Approvato", "Invia").  
      - Declined: Requests for modifications or edits.  
  - *Input:* Feedback text from Gmail.  
  - *Output:* Classification result.  
  - *Edge Cases:* Ambiguous feedback, misclassification risk.

- **Send Email**  
  - *Type:* Email Send (SMTP)  
  - *Role:* Sends the finalized approved email back to the original sender.  
  - *Configuration:*  
    - Sends to original email sender (`from` field).  
    - From original recipient (`to` field).  
    - Subject prefixed with "Re: " and original subject.  
    - Email body is the approved draft email in HTML.  
    - Uses SMTP credentials for `info@n3witalia.com`.  
  - *Input:* Approved email text from Text Classifier (on approval path).  
  - *Output:* Confirmation of sent email.  
  - *Edge Cases:* SMTP authentication failure, invalid email addresses.

- **Email Reviewer**  
  - *Type:* LangChain Agent (AI Language Model)  
  - *Role:* If feedback is "Declined", rewrites the draft email incorporating human suggestions, producing a revised email in HTML format.  
  - *Configuration:*  
    - System message instructs expert review and restructuring of the email body, allowing limited HTML tags (`<br>`, `<b>`, `<i>`, `<p>`), max 100 words.  
    - Input includes original draft email and human feedback.  
  - *Input:* Declined feedback and draft email text.  
  - *Output:* Revised email draft.  
  - *Edge Cases:* AI rewriting errors, HTML formatting issues.

---

#### 2.3 Knowledge Base Setup and Document Vectorization (Supporting Block)

**Overview:**  
Manages the Qdrant vector database collection and ingests documents from Google Drive, embedding them for retrieval during email response generation.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Create collection  
- Refresh collection  
- Get folder  
- Download Files  
- Default Data Loader  
- Token Splitter  
- Embeddings OpenAI1  
- Qdrant Vector Store1  
- Embeddings OpenAI  
- Qdrant Vector Store

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the setup process manually.  
  - *Input:* None.  
  - *Output:* Triggers collection creation and refresh.

- **Create collection**  
  - *Type:* HTTP Request  
  - *Role:* Creates a new collection in Qdrant vector database.  
  - *Configuration:*  
    - POST to `https://QDRANTURL/collections/COLLECTION` with empty filter.  
    - Requires HTTP header authentication.  
  - *Edge Cases:* API errors, authentication failure.

- **Refresh collection**  
  - *Type:* HTTP Request  
  - *Role:* Deletes all points in the Qdrant collection to refresh data.  
  - *Configuration:*  
    - POST to `https://QDRANTURL/collections/COLLECTION/points/delete` with empty filter.  
  - *Edge Cases:* API errors, partial deletion.

- **Get folder**  
  - *Type:* Google Drive node  
  - *Role:* Retrieves files from a specified Google Drive folder (`test-whatsapp`).  
  - *Configuration:* Uses Google Drive OAuth2 credentials.  
  - *Edge Cases:* Permission issues, empty folder.

- **Download Files**  
  - *Type:* Google Drive node  
  - *Role:* Downloads each file from the folder, converting Google Docs to plain text.  
  - *Configuration:* Conversion enabled for Google Docs to `text/plain`.  
  - *Edge Cases:* Unsupported file types, download failures.

- **Default Data Loader**  
  - *Type:* LangChain Document Loader  
  - *Role:* Loads downloaded files as binary data for embedding.  
  - *Edge Cases:* Corrupt files, unsupported formats.

- **Token Splitter**  
  - *Type:* LangChain Text Splitter (Token-based)  
  - *Role:* Splits documents into chunks of 300 tokens with 30 token overlap for embedding.  
  - *Edge Cases:* Incorrect chunk sizes, tokenization errors.

- **Embeddings OpenAI1**  
  - *Type:* LangChain Embeddings (OpenAI)  
  - *Role:* Generates vector embeddings for document chunks.  
  - *Edge Cases:* API rate limits, embedding failures.

- **Qdrant Vector Store1**  
  - *Type:* LangChain Vector Store (Qdrant)  
  - *Role:* Inserts document embeddings into Qdrant collection.  
  - *Edge Cases:* API errors, data insertion failures.

- **Embeddings OpenAI**  
  - *Type:* LangChain Embeddings (OpenAI)  
  - *Role:* Generates embeddings for query vectors during retrieval.  
  - *Edge Cases:* Same as above.

- **Qdrant Vector Store**  
  - *Type:* LangChain Vector Store (Qdrant)  
  - *Role:* Retrieves relevant business knowledge documents for RAG during email reply generation.  
  - *Edge Cases:* Retrieval failures, empty results.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                                  | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                  |
|-------------------------|--------------------------------------|-------------------------------------------------|-------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Email Trigger (IMAP)     | Email Read (IMAP)                    | Listens for incoming emails                      | None                          | Markdown                    |                                                                                                              |
| Markdown                | Markdown converter                   | Converts email HTML to plain text                | Email Trigger (IMAP)           | Email Summarization Chain    | Convert email to Markdown format for better understanding of LLM models                                      |
| Email Summarization Chain| LangChain Summarization Chain        | Summarizes email content concisely               | Markdown                      | Write email                 | Chain that summarizes the received email                                                                     |
| Write email             | LangChain Agent (AI Language Model)  | Generates professional reply based on summary    | Email Summarization Chain      | Edit Fields                 | Agent that retrieves business information from a vector database and processes the response                  |
| Edit Fields             | Set node                            | Prepares draft email text for sending            | Write email                   | Gmail                       |                                                                                                              |
| Gmail                   | Gmail node                         | Sends draft email for human review and waits     | Edit Fields                   | Text Classifier             | IMPORTANT: For the "Send Draft" node, you need to send the draft email to a Gmail address because it is the only one that allows the "Send and wait for response" function. |
| Text Classifier         | LangChain Text Classifier            | Classifies human feedback as Approved or Declined| Gmail                         | Send Email, Email Reviewer  | Based on the suggestion received, the text classifier can understand whether the feedback received approves the generated email or not. |
| Send Email              | Email Send (SMTP)                   | Sends final approved email to original sender    | Text Classifier (Approved)    | None                       |                                                                                                              |
| Email Reviewer          | LangChain Agent (AI Language Model)  | Rewrites email based on human feedback            | Text Classifier (Declined)    | Edit Fields                 | The Email Reviewer agent, taking inspiration from human feedback, rewrites the email                         |
| When clicking ‘Test workflow’ | Manual Trigger                  | Starts setup process                              | None                          | Create collection, Refresh collection |                                                                                                              |
| Create collection       | HTTP Request                       | Creates Qdrant collection                         | When clicking ‘Test workflow’  | Refresh collection          | STEP 1: Create Qdrant Collection - Change QDRANTURL and COLLECTION                                          |
| Refresh collection      | HTTP Request                       | Deletes all points in Qdrant collection           | Create collection             | Get folder                  | STEP 1: Create Qdrant Collection - Change QDRANTURL and COLLECTION                                          |
| Get folder              | Google Drive                       | Retrieves files from Google Drive folder          | Refresh collection            | Download Files              | STEP 2: Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION              |
| Download Files          | Google Drive                       | Downloads and converts files to plain text        | Get folder                   | Qdrant Vector Store1        | STEP 2: Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION              |
| Default Data Loader     | LangChain Document Loader           | Loads downloaded files as binary data              | Token Splitter                | Qdrant Vector Store1        |                                                                                                              |
| Token Splitter          | LangChain Text Splitter (Token)     | Splits documents into token chunks                 | Default Data Loader           | Default Data Loader         |                                                                                                              |
| Embeddings OpenAI1      | LangChain Embeddings (OpenAI)       | Generates embeddings for document chunks           | Token Splitter                | Qdrant Vector Store1        |                                                                                                              |
| Qdrant Vector Store1    | LangChain Vector Store (Qdrant)     | Inserts document embeddings into Qdrant collection| Embeddings OpenAI1            | None                       |                                                                                                              |
| Embeddings OpenAI       | LangChain Embeddings (OpenAI)       | Generates embeddings for queries                    | None                         | Qdrant Vector Store         |                                                                                                              |
| Qdrant Vector Store     | LangChain Vector Store (Qdrant)     | Retrieves relevant documents for RAG                | Embeddings OpenAI             | Write email, Email Reviewer |                                                                                                              |
| DeepSeek Chat Model     | LangChain Chat Model (DeepSeek)     | Alternative AI model for summarization              | None                         | Email Summarization Chain   |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Email Trigger (IMAP) Node**  
   - Type: Email Read (IMAP)  
   - Credentials: Configure IMAP credentials for your email inbox (e.g., `info@n3witalia.com`).  
   - Parameters: Default options to listen for new emails.

2. **Add Markdown Node**  
   - Type: Markdown converter  
   - Parameters: Set input HTML as `={{ $json.textHtml }}` to convert incoming email HTML to plain text.

3. **Add Email Summarization Chain Node**  
   - Type: LangChain Summarization Chain  
   - Parameters:  
     - Summarization prompt: "Write a concise summary of the following in max 100 words: \"{{ $json.data }}\" Do not enter the total number of words used."  
     - Operation mode: Node input binary with key `data`.  
   - Connect Markdown output to this node.

4. **Add Write email Node**  
   - Type: LangChain Agent (AI Language Model)  
   - Parameters:  
     - Text: `Write the text to reply to the following email:\n\n{{ $json.response.text }}`  
     - System message: "You are an expert at answering emails. You need to answer them professionally based on the information you have. This is a business email. Be concise and never exceed 100 words. Only the body of the email, not create the subject."  
     - Prompt type: Define  
     - Enable output parser.  
   - Connect Email Summarization Chain output here.

5. **Add Edit Fields Node**  
   - Type: Set node  
   - Parameters: Assign `email` field to `={{ $json.output }}` (the draft email text).  
   - Connect Write email output here.

6. **Add Gmail Node**  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials for the review email account (e.g., `info@n3w.it`).  
   - Parameters:  
     - Send To: `info@n3w.it` (human reviewer).  
     - Subject: `=[Approval Required] {{ $('Email Trigger (IMAP)').item.json.subject }}`  
     - Message: Include original email HTML and AI response:  
       ```
       <h3>MESSAGE</h3>
       {{ $('Email Trigger (IMAP)').item.json.textHtml }}

       <h3>AI RESPONSE</h3>
       {{ $json.email }}
       ```  
     - Operation: Send and wait for response (free-text).  
   - Connect Edit Fields output here.

7. **Add Text Classifier Node**  
   - Type: LangChain Text Classifier  
   - Parameters:  
     - System prompt: Classify text into "Approved" or "Declined" categories with JSON output only.  
     - Input text: `={{ $json.data.text }}` (human feedback from Gmail).  
     - Categories:  
       - Approved: Explicit or implicit approval (e.g., "Ok", "Approvato", "Invia").  
       - Declined: Requests for modifications or edits.  
   - Connect Gmail output here.

8. **Add Send Email Node**  
   - Type: Email Send (SMTP)  
   - Credentials: Configure SMTP credentials for sending email (e.g., `info@n3witalia.com`).  
   - Parameters:  
     - To Email: `={{ $('Email Trigger (IMAP)').item.json.from }}` (original sender).  
     - From Email: `={{ $('Email Trigger (IMAP)').item.json.to }}` (your email).  
     - Subject: `=Re: {{ $('Email Trigger (IMAP)').item.json.subject }}`  
     - HTML: `={{ $('Edit Fields').item.json.email }}` (approved email body).  
   - Connect Text Classifier output on "Approved" path here.

9. **Add Email Reviewer Node**  
   - Type: LangChain Agent (AI Language Model)  
   - Parameters:  
     - Text:  
       ```
       Review at the following email:
       {{ $('Edit Fields').item.json.email }}

       Feedback from human:
       {{ $json.data.text }}
       ```  
     - System message: "If you are an expert in reviewing emails before sending them. You need to review and structure them in such a way that you can send them. It must be in HTML format and you can insert (if you think it is appropriate) only HTML characters such as <br>, <b>, <i>, <p> where necessary. Be concise and never exceed 100 words. Only the body of the email."  
     - Prompt type: Define  
     - Enable output parser.  
   - Connect Text Classifier output on "Declined" path here.

10. **Connect Email Reviewer output back to Edit Fields**  
    - This allows iterative editing based on human feedback.

---

**Optional: Knowledge Base Setup**

11. **Add Manual Trigger Node**  
    - To start setup manually.

12. **Add Create collection Node**  
    - HTTP Request POST to `https://QDRANTURL/collections/COLLECTION` with empty filter.  
    - Authenticate with HTTP header credentials.

13. **Add Refresh collection Node**  
    - HTTP Request POST to `https://QDRANTURL/collections/COLLECTION/points/delete` with empty filter.

14. **Add Get folder Node**  
    - Google Drive node to list files in folder `test-whatsapp`.

15. **Add Download Files Node**  
    - Download and convert Google Docs to plain text.

16. **Add Default Data Loader Node**  
    - Load downloaded files as binary data.

17. **Add Token Splitter Node**  
    - Split documents into 300 token chunks with 30 token overlap.

18. **Add Embeddings OpenAI1 Node**  
    - Generate embeddings for document chunks.

19. **Add Qdrant Vector Store1 Node**  
    - Insert embeddings into Qdrant collection.

20. **Add Embeddings OpenAI Node**  
    - Generate embeddings for queries.

21. **Add Qdrant Vector Store Node**  
    - Retrieve relevant documents for RAG during email reply generation.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| For the "Send Draft" node, Gmail is required because it uniquely supports the "Send and wait for response" feature. | Sticky Note6                                                                                     |
| The workflow supports integration with both Gmail and Outlook via appropriate trigger nodes.              | Sticky Note3                                                                                     |
| Change placeholders `QDRANTURL` and `COLLECTION` when setting up Qdrant collection and vector store nodes.| Sticky Note3, Sticky Note4                                                                       |
| The Email Reviewer agent rewrites emails incorporating human feedback using limited HTML formatting.      | Sticky Note8                                                                                    |
| The Text Classifier node helps automate decision-making on whether to send or revise emails based on feedback.| Sticky Note7                                                                                    |
| For detailed setup, ensure all credentials (IMAP, SMTP, OpenAI API, Gmail OAuth2, Qdrant API, Google Drive OAuth2) are configured properly in n8n.| Workflow Description and Node Credentials sections                                              |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "Effortless Email Management with AI-Powered Summarization & Review" workflow, enabling advanced users and AI agents to work effectively with it.