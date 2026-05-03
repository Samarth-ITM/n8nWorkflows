Smart Email Assistant: Automate Customer Support with AI & Supabase

https://n8nworkflows.xyz/workflows/smart-email-assistant--automate-customer-support-with-ai---supabase-2929


# Smart Email Assistant: Automate Customer Support with AI & Supabase

### 1. Workflow Overview

This workflow, titled **Smart Email Assistant: Automate Customer Support with AI & Supabase**, is designed to automate customer support email handling by integrating AI-driven natural language processing with a vector database for knowledge management. It targets organizations seeking to streamline email support by automatically classifying incoming emails, retrieving relevant knowledge from a document database, and generating personalized AI responses. Additionally, it manages document ingestion and indexing from Google Drive to keep the knowledge base up to date.

The workflow is logically divided into two main functional blocks:

- **1.1 Email Support System**: Monitors Gmail inbox, classifies incoming emails using AI, routes support requests to an AI response generator that leverages vector similarity search in Supabase, and drafts personalized replies in Gmail.

- **1.2 Document Management System**: Monitors Google Drive for new or updated documents, downloads and extracts text, splits text for efficient embedding, generates OpenAI embeddings, and updates the Supabase vector database to maintain an up-to-date knowledge base.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Support System

**Overview:**  
This block automates the reception and classification of incoming emails, routes support-related emails to an AI-powered response generator that uses vector similarity search for context-aware replies, and creates draft responses in Gmail.

**Nodes Involved:**  
- Email Monitor  
- AI Email Classifier  
- Route Email  
- AI Response Generator  
- OpenAI Chat Model2  
- Vector Store Tool1  
- OpenAI Chat Model3  
- Embeddings OpenAI1  
- Supabase Vector Store  
- Create Draft  

**Node Details:**

- **Email Monitor**  
  - *Type:* Gmail Trigger  
  - *Role:* Watches Gmail inbox for new incoming emails to trigger the workflow.  
  - *Configuration:* Default monitoring of inbox; OAuth2 credentials required for Gmail API.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Passes new email data to AI Email Classifier.  
  - *Edge Cases:* Gmail API quota limits, OAuth token expiration, network issues.  

- **AI Email Classifier**  
  - *Type:* OpenAI Node (LangChain)  
  - *Role:* Classifies incoming emails as either customer support or non-support inquiries using AI NLP.  
  - *Configuration:* Uses OpenAI API with prompt engineering to determine email category.  
  - *Inputs:* Email content from Email Monitor.  
  - *Outputs:* Classification result to Route Email node.  
  - *Edge Cases:* Misclassification, API rate limits, prompt failures.  

- **Route Email**  
  - *Type:* Switch  
  - *Role:* Routes emails based on classification result (support vs non-support).  
  - *Configuration:* Switch condition based on AI Email Classifier output.  
  - *Inputs:* Classification data.  
  - *Outputs:* Support emails routed to AI Response Generator; others can be ignored or handled differently.  
  - *Edge Cases:* Unexpected classification values, missing data.  

- **AI Response Generator**  
  - *Type:* LangChain Agent  
  - *Role:* Generates personalized AI-driven support responses by combining language models and vector search tools.  
  - *Configuration:* Integrates OpenAI Chat Models and Vector Store Tool for knowledge retrieval.  
  - *Inputs:* Routed support emails, AI language model outputs, vector store results.  
  - *Outputs:* Generated response text.  
  - *Edge Cases:* API failures, incomplete vector search results, latency.  
  - *Sub-workflow:* Acts as an orchestrator node combining multiple AI tools.  

- **OpenAI Chat Model2**  
  - *Type:* OpenAI Chat Model (LangChain)  
  - *Role:* Provides language model capabilities for AI Response Generator.  
  - *Configuration:* Uses OpenAI Chat API with configured parameters (model, temperature, etc.).  
  - *Inputs:* Prompts from AI Response Generator.  
  - *Outputs:* Language model responses.  
  - *Edge Cases:* API rate limits, timeouts.  

