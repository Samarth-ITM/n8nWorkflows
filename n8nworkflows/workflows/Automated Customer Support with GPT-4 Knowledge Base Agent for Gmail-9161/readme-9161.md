Automated Customer Support with GPT-4 Knowledge Base Agent for Gmail

https://n8nworkflows.xyz/workflows/automated-customer-support-with-gpt-4-knowledge-base-agent-for-gmail-9161


# Automated Customer Support with GPT-4 Knowledge Base Agent for Gmail

### 1. Workflow Overview

This workflow automates customer support email handling for a Gmail account using GPT-4 and a Pinecone-powered knowledge base. It classifies incoming emails, responds to customer support inquiries by querying a knowledge base, and auto-updates the knowledge base when new documents are added to a designated Google Drive folder.

The workflow is logically divided into two main blocks:

- **1.1 Gmail Customer Support Handling:** Triggered by incoming Gmail messages, this block classifies emails, invokes an AI agent to generate replies using a knowledge base, labels the message as important, and sends the response.

- **1.2 Knowledge Base Auto-Update:** Triggered by new files in a specific Google Drive folder, this block downloads files, processes and splits their textual content, generates vector embeddings, and inserts them into a Pinecone vector store to extend the knowledge base.

---

### 2. Block-by-Block Analysis

#### 2.1 Gmail Customer Support Handling

**Overview:**  
This block listens for new Gmail emails, classifies whether they relate to customer support, and if so, uses an AI agent powered by GPT-4 to generate a reply based on a Pinecone knowledge base. It then labels the email as important and replies automatically.

**Nodes Involved:**  
- Gmail Trigger  
- Text Classifier  
- AI Agent  
- knowledgebase (Pinecone VectorStore)  
- Add label to message  
- Reply to a message  
- No Operation, do nothing (fallback)  
- OpenAI Chat Model  
- Sticky Notes (contextual annotations)

**Node Details:**

- **Gmail Trigger**  
  - Type: Trigger node for new Gmail emails  
  - Config: Polls every minute; no filters applied (captures all incoming emails)  
  - Inputs: External trigger (new email)  
  - Outputs: Provides email JSON, including text and message ID  
  - Credentials: Gmail OAuth2  
  - Failure cases: OAuth token expiration, Gmail API limits  
  - Notes: Entry point for customer support emails.

- **Text Classifier**  
  - Type: Langchain Text Classifier node  
  - Config: Classifies input email text into two categories: "Customer Support" or "Other"  
  - Input Expression: Uses email text from Gmail Trigger `{{$json.text}}`  
  - Outputs: Routes classified emails to AI Agent if "Customer Support" or to No Operation if "Other"  
  - Version: 1.1  
  - Failure cases: Classification errors if input text missing or malformed.

- **AI Agent**  
  - Type: Langchain Agent node  
  - Config: Uses incoming email text as prompt; accesses the Pinecone knowledge base as a tool; system message instructs the agent to only output email body, sign off as "Yasser @ Blueproof", and avoid hallucination  
  - Input Expression: `={{ $('Gmail Trigger').item.json.text }}`  
  - Outputs: AI-generated reply text  
  - Version: 2.2  
  - Credentials: OpenAI API (GPT-4o-mini model)  
  - Failure cases: API errors, knowledge base retrieval failures, prompt errors  
  - Notes: Central AI logic to generate customer support replies.

- **knowledgebase (Pinecone VectorStore)**  
  - Type: Langchain Pinecone Vector Store node  
  - Config: Retrieval mode as a tool, namespace "FAQ", index "n8n"  
  - Credentials: Pinecone API  
  - Inputs: Used as an AI tool by the AI Agent node  
  - Failure cases: Pinecone API errors, connectivity issues.

- **Add label to message**  
  - Type: Gmail node  
  - Config: Adds "IMPORTANT" label to the email message ID from Gmail Trigger  
  - Credentials: Gmail OAuth2  
  - Inputs: AI Agent output connection (triggered only if AI Agent runs)  
  - Outputs: Connected to Reply to a message  
  - Failure cases: Gmail API errors, label ID invalid.

