Build AI-Ready Knowledge Base from Outlook & Notion using Pinecone & GPT-4

https://n8nworkflows.xyz/workflows/build-ai-ready-knowledge-base-from-outlook---notion-using-pinecone---gpt-4-9614


# Build AI-Ready Knowledge Base from Outlook & Notion using Pinecone & GPT-4

### 1. Workflow Overview

This workflow automates the construction of an AI-ready knowledge base by ingesting content from two primary sources: Microsoft Outlook emails and Notion pages. It processes, cleans, embeds, and indexes these contents into Pinecone vector stores under distinct namespaces, enabling an AI agent powered by GPT-4 to retrieve relevant information for answering customer queries. The workflow is structured in three main logical blocks:

- **1.1 Outlook Email Ingestion & Processing:** Watches a specific Outlook folder for new emails or threads, retrieves all related messages, removes duplicates, splits and embeds the email content, and stores it in Pinecone under the "emails" namespace.

- **1.2 Notion Page Ingestion & Processing:** Triggers on new or updated pages in a Notion database, fetches nested content blocks (including synced blocks), removes duplicates, splits and embeds the content, and stores it in Pinecone under the "knowledgebase" namespace.

- **1.3 AI Agent Query Handling:** Listens for chat messages, uses the indexed content from both Pinecone namespaces ("emails" and "knowledgebase") as retrieval tools, and employs GPT-4 to generate context-aware answers based on the stored knowledge base and previous emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Outlook Email Ingestion & Processing

- **Overview:**  
  This block monitors a dedicated Outlook folder for new emails or threads, retrieves all messages in the conversation, removes duplicate content, processes the email body, generates embeddings, and stores vector representations in Pinecone for later retrieval.

- **Nodes Involved:**  
  - Microsoft Outlook Trigger  
  - Get many messages  
  - Remove Duplicates  
  - Embeddings Cohere  
  - Default Data Loader  
  - Recursive Character Text Splitter  
  - Pinecone Vector Store (emails namespace)  
  - Sticky Note1 (documentation note)

- **Node Details:**

  - **Microsoft Outlook Trigger**  
    - *Type & Role:* Trigger node that listens for new emails in a specific Outlook folder (identified by folder ID).  
    - *Configuration:* Polls every minute; downloads attachments; filters messages by a particular folder (likely named "knowledgebase" or similar).  
    - *Input/Output:* Starts workflow on new email data; outputs raw email data including conversationId.  
    - *Potential Failures:* OAuth token expiration, API rate limits, folder ID changes.

  - **Get many messages**  
    - *Type & Role:* Retrieves all messages belonging to the conversationId from the trigger node.  
    - *Configuration:* Uses Microsoft Outlook API with filter `conversationId eq '{{ $json.conversationId }}'`; returns all messages.  
    - *Input/Output:* Input from Outlook Trigger; outputs raw email messages array.  
    - *Potential Failures:* API call limits, invalid conversationId, timeout.

  - **Remove Duplicates**  
    - *Type & Role:* Removes duplicate emails based on the `body.content` field to avoid redundant processing.  
    - *Configuration:* Compares selected field `body.content`.  
    - *Input/Output:* Input from Get many messages; outputs unique emails only.  
    - *Potential Failures:* Incorrect field comparison if email body structure changes.

  - **Embeddings Cohere**  
    - *Type & Role:* Generates multilingual vector embeddings for the text content using Cohere API.  
    - *Configuration:* Model: `embed-multilingual-v3.0`.  
    - *Credentials:* Requires valid Cohere API key.  
    - *Input/Output:* Input: email text from Default Data Loader; outputs embedding vectors.  
    - *Potential Failures:* API limits, invalid API key, network issues.

  - **Default Data Loader**  
    - *Type & Role:* Prepares the document text for embedding by parsing and structuring the email body content.  
    - *Configuration:* Uses JSON expression to extract `body.content`; custom text splitting mode.  
    - *Input/Output:* Input from Recursive Character Text Splitter; outputs structured document data.  
    - *Potential Failures:* Expression errors if JSON structure changes.

  - **Recursive Character Text Splitter**  
    - *Type & Role:* Splits large email text into manageable chunks with overlap for better embedding quality.  
    - *Configuration:* 100 characters overlap between chunks.  
    - *Input/Output:* Input raw text from Remove Duplicates; outputs smaller text chunks.  
    - *Potential Failures:* Incorrect splitting if input text format varies.

  - **Pinecone Vector Store (emails namespace)**  
    - *Type & Role:* Inserts the generated embeddings into Pinecone vector database under namespace "emails".  
    - *Configuration:* Embedding batch size 50; index named "n8n".  
    - *Credentials:* Pinecone API key required.  
    - *Input/Output:* Input embeddings from Embeddings Cohere; stores vectors for retrieval.  
    - *Potential Failures:* API quota exceeded, invalid namespace/index, network errors.

  - **Sticky Note1**  
    - *Content:* Describes this block as the Outlook knowledge base ingestion, emphasizing email thread capture, deduplication, and storage in Pinecone for AI agent use.

