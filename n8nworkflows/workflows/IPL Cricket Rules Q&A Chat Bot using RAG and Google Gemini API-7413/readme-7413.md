IPL Cricket Rules Q&A Chat Bot using RAG and Google Gemini API

https://n8nworkflows.xyz/workflows/ipl-cricket-rules-q-a-chat-bot-using-rag-and-google-gemini-api-7413


# IPL Cricket Rules Q&A Chat Bot using RAG and Google Gemini API

### 1. Workflow Overview

This workflow implements an **IPL Cricket Rules Q&A Chat Bot** leveraging Retrieval-Augmented Generation (RAG) and Google Gemini API. It enables users to ask questions related to IPL and international cricket rules and receive accurate answers grounded strictly on the official IPL “Match Playing Conditions” document.

The workflow is structured into two broad logical blocks:

- **1.1 Reference Data Loading and Vector Store Creation**  
  This block is responsible for fetching the IPL official rulebook PDF, extracting and splitting its text content, generating embeddings using Google Gemini, and storing these embeddings in an in-memory vector store. This prepares the knowledge base for retrieval.

- **1.2 Chat Interface and AI Agent Processing**  
  This block handles receiving user chat messages, managing conversational memory, retrieving relevant rule chunks from the vector store using the user query, and orchestrating the AI agent with Google Gemini to generate answers strictly based on the retrieved data. It ensures context-aware, accurate Q&A.

---

### 2. Block-by-Block Analysis

#### 2.1 Reference Data Loading and Vector Store Creation

##### Overview  
This block runs once (or as needed) to prepare the knowledge repository by downloading the IPL playing conditions PDF, extracting text, splitting it into manageable chunks, embedding these chunks with Google Gemini embeddings, and inserting vectors into an in-memory vector store for future retrieval.

##### Nodes Involved  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- HTTP Request  
- Default Data Loader  
- Recursive Character Text Splitter  
- Embeddings Google Gemini1  
- Simple Vector Store1  

##### Node Details

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Initiates the data loading process manually.  
  - *Input/Output:* No input; triggers the HTTP Request node.  
  - *Potential Failures:* Workflow not triggered; user error.

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the IPL “Match Playing Conditions” PDF from a fixed URL.  
  - *Configuration:* URL set to IPL official PDF; no authentication or special headers.  
  - *Input:* Triggered by Manual Trigger  
  - *Output:* Binary PDF data to Default Data Loader  
  - *Edge Cases:* Network failures, URL unreachable, PDF format change.

- **Default Data Loader**  
  - *Type:* Document Default Data Loader  
  - *Role:* Extracts text content from the binary PDF data.  
  - *Configuration:* Reads binary input, extracts text; uses custom text splitting mode.  
  - *Input:* PDF binary data from HTTP Request  
  - *Output:* Extracted text to Recursive Character Text Splitter  
  - *Edge Cases:* PDF parsing errors, corrupted PDF.

- **Recursive Character Text Splitter**  
  - *Type:* Text Splitter (Recursive Character)  
  - *Role:* Splits the extracted text into overlapping chunks for embedding.  
  - *Configuration:* Chunk overlap set to 200 characters to reduce hallucination risk; chunk size defaults apply.  
  - *Input:* Extracted text from Default Data Loader  
  - *Output:* Text chunks to Embeddings Google Gemini1  
  - *Edge Cases:* Poor splitting causing irrelevant chunks, very large documents causing performance issues.

- **Embeddings Google Gemini1**  
  - *Type:* Google Gemini Embeddings  
  - *Role:* Converts text chunks into vector embeddings using Google Gemini API.  
  - *Configuration:* Uses Google Gemini API credentials; embedding parameters default.  
  - *Input:* Text chunks from Text Splitter  
  - *Output:* Embeddings to Simple Vector Store1  
  - *Edge Cases:* API authentication errors, quota limits, network timeout.

- **Simple Vector Store1**  
  - *Type:* In-Memory Vector Store (Insert mode)  
  - *Role:* Inserts generated embeddings into an in-memory vector store under a specific key.  
  - *Configuration:* Vector store key explicitly defined (must match retrieval step key).  
  - *Input:* Embeddings from Embeddings Google Gemini1  
  - *Output:* None (stores vectors internally)  
  - *Edge Cases:* Memory limits, key mismatches causing retrieval failures later.

---

#### 2.2 Chat Interface and AI Agent Processing

##### Overview  
This block enables the chat interface to receive user queries, maintains contextual conversation memory, retrieves relevant cricket rule chunks from the vector store, and uses the AI agent with Google Gemini to generate precise answers based solely on retrieved data.

##### Nodes Involved  
- When chat message received  
- AI Agent  
- Simple Memory  
- Simple Vector Store  
- Google Gemini Chat Model  
- Embeddings Google Gemini  

##### Node Details

