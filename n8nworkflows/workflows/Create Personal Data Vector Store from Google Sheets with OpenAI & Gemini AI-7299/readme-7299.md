Create Personal Data Vector Store from Google Sheets with OpenAI & Gemini AI

https://n8nworkflows.xyz/workflows/create-personal-data-vector-store-from-google-sheets-with-openai---gemini-ai-7299


# Create Personal Data Vector Store from Google Sheets with OpenAI & Gemini AI

---

### 1. Workflow Overview

This workflow, titled **"Create Personal Data Vector Store from Google Sheets with OpenAI & Gemini AI"**, automates the process of ingesting personal data from a Google Sheets spreadsheet, embedding this data into a vector store hosted on Supabase, and enabling an AI agent to answer queries about the stored personal information using Google Gemini Chat and OpenAI embeddings.

The workflow is logically divided into two main blocks:

- **1.1 Data Ingestion & Vector Store Creation:**  
  Retrieves rows from a Google Sheet, converts the data into a file format, loads it as documents, generates embeddings using OpenAI, and inserts these embeddings into a Supabase vector database.

- **1.2 AI Agent Query Handling:**  
  Listens for incoming chat messages via a LangChain chat trigger, processes queries using an AI agent configured with Google Gemini Chat for language understanding, OpenAI embeddings for vector similarity search, and Postgres for chat memory. The agent retrieves relevant personal data from the Supabase vector store to inform its responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Vector Store Creation

- **Overview:**  
  This block fetches personal data from a specific Google Sheets spreadsheet, processes it into a suitable document format, generates vector embeddings for semantic search, and inserts these embeddings into a Supabase vector store. This allows later retrieval of personal data based on semantic similarity.

- **Nodes Involved:**  
  - Get row(s) in sheet1  
  - Convert to File  
  - Default Data Loader2  
  - Embeddings OpenAI  
  - Vector Store  
  - Supabase Vector Store  
  - Sticky Note (Vector Store Load :file_folder:)

- **Node Details:**

  1. **Get row(s) in sheet1**  
     - *Type:* Google Sheets node (v4.6)  
     - *Role:* Fetches all rows from the first sheet ("Birthday") of the Google Sheets document identified by the provided document ID.  
     - *Configuration:* Uses OAuth2 credentials for Google Sheets API access. Retrieves data from sheet with gid=0.  
     - *Inputs:* None (start node for data ingestion).  
     - *Outputs:* Rows of personal data from the sheet.  
     - *Failures:* Possible issues include authentication errors, API rate limits, spreadsheet access permissions, or empty/invalid sheet data.

  2. **Convert to File**  
     - *Type:* Convert to File node (v1.1)  
     - *Role:* Converts the Google Sheets row data into a file format suitable for the next data loader node.  
     - *Configuration:* Default conversion options applied.  
     - *Inputs:* Data from "Get row(s) in sheet1".  
     - *Outputs:* Binary file data representing the spreadsheet content.  
     - *Failures:* Conversion errors if input data malformed or empty.

  3. **Default Data Loader2**  
     - *Type:* LangChain Document Default Data Loader node (v1.1)  
     - *Role:* Loads the binary CSV file into LangChain documents for embedding generation.  
     - *Configuration:* Uses "csvLoader" indicating CSV format input, dataType set to binary.  
     - *Inputs:* Binary file from "Convert to File".  
     - *Outputs:* Array of document objects representing the spreadsheet content.  
     - *Failures:* Format mismatches, corrupted file, or loader errors.

  4. **Embeddings OpenAI**  
     - *Type:* LangChain OpenAI Embeddings node (v1.2)  
     - *Role:* Generates vector embeddings for the loaded documents using OpenAI API.  
     - *Configuration:* Uses OpenAI API credentials; default or empty options.  
     - *Inputs:* Documents from "Default Data Loader2".  
     - *Outputs:* Document embeddings ready for insertion.  
     - *Failures:* API authentication errors, rate limits, or network issues.

  5. **Vector Store**  
     - *Type:* LangChain Supabase Vector Store node (v1.3)  
     - *Role:* Inserts the generated embeddings into the Supabase vector store table named "personal_data".  
     - *Configuration:* Insert mode enabled, query name "match_personal_data" specified for later retrieval. Uses Supabase API credentials.  
     - *Inputs:* Embeddings from "Embeddings OpenAI". Documents metadata also passed.  
     - *Outputs:* Confirmation of insertion or errors.  
     - *Failures:* Database connectivity, authentication, table schema mismatches, or insertion errors.

  6. **Supabase Vector Store**  
     - *Type:* LangChain Supabase Vector Store node (v1.3)  
     - *Role:* Also configured but used in the query block as a tool for retrieval. Here, it is connected downstream and flagged with embedding input from another embeddings node (Embeddings OpenAI1).  
     - *Note:* This node is more relevant in the AI Agent block but appears connected here due to embedding generation.  

  7. **Embeddings OpenAI1**  
     - *Type:* LangChain OpenAI Embeddings node (v1.2)  
     - *Role:* Generates embeddings likely for query retrieval use, connected to the Supabase Vector Store node for searching.  
     - *Inputs:* Not directly from this block’s flow; used in query flow.

  8. **Sticky Note (Vector Store Load :file_folder:)**  
     - *Content:* "Insert Personal Data to Vector Store"  
     - *Role:* Visual annotation indicating the data ingestion and vector store creation block.  

