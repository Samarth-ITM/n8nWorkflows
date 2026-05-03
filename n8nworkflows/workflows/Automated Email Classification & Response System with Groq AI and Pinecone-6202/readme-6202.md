Automated Email Classification & Response System with Groq AI and Pinecone

https://n8nworkflows.xyz/workflows/automated-email-classification---response-system-with-groq-ai-and-pinecone-6202


# Automated Email Classification & Response System with Groq AI and Pinecone

### 1. Workflow Overview

This workflow automates the classification and response process for incoming emails using a combination of rule-based logic, Large Language Models (LLMs) including Groq AI models, and vector search with Pinecone. It integrates email reception, categorization by topic, AI-assisted decision making, sentiment analysis, and dynamic routing of responses or notifications to relevant teams. The workflow also supports processing business documents into a vector database for enriched AI context retrieval.

**Target Use Cases:**  
- Automated triage of customer emails by category (HR, billing, complaints, feedback, inquiries, sales, unknown)  
- AI-enhanced evaluation of job applications  
- Sentiment analysis of feedback emails  
- Automated replies to customer inquiries using retrieval-augmented generation (RAG) based on company documents  
- Notifications to internal teams depending on category  
- Document ingestion and embedding to support AI responses  

**Logical Blocks:**  
- **1.1 Email Reception and Basic Categorization:** Receiving emails via IMAP, initial rule-based classification  
- **1.2 AI Classification Fallback:** Use of Groq AI and LangChain LLMs to classify emails that don’t match simple rules  
- **1.3 Category-Based Routing:** Switch node routes emails to different processing paths and email notifications  
- **1.4 HR Candidate Evaluation:** AI evaluates candidate suitability and triggers acceptance/rejection workflows  
- **1.5 Feedback Sentiment Analysis:** Sentiment analysis on feedback emails with notifications and social media posting  
- **1.6 Inquiry Handling with RAG:** Retrieval augmented generation for answering customer inquiries using Pinecone vector search  
- **1.7 Document Processing to Vector Store:** Ingest business documents (PDF) into Pinecone for knowledge base enrichment  

---

### 2. Block-by-Block Analysis

#### 1.1 Email Reception and Basic Categorization

- **Overview:**  
  This block listens for new emails via IMAP, downloads attachments, and applies a JavaScript rule-based categorization based on subject keywords.

- **Nodes Involved:**  
  - Email Trigger (IMAP)  
  - Code (Rule-based Category Assignment)  
  - Switch1 (Category Routing)  
  - Sticky Note (EMAIL CLASSIFIER USING SWITCH)

- **Node Details:**

  - **Email Trigger (IMAP)**  
    - Type: Email Read (IMAP)  
    - Role: Listens for incoming emails and downloads attachments  
    - Config: Default IMAP settings, attachments enabled  
    - Inputs: none (trigger node)  
    - Outputs: Email JSON with metadata and content  
    - Edge cases: Connection/authentication failures, attachment download errors  

  - **Code (Rule-based categorization)**  
    - Type: Code node (JavaScript)  
    - Role: Assigns email category by checking subject keywords (e.g., "resume" → hr, "invoice" → billing)  
    - Key expressions: Regex tests on lowercase subject  
    - Inputs: Email JSON from IMAP trigger  
    - Outputs: JSON with added `category` field  
    - Edge cases: Missing subject, unexpected subject formats  

  - **Switch1**  
    - Type: Switch node  
    - Role: Routes emails based on the `category` field computed in the Code node  
    - Categories: hr, billing, complaint, feedback, inquiry, sales, unknown  
    - Inputs: Categorized email JSON  
    - Outputs: Branches per category for further processing  
    - Edge cases: Misclassification if category not matched correctly  

  - **Sticky Note (EMAIL CLASSIFIER USING SWITCH)**  
    - Annotation describing the purpose of the Switch node for visual clarity  

#### 1.2 AI Classification Fallback