#### 2.2 Notion Page Ingestion & Processing

- **Overview:**  
  This block triggers on new or updated pages in a specific Notion database, recursively fetches all nested content blocks (including those within synced blocks), merges content, removes duplicates, splits text, generates embeddings, and indexes them into Pinecone under the "knowledgebase" namespace.

- **Nodes Involved:**  
  - Notion Trigger  
  - Get many child blocks  
  - If (condition to check block type)  
  - Get many child blocks1 (fetch synced block content)  
  - Merge  
  - Split Out  
  - Remove Duplicates1  
  - Embeddings Cohere1  
  - Default Data Loader1  
  - Recursive Character Text Splitter1  
  - Pinecone Vector Store1 (knowledgebase namespace)  
  - Sticky Note4 (documentation note)

- **Node Details:**

  - **Notion Trigger**  
    - *Type & Role:* Watches a specified Notion database for new or updated pages.  
    - *Configuration:* Polls every minute; uses database ID for monitoring.  
    - *Input/Output:* Outputs new page JSON data.  
    - *Potential Failures:* Notion API limits, invalid database ID, OAuth token expiration.

  - **Get many child blocks**  
    - *Type & Role:* Recursively fetches all content blocks from the triggered Notion page, including nested blocks.  
    - *Configuration:* Fetches nested blocks, does not simplify output to preserve structure.  
    - *Input/Output:* Input: page ID from Notion Trigger; output: array of blocks.  
    - *Potential Failures:* Large pages may cause timeout, API errors.

  - **If**  
    - *Type & Role:* Checks if block type is `synced_block` to handle content from synced blocks differently.  
    - *Configuration:* Condition: `$json.type == "synced_block"`.  
    - *Input/Output:* Routes synced blocks to additional fetching.  
    - *Potential Failures:* Expression failures, missing type field.

  - **Get many child blocks1**  
    - *Type & Role:* Retrieves blocks from the original synced block source.  
    - *Configuration:* Uses synced block's source block ID.  
    - *Input/Output:* Input: synced block data; output: additional nested blocks.  
    - *Potential Failures:* Missing synced_from data, API errors.

  - **Merge**  
    - *Type & Role:* Combines normal and synced block content into a single array for processing.  
    - *Configuration:* Default merge mode (append).  
    - *Input/Output:* Inputs: outputs from If node branches; output: merged blocks.  
    - *Potential Failures:* Data structure mismatches.

  - **Split Out**  
    - *Type & Role:* Extracts the `paragraph.text` field from each block, splitting the array into individual items for deduplication.  
    - *Configuration:* Field to split out: `paragraph.text`.  
    - *Input/Output:* Input: merged blocks; output: individual paragraph texts.  
    - *Potential Failures:* Missing `paragraph.text` field on some blocks.

  - **Remove Duplicates1**  
    - *Type & Role:* Removes duplicate paragraphs by comparing `plain_text` fields.  
    - *Configuration:* Compares selected field `plain_text`.  
    - *Input/Output:* Input: individual paragraphs; output: unique paragraphs.  
    - *Potential Failures:* Field missing or inconsistent text formatting.

  - **Embeddings Cohere1**  
    - *Type & Role:* Generates multilingual embeddings for Notion page content using Cohere API.  
    - *Configuration:* Model: `embed-multilingual-v3.0`.  
    - *Credentials:* Cohere API key required.  
    - *Input/Output:* Input: text chunks; output: embedding vectors.  
    - *Potential Failures:* API limits, invalid credentials.

  - **Default Data Loader1**  
    - *Type & Role:* Prepares Notion paragraph text for embedding.  
    - *Configuration:* JSON expression extracting `plain_text`; custom text splitting.  
    - *Input/Output:* Input: text chunks; output: structured document data.  
    - *Potential Failures:* Expression errors if input format changes.

  - **Recursive Character Text Splitter1**  
    - *Type & Role:* Splits Notion content into overlapping chunks for embedding.  
    - *Configuration:* 100 characters overlap.  
    - *Input/Output:* Input: paragraph text; output: text chunks.  
    - *Potential Failures:* Incorrect splitting if text format varies.

  - **Pinecone Vector Store1 (knowledgebase namespace)**  
    - *Type & Role:* Inserts Notion content embeddings into Pinecone under the "knowledgebase" namespace.  
    - *Configuration:* Batch size 50; index "n8n".  
    - *Credentials:* Valid Pinecone API key.  
    - *Input/Output:* Input embeddings; output: stored vectors for retrieval.  
    - *Potential Failures:* API quota, network issues.

  - **Sticky Note4**  
    - *Content:* Describes this block as the Notion knowledge base ingestion, explaining the trigger, content capture, storage in Pinecone, and its use by the AI agent.