---

#### 2.2 AI Agent Query Handling

- **Overview:**  
  This block listens for incoming chat messages, uses a LangChain AI Agent to process the input with memory support and vector search over the personal data vector store, and responds by leveraging Google Gemini Chat Model as the language model.

- **Nodes Involved:**  
  - When chat message received  
  - AI Agent  
  - Google Gemini Chat Model  
  - Postgres Chat Memory  
  - Supabase Vector Store  
  - Embeddings OpenAI1  
  - Sticky Note1 (Agent :information_desk_person:)

- **Node Details:**

  1. **When chat message received**  
     - *Type:* LangChain Chat Trigger node (v1.1)  
     - *Role:* Acts as the webhook entry point, triggering the workflow when a new chat message arrives.  
     - *Configuration:* Default options; webhook ID assigned.  
     - *Inputs:* External chat message via webhook.  
     - *Outputs:* Chat message payload forwarded to the AI Agent.  
     - *Failures:* Webhook misconfiguration, network errors, malformed messages.

  2. **AI Agent**  
     - *Type:* LangChain Agent node (v2.1)  
     - *Role:* Central intelligent agent processing chat messages, managing tools, memory, and language model interaction.  
     - *Configuration:* Default options; connected to language model, memory, and tools nodes.  
     - *Inputs:* Chat message from "When chat message received", memory data, tool outputs.  
     - *Outputs:* Agent responses to chat messages.  
     - *Failures:* Agent internal errors, model API errors, memory or tool integration failures.

  3. **Google Gemini Chat Model**  
     - *Type:* LangChain Google Gemini Chat Model node (v1)  
     - *Role:* Provides advanced language understanding and response generation using Google PaLM API.  
     - *Configuration:* Uses Google Palm API credentials (Google Gemini).  
     - *Inputs:* Chat prompt from AI Agent.  
     - *Outputs:* Generated language model responses.  
     - *Failures:* API authentication, quota limits, network errors.

  4. **Postgres Chat Memory**  
     - *Type:* LangChain Postgres Chat Memory node (v1.3)  
     - *Role:* Stores and retrieves chat history to maintain conversational context.  
     - *Configuration:* Uses Postgres database credentials.  
     - *Inputs:* Chat messages and agent responses.  
     - *Outputs:* Contextual memory for the AI Agent.  
     - *Failures:* Database connectivity, query errors, data corruption.

  5. **Supabase Vector Store**  
     - *Type:* LangChain Supabase Vector Store node (v1.3)  
     - *Role:* Acts as a retrieval tool for the AI Agent to semantically search personal data embeddings during queries.  
     - *Configuration:* Retrieval mode enabled with query name "match_personal_data", uses Supabase API credentials.  
     - *Inputs:* Embeddings from "Embeddings OpenAI1" for similarity search.  
     - *Outputs:* Matching personal data entries for the AI Agent.  
     - *Failures:* Vector search failures, API errors, authentication issues.

  6. **Embeddings OpenAI1**  
     - *Type:* LangChain OpenAI Embeddings node (v1.2)  
     - *Role:* Generates embeddings for incoming queries that the Supabase vector store uses to find relevant personal data.  
     - *Configuration:* Uses OpenAI API credentials.  
     - *Inputs:* Query text from AI Agent.  
     - *Outputs:* Query embeddings for vector search.  
     - *Failures:* API issues, rate limits, invalid inputs.

  7. **Sticky Note1 (Agent :information_desk_person:)**  
     - *Content:* "Agent can answer any personal information on Vector Store"  
     - *Role:* Visual annotation indicating the agent’s capability to answer queries based on the vector store.

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                           | Input Node(s)           | Output Node(s)            | Sticky Note                                  |
|-----------------------------|-----------------------------------------|-----------------------------------------|-------------------------|---------------------------|----------------------------------------------|
| When chat message received   | LangChain Chat Trigger (v1.1)           | Entry point for chat messages            | None                    | AI Agent                  | Agent can answer any personal information on Vector Store |
| AI Agent                    | LangChain Agent (v2.1)                   | Core agent processing queries            | When chat message received, Postgres Chat Memory, Supabase Vector Store, Google Gemini Chat Model | None                      | Agent can answer any personal information on Vector Store |
| Google Gemini Chat Model    | LangChain Google Gemini Chat Model (v1) | Language model for response generation   | AI Agent                | AI Agent                  | Agent can answer any personal information on Vector Store |
| Postgres Chat Memory        | LangChain Postgres Chat Memory (v1.3)   | Maintains conversational memory          | AI Agent                | AI Agent                  | Agent can answer any personal information on Vector Store |
| Supabase Vector Store       | LangChain Supabase Vector Store (v1.3)  | Vector store retrieval for personal data | Embeddings OpenAI1      | AI Agent                  | Agent can answer any personal information on Vector Store |
| Embeddings OpenAI1          | LangChain OpenAI Embeddings (v1.2)      | Embeddings for query vectors              | AI Agent                | Supabase Vector Store     | Agent can answer any personal information on Vector Store |
| Get row(s) in sheet1         | Google Sheets (v4.6)                     | Fetch personal data from Google Sheets    | None                    | Convert to File           | Insert Personal Data to Vector Store          |
| Convert to File             | Convert to File (v1.1)                   | Converts sheet data to file format        | Get row(s) in sheet1    | Default Data Loader2      | Insert Personal Data to Vector Store          |
| Default Data Loader2        | LangChain Document Default Data Loader (v1.1) | Loads file as documents for embedding      | Convert to File         | Embeddings OpenAI         | Insert Personal Data to Vector Store          |
| Embeddings OpenAI           | LangChain OpenAI Embeddings (v1.2)      | Generates embeddings for documents        | Default Data Loader2    | Vector Store              | Insert Personal Data to Vector Store          |
| Vector Store               | LangChain Supabase Vector Store (v1.3)  | Inserts embeddings into Supabase vector DB | Embeddings OpenAI       | None                     | Insert Personal Data to Vector Store          |
| Supabase Vector Store       | LangChain Supabase Vector Store (v1.3)  | Used as retrieval tool in AI Agent block  | Embeddings OpenAI1      | AI Agent                  | Agent can answer any personal information on Vector Store |
| Embeddings OpenAI1          | LangChain OpenAI Embeddings (v1.2)      | Embeddings for retrieval queries          | AI Agent                | Supabase Vector Store     | Agent can answer any personal information on Vector Store |
| Sticky Note                 | Sticky Note                             | Annotation                               | None                    | None                      | Vector Store Load :file_folder: Insert Personal Data to Vector Store |
| Sticky Note1                | Sticky Note                             | Annotation                               | None                    | None                      | Agent :information_desk_person: Agent can answer any personal information on Vector Store |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheets Node ("Get row(s) in sheet1")**  
   - Node Type: Google Sheets (v4.6)  
   - Configure OAuth2 credentials for Google Sheets API.  
   - Set Document ID to your Google Sheets document containing personal data.  
   - Set Sheet Name to the first sheet (gid=0 or sheet named "Birthday").  
   - Fetch all rows.