- **Reply to a message**  
  - Type: Gmail node  
  - Config: Sends a text reply using AI Agent output as message body, replies to original message ID  
  - Credentials: Gmail OAuth2  
  - Inputs: From Add label to message  
  - Failure cases: Sending failures, Gmail API rate limits.

- **No Operation, do nothing**  
  - Type: NoOp node  
  - Config: Executes when email classified as "Other" (non-customer support)  
  - Inputs: From Text Classifier (if category "Other")  
  - Outputs: None  
  - Failure cases: None (safe fallback).

- **OpenAI Chat Model**  
  - Type: Langchain Chat Model (GPT-4o-mini)  
  - Config: Used internally by Text Classifier for classification  
  - Credentials: OpenAI API  
  - Inputs: Connected from Text Classifier for language model calls  
  - Failure cases: API errors, model unavailability.

- **Sticky Notes**  
  - Provide contextual explanations and documentation for classification, AI agent, and overall Gmail customer support logic.

---

#### 2.2 Knowledge Base Auto-Update

**Overview:**  
This block monitors a specific Google Drive folder for new files, downloads them, processes and splits their textual content, embeds the text using OpenAI embeddings, and inserts vector data into Pinecone to update the knowledge base automatically.

**Nodes Involved:**  
- Google Drive Trigger  
- Download file  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI1  
- Pinecone Vector Store  
- Sticky Note2 (annotation)

**Node Details:**

- **Google Drive Trigger**  
  - Type: Trigger node for new files in Google Drive  
  - Config: Watches a specific folder ID, polling every minute  
  - Credentials: Google Drive OAuth2  
  - Inputs: External (new file created)  
  - Outputs: File metadata JSON including file ID  
  - Failure cases: OAuth token expiration, API limits.

- **Download file**  
  - Type: Google Drive node  
  - Config: Downloads file using file ID from trigger  
  - Credentials: Google Drive OAuth2  
  - Inputs: From Google Drive Trigger  
  - Outputs: Binary file content  
  - Failure cases: File not found, download errors.

- **Recursive Character Text Splitter**  
  - Type: Langchain text splitter  
  - Config: Splits downloaded document text into chunks for embedding  
  - Inputs: Extracted text from Default Data Loader  
  - Outputs: Text chunks for embedding  
  - Failure cases: Text splitting errors if input is malformed.

- **Default Data Loader**  
  - Type: Langchain document loader  
  - Config: Reads binary file data, converts to text for processing  
  - Inputs: Binary data from Download file  
  - Outputs: Text content for splitting  
  - Failure cases: Unsupported file types, decoding errors.

- **Embeddings OpenAI1**  
  - Type: Langchain OpenAI Embeddings node  
  - Config: Generates vector embeddings for text chunks  
  - Credentials: OpenAI API  
  - Inputs: From Recursive Character Text Splitter  
  - Outputs: Embeddings for Pinecone insertion  
  - Failure cases: API limits, embedding errors.

- **Pinecone Vector Store**  
  - Type: Langchain Pinecone Vector Store node (Insert mode)  
  - Config: Inserts embeddings into Pinecone index "n8n" under namespace "FAQ"  
  - Credentials: Pinecone API  
  - Inputs: Embeddings from Embeddings OpenAI1, documents from Default Data Loader  
  - Outputs: None  
  - Failure cases: API insertion errors, index limits.

- **Sticky Note2**  
  - Describes the auto-update logic for the knowledge base when adding files to Google Drive.

---

### 3. Summary Table