#### 2.3 AI Agent Query Handling

- **Overview:**  
  This block listens for incoming chat messages, uses Pinecone vector stores populated by Outlook emails and Notion pages as retrieval tools, applies contextual memory, and utilizes GPT-4 to generate informed responses based on the indexed knowledge.

- **Nodes Involved:**  
  - When chat message received  
  - Pinecone Vector Store2 (knowledgebase retrieval tool)  
  - Pinecone Vector Store3 (emails retrieval tool)  
  - Embeddings Cohere2  
  - Simple Memory  
  - OpenAI Chat Model (GPT-4)  
  - AI Agent  
  - Sticky Note3 (documentation note)

- **Node Details:**

  - **When chat message received**  
    - *Type & Role:* Trigger node activated when a chat message arrives.  
    - *Configuration:* Listens on webhook ID; no additional filters.  
    - *Input/Output:* Receives chat input; outputs user message data.  
    - *Potential Failures:* Webhook misconfiguration, network issues.

  - **Pinecone Vector Store2 (knowledgebase retrieval tool)**  
    - *Type & Role:* Retrieval tool to query the "knowledgebase" namespace in Pinecone for relevant Notion content.  
    - *Configuration:* Mode `retrieve-as-tool`; index "n8n"; description "use this tool to look at your knowledge base."  
    - *Credentials:* Pinecone API key.  
    - *Input/Output:* Receives embedding input from Embeddings Cohere2; outputs relevant vector search results.  
    - *Potential Failures:* API errors, invalid namespace.

  - **Pinecone Vector Store3 (emails retrieval tool)**  
    - *Type & Role:* Retrieval tool to query the "emails" namespace in Pinecone for similar emails.  
    - *Configuration:* Mode `retrieve-as-tool`; index "n8n"; description "use this tool to look at similar emails."  
    - *Credentials:* Pinecone API key.  
    - *Input/Output:* Receives embedding input; outputs relevant email vectors.  
    - *Potential Failures:* API errors, invalid namespace.

  - **Embeddings Cohere2**  
    - *Type & Role:* Generates embeddings for the incoming chat message to query Pinecone indexes.  
    - *Configuration:* Model: `embed-multilingual-v3.0`.  
    - *Credentials:* Cohere API key.  
    - *Input/Output:* Input: chat message; output: embedding vector.  
    - *Potential Failures:* API limits, invalid key.

  - **Simple Memory**  
    - *Type & Role:* Maintains a sliding window buffer memory for the AI agent conversation to preserve context.  
    - *Configuration:* Default buffer window parameters.  
    - *Input/Output:* Connects as AI memory input to AI Agent.  
    - *Potential Failures:* Memory overflow if conversation is too long without pruning.

  - **OpenAI Chat Model**  
    - *Type & Role:* GPT-4 based language model providing natural language generation for responses.  
    - *Configuration:* Model: `gpt-4.1-mini`; default options.  
    - *Credentials:* OpenAI API key.  
    - *Input/Output:* Input: prompt from AI Agent; output: generated responses.  
    - *Potential Failures:* API rate limits, invalid API key, network issues.

  - **AI Agent**  
    - *Type & Role:* Central agent orchestrating retrieval tools, memory, and language model to answer user questions.  
    - *Configuration:* System message instructs the agent to only answer based on knowledge base or emails; otherwise, respond with no information available.  
    - *Input/Output:* Inputs from chat trigger, retrieval tools, memory, and language model; outputs final answer.  
    - *Potential Failures:* Tool integration errors, memory inconsistencies.

  - **Sticky Note3**  
    - *Content:* Explains that the AI agent uses the knowledge base and emails stored in Pinecone to answer customer questions.