- **Overview:**  
  For emails that cannot be categorized confidently by rules, this block uses Groq AI and LangChain LLM chains to classify emails and normalize AI outputs.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - Groq Chat Model  
  - CLEAN AI AGENT OUTPUT (Code)  
  - Sticky Note1 (USE FOR AI AGENT WHEN SWITCH FAILS)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain LLM chain  
    - Role: Given subject and body, return exact category from fixed list  
    - Config: Prompt specifying categories and instructions for classification  
    - Inputs: Email JSON from Switch1’s unknown category branch  
    - Outputs: LLM response text with predicted category  
    - Edge cases: LLM response not matching expected categories  

  - **Groq Chat Model**  
    - Type: Groq AI chat model node  
    - Role: Underlying language model powering the Basic LLM Chain  
    - Config: Model "llama-3.3-70b-versatile" selected  
    - Inputs: Text prompt from Basic LLM Chain  
    - Outputs: Model-generated classification text  
    - Edge cases: API timeout, rate limits, connectivity issues  

  - **CLEAN AI AGENT OUTPUT (Code)**  
    - Type: Code node  
    - Role: Normalizes LLM output text to one of known categories or "unknown"  
    - Key expressions: Lowercase trim, inclusion check against category list  
    - Inputs: Raw LLM output  
    - Outputs: JSON with standardized `category` field  
    - Edge cases: Unexpected output formats, empty responses   

  - **Sticky Note1**  
    - Explains this block is used when the rule-based switch fails to classify emails  

#### 1.3 Category-Based Routing and Notifications

- **Overview:**  
  Routes emails to different teams or internal workflows based on category, triggering email notifications or further AI processing.

- **Nodes Involved:**  
  - Multiple emailSend nodes: to hr, to sales, send to customer, send to support team, send email to team  
  - Sentiment Analysis of feedback  
  - Switch node for HR acceptance/rejection  
  - Switch1 main branches  
  - Sticky Note2 (check if feedback is positive or negative)  

- **Node Details:**

  - **Send to Customer / Support / Teams (emailSend nodes)**  
    - Type: Email Send nodes  
    - Role: Send category-specific emails (e.g., complaint acknowledgment, billing notification, sales lead forwarding)  
    - Config: Dynamic email subjects and bodies based on JSON data, emails sent from configured credentials  
    - Inputs: JSON from Switch1 or downstream nodes  
    - Outputs: None (side effect nodes)  
    - Edge cases: SMTP authentication issues, invalid recipient emails  

  - **Sentiment Analysis of feedback**  
    - Type: LangChain Sentiment Analysis node  
    - Role: Analyze feedback emails to categorize sentiment as Positive, Neutral, or Negative  
    - Inputs: Plain text of feedback email  
    - Outputs: Sentiment category for further routing or notification  
    - Edge cases: Ambiguous sentiment, empty text  

  - **Switch (HR decision)**  
    - Routes AI candidate evaluation to acceptance or rejection email workflows  

  - **Sticky Note2**  
    - Notes the sentiment analysis step on feedback emails  

#### 1.4 HR Candidate Evaluation

- **Overview:**  
  Evaluates job application emails via AI, deciding acceptance or rejection and notifying HR and the candidate accordingly.

- **Nodes Involved:**  
  - Basic LLM Chain1 (HR evaluation prompt)  
  - Groq Chat Model3 (AI engine for HR evaluation)  
  - Code2 (parse AI decision)  
  - Switch (decision accept/reject)  
  - accepted confirm to candidate (emailSend)  
  - rejection email (emailSend)  
  - to hr (emailSend)  

- **Node Details:**

  - **Basic LLM Chain1**  
    - Role: AI prompt to evaluate candidate email/resume against job description  
    - Inputs: Candidate email text  
    - Outputs: AI decision text (ACCEPT/REJECT with explanation)  

  - **Groq Chat Model3**  
    - Groq AI model powering the candidate evaluation prompt  

  - **Code2**  
    - Parses AI response to decide acceptance or rejection  
    - Outputs decision flag and response text  

  - **Switch**  
    - Routes based on decision field to acceptance or rejection email flows  

  - **accepted confirm to candidate / rejection email**  
    - Sends email notifications to candidate about application status  

  - **to hr**  
    - Notifies HR team about accepted candidates  

#### 1.5 Feedback Sentiment Analysis and Social Media Posting

- **Overview:**  
  Analyzes feedback sentiment, sends notifications, and posts positive feedback to Twitter.

- **Nodes Involved:**  
  - Sentiment Analysis of feedback  
  - Groq Chat Model1 (sentiment AI)  
  - X (Twitter node)  
  - send email to team (for negative feedback notifications)  

- **Node Details:**

  - **Sentiment Analysis of feedback & Groq Chat Model1**  
    - Combined to analyze and confirm sentiment category  

  - **X (Twitter)**  
    - Posts positive customer feedback as tweets  
    - Configured with Twitter credentials  
    - Inputs: Feedback text  

  - **send email to team**  
    - Sends email notifications on negative feedback to internal team  

#### 1.6 Inquiry Handling with RAG (Retrieval Augmented Generation)

- **Overview:**  
  For inquiry category emails, retrieves relevant context from Pinecone vector store and generates AI email replies.

