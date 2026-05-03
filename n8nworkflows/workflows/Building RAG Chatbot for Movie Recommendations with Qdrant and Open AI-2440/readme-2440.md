Building RAG Chatbot for Movie Recommendations with Qdrant and Open AI

https://n8nworkflows.xyz/workflows/building-rag-chatbot-for-movie-recommendations-with-qdrant-and-open-ai-2440


# Building RAG Chatbot for Movie Recommendations with Qdrant and Open AI

### 1. Workflow Overview

This workflow builds a Retrieval-Augmented Generation (RAG) chatbot specialized in movie recommendations using the Qdrant vector database and OpenAI’s API. It is designed to provide accurate movie recommendations from the IMDB Top 1000 dataset without hallucinations by leveraging vector similarity search on movie descriptions. Users can specify positive preferences and negative exclusions (e.g., "a movie about wizards but not Harry Potter") to get personalized top-3 movie suggestions.

The workflow is logically divided into two main blocks:

- **1.1 Data Ingestion and Vector Store Setup**  
  This block imports the IMDB dataset from GitHub, extracts movie descriptions, creates embeddings using OpenAI, and uploads these embeddings with metadata to the Qdrant vector store.

- **1.2 Chat Interaction and Recommendation Retrieval**  
  This block handles chat inputs, generates embeddings for user queries (both positive and negative examples), queries the Qdrant Recommendation API to fetch relevant movie vectors, retrieves metadata for recommended movies, and formats the results for presentation by an AI agent.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion and Vector Store Setup

**Overview:**  
This block loads the IMDB Top 1000 movie dataset from a GitHub repository, processes the movie descriptions into embeddings via OpenAI, enriches them with metadata, and inserts the data into the Qdrant vector database for later retrieval.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- GitHub  
- Extract from File  
- Embeddings OpenAI  
- Default Data Loader  
- Token Splitter  
- Qdrant Vector Store  
- Sticky Note1  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start ingestion manually via UI test  
  - Inputs: None  
  - Outputs: Triggers GitHub node  
  - Failure modes: None typical; user interaction required  

- **GitHub**  
  - Type: GitHub File Node  
  - Role: Fetches the CSV dataset file "Top_1000_IMDB_movies.csv" from the `mrscoopers/n8n_demo` repository  
  - Config: Reads file from repository root  
  - Inputs: Manual trigger  
  - Outputs: File data to Extract from File  
  - Failure modes: Auth errors (invalid GitHub credentials), file not found, rate limits  

- **Extract from File**  
  - Type: File Extractor  
  - Role: Parses the CSV file content into structured JSON records per movie  
  - Inputs: GitHub file content  
  - Outputs: JSON objects for each movie to Qdrant Vector Store and Embeddings pipeline  
  - Failure modes: File format issues, parsing errors  

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings (LangChain wrapper)  
  - Role: Creates vector embeddings for movie descriptions using OpenAI’s `text-embedding-3-small` model  
  - Inputs: Movie description text from Default Data Loader  
  - Outputs: Embeddings to Qdrant Vector Store  
  - Credentials: OpenAI API key required  
  - Failure modes: API key invalid, rate limits, network errors  

- **Default Data Loader**  
  - Type: LangChain Document Loader  
  - Role: Structures movie description data into document format enriched with metadata fields (`movie_name`, `movie_release_date`, `movie_description`)  
  - Inputs: JSON movie data from Extract from File  
  - Outputs: Structured document for embedding generation  
  - Failure modes: Expression errors in metadata extraction  

- **Token Splitter**  
  - Type: Text Splitter (Token-based)  
  - Role: Splits movie descriptions into token chunks if needed for embedding processing (helps manage large text size)  
  - Inputs: Document data from Default Data Loader  
  - Outputs: Chunked text documents to Embeddings OpenAI  
  - Failure modes: Misconfiguration leading to improper chunking  

- **Qdrant Vector Store**  
  - Type: Qdrant Vector Store Node (LangChain wrapper)  
  - Role: Inserts movie embeddings and metadata into the Qdrant collection named `imdb`  
  - Inputs: Embeddings and metadata from Embeddings OpenAI and Default Data Loader  
  - Outputs: Confirmation of insert  
  - Credentials: Qdrant API credentials required  
  - Failure modes: Auth errors, API endpoint issues, collection not found  

