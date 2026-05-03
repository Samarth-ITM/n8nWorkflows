Build a Knowledge Base Chatbot with Jotform, RAG on Supabase, Together AI & Gemini

https://n8nworkflows.xyz/workflows/build-a-knowledge-base-chatbot-with-jotform--rag-on-supabase--together-ai---gemini-9626


# Build a Knowledge Base Chatbot with Jotform, RAG on Supabase, Together AI & Gemini

---

## 1. Workflow Overview

This workflow implements a **Knowledge Base Chatbot** that integrates multiple services to enable users to upload PDF documents via Jotform and subsequently ask questions based on the content of those documents. The system uses Retrieval-Augmented Generation (RAG) techniques on a Supabase vector database combined with Together AI embeddings and Google Gemini (PaLM) for answering user queries accurately and contextually.

The workflow is logically divided into two main blocks:

- **1.1 Feeding the AI Knowledge ("Librarian" part)**  
  Handles new PDF uploads from Jotform, extracts text, splits it into chunks, generates embeddings for each chunk, and stores them in Supabase for fast semantic search.

- **1.2 Asking the AI a Question ("Researcher" part)**  
  Triggered by user chat messages, it embeds the question, searches the Supabase vector DB for relevant chunks, and uses an AI agent powered by Google Gemini to generate a precise, context-based answer.

---

## 2. Block-by-Block Analysis

### 2.1 Feeding the AI Knowledge ("Librarian" part)

**Overview:**  
This block activates when a user uploads a new PDF contract through a Jotform form. It processes the uploaded file by extracting text, splitting it into manageable chunks, generating semantic embeddings for each chunk, and saving these embeddings along with the text chunks into a Supabase vector database. This builds a searchable knowledge base.

**Nodes Involved:**  
- JotForm Trigger  
- Grab New knowledgebase  
- Grab the uploaded knowledgebase file link  
- Extract Text from PDF File  
- Splitting into Chunks  
- Embedding Uploaded document  
- Save the embedding in DB  
- Sticky Note (explaining this part)

**Node Details:**

- **JotForm Trigger**  
  - Type: Trigger node for Jotform form submissions  
  - Configuration: Linked to a specific form ID (`252862840518058`), listens for any submission  
  - Inputs: None (webhook trigger)  
  - Outputs: Submission JSON data including submission ID and answers  
  - Possible failures: Network issues, invalid form ID, webhook misconfiguration  
  - Credentials: JotForm API with read permissions  
  - Notes: Entry point for document uploads

- **Grab New knowledgebase**  
  - Type: HTTP Request  
  - Configuration: Fetches submission details from Jotform API using `submissionID` from trigger  
  - URL: `https://api.jotform.com/submission/{{ $json.submissionID }}?apiKey=...`  
  - Inputs: Submission ID from previous node  
  - Outputs: Submission data including answers and uploaded file links  
  - Possible failures: API key invalid/expired, rate limits, network errors

- **Grab the uploaded knowledgebase file link**  
  - Type: HTTP Request  
  - Configuration: Downloads the actual uploaded file using the file URL found in submission answers (`answers['6'].answer[0]`)  
  - Response format: File download (binary)  
  - Headers: Includes API key for authentication  
  - Inputs: File URL from previous node  
  - Outputs: Binary file data (PDF)  
  - Possible failures: File URL invalid, permission denied, API key issues

- **Extract Text from PDF File**  
  - Type: Extract from File  
  - Configuration: Extracts raw text content from the downloaded PDF file  
  - Inputs: Binary PDF file  
  - Outputs: Extracted text JSON field  
  - Possible failures: PDF corrupt or unsupported format, extraction errors

- **Splitting into Chunks**  
  - Type: Code node (JavaScript)  
  - Configuration: Splits the extracted text into 1000-character chunks for manageable processing  
  - Key logic: Iterates over text string, slicing into array of chunk objects  
  - Inputs: Extracted text JSON  
  - Outputs: Multiple items, each containing a text chunk  
  - Possible failures: Empty or very short text, malformed input

- **Embedding Uploaded document**  
  - Type: HTTP Request  
  - Configuration: Calls Together AI embedding API with model `BAAI/bge-large-en-v1.5` to generate embeddings for each text chunk  
  - Input parameter: chunk text from previous node  
  - Authentication: HTTP Bearer token with Together API credentials  
  - Outputs: JSON containing embedding vector  
  - Possible failures: API limits, network issues, invalid API token

