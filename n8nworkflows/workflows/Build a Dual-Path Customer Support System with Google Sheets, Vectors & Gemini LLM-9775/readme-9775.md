Build a Dual-Path Customer Support System with Google Sheets, Vectors & Gemini LLM

https://n8nworkflows.xyz/workflows/build-a-dual-path-customer-support-system-with-google-sheets--vectors---gemini-llm-9775


# Build a Dual-Path Customer Support System with Google Sheets, Vectors & Gemini LLM

### 1. Workflow Overview

This workflow, titled **"Build a Dual-Path Customer Support System with Google Sheets, Vectors & Gemini LLM"**, is designed to provide automated customer support by combining a knowledge base FAQ retrieval system with a fallback conversational AI assistant. It targets customer support use cases where users submit queries that can be answered either via predefined FAQ answers stored in Google Sheets or via a conversational Large Language Model (LLM) when no suitable FAQ answer exists.

The workflow is logically divided into these blocks:

- **1.1 Initialization & Data Preparation:** Loading FAQ data from Google Sheets and generating persistent embeddings stored in an in-memory vector store to support similarity search.
- **1.2 User Input Reception:** Receiving chat messages from users via a chat trigger webhook.
- **1.3 Query Processing & Similarity Scoring:** Generating embeddings for user queries, retrieving closest matching FAQ questions from the vector store, and scoring similarity.
- **1.4 Response Decision & Delivery:** Using an If/Else decision node to determine if a matching FAQ answer exists above a similarity threshold. If yes, retrieving the pre-defined FAQ answer and replying; if no, forwarding the query to a conversational LLM (Google Gemini) and responding with its output.
- **1.5 Manual Trigger for Data Loading:** A manual node to kick off FAQ data loading and embedding generation.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Data Preparation

- **Overview:**  
  This block loads FAQ questions from a Google Sheets document, extracts relevant fields, generates embeddings for them using HuggingFace, and stores them into an in-memory vector store for quick similarity search.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Knowledge Database (Google Sheets)  
  - Extract Questions (Set)  
  - Embeddings HuggingFace Inference  
  - Default Data Loader  
  - Generate & Store Embeddings (Vector Store In Memory)

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually to initiate FAQ data loading.  
    - Inputs: None  
    - Outputs: To "Knowledge Database"  
    - Potential Failures: None typical; manual trigger requires user interaction.

  - **Knowledge Database**  
    - Type: Google Sheets  
    - Role: Reads FAQ data from a specific Google Sheets tab ("Stripe_Dummy_FAQ") containing questions and answers.  
    - Config: Uses OAuth2 credentials for Google Sheets access, reads all rows without first-match filtering.  
    - Inputs: From manual trigger  
    - Outputs: To "Extract Questions"  
    - Potential Failures: OAuth token expiration, permission errors, sheet name or documentId misconfiguration.

  - **Extract Questions**  
    - Type: Set node  
    - Role: Extracts and reformats relevant columns ("row_number" and "Question") from the Google Sheets data for embedding generation.  
    - Inputs: From "Knowledge Database"  
    - Outputs: To "Embeddings HuggingFace Inference"  
    - Configuration: Assigns extracted fields as variables for downstream nodes.  
    - Potential Failures: Expression errors if expected fields are missing.

  - **Embeddings HuggingFace Inference**  
    - Type: LangChain Embeddings HuggingFace Inference  
    - Role: Generates vector embeddings for each FAQ question text using HuggingFace API.  
    - Config: Authenticated with a HuggingFace API key; default options used.  
    - Inputs: From "Extract Questions"  
    - Outputs: To "Generate & Store Embeddings" (via ai_embedding connection)  
    - Potential Failures: API rate limits, invalid credentials, network timeouts.

  - **Default Data Loader**  
    - Type: LangChain Document Default Data Loader  
    - Role: Also connected as an ai_document input to "Generate & Store Embeddings" (likely for fallback or batch data loading).  
    - Inputs: None explicitly triggered here; auxiliary.  
    - Outputs: To "Generate & Store Embeddings" (ai_document input)  
    - Potential Failures: Data format issues.

  - **Generate & Store Embeddings**  
    - Type: LangChain Vector Store In Memory (mode = insert)  
    - Role: Inserts embeddings into the in-memory vector store keyed by "vector_store_key" for later similarity retrieval.  
    - Inputs: From "Embeddings HuggingFace Inference" (ai_embedding) and "Default Data Loader" (ai_document)  
    - Outputs: None (terminal for this chain)  
    - Potential Failures: Memory overload if dataset is large, key misconfiguration.

---

#### 1.2 User Input Reception