- **Vector Store Tool1**  
  - *Type:* Vector Store Tool (LangChain)  
  - *Role:* Performs similarity search in Supabase vector database to retrieve relevant knowledge snippets.  
  - *Configuration:* Connects to Supabase vector store, uses OpenAI embeddings for query conversion.  
  - *Inputs:* Query embeddings from AI Response Generator.  
  - *Outputs:* Retrieved documents or knowledge snippets.  
  - *Edge Cases:* Database connection errors, empty search results.  

- **OpenAI Chat Model3**  
  - *Type:* OpenAI Chat Model (LangChain)  
  - *Role:* Supports Vector Store Tool1 by generating embeddings or processing queries.  
  - *Configuration:* Similar to OpenAI Chat Model2, configured for vector search context.  
  - *Inputs:* Queries or prompts from Vector Store Tool1.  
  - *Outputs:* Processed embeddings or responses.  
  - *Edge Cases:* API failures.  

- **Embeddings OpenAI1**  
  - *Type:* OpenAI Embeddings (LangChain)  
  - *Role:* Generates vector embeddings for AI Response Generator inputs.  
  - *Configuration:* Uses OpenAI embedding model (e.g., text-embedding-ada-002).  
  - *Inputs:* Text data from AI Response Generator.  
  - *Outputs:* Embeddings for vector search.  
  - *Edge Cases:* API limits, malformed input text.  

- **Supabase Vector Store**  
  - *Type:* Supabase Vector Store (LangChain)  
  - *Role:* Stores and retrieves vector embeddings in Supabase database.  
  - *Configuration:* Connects to Supabase project with credentials, manages vector data.  
  - *Inputs:* Embeddings from Embeddings OpenAI1.  
  - *Outputs:* Vector search results to Vector Store Tool1.  
  - *Edge Cases:* Database connectivity, query performance.  

- **Create Draft**  
  - *Type:* Gmail Tool  
  - *Role:* Creates draft emails in Gmail with AI-generated responses.  
  - *Configuration:* Uses Gmail API with OAuth2 credentials; drafts are created but not sent automatically.  
  - *Inputs:* AI-generated response text from AI Response Generator.  
  - *Outputs:* Draft email in Gmail.  
  - *Edge Cases:* Gmail API quota, OAuth token expiration, draft creation failures.  

---

#### 2.2 Document Management System

**Overview:**  
This block monitors Google Drive for new or updated documents, downloads and extracts their text content, splits text into manageable chunks, generates vector embeddings, and updates the Supabase vector database to maintain an accurate and searchable knowledge base.

**Nodes Involved:**  
- File Created  
- File Updated  
- Set File ID  
- Delete Old Doc Rows  
- Download File  
- Extract Document Text  
- Insert into Supabase Vectorstore  
- Embeddings OpenAI  
- Default Data Loader  
- Recursive Character Text Splitter  

**Node Details:**

- **File Created**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches for new files added to monitored Google Drive folders.  
  - *Configuration:* Monitors specific folder IDs configured in Google Drive API credentials.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Passes new file metadata to Set File ID.  
  - *Edge Cases:* API quota, folder permission issues.  

- **File Updated**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches for updates to existing files in monitored folders.  
  - *Configuration:* Same as File Created, but triggers on file updates.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Passes updated file metadata to Set File ID.  
  - *Edge Cases:* Same as File Created.  

- **Set File ID**  
  - *Type:* Set  
  - *Role:* Extracts and sets the file ID from trigger data for downstream processing.  
  - *Configuration:* Maps file metadata fields to a standardized variable for use in subsequent nodes.  
  - *Inputs:* File metadata from File Created or File Updated.  
  - *Outputs:* File ID to Delete Old Doc Rows.  
  - *Edge Cases:* Missing or malformed file metadata.  

- **Delete Old Doc Rows**  
  - *Type:* Supabase Node  
  - *Role:* Deletes existing vector entries in Supabase related to the updated or new file to avoid duplication.  
  - *Configuration:* Executes a delete query filtering by file ID or metadata.  
  - *Inputs:* File ID from Set File ID.  
  - *Outputs:* Confirmation to Download File.  
  - *Edge Cases:* Database connectivity, permission errors, query failures.  