- **Nodes Involved:**  
  - Pinecone Vector Store (retrieve mode)  
  - Embeddings Cohere (embeddings generation)  
  - RAG INQURY REPLY (LangChain agent)  
  - Groq Chat Model2 (underlying AI model)  
  - send reply to customer (emailSend)  

- **Node Details:**

  - **Embeddings Cohere**  
    - Generates embeddings for incoming inquiry email text for similarity search  

  - **Pinecone Vector Store**  
    - Retrieves relevant documents based on embeddings as context for AI reply  

  - **RAG INQURY REPLY**  
    - Agent node uses retrieved context and inquiry text to generate professional response email  
    - Has output parser to structure AI output  

  - **Groq Chat Model2**  
    - AI engine for RAG response generation  

  - **send reply to customer**  
    - Sends AI-generated response email to the customer  

#### 1.7 Document Processing to Vector Store

- **Overview:**  
  Imports business documents (PDF) from Dropbox, extracts text, cleans it, splits for chunking, generates embeddings, and inserts into Pinecone vector database to enrich company knowledge base.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (manual trigger)  
  - HTTP Request (downloads PDF from Dropbox)  
  - Extract from File (PDF text extraction)  
  - Code3 (text cleaning)  
  - Pinecone Vector Store1 (insert mode)  
  - Embeddings Cohere1 (multilingual embeddings)  
  - Default Data Loader (prepares document for vector store)  
  - Recursive Character Text Splitter (splits text into chunks)  
  - Sticky Note3 (details of business in pdf form to vector db)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Manual trigger to start document ingestion  

  - **HTTP Request**  
    - Downloads PDF document from provided Dropbox URL  
    - Edge cases: network failures, invalid URL  

  - **Extract from File**  
    - Extracts text content from PDF file  

  - **Code3**  
    - Cleans extracted text by trimming and removing extra whitespace  

  - **Recursive Character Text Splitter**  
    - Splits cleaned text into manageable chunks for vector embedding  

  - **Default Data Loader**  
    - Prepares document chunks for embedding and insertion  

  - **Embeddings Cohere1**  
    - Generates multilingual embeddings for document chunks  

  - **Pinecone Vector Store1**  
    - Inserts embeddings into Pinecone knowledge base index "demokb"  

  - **Sticky Note3**  
    - Describes purpose of this document ingestion flow  

---

### 3. Summary Table