- **When chat message received**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Native n8n chat interface trigger that receives user messages to start processing.  
  - *Configuration:* Default settings; webhook based for chat messages.  
  - *Input:* Chat messages from user interface  
  - *Output:* Triggers AI Agent node  
  - *Edge Cases:* Webhook failures, message format errors.

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Orchestrates the entire chat response generation pipeline.  
  - *Configuration:*  
    - System message defines it as a cricket expert who strictly answers using provided database info.  
    - Uses Simple Memory for conversational history (last 20 turns).  
    - Accesses Simple Vector Store in retrieval-as-tool mode for RAG.  
    - Uses Google Gemini Chat Model for generating final answers.  
  - *Input:* User query from chat trigger, memory context, and vector store tool output  
  - *Output:* Generated chat response  
  - *Edge Cases:* Model API errors, memory overflow, inconsistent context, unavailable vector store data.

- **Simple Memory**  
  - *Type:* Memory Buffer Window  
  - *Role:* Maintains last 20 chat turns for conversational context.  
  - *Configuration:* Context window length set to 20 messages.  
  - *Input:* Conversation data from AI Agent  
  - *Output:* Provides memory context back to AI Agent  
  - *Edge Cases:* Memory overflow, lost context on restart.

- **Simple Vector Store**  
  - *Type:* In-Memory Vector Store (Retrieve-as-tool mode)  
  - *Role:* Retrieves top-10 relevant document chunks matching user query embeddings.  
  - *Configuration:*  
    - Mode set to “retrieve-as-tool”  
    - Top K results = 10  
    - Uses same vector store key as insertion phase (Simple Vector Store1)  
    - Tool description clarifies it stores IPL and international cricket rules for grounding.  
  - *Input:* User query embeddings from Embeddings Google Gemini  
  - *Output:* Relevant document chunks as tool output to AI Agent  
  - *Edge Cases:* No results found, key mismatch, stale data.

- **Google Gemini Chat Model**  
  - *Type:* Langchain Google Gemini Chat Model (LLM)  
  - *Role:* Generates language model output for the AI Agent during chat.  
  - *Configuration:*  
    - Top P (nucleus sampling) set to 0.3 for focused, precise generation.  
    - Uses Google Gemini API credentials.  
  - *Input:* Prompts from AI Agent  
  - *Output:* Text responses to AI Agent  
  - *Edge Cases:* API authentication failures, rate limits, model latency.

- **Embeddings Google Gemini**  
  - *Type:* Google Gemini Embeddings  
  - *Role:* Converts user chat queries into embeddings for vector search.  
  - *Configuration:* Uses same Google Gemini API credentials as embedding node in block 1.  
  - *Input:* User chat query text  
  - *Output:* Query embedding to Simple Vector Store retrieval node  
  - *Edge Cases:* API errors, inconsistent embedding versions causing retrieval issues.

---

### 3. Summary Table

| Node Name                     | Node Type                                         | Functional Role                                 | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                         |
|-------------------------------|--------------------------------------------------|------------------------------------------------|-------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                                   | Initiates reference data loading                | —                             | HTTP Request                 | ## Step 1.1                                                                                                         |
| HTTP Request                  | HTTP Request                                     | Downloads IPL “Match Playing Conditions” PDF   | When clicking ‘Execute workflow’ | Simple Vector Store1          | ## Step 1                                                                                                           |
| Default Data Loader           | Document Default Data Loader                      | Extracts text from downloaded PDF               | Recursive Character Text Splitter | Simple Vector Store1          | ## Step 1.2 and 1.3                                                                                                |
| Recursive Character Text Splitter | Text Splitter (Recursive Character)            | Splits extracted text into overlapping chunks   | Default Data Loader             | Embeddings Google Gemini1     | ## Step 1.2 and 1.3                                                                                                |
| Embeddings Google Gemini1     | Google Gemini Embeddings                          | Converts text chunks to vector embeddings       | Recursive Character Text Splitter | Simple Vector Store1          | ## Step 1.4                                                                                                         |
| Simple Vector Store1          | Vector Store In-Memory (Insert mode)             | Inserts embeddings to in-memory vector store    | Embeddings Google Gemini1       | —                            | ## Step 1.5                                                                                                         |
| When chat message received    | Langchain Chat Trigger                           | Receives user chat messages                      | —                             | AI Agent                     | ## Step 2.1                                                                                                         |
| AI Agent                     | Langchain Agent                                  | Orchestrates chat Q&A using memory, vector store, and LLM | When chat message received      | —                            | ## Step 2.5                                                                                                         |
| Simple Memory                | Memory Buffer Window                             | Keeps last 20 chat turns for context            | AI Agent (ai_memory)           | AI Agent                     | ## Step 2.2                                                                                                         |
| Simple Vector Store          | Vector Store In-Memory (Retrieve-as-tool mode)  | Retrieves top-10 relevant chunks for RAG        | Embeddings Google Gemini (query embedding) | AI Agent                     | ## Step 2.3                                                                                                         |
| Google Gemini Chat Model     | Google Gemini Chat Model (LLM)                    | Generates AI answers based on prompts           | AI Agent (ai_languageModel)    | AI Agent                     | ## Step 2.4                                                                                                         |
| Embeddings Google Gemini     | Google Gemini Embeddings                          | Converts user query text to embeddings           | When chat message received     | Simple Vector Store          | ## Step 2.3                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the data loading process.

2. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://documents.iplt20.com/bcci/documents/1742707993986_Match_Playing_Conditions.pdf`  
     - No authentication or headers needed.  
   - Connect Manual Trigger node output to this node input.

3. **Create Default Data Loader Node**  
   - Type: Document Default Data Loader  
   - Parameters:  
     - Data Type: Binary (PDF)  
     - Text Splitting Mode: Custom  
   - Connect HTTP Request output to this node input.

4. **Create Recursive Character Text Splitter Node**  
   - Type: Recursive Character Text Splitter  
   - Parameters:  
     - Chunk Overlap: 200 characters (to ensure chunk context overlap)  
     - Default chunk size (leave as default or adjust as needed)  
   - Connect Default Data Loader output to this node input.

5. **Create Embeddings Google Gemini Node for Reference Data**  
   - Type: Langchain Embeddings Google Gemini  
   - Credentials: Configure Google Gemini (PaLM) API Credential (API key required)  
   - Connect Recursive Character Text Splitter output to this node input.

6. **Create Simple Vector Store Node (Insert Mode)**  
   - Type: Vector Store In-Memory  
   - Parameters:  
     - Mode: Insert  
     - Memory Key: Define a unique key, e.g., "vector_store_key" (must be consistent with retrieval)  
   - Connect Embeddings Google Gemini output to this node input.

7. **Create Chat Trigger Node**  
   - Type: Langchain Chat Trigger  
   - Purpose: To receive chat messages from users.  
   - No special configuration required.

8. **Create Embeddings Google Gemini Node for Queries**  
   - Type: Langchain Embeddings Google Gemini  
   - Credentials: Same Google Gemini API Credential as above  
   - Connect Chat Trigger output to this node input (for query embedding).

9. **Create Simple Vector Store Node (Retrieve-as-Tool Mode)**  
   - Type: Vector Store In-Memory  
   - Parameters:  
     - Mode: Retrieve-as-tool  
     - Top K: 10 (number of relevant chunks to retrieve)  
     - Memory Key: Same key as insertion, e.g., "vector_store_key"  
     - Tool Description: "This is a repository of IPL cricket rules and international cricket rules"  
   - Connect Embeddings Google Gemini (query) output to this node input.

10. **Create Simple Memory Node**  
    - Type: Memory Buffer Window  
    - Parameters:  
      - Context Window Length: 20 (retain last 20 messages for context)  
    - Connect AI Agent memory input to this node.

11. **Create Google Gemini Chat Model Node**  
    - Type: Langchain Google Gemini Chat Model  
    - Parameters:  
      - Top P: 0.3 (to focus generation)  
    - Credentials: Google Gemini API Credential  

12. **Create AI Agent Node**  
    - Type: Langchain Agent  
    - Parameters:  
      - System Message:  
        ```
        You are a cricket expert. 
        
        You are tasked with answering questions on IPL cricket queries. Information should only be referred to and provided if it is provided explicitly in the database to you. Your goal is to provide accurate information based on this information.
        
        If information is not provided to you explicitly or if you cannot answer the question using the provided information, say "Sorry I donot know"
        ```  
      - Memory Input: Connect to Simple Memory node output  
      - Tool Input: Connect to Simple Vector Store (retrieve) node output  
      - Language Model Input: Connect to Google Gemini Chat Model node output  
    - Connect Chat Trigger node main output to AI Agent main input.

13. **Connect Nodes for Data Flow:**  
    - Chat Trigger → Embeddings Google Gemini (query) → Simple Vector Store (retrieve) → AI Agent (tool input)  
    - Simple Memory → AI Agent (memory input)  
    - Google Gemini Chat Model → AI Agent (language model input)  
    - AI Agent → Output Chat Response  

14. **Configure Credentials:**  
    - Set up Google Gemini API credentials with valid API key for both embeddings and chat model nodes.

15. **Run Manual Trigger to Load Data:**  
    - Execute Manual Trigger → HTTP Request → Data Loader → Text Splitter → Embeddings → Vector Store Insert.

16. **Test Chat Interface:**  
    - Send chat messages to the chat trigger webhook; verify responses are accurate and grounded on IPL rules.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is divided into two main steps: (1) Vector store creation with IPL rules using Google Gemini embeddings, and (2) Chat interface connected with AI agent using Google Gemini chat API.                                          | Sticky Note7                                                                                                                         |
| Google Gemini API key credential is required for embeddings and chat model nodes. Use n8n’s native credential management for secure API key storage.                                                                                       | Sticky Notes (multiple)                                                                                                              |
| Using simple memory store and in-memory vector store nodes is ideal for proof of concept and testing before upgrading to scalable enterprise-grade vector stores.                                                                           | Sticky Note (Step 2), Sticky Note (Step 1)                                                                                           |
| The chunk overlap in text splitting is set to 200 to reduce hallucinations by ensuring context continuity across text chunks.                                                                                                             | Sticky Note (Step 1.3)                                                                                                               |
| Official IPL “Match Playing Conditions” PDF URL used is: https://documents.iplt20.com/bcci/documents/1742707993986_Match_Playing_Conditions.pdf                                                                                              | HTTP Request node                                                                                                                    |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow built with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.