---

### 3. Summary Table

| Node Name                 | Node Type                                       | Functional Role                                    | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                  |
|---------------------------|------------------------------------------------|---------------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Microsoft Outlook Trigger  | n8n-nodes-base.microsoftOutlookTrigger         | Trigger new emails in specific Outlook folder     | -                                | Get many messages                 |                                                                                              |
| Get many messages          | n8n-nodes-base.microsoftOutlook                 | Retrieve all messages in the conversation         | Microsoft Outlook Trigger         | Remove Duplicates                 |                                                                                              |
| Remove Duplicates          | n8n-nodes-base.removeDuplicates                  | Remove duplicate emails by body content            | Get many messages                 | Pinecone Vector Store             |                                                                                              |
| Embeddings Cohere          | @n8n/n8n-nodes-langchain.embeddingsCohere      | Generate multilingual embeddings for emails        | Default Data Loader               | Pinecone Vector Store             |                                                                                              |
| Default Data Loader        | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Prepare email content for embedding                 | Recursive Character Text Splitter | Pinecone Vector Store             |                                                                                              |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Split email text into chunks                         | Remove Duplicates                 | Default Data Loader               |                                                                                              |
| Pinecone Vector Store      | @n8n/n8n-nodes-langchain.vectorStorePinecone   | Store email embeddings in Pinecone (emails ns)     | Embeddings Cohere                | -                                |                                                                                              |
| Sticky Note1               | n8n-nodes-base.stickyNote                        | Doc note for Outlook knowledge base                 | -                                | -                                | Outlook knowledge base: triggers on email/thread in folder, deduplicates, stores in Pinecone.|
| Notion Trigger            | n8n-nodes-base.notionTrigger                     | Trigger on new/updated Notion database page         | -                                | Get many child blocks             |                                                                                              |
| Get many child blocks      | n8n-nodes-base.notion                            | Fetch all blocks recursively from Notion page       | Notion Trigger                   | If                              |                                                                                              |
| If                        | n8n-nodes-base.if                                | Check if block is synced_block                        | Get many child blocks             | Get many child blocks1 (true), Merge (false) |                                                                                              |
| Get many child blocks1     | n8n-nodes-base.notion                            | Fetch blocks from synced block source                | If (true path)                   | Merge                           |                                                                                              |
| Merge                     | n8n-nodes-base.merge                             | Combine normal and synced blocks                      | If (false), Get many child blocks1 | Split Out                      |                                                                                              |
| Split Out                 | n8n-nodes-base.splitOut                          | Extract paragraph.text from merged blocks             | Merge                           | Remove Duplicates1                |                                                                                              |
| Remove Duplicates1         | n8n-nodes-base.removeDuplicates                  | Remove duplicate paragraphs by plain_text            | Split Out                      | Pinecone Vector Store1            |                                                                                              |
| Embeddings Cohere1         | @n8n/n8n-nodes-langchain.embeddingsCohere      | Generate multilingual embeddings for Notion content | Default Data Loader1             | Pinecone Vector Store1            |                                                                                              |
| Default Data Loader1       | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Prepare Notion content for embedding                  | Recursive Character Text Splitter1 | Pinecone Vector Store1            |                                                                                              |
| Recursive Character Text Splitter1 | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Split Notion text into chunks                         | Remove Duplicates1               | Default Data Loader1              |                                                                                              |
| Pinecone Vector Store1     | @n8n/n8n-nodes-langchain.vectorStorePinecone   | Store Notion embeddings in Pinecone (knowledgebase ns) | Embeddings Cohere1              | -                                |                                                                                              |
| Sticky Note4               | n8n-nodes-base.stickyNote                        | Doc note for Notion knowledge base                    | -                                | -                                | Notion knowledge base: triggers on page add, captures content, stores in Pinecone.          |
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger             | Trigger on incoming chat messages                     | -                                | AI Agent                        |                                                                                              |
| Pinecone Vector Store2     | @n8n/n8n-nodes-langchain.vectorStorePinecone   | Retrieval tool for Notion knowledge base              | Embeddings Cohere2              | AI Agent                       |                                                                                              |
| Pinecone Vector Store3     | @n8n/n8n-nodes-langchain.vectorStorePinecone   | Retrieval tool for similar emails                      | Embeddings Cohere2              | AI Agent                       |                                                                                              |
| Embeddings Cohere2         | @n8n/n8n-nodes-langchain.embeddingsCohere      | Generate embeddings for chat messages                  | When chat message received      | Pinecone Vector Store2, Pinecone Vector Store3 |                                                                                              |
| Simple Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow     | Maintain conversation context memory                   | AI Agent                       | AI Agent                       |                                                                                              |
| OpenAI Chat Model          | @n8n/n8n-nodes-langchain.lmChatOpenAi           | GPT-4 language model for generating answers           | AI Agent                       | AI Agent                       |                                                                                              |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent                   | Orchestrates retrieval, memory, and language model    | When chat message received, Pinecone Vector Store2, Pinecone Vector Store3, Simple Memory, OpenAI Chat Model | - | Customer Support Agent: uses knowledge base and emails in Pinecone to answer questions.     |
| Sticky Note3               | n8n-nodes-base.stickyNote                        | Documentation note for AI Agent                         | -                                | -                                | Customer Support Agent uses Pinecone knowledge base and emails to answer queries.            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Microsoft Outlook Trigger node**  
   - Type: Microsoft Outlook Trigger  
   - Configure OAuth2 credentials for Microsoft Outlook.  
   - Set folder to monitor by folder ID (specific Outlook folder, e.g., "knowledgebase").  
   - Enable "Download Attachments".  
   - Poll mode: every minute.