- **Download File**  
  - *Type:* Google Drive  
  - *Role:* Downloads the actual file content from Google Drive for processing.  
  - *Configuration:* Uses file ID to fetch file; supports various file formats.  
  - *Inputs:* Confirmation from Delete Old Doc Rows.  
  - *Outputs:* File binary data to Extract Document Text.  
  - *Edge Cases:* File access permissions, large file size, network errors.  

- **Extract Document Text**  
  - *Type:* Extract From File  
  - *Role:* Extracts text content from the downloaded file binary.  
  - *Configuration:* Supports multiple file types (PDF, DOCX, TXT, etc.); configured for always outputting data.  
  - *Inputs:* File binary from Download File.  
  - *Outputs:* Extracted text to Insert into Supabase Vectorstore.  
  - *Edge Cases:* Unsupported file formats, extraction errors, corrupted files.  

- **Insert into Supabase Vectorstore**  
  - *Type:* Supabase Vector Store (LangChain)  
  - *Role:* Inserts processed document chunks and their embeddings into the Supabase vector database.  
  - *Configuration:* Accepts document text chunks and embeddings; associates metadata including file ID.  
  - *Inputs:* Document chunks and embeddings from Default Data Loader and Embeddings OpenAI.  
  - *Outputs:* Confirmation of insertion.  
  - *Edge Cases:* Database insertion errors, duplicate entries, connectivity issues.  

- **Embeddings OpenAI**  
  - *Type:* OpenAI Embeddings (LangChain)  
  - *Role:* Generates vector embeddings for document text chunks.  
  - *Configuration:* Uses OpenAI embedding model; configured for batch processing of text chunks.  
  - *Inputs:* Text chunks from Recursive Character Text Splitter.  
  - *Outputs:* Embeddings to Insert into Supabase Vectorstore.  
  - *Edge Cases:* API rate limits, malformed text input.  

- **Default Data Loader**  
  - *Type:* Document Default Data Loader (LangChain)  
  - *Role:* Loads and prepares document chunks for embedding and storage.  
  - *Configuration:* Receives text chunks from Recursive Character Text Splitter; formats data for vector store insertion.  
  - *Inputs:* Text chunks from Recursive Character Text Splitter.  
  - *Outputs:* Document chunks to Insert into Supabase Vectorstore.  
  - *Edge Cases:* Data formatting errors.  