| Node Name                 | Node Type                                     | Functional Role                        | Input Node(s)          | Output Node(s)                 | Sticky Note                                                                                                       |
|---------------------------|-----------------------------------------------|-------------------------------------|-----------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger             | n8n-nodes-base.gmailTrigger                    | Entry point: detects new emails     | External               | Text Classifier                |                                                                                                                  |
| Text Classifier           | @n8n/n8n-nodes-langchain.textClassifier       | Classifies emails as support/other  | Gmail Trigger          | AI Agent, No Operation         | ## Classification: AI checks if the email is customer support or not.                                           |
| AI Agent                 | @n8n/n8n-nodes-langchain.agent                 | Generates AI reply using knowledgebase | Text Classifier       | Add label to message           | ## AI Agent: Searches your knowledge base and drafts a reply.                                                   |
| knowledgebase            | @n8n/n8n-nodes-langchain.vectorStorePinecone  | Provides knowledge base retrieval   | Embeddings OpenAI      | AI Agent (as tool)             |                                                                                                                  |
| Add label to message     | n8n-nodes-base.gmail                           | Labels email as "IMPORTANT"         | AI Agent               | Reply to a message             |                                                                                                                  |
| Reply to a message       | n8n-nodes-base.gmail                           | Sends AI-generated reply            | Add label to message    | None                          |                                                                                                                  |
| No Operation, do nothing | n8n-nodes-base.noOp                            | No action for non-customer support  | Text Classifier        | None                          |                                                                                                                  |
| OpenAI Chat Model        | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Supports Text Classifier             |                       | Text Classifier                |                                                                                                                  |
| Google Drive Trigger     | n8n-nodes-base.googleDriveTrigger              | Triggers on new files in Drive      | External               | Download file                 | ## Auto Update Knowledge Base: When you add a file to a specific folder in your Google Drive it automatically updates the vector database and adds the file to the knowledge base. |
| Download file            | n8n-nodes-base.googleDrive                      | Downloads new files                  | Google Drive Trigger   | Default Data Loader           |                                                                                                                  |
| Default Data Loader      | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Converts binary file to text         | Download file          | Recursive Character Text Splitter |                                                                                                                  |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits document text into chunks    | Default Data Loader     | Embeddings OpenAI1            |                                                                                                                  |
| Embeddings OpenAI1       | @n8n/n8n-nodes-langchain.embeddingsOpenAi       | Creates vector embeddings            | Recursive Character Text Splitter | Pinecone Vector Store     |                                                                                                                  |
| Pinecone Vector Store    | @n8n/n8n-nodes-langchain.vectorStorePinecone    | Inserts embeddings into Pinecone    | Embeddings OpenAI1, Default Data Loader | None                     |                                                                                                                  |
| Sticky Note1             | n8n-nodes-base.stickyNote                       | Documentation                       |                       |                               | ## AI Gmail Customer Support: This template auto-replies to customer emails in Gmail using AI + your knowledge base. |
| Sticky Note2             | n8n-nodes-base.stickyNote                       | Documentation                       |                       |                               | ## Auto Update Knowledge Base: When you add a file to a specific folder in your Google Drive it automatically update the vector database and add the file to the knowledge base. |
| Sticky Note3             | n8n-nodes-base.stickyNote                       | Documentation                       |                       |                               |                                                                                                                  |
| Sticky Note4             | n8n-nodes-base.stickyNote                       | Documentation                       |                       |                               |                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Poll every minute, no filters  
   - Connect Gmail OAuth2 credentials  
   - Purpose: Trigger on every new email.

2. **Add Text Classifier Node**  
   - Type: Langchain Text Classifier  
   - Configure categories:  
     - "Customer Support": emails related to product/service inquiries  
     - "Other": all other emails  
   - Set input text to `={{ $json.text }}` from Gmail Trigger  
   - Connect output branches: "Customer Support" → AI Agent; "Other" → No Operation.

3. **Add OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: gpt-4o-mini (GPT-4 optimized mini)  
   - Connect as language model for Text Classifier node  
   - Provide OpenAI API credentials.