- **Sticky Note1**  
  - Content: "Uploading data (movies and their descriptions) to Qdrant Vector Store"  
  - Linked nodes: above ingestion nodes  

---

#### 2.2 Chat Interaction and Recommendation Retrieval

**Overview:**  
This block handles live chat messages, where users specify positive and negative movie preferences. It converts these preferences into embeddings, queries Qdrant’s Recommendation API to find the top-3 matching movies, retrieves metadata, and formats a user-friendly response using an AI agent.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Window Buffer Memory  
- Call n8n Workflow Tool  
- Execute Workflow Trigger  
- Embedding Recommendation Request with Open AI  
- Embedding Anti-Recommendation Request with Open AI  
- Extracting Embedding  
- Extracting Embedding1  
- Merge  
- Calling Qdrant Recommendation API  
- Retrieving Recommended Movies Meta Data  
- Split Out1  
- Split Out  
- Merge1  
- Selecting Fields Relevant for Agent  
- Aggregate  
- Sticky Note  

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger (webhook)  
  - Role: Entry point for receiving chat messages from user  
  - Inputs: Incoming chat message, containing positive and negative example text  
  - Outputs: Passes message to AI Agent  
  - Failure modes: Webhook misconfiguration, missing input data  

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Central chatbot logic handling user interaction; calls tools and memory  
  - Config: System message instructing it to provide top-3 recommendations from vector DB without showing scores  
  - Inputs: Chat messages, memory, tool response  
  - Outputs: Chat response text  
  - Failure modes: Model API errors, logic errors in tool calling  

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Completion model (GPT-4o-mini)  
  - Role: Provides conversational AI responses within the agent  
  - Credentials: OpenAI API key  
  - Failure modes: API rate limits, invalid credentials  

- **Window Buffer Memory**  
  - Type: LangChain Memory Buffer (windowed)  
  - Role: Manages recent conversation context for the agent  
  - Inputs/Outputs: Conversation history chunking  
  - Failure modes: Memory state corruption  

- **Call n8n Workflow Tool**  
  - Type: LangChain Tool Node  
  - Role: Calls a sub-workflow (`movie_recommender`) passing user positive and negative examples for recommendations  
  - Inputs: User query data structured as JSON with `positive_example` and `negative_example` strings  
  - Outputs: Recommendation results to AI Agent  
  - Failure modes: Workflow not found, input schema mismatch  

- **Execute Workflow Trigger**  
  - Type: n8n Execute Workflow Trigger  
  - Role: Invokes the internal workflow that handles embedding and recommendation querying  
  - Inputs: Positive and negative example text  
  - Outputs: Embedding requests to OpenAI nodes  
  - Failure modes: Workflow execution errors, timeout  

- **Embedding Recommendation Request with Open AI**  
  - Type: HTTP Request (OpenAI Embeddings API)  
  - Role: Sends positive example text to OpenAI API to create embedding vector  
  - Inputs: Positive example text from incoming query  
  - Outputs: Embedding vector JSON  
  - Credentials: OpenAI API key  
  - Failure modes: HTTP errors, API key invalid  

- **Embedding Anti-Recommendation Request with Open AI**  
  - Type: HTTP Request (OpenAI Embeddings API)  
  - Role: Sends negative example text to OpenAI API to create embedding vector  
  - Inputs: Negative example text from incoming query  
  - Outputs: Embedding vector JSON  
  - Credentials: OpenAI API key  
  - Failure modes: HTTP errors, API key invalid  

- **Extracting Embedding**  
  - Type: Set Node  
  - Role: Extracts the positive embedding array from OpenAI response JSON  
  - Inputs: OpenAI embeddings response  
  - Outputs: Simplified embedding array with key `positive_example`  
  - Failure modes: JSON path errors  

- **Extracting Embedding1**  
  - Type: Set Node  
  - Role: Extracts the negative embedding array similarly with key `negative_example`  
  - Failure modes: Same as above  