2. **Add Get many messages node**  
   - Type: Microsoft Outlook node (getAll operation)  
   - Use same Outlook OAuth2 credentials.  
   - Filter: `conversationId eq '{{ $json.conversationId }}'` (dynamic from trigger).  
   - Return all messages.  
   - Connect Microsoft Outlook Trigger → Get many messages.

3. **Add Remove Duplicates node**  
   - Type: Remove Duplicates  
   - Compare field: `body.content`.  
   - Connect Get many messages → Remove Duplicates.

4. **Add Recursive Character Text Splitter node**  
   - Type: Recursive Character Text Splitter  
   - Chunk overlap: 100 characters.  
   - Connect Remove Duplicates → Recursive Character Text Splitter.

5. **Add Default Data Loader node**  
   - Type: Document Default Data Loader  
   - JSON data expression: `={{ $json.body.content }}`.  
   - Text splitting mode: Custom.  
   - Connect Recursive Character Text Splitter → Default Data Loader.

6. **Add Embeddings Cohere node**  
   - Type: Cohere Embeddings  
   - Model: `embed-multilingual-v3.0`.  
   - Configure Cohere API credentials.  
   - Connect Default Data Loader → Embeddings Cohere.

7. **Add Pinecone Vector Store node**  
   - Type: Pinecone Vector Store (Insert Mode)  
   - Pinecone index: select your "n8n" index.  
   - Namespace: `emails`.  
   - Embedding batch size: 50.  
   - Configure Pinecone API credentials.  
   - Connect Embeddings Cohere → Pinecone Vector Store.