4. **Add AI Agent Node**  
   - Type: Langchain Agent  
   - Set input text: `={{ $('Gmail Trigger').item.json.text }}`  
   - Configure system message:  
     - Role: Customer support agent from Blueproof  
     - Instructions: Output only email body, sign off as "Yasser @ Blueproof", avoid hallucinations  
   - Set prompt type: define  
   - Connect knowledgebase node as AI tool (see next step)  
   - Add OpenAI Chat Model (gpt-4.1-mini) as language model for AI Agent  
   - Provide OpenAI API credentials.

5. **Add knowledgebase Node (Pinecone Vector Store)**  
   - Type: Langchain Pinecone Vector Store  
   - Mode: Retrieve-as-tool  
   - Pinecone Index: "n8n"  
   - Namespace: "FAQ"  
   - Connect Pinecone API credentials  
   - Connect as AI tool input to AI Agent.

6. **Add "Add label to message" Node**  
   - Type: Gmail  
   - Operation: Add label  
   - Label: "IMPORTANT"  
   - Message ID: `={{ $('Gmail Trigger').item.json.id }}`  
   - Connect output from AI Agent to this node  
   - Connect Gmail OAuth2 credentials.

7. **Add "Reply to a message" Node**  
   - Type: Gmail  
   - Operation: Reply  
   - Message: `={{ $('AI Agent').item.json.output }}`  
   - Message ID: `={{ $('Gmail Trigger').item.json.id }}`  
   - Connect output from "Add label to message"  
   - Use Gmail OAuth2 credentials.

8. **Add No Operation Node**  
   - Type: NoOp  
   - Connect from Text Classifier output branch for "Other" category  
   - Provides safe fallback doing nothing.

9. **Setup Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Event: File created  
   - Folder to watch: specific folder ID (e.g., your FAQ docs folder)  
   - Poll every minute  
   - Connect Google Drive OAuth2 credentials.

10. **Add Download File Node**  
    - Type: Google Drive  
    - Operation: Download  
    - File ID: `={{ $json.id }}` from Google Drive Trigger  
    - Connect Google Drive OAuth2 credentials.

11. **Add Default Data Loader Node**  
    - Type: Langchain Document Default Data Loader  
    - Input: Binary data from Download file  
    - Data type: Binary  
    - Text splitting mode: Custom.

12. **Add Recursive Character Text Splitter Node**  
    - Type: Langchain Recursive Character Text Splitter  
    - Input: Text from Default Data Loader  
    - Configure default options (no custom changes needed).

13. **Add Embeddings OpenAI Node**  
    - Type: Langchain OpenAI Embeddings  
    - Input: Text chunks from Recursive Character Text Splitter  
    - Connect OpenAI API credentials.

14. **Add Pinecone Vector Store Node**  
    - Type: Langchain Pinecone Vector Store  
    - Mode: Insert  
    - Pinecone Index: "n8n"  
    - Namespace: "FAQ"  
    - Connect Pinecone API credentials  
    - Inputs: Embeddings from Embeddings OpenAI, documents from Default Data Loader.

15. **Connect nodes in sequence:**  
    Google Drive Trigger → Download file → Default Data Loader → Recursive Character Text Splitter → Embeddings OpenAI → Pinecone Vector Store.

16. **Add Sticky Notes** (optional for documentation) to explain blocks:  
    - AI Gmail Customer Support overview  
    - Classification explanation  
    - AI Agent description  
    - Auto Update Knowledge Base description.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This template auto-replies to customer emails in Gmail using AI + your knowledge base.                                                   | Sticky Note1                                                                                        |
| When you add a file to a specific folder in your Google Drive it automatically updates the vector database and adds the file to the knowledge base. | Sticky Note2                                                                                        |
| AI checks if the email is customer support or not.                                                                                      | Sticky Note3                                                                                        |
| AI Agent searches your knowledge base and drafts a reply.                                                                               | Sticky Note4                                                                                        |

---

**Disclaimer:** The provided content is generated from an automated workflow built with n8n, respecting all content policies and handling only legal and public data.