- **Merge**  
  - Type: Merge Node (combine mode)  
  - Role: Combines positive and negative embeddings into a single JSON for recommendation query  
  - Inputs: Outputs from both embedding extraction nodes  
  - Outputs: Merged JSON with both embeddings  
  - Failure modes: Data mismatch, missing inputs  

- **Calling Qdrant Recommendation API**  
  - Type: HTTP Request  
  - Role: Calls Qdrant’s Recommendation API endpoint with positive and negative embeddings, using the “average_vector” strategy, limiting to top 3 results  
  - Inputs: Merged embedding JSON  
  - Outputs: Recommendation result points with IDs and scores  
  - Credentials: Qdrant API key  
  - Failure modes: Network errors, auth failure, invalid payload  

- **Retrieving Recommended Movies Meta Data**  
  - Type: HTTP Request  
  - Role: Fetches full metadata for the recommended movie IDs from Qdrant collection `imdb_1000_open_ai`  
  - Inputs: IDs from recommendation result points  
  - Outputs: Movie metadata including names, descriptions, release years  
  - Credentials: Qdrant API key  
  - Failure modes: Invalid IDs, API errors  

- **Split Out1**  
  - Type: Split Out Node  
  - Role: Extracts `result.points` array from recommendation API response for parallel processing  
  - Outputs: Individual points to Merge1 node  
  - Failure modes: Field missing  

- **Split Out**  
  - Type: Split Out Node  
  - Role: Extracts `result` array from metadata retrieval response for processing  
  - Outputs: Individual movie metadata items to Merge1  
  - Failure modes: Field missing  

- **Merge1**  
  - Type: Merge Node (combine by id)  
  - Role: Combines recommendation points and metadata by matching movie IDs  
  - Inputs: Points from Split Out1 and metadata from Split Out  
  - Outputs: Combined movie data with score and metadata  
  - Failure modes: ID mismatch  

- **Selecting Fields Relevant for Agent**  
  - Type: Set Node  
  - Role: Selects and renames relevant fields (`movie_recommendation_score`, `movie_description`, `movie_name`, `movie_release_year`) to simplify data for AI Agent  
  - Inputs: Combined movie data from Merge1  
  - Outputs: Cleaned movie recommendation objects  
  - Failure modes: Expression errors  

- **Aggregate**  
  - Type: Aggregate Node  
  - Role: Aggregates all movie recommendation items into a single JSON array under `response` field for final output  
  - Inputs: Set node output  
  - Outputs: Single aggregated object with all recommendations  
  - Failure modes: Empty input, aggregation failure  

- **Sticky Note**  
  - Content: "Tool, calling Qdrant's recommendation API based on user's request, transformed by AI agent"  
  - Linked nodes: All nodes in this recommendation retrieval block  

---

### 3. Summary Table