- **Recursive Character Text Splitter**  
  - *Type:* Recursive Character Text Splitter (LangChain)  
  - *Role:* Splits extracted document text into smaller chunks with overlap for better embedding quality.  
  - *Configuration:* Configured chunk size and overlap parameters for optimal indexing.  
  - *Inputs:* Extracted text from Extract Document Text.  
  - *Outputs:* Text chunks to Default Data Loader and Embeddings OpenAI.  
  - *Edge Cases:* Improper chunking causing loss of context or too large chunks.  

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                                | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                      |
|-------------------------------|--------------------------------------|------------------------------------------------|-------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Email Monitor                 | Gmail Trigger                        | Monitors Gmail inbox for new emails            | None                          | AI Email Classifier             |                                                                                                 |
| AI Email Classifier           | OpenAI (LangChain)                   | Classifies emails as support or non-support    | Email Monitor                 | Route Email                    | Uses AI to classify incoming emails as customer support or non-support                          |
| Route Email                  | Switch                              | Routes emails based on classification          | AI Email Classifier           | AI Response Generator          | Routes emails based on AI classification results                                                |
| AI Response Generator         | LangChain Agent                     | Generates AI-driven personalized responses     | Route Email                   | Create Draft                  | Generates personalized support responses using AI                                              |
| OpenAI Chat Model2            | OpenAI Chat Model (LangChain)       | Provides language model for response generation| AI Response Generator         | AI Response Generator          |                                                                                                 |
| Vector Store Tool1            | Vector Store Tool (LangChain)        | Performs vector similarity search               | Supabase Vector Store, OpenAI Chat Model3 | AI Response Generator          |                                                                                                 |
| OpenAI Chat Model3            | OpenAI Chat Model (LangChain)       | Supports vector search with embeddings         | Vector Store Tool1            | Vector Store Tool1             |                                                                                                 |
| Embeddings OpenAI1            | OpenAI Embeddings (LangChain)        | Generates embeddings for AI response inputs    | AI Response Generator         | Supabase Vector Store          |                                                                                                 |
| Supabase Vector Store         | Supabase Vector Store (LangChain)    | Stores and retrieves vector embeddings          | Embeddings OpenAI1            | Vector Store Tool1             |                                                                                                 |
| Create Draft                 | Gmail Tool                         | Creates draft email in Gmail with AI response  | AI Response Generator         | None                         |                                                                                                 |
| File Created                 | Google Drive Trigger                | Triggers on new files in Google Drive           | None                          | Set File ID                   |                                                                                                 |
| File Updated                 | Google Drive Trigger                | Triggers on updated files in Google Drive       | None                          | Set File ID                   |                                                                                                 |
| Set File ID                  | Set                                | Extracts file ID for downstream processing      | File Created, File Updated    | Delete Old Doc Rows           |                                                                                                 |
| Delete Old Doc Rows          | Supabase Node                      | Deletes old vector entries for updated files    | Set File ID                   | Download File                 |                                                                                                 |
| Download File                | Google Drive                       | Downloads file content for processing            | Delete Old Doc Rows           | Extract Document Text         |                                                                                                 |
| Extract Document Text        | Extract From File                  | Extracts text from downloaded files              | Download File                 | Recursive Character Text Splitter, Insert into Supabase Vectorstore |                                                                                                 |
| Recursive Character Text Splitter | Recursive Character Text Splitter (LangChain) | Splits text into chunks for embedding            | Extract Document Text         | Default Data Loader, Embeddings OpenAI |                                                                                                 |
| Default Data Loader          | Document Default Data Loader (LangChain) | Prepares document chunks for vector store        | Recursive Character Text Splitter | Insert into Supabase Vectorstore |                                                                                                 |
| Embeddings OpenAI            | OpenAI Embeddings (LangChain)        | Generates embeddings for document chunks         | Recursive Character Text Splitter | Insert into Supabase Vectorstore |                                                                                                 |
| Insert into Supabase Vectorstore | Supabase Vector Store (LangChain)    | Inserts document chunks and embeddings into DB  | Default Data Loader, Embeddings OpenAI, Extract Document Text | None                         |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node ("Email Monitor")**  
   - Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail API.  
   - Set to monitor inbox for new emails.  

2. **Add OpenAI Node ("AI Email Classifier")**  
   - Type: OpenAI (LangChain)  
   - Configure with OpenAI API key.  
   - Set prompt to classify emails as support or non-support.  
   - Connect output of "Email Monitor" to this node.  

3. **Add Switch Node ("Route Email")**  
   - Type: Switch  
   - Configure condition based on classification output from "AI Email Classifier".  
   - Route support emails to next block; optionally handle others differently.  
   - Connect "AI Email Classifier" output to this node.  

4. **Add LangChain Agent Node ("AI Response Generator")**  
   - Type: LangChain Agent  
   - Configure to integrate OpenAI Chat Models and Vector Store Tool.  
   - Connect "Route Email" support output to this node.  

5. **Add OpenAI Chat Model Node ("OpenAI Chat Model2")**  
   - Type: OpenAI Chat Model (LangChain)  
   - Configure with OpenAI API key and desired model parameters.  
   - Connect to "AI Response Generator" as language model input.  

6. **Add OpenAI Chat Model Node ("OpenAI Chat Model3")**  
   - Type: OpenAI Chat Model (LangChain)  
   - Configure similarly for vector search context.  
   - Connect to "Vector Store Tool1".  