- **Save the embedding in DB**  
  - Type: Supabase node  
  - Configuration: Inserts each chunk and its embedding (stringified JSON) into the `RAG` table in Supabase  
  - Inputs: chunk text, embedding from Together AI  
  - Outputs: Confirmation of DB insert  
  - Credentials: Supabase API key with write access  
  - Possible failures: DB connection issues, permission denied, malformed data

- **Sticky Note**  
  - Content: Describes the entire "Librarian" block purpose and flow for user clarity

---

### 2.2 Asking the AI a Question ("Researcher" part)

**Overview:**  
Triggered by incoming chat messages, this block embeds the user's query, searches Supabase for the most relevant stored text chunks, aggregates these chunks, and feeds them as context to an AI Agent powered by Google Gemini. The AI Agent then produces a WhatsApp-style formatted answer strictly based on retrieved knowledge.

**Nodes Involved:**  
- When chat message received  
- Embend User Message  
- Search Embeddings  
- Aggregate  
- AI Agent  
- Google Gemini Chat Model  
- Sticky Note (explaining this part)

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Configuration: Webhook listening for chat messages  
  - Inputs: Incoming chat message JSON with `chatInput` field  
  - Outputs: User message text for embedding  
  - Possible failures: Webhook misconfiguration, payload errors

- **Embend User Message**  
  - Type: HTTP Request  
  - Configuration: Calls Together AI embedding API with `BAAI/bge-large-en-v1.5` model to embed user's chat input  
  - Input: `chatInput` from chat message  
  - Authentication: Bearer token for Together API  
  - Outputs: Embedding vector for query  
  - Possible failures: API quota limits, invalid input

- **Search Embeddings**  
  - Type: HTTP Request (Supabase RPC)  
  - Configuration: Calls Supabase RPC function `matchembeddings1` with user query embedding and `match_count`=5 to get top 5 relevant chunks  
  - Inputs: embedding vector from previous node  
  - Outputs: List of matching text chunks  
  - Credentials: Supabase API key with read access  
  - Possible failures: DB connectivity, RPC function errors

- **Aggregate**  
  - Type: Aggregate node  
  - Configuration: Aggregates the matched chunks by concatenating all `chunk` fields into one combined context string  
  - Inputs: Array of matched chunks  
  - Outputs: Single aggregated context string  
  - Possible failures: Empty results, aggregation errors

- **AI Agent**  
  - Type: Langchain agent node  
  - Configuration: Receives aggregated chunks as context and user message. Uses a prompt template instructing the AI to answer based purely on the context, formatting replies in WhatsApp style (italics, bold, bullet points) and responding with "I don't know..." if information is missing.  
  - Inputs: Aggregated context, user chat message  
  - Outputs: AI-generated reply string  
  - Possible failures: AI model errors, prompt formatting issues

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini (PaLM) LLM node  
  - Configuration: Provides the language model backend for the AI Agent node  
  - Credentials: Google PaLM API key  
  - Inputs: AI Agent prompt and context  
  - Outputs: AI-generated language model completion  
  - Possible failures: Google API quota, auth errors

- **Sticky Note**  
  - Content: Describes the "Researcher" block's purpose and flow for user understanding

---