2. **Add "Convert to File" Node**  
   - Node Type: Convert to File (v1.1)  
   - Connect input from "Get row(s) in sheet1".  
   - Use default conversion options to convert rows to a CSV file.

3. **Add "Default Data Loader2" Node**  
   - Node Type: LangChain Document Default Data Loader (v1.1)  
   - Connect input from "Convert to File".  
   - Set Loader to "csvLoader".  
   - Set Data Type to "binary".

4. **Add "Embeddings OpenAI" Node**  
   - Node Type: LangChain OpenAI Embeddings (v1.2)  
   - Connect input from "Default Data Loader2".  
   - Configure OpenAI API credentials.  
   - Use default options for embedding generation.

5. **Add "Vector Store" Node**  
   - Node Type: LangChain Supabase Vector Store (v1.3)  
   - Connect input from "Embeddings OpenAI".  
   - Set Mode to "insert".  
   - Set Table Name to "personal_data" (must exist in Supabase).  
   - Set Query Name to "match_personal_data" for retrieval queries.  
   - Configure Supabase API credentials.

6. **Add "When chat message received" Node**  
   - Node Type: LangChain Chat Trigger (v1.1)  
   - Configure webhook URL for receiving chat messages.

7. **Add "Google Gemini Chat Model" Node**  
   - Node Type: LangChain Google Gemini Chat Model (v1)  
   - Configure Google PaLM API credentials.  
   - Connect to the AI Agent node later.