7. **Add Vector Store Tool Node ("Vector Store Tool1")**  
   - Type: Vector Store Tool (LangChain)  
   - Configure to connect to Supabase vector store.  
   - Connect "OpenAI Chat Model3" output to this node.  
   - Connect output to "AI Response Generator" as tool input.  

8. **Add OpenAI Embeddings Node ("Embeddings OpenAI1")**  
   - Type: OpenAI Embeddings (LangChain)  
   - Configure with OpenAI API key and embedding model.  
   - Connect "AI Response Generator" output to this node.  

9. **Add Supabase Vector Store Node ("Supabase Vector Store")**  
   - Type: Supabase Vector Store (LangChain)  
   - Configure with Supabase project URL and API key.  
   - Connect "Embeddings OpenAI1" output to this node.  
   - Connect output to "Vector Store Tool1".  

10. **Add Gmail Tool Node ("Create Draft")**  
    - Type: Gmail Tool  
    - Configure OAuth2 credentials for Gmail API.  
    - Connect "AI Response Generator" output to this node to create draft emails.  

---

11. **Create Google Drive Trigger Node ("File Created")**  
    - Type: Google Drive Trigger  
    - Configure OAuth2 credentials for Google Drive API.  
    - Set to trigger on new files in monitored folders.  

12. **Create Google Drive Trigger Node ("File Updated")**  
    - Type: Google Drive Trigger  
    - Configure similarly for file updates.  

13. **Add Set Node ("Set File ID")**  
    - Type: Set  
    - Extract file ID from trigger data.  
    - Connect outputs of "File Created" and "File Updated" to this node.  

14. **Add Supabase Node ("Delete Old Doc Rows")**  
    - Type: Supabase  
    - Configure with Supabase credentials.  
    - Set to delete vector entries matching file ID.  
    - Connect "Set File ID" output to this node.  

15. **Add Google Drive Node ("Download File")**  
    - Type: Google Drive  
    - Configure with OAuth2 credentials.  
    - Use file ID from "Delete Old Doc Rows" to download file.  
    - Connect "Delete Old Doc Rows" output to this node.  

16. **Add Extract From File Node ("Extract Document Text")**  
    - Type: Extract From File  
    - Configure supported file types.  
    - Connect "Download File" output to this node.  

17. **Add Recursive Character Text Splitter Node ("Recursive Character Text Splitter")**  
    - Type: Recursive Character Text Splitter (LangChain)  
    - Configure chunk size and overlap parameters.  
    - Connect "Extract Document Text" output to this node.  

18. **Add Document Default Data Loader Node ("Default Data Loader")**  
    - Type: Document Default Data Loader (LangChain)  
    - Connect "Recursive Character Text Splitter" output to this node.  

19. **Add OpenAI Embeddings Node ("Embeddings OpenAI")**  
    - Type: OpenAI Embeddings (LangChain)  
    - Configure with OpenAI API key and embedding model.  
    - Connect "Recursive Character Text Splitter" output to this node.  

20. **Add Supabase Vector Store Node ("Insert into Supabase Vectorstore")**  
    - Type: Supabase Vector Store (LangChain)  
    - Configure with Supabase credentials.  
    - Connect outputs of "Default Data Loader", "Embeddings OpenAI", and "Extract Document Text" to this node.  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Supabase vector extension and table creation SQL provided in setup instructions.                 | See section "2️⃣ Supabase Database Setup" in workflow description.                             |
| Google Drive folders must be configured with correct permissions and folder IDs added to workflow.| See section "3️⃣ Google Drive Setup" in workflow description.                                 |
| OAuth2 credentials required for Gmail and Google Drive API integration.                          | Setup instructions in workflow prerequisites.                                                 |
| Regular maintenance recommended: monitor AI classification accuracy, update knowledge base, optimize vector search parameters. | See "🔍 Maintenance & Optimization" section.                                                  |
| Troubleshooting tips include verifying API credentials, monitoring AI service uptime, and checking permissions. | See "🛠️ Troubleshooting" section.                                                             |

---

This documentation provides a comprehensive understanding of the workflow’s structure and logic, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.