| Node Name                                   | Node Type                                  | Functional Role                                    | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                   |
|---------------------------------------------|--------------------------------------------|---------------------------------------------------|-------------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                | Manual Trigger                             | Manual start of data ingestion                     | None                                | GitHub                              |                                                                                               |
| GitHub                                       | GitHub File Node                           | Fetch IMDB CSV dataset from GitHub repo            | When clicking ‘Test workflow’       | Extract from File                   |                                                                                               |
| Extract from File                            | Extract from File                          | Parse CSV to JSON objects                           | GitHub                             | Qdrant Vector Store, Default Data Loader |                                                                                               |
| Token Splitter                              | Text Splitter                             | Split movie descriptions into tokens               | Default Data Loader                 | Embeddings OpenAI                   |                                                                                               |
| Default Data Loader                         | LangChain Document Loader                  | Structure movie data with metadata                  | Extract from File                  | Token Splitter                     |                                                                                               |
| Embeddings OpenAI                           | OpenAI Embeddings Node                      | Generate vector embeddings for movie descriptions  | Token Splitter                    | Qdrant Vector Store                |                                                                                               |
| Qdrant Vector Store                         | Qdrant Vector Store Node                    | Insert embeddings and metadata into Qdrant         | Embeddings OpenAI, Extract from File | None                            |                                                                                               |
| Sticky Note1                               | Sticky Note                               | Describes data upload to Qdrant                     | None                              | None                              | Uploading data (movies and their descriptions) to Qdrant Vector Store                         |
| When chat message received                  | LangChain Chat Trigger                      | Entry point for user chat input                      | None                              | AI Agent                          |                                                                                               |
| AI Agent                                   | LangChain Agent Node                       | Orchestrates chat response, calls tools and memory | When chat message received         | OpenAI Chat Model, Call n8n Workflow Tool | Tool, calling Qdrant's recommendation API based on user's request, transformed by AI agent  |
| OpenAI Chat Model                          | OpenAI Chat Completion                      | Generates AI conversational responses               | AI Agent                         | AI Agent                         |                                                                                               |
| Window Buffer Memory                       | LangChain Memory Buffer                      | Maintains chat context                               | AI Agent                         | AI Agent                         |                                                                                               |
| Call n8n Workflow Tool                     | LangChain Tool Node                         | Invokes sub-workflow for movie recommendation       | AI Agent                         | AI Agent                         |                                                                                               |
| Execute Workflow Trigger                   | Execute Workflow Trigger                    | Starts embedding and recommendation workflow        | Call n8n Workflow Tool            | Embedding Recommendation Request with Open AI, Embedding Anti-Recommendation Request with Open AI |                                                                                               |
| Embedding Recommendation Request with Open AI | HTTP Request                             | Creates embedding for positive example              | Execute Workflow Trigger           | Extracting Embedding              |                                                                                               |
| Extracting Embedding                       | Set Node                                  | Extracts positive example embedding vector          | Embedding Recommendation Request with Open AI | Merge                          |                                                                                               |
| Embedding Anti-Recommendation Request with Open AI | HTTP Request                             | Creates embedding for negative example              | Execute Workflow Trigger           | Extracting Embedding1             |                                                                                               |
| Extracting Embedding1                      | Set Node                                  | Extracts negative example embedding vector          | Embedding Anti-Recommendation Request with Open AI | Merge                          |                                                                                               |
| Merge                                     | Merge Node (combine)                       | Combines positive and negative embeddings           | Extracting Embedding, Extracting Embedding1 | Calling Qdrant Recommendation API |                                                                                               |
| Calling Qdrant Recommendation API          | HTTP Request                             | Queries Qdrant Recommendation API for top-3 movies | Merge                            | Retrieving Recommended Movies Meta Data, Split Out1 |                                                                                               |
| Retrieving Recommended Movies Meta Data    | HTTP Request                             | Fetches metadata for recommended movies              | Calling Qdrant Recommendation API | Split Out                         |                                                                                               |
| Split Out1                                | Split Out                                 | Extracts points array from recommendation response  | Calling Qdrant Recommendation API | Merge1                           |                                                                                               |
| Split Out                                 | Split Out                                 | Extracts result array from metadata response         | Retrieving Recommended Movies Meta Data | Merge1                           |                                                                                               |
| Merge1                                    | Merge Node (combine by id)                  | Combines recommendation points and metadata         | Split Out1, Split Out            | Selecting Fields Relevant for Agent |                                                                                               |
| Selecting Fields Relevant for Agent        | Set Node                                  | Selects and renames fields for AI agent              | Merge1                           | Aggregate                        |                                                                                               |
| Aggregate                                 | Aggregate Node                             | Aggregates movie recommendations into single array  | Selecting Fields Relevant for Agent | AI Agent                         |                                                                                               |
| Sticky Note                               | Sticky Note                               | Describes recommendation tool workflow               | None                              | None                              | Tool, calling Qdrant's recommendation API based on user's request, transformed by AI agent   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Test workflow’` to start data ingestion manually.

2. **Add a GitHub node** configured to:  
   - Operation: Get file  
   - Repository owner: `mrscoopers`  
   - Repository name: `n8n_demo`  
   - File path: `Top_1000_IMDB_movies.csv`  
   - Credentials: Setup GitHub API credentials.

3. **Add an Extract from File node** connected to GitHub:  
   - Use default CSV parsing options to convert file content to JSON.

4. **Add a Default Data Loader (LangChain Document Loader) node**:  
   - Input: JSON movie records from Extract from File  
   - Configure metadata fields:  
     - `movie_name` from `Movie Name` field  
     - `movie_release_date` from `Year of Release` field  
     - `movie_description` from `Description` field  
   - JSON data source: movie description field.

5. **Add a Token Splitter node** connected to Default Data Loader:  
   - Use default token-based splitting to chunk long movie descriptions.

6. **Add an Embeddings OpenAI node** connected to Token Splitter:  
   - Model: `text-embedding-3-small`  
   - Credentials: OpenAI API key.

7. **Add a Qdrant Vector Store node** connected to Embeddings OpenAI:  
   - Mode: Insert  
   - Collection: Select or create a Qdrant collection named `imdb`  
   - Credentials: Setup Qdrant API credentials.

8. **Connect Extract from File node to Qdrant Vector Store in parallel** to pass metadata for embedding storage.

9. **Add a LangChain Chat Trigger node** named `When chat message received` for live chat input:  
   - Configure webhook settings.

10. **Add an AI Agent node** connected to the chat trigger:  
    - System message: "You are a Movie Recommender Tool using a Vector Database under the hood. Provide top-3 movie recommendations returned by the database, ordered by their recommendation score, but not showing the score to the user."

11. **Add an OpenAI Chat Model node** connected to AI Agent:  
    - Model: `gpt-4o-mini`  
    - Credentials: OpenAI API key.

12. **Add a Window Buffer Memory node** connected to AI Agent for conversation context.

13. **Add a Call n8n Workflow Tool node** connected to AI Agent:  
    - Name: `movie_recommender` (this workflow's ID)  
    - Input schema: JSON with properties `positive_example` (string), `negative_example` (string).

14. **Add an Execute Workflow Trigger node** to start embedding and recommendation subworkflow.

15. **Add two HTTP Request nodes for embeddings**:  
    - One for positive example embedding request:  
      - POST to OpenAI embeddings endpoint  
      - Body with `input` as user's positive example text  
      - Model `text-embedding-3-small`  
      - Auth: OpenAI credentials  
    - One for negative example embedding request:  
      - Same configuration but input is user's negative example text.

16. **Add two Set nodes** to extract embeddings arrays from responses:  
    - One assigns `positive_example` embedding array  
    - One assigns `negative_example` embedding array.

17. **Add a Merge node** to combine the two embeddings into one JSON.

18. **Add an HTTP Request node** to call Qdrant Recommendation API:  
    - POST URL to Qdrant recommendation endpoint for `imdb_1000_open_ai` collection  
    - JSON body with positive and negative embedding arrays under `recommend` key, strategy `average_vector`, limit 3  
    - Auth: Qdrant credentials.

19. **Add an HTTP Request node** to fetch metadata for recommended movie points:  
    - POST to Qdrant points endpoint with IDs from recommendation results  
    - Request with `with_payload` set true  
    - Auth: Qdrant credentials.

20. **Add Split Out nodes** to split recommended points and metadata arrays.

21. **Add a Merge node** to combine points and metadata by matching IDs.

22. **Add a Set node** to select and rename fields for the agent:  
    - Extract `movie_recommendation_score`, `movie_description`, `movie_name`, `movie_release_year`.

23. **Add an Aggregate node** to combine the selected movie recommendations into a single array under a `response` field.

24. **Connect the aggregate output back to the AI Agent** to complete the chat response cycle.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                        |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| Full design process video explaining workflow construction and logic                                         | https://www.youtube.com/watch?v=O5mT8M7rqQQ                           |
| Qdrant Recommendation API documentation for vector similarity queries                                        | https://qdrant.tech/articles/new-recommendation-api/                  |
| IMDB Top 1000 Kaggle dataset used as data source                                                             | https://www.kaggle.com/datasets/omarhanyy/imdb-top-1000              |
| Requires free tier Qdrant cluster setup with API credentials                                                  | https://cloud.qdrant.io/                                              |
| OpenAI credentials needed for embeddings and chat model                                                      | https://platform.openai.com/account/api-keys                          |
| GitHub repository hosting the dataset must be accessible with valid GitHub API credentials                    | https://github.com/mrscoopers/n8n_demo                               |

---

This document comprehensively describes all nodes, their connections, configuration, and the logic for a reproducible RAG movie recommendation chatbot using n8n, OpenAI, and Qdrant.