- **Overview:**  
  This block listens for incoming chat messages from users via a webhook and initiates processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: LangChain Chat Trigger (webhook)  
    - Role: Receives chat input messages, triggering the workflow on user interaction.  
    - Configuration: Webhook with responseMode set to "responseNodes" to enable asynchronous responses.  
    - Inputs: External webhook calls  
    - Outputs: To "Retrieve & Score Embeddings" and "Forward Chat Message" (parallel branches)  
    - Potential Failures: Webhook misconfiguration, connectivity issues.

---

#### 1.3 Query Processing & Similarity Scoring

- **Overview:**  
  This block generates embeddings for the user query, retrieves the top matching FAQ questions from the vector store, and scores the similarity to decide if the FAQ contains a suitable answer.

- **Nodes Involved:**  
  - Embeddings HuggingFace Inference2  
  - Retrieve & Score Embeddings  
  - Determine Question Type (If/Else node)

- **Node Details:**  
  - **Embeddings HuggingFace Inference2**  
    - Type: LangChain Embeddings HuggingFace Inference  
    - Role: Generates embeddings for the incoming user chat message using the same HuggingFace API model as for FAQ questions.  
    - Inputs: From "When chat message received" (implied via connections)  
    - Outputs: To "Retrieve & Score Embeddings" (ai_embedding input)  
    - Potential Failures: Same as first embeddings node, including API issues.

  - **Retrieve & Score Embeddings**  
    - Type: LangChain Vector Store In Memory (mode = load)  
    - Role: Loads the top 2 most similar FAQ questions from the in-memory vector store using the user query embeddings.  
    - Inputs: From "When chat message received" (main) and "Embeddings HuggingFace Inference2" (ai_embedding)  
    - Outputs: To "Determine Question Type" (main)  
    - Potential Failures: Empty vector store, memory errors, key mismatch.

  - **Determine Question Type**  
    - Type: If Node  
    - Role: Compares similarity score of top retrieved FAQ question with threshold (≥ 0.8) to decide if FAQ is relevant.  
    - Inputs: From "Retrieve & Score Embeddings"  
    - Outputs: Two branches:  
      - True (similarity >= 0.8): to "Get Respective Answers"  
      - False: to "Forward Chat Message" (fallback to LLM)  
    - Potential Failures: Expression evaluation errors, missing score field.

---

#### 1.4 Response Decision & Delivery

- **Overview:**  
  This block handles delivering the final answer to the user, either from the FAQ database or via a conversational AI fallback.

- **Nodes Involved:**  
  - Get Respective Answers  
  - Respond to Chat  
  - Forward Chat Message  
  - Chat Model (Google Gemini)  
  - Respond to Chat1

- **Node Details:**  
  - **Get Respective Answers**  
    - Type: Google Sheets  
    - Role: Retrieves the static FAQ answer corresponding to the matched question from the Google Sheets knowledge base.  
    - Config: Filters rows where "Question" equals the matched FAQ question content.  
    - Inputs: From "Determine Question Type" (true branch)  
    - Outputs: To "Respond to Chat"  
    - Potential Failures: Same as previous Google Sheets node; also risk of no matching answer found.

  - **Respond to Chat**  
    - Type: LangChain Chat  
    - Role: Sends the retrieved FAQ answer back to the user.  
    - Config: Message content set dynamically to the answer field from Google Sheets.  
    - Inputs: From "Get Respective Answers"  
    - Outputs: Terminal (sends response)  
    - Potential Failures: Expression errors if answer content missing.

  - **Forward Chat Message**  
    - Type: Merge (Choose Branch)  
    - Role: Merges two branches — the false branch from FAQ decision (fallback to LLM) and one from initial retrieval — to forward to the LLM.  
    - Inputs: From "When chat message received" and "Determine Question Type" (false branch)  
    - Outputs: To "Chat Model"  
    - Potential Failures: Merge misconfiguration.

  - **Chat Model**  
    - Type: LangChain Google Gemini LLM  
    - Role: Generates a conversational AI response for queries not answered by FAQ.  
    - Config: Uses a preset system message to enforce strict scope and disclaimers for sensitive queries; model "gemini-2.5-flash-lite" selected.  
    - Inputs: From "Forward Chat Message"  
    - Outputs: To "Respond to Chat1"  
    - Credential: Google Palm API (Gemini) OAuth2  
    - Potential Failures: API limits, incorrect credentials, network issues.

  - **Respond to Chat1**  
    - Type: LangChain Chat  
    - Role: Sends the LLM-generated response back to the user.  
    - Config: Message content extracted from the LLM response JSON structure.  
    - Inputs: From "Chat Model"  
    - Outputs: Terminal (sends response)  
    - Potential Failures: Expression errors if response structure changes.