## 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                                  | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                                                                          |
|--------------------------------|----------------------------------|-------------------------------------------------|----------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger                | JotForm Trigger                  | Trigger on new PDF contract upload via Jotform  | None                             | Grab New knowledgebase          | Part 1: Feeding the AI Knowledge (The "Librarian" part) - handles PDF upload and processing                                                           |
| Grab New knowledgebase         | HTTP Request                    | Fetch form submission details from Jotform API  | JotForm Trigger                 | Grab the uploaded knowledgebase file link | Part 1: Feeding the AI Knowledge (The "Librarian" part)                                                                                              |
| Grab the uploaded knowledgebase file link | HTTP Request                    | Download uploaded PDF file                        | Grab New knowledgebase          | Extract Text from PDF File      | Part 1: Feeding the AI Knowledge (The "Librarian" part)                                                                                              |
| Extract Text from PDF File     | Extract from File                | Extract raw text from PDF                         | Grab the uploaded knowledgebase file link | Splitting into Chunks          | Part 1: Feeding the AI Knowledge (The "Librarian" part)                                                                                              |
| Splitting into Chunks          | Code                            | Split extracted text into 1000-character chunks | Extract Text from PDF File       | Embedding Uploaded document     | Part 1: Feeding the AI Knowledge (The "Librarian" part)                                                                                              |
| Embedding Uploaded document    | HTTP Request                    | Generate embeddings for each text chunk          | Splitting into Chunks           | Save the embedding in DB        | Part 1: Feeding the AI Knowledge (The "Librarian" part)                                                                                              |
| Save the embedding in DB       | Supabase                       | Store chunk text and embedding vector in DB      | Embedding Uploaded document     | None                          | Part 1: Feeding the AI Knowledge (The "Librarian" part)                                                                                              |
| When chat message received     | Langchain Chat Trigger          | Trigger on user chat message                      | None                            | Embend User Message            | Part 2: Asking the AI a Question (The "Researcher" part)                                                                                            |
| Embend User Message            | HTTP Request                    | Generate embedding vector for user question      | When chat message received      | Search Embeddings             | Part 2: Asking the AI a Question (The "Researcher" part)                                                                                            |
| Search Embeddings              | HTTP Request                   | Search Supabase DB for relevant chunks            | Embend User Message             | Aggregate                    | Part 2: Asking the AI a Question (The "Researcher" part)                                                                                            |
| Aggregate                     | Aggregate                      | Aggregate matched chunks into single context      | Search Embeddings               | AI Agent                    | Part 2: Asking the AI a Question (The "Researcher" part)                                                                                            |
| AI Agent                      | Langchain Agent                | Generate AI answer based on context and user query | Aggregate                       | None (final output)           | Part 2: Asking the AI a Question (The "Researcher" part)                                                                                            |
| Google Gemini Chat Model       | Langchain Google Gemini LLM     | LLM backend model for AI Agent                    | AI Agent (languageModel input)  | AI Agent                     | Part 2: Asking the AI a Question (The "Researcher" part)                                                                                            |
| Sticky Note                   | Sticky Note                    | Explanatory note for "Librarian" part             | None                            | None                         | Describes Part 1: Feeding the AI Knowledge                                                                                                          |
| Sticky Note1                  | Sticky Note                    | Explanatory note for "Researcher" part             | None                            | None                         | Describes Part 2: Asking the AI a Question                                                                                                          |

---

## 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger node**  
   - Type: JotForm Trigger  
   - Configure webhook for form ID `252862840518058`  
   - Connect credentials with JotForm API key having read access  
   - No input nodes; output connects to next node

2. **Add HTTP Request node "Grab New knowledgebase"**  
   - Method: GET  
   - URL: `https://api.jotform.com/submission/{{ $json.submissionID }}?apiKey=YOUR_JOTFORM_API_KEY` (replace placeholder)  
   - Connect input from JotForm Trigger  
   - No authentication required if API key in URL, or configure as needed  
   - Output connects to next node

3. **Add HTTP Request node "Grab the uploaded knowledgebase file link"**  
   - Method: GET  
   - URL: `={{ $json.content.answers['6'].answer[0] }}` (adjust path to file URL in form answers)  
   - Set Response Format: File (binary)  
   - Add HTTP header `APIKEY: YOUR_JOTFORM_API_KEY`  
   - Connect input from previous node  
   - Output connects to next node

4. **Add Extract From File node "Extract Text from PDF File"**  
   - Operation: PDF  
   - Input: binary file from previous node  
   - Output: JSON with field containing extracted text  
   - Connect input from previous node

5. **Add Code node "Splitting into Chunks"**  
   - Paste this JavaScript code:  
     ```javascript
     const text = $input.first().json.text;
     const chunkSize = 1000;

     let chunks = [];
     for (let i = 0; i < text.length; i += chunkSize) {
       chunks.push({
         json: { chunk: text.slice(i, i + chunkSize) }
       });
     }

     return chunks;
     ```  
   - Input: JSON text from Extract node  
   - Output: multiple items with `chunk` field