8. **Add "Embeddings OpenAI1" Node**  
   - Node Type: LangChain OpenAI Embeddings (v1.2)  
   - Configure OpenAI API credentials.  
   - Use this for query embedding generation.

9. **Add "Supabase Vector Store" Node (for retrieval)**  
   - Node Type: LangChain Supabase Vector Store (v1.3)  
   - Set Mode to "retrieve-as-tool".  
   - Set Table Name to "personal_data".  
   - Set Query Name to "match_personal_data".  
   - Configure Supabase API credentials.  
   - Connect embedding input from "Embeddings OpenAI1".

10. **Add "Postgres Chat Memory" Node**  
    - Node Type: LangChain Postgres Chat Memory (v1.3)  
    - Configure Postgres database credentials for chat history storage.

11. **Add "AI Agent" Node**  
    - Node Type: LangChain Agent (v2.1)  
    - Connect main input from "When chat message received".  
    - Connect ai_languageModel input to "Google Gemini Chat Model".  
    - Connect ai_embedding input to "Embeddings OpenAI1".  
    - Connect ai_tool input to "Supabase Vector Store".  
    - Connect ai_memory input to "Postgres Chat Memory".  
    - Use default agent options.

12. **Connect all nodes according to the described flow:**  
    - Data Ingestion Flow: Get row(s) in sheet1 → Convert to File → Default Data Loader2 → Embeddings OpenAI → Vector Store  
    - Query Handling Flow: When chat message received → AI Agent → Google Gemini Chat Model, Postgres Chat Memory, Supabase Vector Store (with Embeddings OpenAI1)  

13. **Add Sticky Notes** (optional but recommended for clarity):  
    - Over Data Ingestion block: "Insert Personal Data to Vector Store"  
    - Over AI Agent block: "Agent can answer any personal information on Vector Store"

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow uses both OpenAI and Google Gemini (PaLM API) for embeddings and language model. | Credentials for both APIs must be correctly configured with sufficient quota and permissions.             |
| Supabase is used as the vector store backend; ensure Supabase table "personal_data" is set up. | Supabase project must have the vector extension enabled and proper API access configured.                 |
| Postgres is used for chat memory persistence to maintain conversational context.               | Postgres database must be accessible and schema compatible with LangChain memory requirements.           |
| Google Sheets OAuth2 credentials must have read access to the target spreadsheet.               | Spreadsheet permissions should allow API access for the account used in credentials.                       |
| Sticky notes visually cluster nodes for easier understanding in the editor UI.                 | Useful for team collaboration and workflow documentation within n8n.                                     |

---

**Disclaimer:**  
The provided text exclusively derives from an automated workflow created in n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---