---

### 3. Summary Table

| Node Name                     | Node Type                                    | Functional Role                                     | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                    |
|-------------------------------|----------------------------------------------|----------------------------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                              | Manual start for loading FAQ data                   | None                          | Knowledge Database           | ## 0. Create Embeddings - Get common questions from your own knowledge base - Use persistent vector store for embeddings |
| Knowledge Database             | Google Sheets                                | Reads FAQ questions and answers from Google Sheets | When clicking ‘Execute workflow’ | Extract Questions            |                                                                                                               |
| Extract Questions             | Set                                          | Extracts question text and row number from sheet    | Knowledge Database             | Embeddings HuggingFace Inference |                                                                                                               |
| Embeddings HuggingFace Inference | LangChain Embeddings HuggingFace Inference | Generates embeddings for FAQ questions              | Extract Questions              | Generate & Store Embeddings  | ## 2. Evaluate user message - Generate embeddings for the user message - Check against your vector store for relevant questions - IMPORTANT: You need to fine-tune your similarity score threshold within the If/Else-Node |
| Default Data Loader           | LangChain Document Default Data Loader       | Auxiliary data loader for embeddings                | None                          | Generate & Store Embeddings  |                                                                                                               |
| Generate & Store Embeddings   | LangChain Vector Store In Memory              | Stores FAQ question embeddings in vector store      | Embeddings HuggingFace Inference, Default Data Loader | None                        |                                                                                                               |
| When chat message received    | LangChain Chat Trigger (Webhook)              | Receives user chat messages                          | External webhook              | Retrieve & Score Embeddings, Forward Chat Message | ## 1. Receive a user message                                                                                   |
| Embeddings HuggingFace Inference2 | LangChain Embeddings HuggingFace Inference | Generates embeddings for user messages               | When chat message received     | Retrieve & Score Embeddings  |                                                                                                               |
| Retrieve & Score Embeddings   | LangChain Vector Store In Memory              | Retrieves and scores top matching FAQ questions     | When chat message received, Embeddings HuggingFace Inference2 | Determine Question Type       |                                                                                                               |
| Determine Question Type       | If Node                                       | Checks similarity threshold to choose response path | Retrieve & Score Embeddings    | Get Respective Answers (true), Forward Chat Message (false) |                                                                                                               |
| Get Respective Answers        | Google Sheets                                 | Retrieves the matching FAQ answer                    | Determine Question Type (true) | Respond to Chat             | ## 3.1 Reply from FAQ - If the user query is similar to a question from your knowledge base, the pre-defined, static answer is returned |
| Respond to Chat               | LangChain Chat                                | Sends FAQ answer back to user                        | Get Respective Answers         | None                       |                                                                                                               |
| Forward Chat Message          | Merge (Choose Branch)                          | Merges fallback chat message for LLM                 | When chat message received, Determine Question Type (false) | Chat Model                  |                                                                                                               |
| Chat Model                   | LangChain Google Gemini LLM                    | Fallback conversational AI for unmatched queries    | Forward Chat Message           | Respond to Chat1            | ## 3.2 Reply via LLM - If the user query is not found within the knowledge base, a smaller LLM can answer      |
| Respond to Chat1              | LangChain Chat                                | Sends LLM-generated response                         | Chat Model                    | None                       |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Add a "Manual Trigger" node named `When clicking ‘Execute workflow’`.  
   - No special configuration needed.

2. **Add Google Sheets Node for FAQ:**  
   - Add a "Google Sheets" node named `Knowledge Database`.  
   - Configure with OAuth2 credentials to access your FAQ spreadsheet.  
   - Set Document ID to your FAQ Google Sheet (e.g., "1tSEu_-7KAupdkFJGzoPBc-zqBBAmPgGoc89JyvqhY90").  
   - Set Sheet Name to the tab containing FAQ questions (e.g., "Stripe_Dummy_FAQ").  
   - Configure to fetch all rows (no filtering).  
   - Connect `When clicking ‘Execute workflow’` main output to this node.

3. **Add Set Node to Extract Questions:**  
   - Add a "Set" node named `Extract Questions`.  
   - Configure to assign two fields:  
     - `row_number` as `{{$json.row_number}}` (number)  
     - `Question` as `{{$json.Question}}` (string)  
   - Connect `Knowledge Database` main output to this node.

4. **Add Embeddings HuggingFace Inference Node:**  
   - Add a LangChain "Embeddings HuggingFace Inference" node named `Embeddings HuggingFace Inference`.  
   - Select or create HuggingFace API credentials (ensure API key valid).  
   - Default options are sufficient unless specific model tuning required.  
   - Connect `Extract Questions` main output to this node as ai_embedding input.