8. **Create Notion Trigger node**  
   - Type: Notion Trigger  
   - Configure Notion API credentials.  
   - Set database ID to your Notion database.  
   - Poll mode: every minute.

9. **Add Get many child blocks node**  
   - Type: Notion node (Get all blocks)  
   - Block ID: `={{ $json.id }}` from Notion Trigger.  
   - Return all blocks, fetch nested blocks, do not simplify output.  
   - Connect Notion Trigger → Get many child blocks.

10. **Add If node**  
    - Type: If  
    - Condition: Check if `$json.type == "synced_block"`.  
    - Connect Get many child blocks → If.

11. **Add Get many child blocks1 node**  
    - Type: Notion node (Get all blocks)  
    - Block ID: `={{ $json.synced_block.synced_from.block_id }}` from If True output.  
    - Return all blocks, fetch nested blocks.  
    - Connect If (True) → Get many child blocks1.

12. **Add Merge node**  
    - Type: Merge  
    - Merge outputs from: If (False) and Get many child blocks1.  
    - Connect If (False) → Merge (input 1).  
    - Connect Get many child blocks1 → Merge (input 2).

13. **Add Split Out node**  
    - Type: Split Out  
    - Field to split: `paragraph.text`.  
    - Connect Merge → Split Out.

14. **Add Remove Duplicates1 node**  
    - Type: Remove Duplicates  
    - Compare field: `plain_text`.  
    - Connect Split Out → Remove Duplicates1.

15. **Add Recursive Character Text Splitter1 node**  
    - Type: Recursive Character Text Splitter  
    - Chunk overlap: 100.  
    - Connect Remove Duplicates1 → Recursive Character Text Splitter1.

16. **Add Default Data Loader1 node**  
    - Type: Document Default Data Loader  
    - JSON data expression: `={{ $json.plain_text }}`.  
    - Text splitting mode: Custom.  
    - Connect Recursive Character Text Splitter1 → Default Data Loader1.

17. **Add Embeddings Cohere1 node**  
    - Type: Cohere Embeddings  
    - Model: `embed-multilingual-v3.0`.  
    - Use existing Cohere credentials.  
    - Connect Default Data Loader1 → Embeddings Cohere1.

18. **Add Pinecone Vector Store1 node**  
    - Type: Pinecone Vector Store (Insert Mode)  
    - Pinecone index: "n8n".  
    - Namespace: `knowledgebase`.  
    - Batch size: 50.  
    - Use existing Pinecone credentials.  
    - Connect Embeddings Cohere1 → Pinecone Vector Store1.