| Node Name                    | Node Type                             | Functional Role                             | Input Node(s)                | Output Node(s)                 | Sticky Note                                                    |
|------------------------------|-------------------------------------|---------------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------|
| Email Trigger (IMAP)          | n8n-nodes-base.emailReadImap         | Incoming email reception                     | -                           | Code                          |                                                               |
| Code                         | n8n-nodes-base.code                  | Rule-based email categorization              | Email Trigger (IMAP)          | Switch1                       |                                                               |
| Switch1                      | n8n-nodes-base.switch                | Routes emails by category                     | Code                        | Multiple category branches     | EMAIL CLASSIFIER USING SWITCH                                  |
| Basic LLM Chain              | @n8n/n8n-nodes-langchain.chainLlm   | AI fallback email classification             | Switch1 (unknown branch)      | CLEAN AI AGENT OUTPUT          | USE FOR AI AGENT WHEN SWITCH FAILS                            |
| Groq Chat Model              | @n8n/n8n-nodes-langchain.lmChatGroq | LLM engine for classification                 | Basic LLM Chain               | Basic LLM Chain               |                                                               |
| CLEAN AI AGENT OUTPUT        | n8n-nodes-base.code                  | Normalize AI classification output            | Basic LLM Chain              | Switch1                       | USE FOR AI AGENT WHEN SWITCH FAILS                            |
| Basic LLM Chain1             | @n8n/n8n-nodes-langchain.chainLlm   | Evaluate job application suitability          | Switch1 (hr branch)           | Code2                         |                                                               |
| Groq Chat Model3             | @n8n/n8n-nodes-langchain.lmChatGroq | LLM engine for HR candidate evaluation        | Basic LLM Chain1              | Basic LLM Chain1              |                                                               |
| Code2                        | n8n-nodes-base.code                  | Parse AI decision (accept/reject)             | Basic LLM Chain1             | Switch                        |                                                               |
| Switch                       | n8n-nodes-base.switch                | Routes candidate decision to email flows      | Code2                       | accepted confirm to candidate, rejection email |                                                               |
| accepted confirm to candidate| n8n-nodes-base.emailSend             | Send acceptance email to candidate            | Switch                      | to hr                         |                                                               |
| rejection email             | n8n-nodes-base.emailSend             | Send rejection email to candidate             | Switch                      | -                            |                                                               |
| to hr                       | n8n-nodes-base.emailSend             | Notify HR team of accepted candidate          | accepted confirm to candidate| -                            |                                                               |
| Sentiment Analysis of feedback| @n8n/n8n-nodes-langchain.sentimentAnalysis| Analyze sentiment of feedback emails          | Switch1 (feedback branch)    | X, send email to team         | check if feedback is positive or negative                     |
| Groq Chat Model1             | @n8n/n8n-nodes-langchain.lmChatGroq | LLM engine for sentiment analysis              | Sentiment Analysis of feedback| Sentiment Analysis of feedback|                                                               |
| X                           | n8n-nodes-base.twitter               | Post positive feedback tweets                   | Sentiment Analysis of feedback| -                            |                                                               |
| send email to team           | n8n-nodes-base.emailSend             | Notify team of negative feedback                | Sentiment Analysis of feedback| -                            |                                                               |
| Send to customer             | n8n-nodes-base.emailSend             | Acknowledge complaints to customers             | Switch1 (complaint branch)   | send to support team          |                                                               |
| send to support team         | n8n-nodes-base.emailSend             | Notify support team of complaints                | Send to customer             | -                            |                                                               |
| TO SALES TEAM                | n8n-nodes-base.emailSend             | Forward sales emails to sales team               | Switch1 (sales branch)       | -                            |                                                               |
| bill send to team            | n8n-nodes-base.emailSend             | Notify billing team of billing emails            | Switch1 (billing branch)     | -                            |                                                               |
| RAG INQURY REPLY             | @n8n/n8n-nodes-langchain.agent       | Generate AI replies to inquiries from RAG context | Pinecone Vector Store       | send reply to customer         |                                                               |
| Groq Chat Model2             | @n8n/n8n-nodes-langchain.lmChatGroq | LLM engine for RAG-based inquiry reply          | RAG INQURY REPLY            | RAG INQURY REPLY             |                                                               |
| Pinecone Vector Store        | @n8n/n8n-nodes-langchain.vectorStorePinecone| Retrieve documents for inquiry context          | Embeddings Cohere           | RAG INQURY REPLY             |                                                               |
| Embeddings Cohere            | @n8n/n8n-nodes-langchain.embeddingsCohere| Generate embeddings for inquiry text             | Switch1 (inquiry branch)    | Pinecone Vector Store         |                                                               |
| send reply to customer      | n8n-nodes-base.emailSend             | Send AI-generated replies to customers            | RAG INQURY REPLY            | -                            |                                                               |
| When clicking ‘Execute workflow’| n8n-nodes-base.manualTrigger        | Manual trigger to start document ingestion        | -                           | HTTP Request                  | details of business in pdf form to vector db                 |
| HTTP Request                | n8n-nodes-base.httpRequest           | Download PDF business document                     | When clicking ‘Execute workflow’| Extract from File            |                                                               |
| Extract from File           | n8n-nodes-base.extractFromFile       | Extract text from downloaded PDF                    | HTTP Request                | Code3                         |                                                               |
| Code3                       | n8n-nodes-base.code                  | Clean extracted text                                 | Extract from File           | Pinecone Vector Store1        |                                                               |
| Recursive Character Text Splitter| @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter| Break text into chunks for embedding          | Default Data Loader         | Default Data Loader           |                                                               |
| Default Data Loader         | @n8n/n8n-nodes-langchain.documentDefaultDataLoader| Prepare text chunks for embedding                  | Recursive Character Text Splitter| Pinecone Vector Store1     |                                                               |
| Embeddings Cohere1          | @n8n/n8n-nodes-langchain.embeddingsCohere| Generate multilingual embeddings for document chunks| Default Data Loader         | Pinecone Vector Store1        |                                                               |
| Pinecone Vector Store1      | @n8n/n8n-nodes-langchain.vectorStorePinecone| Insert embeddings into Pinecone vector DB          | Embeddings Cohere1          | -                            |                                                               |
| Sticky Note                 | n8n-nodes-base.stickyNote            | Workflow annotation                                  | -                           | -                            | EMAIL CLASSIFIER USING SWITCH                                  |
| Sticky Note1                | n8n-nodes-base.stickyNote            | Workflow annotation                                  | -                           | -                            | USE FOR AI AGENT WHEN SWITCH FAILS                            |
| Sticky Note2                | n8n-nodes-base.stickyNote            | Workflow annotation                                  | -                           | -                            | check if feedback is positive or negative                     |
| Sticky Note3                | n8n-nodes-base.stickyNote            | Workflow annotation                                  | -                           | -                            | details of business in pdf form to vector db                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Email Trigger (IMAP):**  
   - Type: Email Read (IMAP)  
   - Settings: Enable download attachments, configure IMAP credentials and folder  