5. **Add Default Data Loader (Optional):**  
   - Add LangChain "Document Default Data Loader" node named `Default Data Loader`.  
   - No special parameters needed; acts as auxiliary data input.  
   - Connect this node’s ai_document output to `Generate & Store Embeddings`.

6. **Add Vector Store In Memory Node to Store Embeddings:**  
   - Add LangChain "Vector Store In Memory" node named `Generate & Store Embeddings`.  
   - Set mode to `insert`.  
   - Use a consistent memory key (e.g., "vector_store_key").  
   - Connect `Embeddings HuggingFace Inference` (ai_embedding) and `Default Data Loader` (ai_document) inputs to this node.  
   - No outputs required.

7. **Add LangChain Chat Trigger (Webhook):**  
   - Add LangChain "Chat Trigger" node named `When chat message received`.  
   - Configure webhook with `responseMode` set to `"responseNodes"`.  
   - This node starts on user chat messages.

8. **Add Embeddings HuggingFace Inference Node for User Query:**  
   - Add another LangChain "Embeddings HuggingFace Inference" node named `Embeddings HuggingFace Inference2`.  
   - Use same HuggingFace API credentials and settings as before.  
   - Connect the `When chat message received` node to this node (ai_embedding input).

9. **Add Vector Store In Memory Node to Retrieve & Score:**  
   - Add LangChain "Vector Store In Memory" node named `Retrieve & Score Embeddings`.  
   - Set mode to `load`.  
   - Configure topK to 2 for retrieving top 2 matches.  
   - Use the same memory key as before ("vector_store_key").  
   - Connect `When chat message received` (main) and `Embeddings HuggingFace Inference2` (ai_embedding) inputs.

10. **Add If Node to Determine Question Type:**  
    - Add an "If" node named `Determine Question Type`.  
    - Configure condition to check if `{{$json.score}} >= 0.8`.  
    - Connect `Retrieve & Score Embeddings` main output to this node.

11. **Add Google Sheets Node to Get Answers:**  
    - Add a "Google Sheets" node named `Get Respective Answers`.  
    - Use same OAuth2 credentials and spreadsheet as `Knowledge Database`.  
    - Configure filter to match `Question` column to `{{$json.document.pageContent}}` (the matched FAQ question).  
    - Connect `Determine Question Type` true branch output to this node.

12. **Add LangChain Chat Node to Respond with FAQ Answer:**  
    - Add LangChain "Chat" node named `Respond to Chat`.  
    - Set message to `{{$json.Answer}}` from Google Sheets data.  
    - Connect `Get Respective Answers` output to this node.

13. **Add Merge Node to Forward Chat Message:**  
    - Add a "Merge" node named `Forward Chat Message`.  
    - Set mode to "Choose Branch".  
    - Connect `When chat message received` main output to one input branch.  
    - Connect `Determine Question Type` false branch output to second input branch.

14. **Add LangChain Google Gemini LLM Node:**  
    - Add LangChain "Google Gemini" node named `Chat Model`.  
    - Select model ID `"models/gemini-2.5-flash-lite"`.  
    - Provide Google Palm API credentials (OAuth2).  
    - Insert the system message that restricts scope and sets disclaimers as per the original workflow.  
    - Pass user message as input.  
    - Connect `Forward Chat Message` output to this node.

15. **Add LangChain Chat Node to Respond with LLM Answer:**  
    - Add LangChain "Chat" node named `Respond to Chat1`.  
    - Set message to `{{$json.content.parts[0].text}}` extracted from Gemini response.  
    - Connect `Chat Model` output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| ## 0. Create Embeddings - Get common questions from your own knowledge base - Use persistent vector store for embeddings | Sticky Note on Initialization & Data Preparation block                                                  |
| ## 1. Receive a user message                                                                                         | Sticky Note on "When chat message received" node                                                        |
| ## 2. Evaluate user message - Generate embeddings for the user message - Check against your vector store for relevant questions - IMPORTANT: You need to fine-tune your similarity score threshold within the If/Else-Node | Sticky Note on Embeddings and similarity scoring nodes                                                  |
| ## 3.1 Reply from FAQ - If the user query is similar to a question from your knowledge base, the pre-defined, static answer is returned | Sticky Note on FAQ answer retrieval nodes                                                              |
| ## 3.2 Reply via LLM - If the user query is not found within the knowledge base, a smaller LLM can answer             | Sticky Note on fallback conversational AI nodes                                                        |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow built with n8n, an integration and automation tool. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.