6. **Add HTTP Request node "Embedding Uploaded document"**  
   - Method: POST  
   - URL: `https://api.together.xyz/v1/embeddings`  
   - Body (JSON):  
     ```json
     {
       "model": "BAAI/bge-large-en-v1.5",
       "input": "={{ $json.chunk }}"
     }
     ```  
   - Headers: `Content-Type: application/json`  
   - Authentication: HTTP Bearer Token with Together API credentials  
   - Input: chunk from previous node  
   - Output connects to next node

7. **Add Supabase node "Save the embedding in DB"**  
   - Operation: Insert  
   - Table: `RAG` (vector DB table)  
   - Map fields:  
     - `chunk` = `={{ $('Splitting into Chunks').item.json.chunk }}`  
     - `embeddings` = `={{ JSON.stringify($json.data[0].embedding) }}`  
   - Connect credentials for Supabase API with write access  
   - Input: output from embedding node

---

8. **Add Langchain Chat Trigger node "When chat message received"**  
   - Webhook trigger to receive user chat messages  
   - No credentials required  
   - Output: user message JSON with `chatInput` field

9. **Add HTTP Request node "Embend User Message"**  
   - Method: POST  
   - URL: `https://api.together.xyz/v1/embeddings`  
   - Body (JSON):  
     ```json
     {
       "model": "BAAI/bge-large-en-v1.5",
       "input": "={{ $json.chatInput }}"
     }
     ```  
   - Headers: `Content-Type: application/json`  
   - Authentication: HTTP Bearer Token with Together API credentials  
   - Connect input from chat trigger node

10. **Add HTTP Request node "Search Embeddings"**  
    - Method: POST  
    - URL: `https://your-supabase-host/rest/v1/rpc/matchembeddings1` (replace with actual Supabase host)  
    - Body (JSON):  
      ```json
      {
        "query_embedding": "={{ $json.data[0].embedding }}",
        "match_count": 5
      }
      ```  
    - Authentication: Supabase API with read access  
    - Connect input from user embedding node

11. **Add Aggregate node "Aggregate"**  
    - Aggregate field: `chunk`  
    - Operation: concatenate all matched chunks into one string  
    - Connect input from Search Embeddings node

12. **Add Langchain Agent node "AI Agent"**  
    - Set prompt text as:  
      ```
      You are a helpful and professional customer support agent. Use the following context to answer the user's question.

      Handle greetings without the need of the context...

      Context:
      {{ $json.chunk }}

      User's message:
      {{ $('When chat message received').item.json.chatInput }}

      Format your reply in WhatsApp style:
      - Use _italics_ for emphasis
      - Use *bold* for key points
      - Use • for bullet lists (no markdown dashes or hashes)
      - Keep responses short, clear, and conversational, like real WhatsApp support
      - Avoid markdown headers or code blocks

      Give a clear, accurate, and friendly response based only on the context.
      If the answer cannot be found in the context, reply: _"I don't know based on the provided information."_
      ```
    - Connect input from Aggregate node

13. **Add Google Gemini Chat Model node**  
    - Connect to AI Agent as the language model backend  
    - Configure credentials with Google PaLM API key

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                          | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Part 1: Feeding the AI Knowledge ("Librarian") describes the ingestion and vectorization pipeline for new PDF knowledgebase docs uploaded via Jotform.                              | Sticky Note content in workflow                                                                              |
| Part 2: Asking the AI a Question ("Researcher") describes how user questions are embedded, matched to stored chunks, and answered by the AI agent using Google Gemini.               | Sticky Note1 content in workflow                                                                             |
| Together AI embeddings use model `BAAI/bge-large-en-v1.5` for semantic vectorization.                                                                                                  | Together AI API documentation                                                                                 |
| Supabase vector database requires a table named `RAG` with fields for `chunk` (text) and `embeddings` (JSON string).                                                                 | Supabase documentation                                                                                        |
| Google Gemini (PaLM) API credentials are required for the AI language model node.                                                                                                      | Google Cloud PaLM API documentation                                                                           |
| The prompt to the AI agent enforces WhatsApp-style formatting with italics, bold, and bullet points for conversational clarity.                                                      | Prompt template embedded in AI Agent node                                                                     |
| The workflow expects Jotform file upload question to be at answer key `'6'`; adjust if your form differs.                                                                             | Jotform submission JSON structure                                                                              |
| Replace placeholders for API keys and Supabase host URLs with your actual credentials before running.                                                                                   | Security best practice                                                                                          |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.