19. **Add When chat message received node**  
    - Type: LangChain Chat Trigger  
    - Configure webhook ID and parameters.  
    - Connect to AI Agent node.

20. **Add Embeddings Cohere2 node**  
    - Type: Cohere Embeddings  
    - Model: `embed-multilingual-v3.0`.  
    - Use Cohere credentials.  
    - Connect When chat message received → Embeddings Cohere2.

21. **Add Pinecone Vector Store2 node**  
    - Type: Pinecone Vector Store (Retrieve as tool)  
    - Index: "n8n"  
    - Namespace: `knowledgebase`  
    - Tool description: "use this tool to look at your knowledge base."  
    - Connect Embeddings Cohere2 → Pinecone Vector Store2.

22. **Add Pinecone Vector Store3 node**  
    - Type: Pinecone Vector Store (Retrieve as tool)  
    - Index: "n8n"  
    - Namespace: `emails`  
    - Tool description: "use this tool to look at similar emails."  
    - Connect Embeddings Cohere2 → Pinecone Vector Store3.

23. **Add Simple Memory node**  
    - Type: LangChain Memory Buffer Window  
    - Default parameters.  
    - Connect to AI Agent memory input.

24. **Add OpenAI Chat Model node**  
    - Type: LangChain Chat OpenAI  
    - Model: `gpt-4.1-mini`.  
    - Configure OpenAI credentials.  
    - Connect to AI Agent language model input.

25. **Add AI Agent node**  
    - Type: LangChain Agent  
    - System message: "You are a helpful assistant. Your role is to answer the customer questions based on the knowledge base you have access to through the tools you have. If you don't find the answer in the knowledge base or previous emails don't come up with answer just tell the customer that you don't have this information."  
    - Connect inputs: When chat message received, Pinecone Vector Store2 (knowledgebase), Pinecone Vector Store3 (emails), Simple Memory, OpenAI Chat Model.  
    - Final output is the answer to the chat message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow builds AI knowledge base from Outlook emails and Notion pages, indexed in Pinecone vector stores for GPT-4 based question answering.                                                                                            | Workflow description.                                                                                        |
| Outlook knowledge base triggers when emails or threads are moved into a dedicated folder, capturing conversation context and storing embeddings in Pinecone under "emails".                                                                | Sticky Note1 content.                                                                                        |
| Notion knowledge base triggers on page addition or update, recursively fetching page content including synced blocks, storing embeddings in Pinecone under "knowledgebase".                                                                | Sticky Note4 content.                                                                                        |
| AI Agent uses both "emails" and "knowledgebase" Pinecone namespaces as retrieval tools with GPT-4 to answer customer support questions, refusing to guess when info is missing.                                                             | Sticky Note3 content.                                                                                        |
| Cohere Embeddings model used is `embed-multilingual-v3.0` for multilingual capability.                                                                                                                                                     | Cohere node configuration.                                                                                   |
| Pinecone vector store uses index named "n8n" with namespaces "emails" and "knowledgebase" to separate data sources.                                                                                                                      | Pinecone nodes configuration.                                                                                |
| GPT-4 model variant used is `gpt-4.1-mini` for chat completion.                                                                                                                                                                           | OpenAI Chat Model node configuration.                                                                        |
| Memory buffer window node maintains conversational context for the AI agent.                                                                                                                                                              | Simple Memory node configuration.                                                                             |
| The workflow uses OAuth2 credentials for Microsoft Outlook, Notion API keys, Cohere API keys, Pinecone API keys, and OpenAI API keys; ensure all credentials are configured with appropriate scopes and permissions.                         | Credential requirements.                                                                                      |
| For detailed Pinecone usage and LangChain integration, refer to n8n official docs: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                                                                                          | n8n documentation.                                                                                            |

---

**Disclaimer:** The provided content originates exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected material. All data processed is legal and publicly accessible.