2. **Add Code node (Rule-based Categorization):**  
   - Run once for each item  
   - JS code tests email subject for keywords to assign a category field (hr, billing, complaint, feedback, inquiry, sales, unknown)  

3. **Add Switch node (Switch1):**  
   - Input: category field  
   - Add output branches for categories: hr, billing, complaint, feedback, inquiry, sales, unknown  

4. **For unknown category branch:**  
   - Add Basic LLM Chain node:  
     - Prompt: Provide subject and body, request one of known categories  
   - Add Groq Chat Model node:  
     - Select model "llama-3.3-70b-versatile"  
   - Connect Groq Chat Model as AI model for Basic LLM Chain  
   - Add Code node to normalize AI output to known categories or unknown  
   - Connect output to Switch1 again for rerouting  

5. **For hr category branch:**  
   - Add Basic LLM Chain1 node with prompt to evaluate candidate email suitability  
   - Add Groq Chat Model3 node as AI model  
   - Connect Groq Chat Model3 to Basic LLM Chain1  
   - Add Code2 node to parse AI decision (accept/reject)  
   - Add Switch node to route accept or reject outputs  
   - For accept branch: add emailSend node "accepted confirm to candidate" to notify candidate  
     - Configure fromEmail, to candidate’s email (from original email trigger)  
   - Add another emailSend node "to hr" to notify HR team of acceptance  
   - For reject branch: add emailSend node "rejection email" to notify candidate  

6. **For billing category branch:**  
   - Add emailSend node "bill send to team" to notify billing team  

7. **For complaint category branch:**  
   - Add emailSend node "Send to customer" to acknowledge complaint to customer  
   - Connect to emailSend node "send to support team" to notify internal support  

8. **For feedback category branch:**  
   - Add Sentiment Analysis node: configure categories Positive, Neutral, Negative  
   - Add Groq Chat Model1 node as AI model for sentiment analysis  
   - Add Twitter node "X" configured with Twitter credentials to post positive feedback tweets  
   - Add emailSend node "send email to team" to notify internal team on negative feedback  

9. **For inquiry category branch:**  
   - Add Embeddings Cohere node to generate embeddings for inquiry text  
   - Add Pinecone Vector Store node in retrieve mode, linked to "demokb" index  
   - Add RAG INQURY REPLY LangChain agent node with prompt to reply using retrieved context  
   - Add Groq Chat Model2 as AI model  
   - Add emailSend node "send reply to customer" to send generated reply  

10. **For sales category branch:**  
    - Add emailSend node "TO SALES TEAM" to forward sales emails  

11. **Document ingestion workflow:**  
    - Add Manual Trigger node "When clicking ‘Execute workflow’"  
    - Add HTTP Request node to download PDF from Dropbox URL  
    - Add Extract from File node to extract text from PDF  
    - Add Code node (Code3) to clean extracted text  
    - Add Recursive Character Text Splitter node to chunk text  
    - Add Default Data Loader node to prepare chunks  
    - Add Embeddings Cohere1 node for multilingual embeddings  
    - Add Pinecone Vector Store1 node in insert mode to add documents to "demokb" index  

12. **Set up credentials for:**  
    - IMAP email account  
    - SMTP email for sending (used in all emailSend nodes)  
    - Groq AI API for language model nodes  
    - Cohere API for embeddings nodes  
    - Pinecone API for vector store nodes  
    - Twitter API for posting tweets  

13. **Test end-to-end:**  
    - Send test emails for each category  
    - Trigger manual ingestion for documents  
    - Verify AI classification fallback triggers on unknown subjects  
    - Check email notifications and Twitter posting  

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow integrates Groq AI LLMs with Pinecone vector database for advanced email automation                           |                                                                                                 |
| Uses LangChain nodes for chaining prompts and handling AI outputs                                                     |                                                                                                 |
| Twitter node posts positive customer feedback as social proof                                                        | Twitter API documentation: https://developer.twitter.com/en/docs/twitter-api                    |
| Pinecone used for vector similarity search and document retrieval                                                    | Pinecone docs: https://docs.pinecone.io/                                                        |
| Cohere embeddings used for both English and multilingual text                                                         | Cohere embeddings docs: https://docs.cohere.ai/                                                |
| Manual trigger allows batch document ingestion to update knowledge base                                                |                                                                                                 |
| Email credentials must be securely configured with OAuth2 or app passwords depending on provider                      |                                                                                                 |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. All content complies with applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.