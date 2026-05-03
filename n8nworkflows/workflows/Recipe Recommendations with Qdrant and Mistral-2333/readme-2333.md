Recipe Recommendations with Qdrant and Mistral

https://n8nworkflows.xyz/workflows/recipe-recommendations-with-qdrant-and-mistral-2333


# Recipe Recommendations with Qdrant and Mistral

### 1. Workflow Overview

This workflow implements a recipe recommendation chatbot leveraging Qdrant’s vector store and Mistral AI services. It scrapes HelloFresh's weekly menu, processes the recipes into vector embeddings, stores them in a Qdrant collection and an SQLite database, then enables an AI agent to interactively recommend recipes based on user preferences, including ingredients or recipes to avoid.

Logical blocks include:

- **1.1 Data Acquisition:** Fetches HelloFresh's current weekly menu and extracts available courses and detailed recipe data.
- **1.2 Recipe Document Preparation:** Combines metadata and recipe details into structured documents.
- **1.3 Document Vectorization and Storage:** Splits documents into chunks, generates embeddings using Mistral, and inserts them into Qdrant vector store.
- **1.4 Database Storage:** Saves full recipe documents in SQLite for retrieval.
- **1.5 User Interaction & Recommendation:** Uses an AI Agent powered by Mistral chat model and Qdrant's Recommend API to suggest recipes based on positive and negative user preferences.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition

- **Overview:** 
  Fetches HelloFresh's menu for the current week, extracts the available courses and individual recipe URLs, then obtains detailed recipe data from each recipe page.

- **Nodes Involved:**
  - When clicking "Test workflow"
  - Get This Week's Menu
  - Extract Server Data
  - Extract Available Courses
  - Get Course Metadata
  - Get Recipe
  - Extract Recipe Details

- **Node Details:**

  - **When clicking "Test workflow"**
    - Type: Manual Trigger
    - Role: Starts the workflow manually for testing.
    - Inputs: None
    - Outputs: Triggers "Get This Week's Menu."
    - Edge cases: None (manual start).

  - **Get This Week's Menu**
    - Type: HTTP Request
    - Role: Fetches HelloFresh weekly menu page for the current year and week.
    - Config: URL dynamically constructed as `https://www.hellofresh.co.uk/menus/{{ $now.year }}-W{{ $now.weekNumber }}`
    - Outputs: Full page HTML.
    - Possible failures: Network errors, 404 if menu unavailable.

  - **Extract Server Data**
    - Type: HTML Extract
    - Role: Extracts JSON data embedded in the page within the `<script id="__NEXT_DATA__">` tag.
    - Config: Extracts the full script content.
    - Outputs: JSON string with server-side rendered data.
    - Failures: Selector not found or JSON parse errors.

  - **Extract Available Courses**
    - Type: Code (JavaScript)
    - Role: Parses the extracted JSON and extracts the first 10 courses from `pageData.props.pageProps.ssrPayload.courses`.
    - Inputs: JSON string from previous node.
    - Outputs: Array of course objects.
    - Failures: JSON parsing or data structure changes.

  - **Get Course Metadata**
    - Type: Set
    - Role: Maps recipe metadata fields (name, cuisines, category, tags, nutrition, url, id) from the course data for downstream use.
    - Inputs: Single course object.
    - Outputs: Flattened metadata fields.
    - Expressions: Uses `.map()` and `.reduce()` to transform arrays.
    - Edge cases: Missing fields or empty arrays.

  - **Get Recipe**
    - Type: HTTP Request
    - Role: Fetches the full recipe page using the URL from course metadata.
    - Inputs: URL string.
    - Outputs: Recipe HTML content.
    - Failures: HTTP errors, page not found.

  - **Extract Recipe Details**
    - Type: HTML Extract
    - Role: Extracts recipe description, ingredients, utensils, and instructions using CSS selectors.
    - Config:
      - Description: `[data-test-id="recipe-description"]`
      - Ingredients: `[data-test-id="ingredients-list"]`
      - Utensils: `[data-test-id="utensils"]`
      - Instructions: `[data-test-id="instructions"]` excluding images and links.
    - Outputs: Structured recipe details.
    - Failures: Selector changes or missing elements.

---

#### 2.2 Recipe Document Preparation

- **Overview:**
  Merges course metadata with extracted recipe details and formats them into a single structured document string for vectorization.

- **Nodes Involved:**
  - Merge Course & Recipe
  - Prepare Documents

- **Node Details:**

  - **Merge Course & Recipe**
    - Type: Merge
    - Role: Combines course metadata and recipe details arrays by position (index).
    - Inputs: Outputs from "Get Course Metadata" and "Extract Recipe Details"
    - Outputs: Merged object with both metadata and recipe content.
    - Edge cases: Unequal array lengths.

  - **Prepare Documents**
    - Type: Set
    - Role: Constructs a formatted document string using markdown-like syntax including headers for name, description, website URL, ingredients, utensils, nutrition facts, and instructions.
    - Expressions: Uses `.replaceAll()` to clean text, `.map()` to format nutrition, and string interpolation for all fields.
    - Outputs: Document string plus metadata fields (cuisine, category, tag, week, id, name).
    - Edge cases: Missing or empty fields, special characters.

---

#### 2.3 Document Vectorization and Storage

- **Overview:**
  Splits the prepared documents into text chunks, generates embeddings with Mistral, and inserts them into the Qdrant vector store.

- **Nodes Involved:**
  - Recursive Character Text Splitter
  - Default Data Loader
  - Embeddings Mistral Cloud
  - Qdrant Vector Store

- **Node Details:**

  - **Recursive Character Text Splitter**
    - Type: Text Splitter (Recursive Character)
    - Role: Splits long recipe documents into smaller chunks suitable for embedding.
    - Inputs: Document string from "Prepare Documents".
    - Outputs: Array of text chunks.
    - Edge cases: Very short or very long texts.

  - **Default Data Loader**
    - Type: Document Loader
    - Role: Converts text chunks into document objects with associated metadata (week, cuisine, category, tag, recipe_id).
    - Inputs: Text chunks.
    - Outputs: Document objects for embedding.
    - Edge cases: Metadata missing or mismatched.

  - **Embeddings Mistral Cloud**
    - Type: Mistral Embeddings Node (Langchain)
    - Role: Generates vector embeddings for each document chunk using Mistral AI embeddings service.
    - Credentials: Requires configured Mistral Cloud account.
    - Inputs: Document objects.
    - Outputs: Embeddings attached to documents.
    - Failures: API authentication, rate limits, timeouts.

  - **Qdrant Vector Store**
    - Type: Qdrant Vectorstore Insert
    - Role: Inserts embedded documents into the `hello_fresh` collection in Qdrant.
    - Credentials: Requires Qdrant API credentials.
    - Inputs: Embedded documents.
    - Pre-requisite: Qdrant collection `hello_fresh` must exist with Cosine distance and vector size 1024.
    - Failures: Qdrant API errors, collection not found.

---

#### 2.4 Database Storage

- **Overview:**
  Stores the full original recipe documents along with metadata in an SQLite database for later retrieval.

- **Nodes Involved:**
  - Save Recipes to DB

- **Node Details:**

  - **Save Recipes to DB**
    - Type: Code (Python)
    - Role: Creates `recipes` table if not exists and inserts or replaces recipe records with id, name, full text data, cuisine, category, tag, and week.
    - Inputs: Prepared documents with metadata.
    - Outputs: Number of affected rows.
    - Failures: SQLite connection errors, data insertion conflicts.

---

#### 2.5 User Interaction & Recommendation

- **Overview:**
  Provides a chat interface for users to request recipe recommendations based on positive and negative preferences. Uses Mistral chat model and Qdrant Recommend API to produce personalized suggestions.

- **Nodes Involved:**
  - Chat Trigger
  - AI Agent
  - Qdrant Recommend API (Tool Workflow)
  - Get Mistral Embeddings
  - Use Qdrant Recommend API (HTTP Request)
  - Get Recipes From DB
  - Get Tool Response
  - Execute Workflow Trigger
  - Wait for Rate Limits
  - Mistral Cloud Chat Model

- **Node Details:**

  - **Chat Trigger**
    - Type: Langchain Chat Trigger
    - Role: Webhook that receives chat messages from users to start recommendation.
    - Webhook ID: Configured.
    - Outputs: Passes user input to AI Agent.
    - Failures: Webhook misconfiguration.

  - **AI Agent**
    - Type: Langchain Agent Node
    - Role: Orchestrates the conversational flow, calling Qdrant Recommend API tool and Mistral chat model.
    - Parameters: 
      - System message instructs it to recommend only current week's HelloFresh recipes.
    - Inputs: Chat trigger messages.
    - Outputs: Final recipe recommendations.
    - Failures: Model API errors, input parsing errors.

  - **Qdrant Recommend API (Tool Workflow)**
    - Type: Tool Workflow node
    - Role: Defines a tool interface that accepts positive and negative user prompts for recommendation.
    - Input Schema: JSON object with `positive` and `negative` string fields.
    - Description: Explains usage for positive and negative preferences.
    - Invoked by AI Agent.
    - Failures: Schema validation errors.

  - **Get Mistral Embeddings**
    - Type: HTTP Request
    - Role: Generates embeddings for the user's positive and negative query inputs.
    - URL: `https://api.mistral.ai/v1/embeddings`
    - Method: POST
    - Body: JSON with model `mistral-embed`, encoding format `float`, input is array of positive & negative queries.
    - Credentials: Mistral Cloud account.
    - Failures: API auth, rate limits.

  - **Use Qdrant Recommend API**
    - Type: HTTP Request
    - Role: Calls Qdrant's `/points/recommend/groups` endpoint to get recommended recipe groups.
    - Method: POST
    - Body Parameters:
      - strategy: average_vector
      - limit: 3
      - positive: embedding vector of positive query
      - negative: embedding vector of negative query
      - filter: matches current week's recipes
      - with_payload: true
      - group_by: metadata.recipe_id
      - group_size: 3
    - Credentials: Qdrant API
    - Failures: API errors, invalid embeddings.

  - **Get Recipes From DB**
    - Type: Code (Python)
    - Role: Queries SQLite database for full recipes matching recommended recipe IDs with score > 0.5.
    - Inputs: Recommendation results.
    - Outputs: Array of full recipe texts.
    - Failures: DB connection, query execution errors.

  - **Get Tool Response**
    - Type: Set
    - Role: Formats the retrieved recipes into a JSON string response for the AI Agent.
    - Outputs: JSON stringified recipe results.
    - Failures: Expression errors.

  - **Execute Workflow Trigger**
    - Type: Execute Workflow Trigger
    - Role: Triggers the wait node to handle rate limits before calling embedding API.
    - Outputs: Controls flow timing.

  - **Wait for Rate Limits**
    - Type: Wait
    - Role: Delays execution by 1.1 seconds to avoid API rate limiting.
    - Failures: None typical.

  - **Mistral Cloud Chat Model**
    - Type: Langchain Chat Model Node
    - Role: Provides the language model responses for AI Agent chat.
    - Model: `mistral-large-2402`
    - Credentials: Mistral Cloud account.
    - Failures: API errors, timeouts.

---

### 3. Summary Table

| Node Name                  | Node Type                                  | Functional Role                               | Input Node(s)                        | Output Node(s)                             | Sticky Note                                                                                          |
|----------------------------|--------------------------------------------|-----------------------------------------------|------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking "Test workflow" | Manual Trigger                           | Manual start trigger                           | -                                  | Get This Week's Menu                        |                                                                                                    |
| Get This Week's Menu        | HTTP Request                              | Fetch HelloFresh weekly menu                    | When clicking "Test workflow"       | Extract Server Data                         | Step 1. Fetch Available Courses For the Current Week. Scrapes HelloFresh website.                   |
| Extract Server Data         | HTML Extract                             | Extract embedded JSON from page                 | Get This Week's Menu                | Extract Available Courses                   |                                                                                                    |
| Extract Available Courses   | Code (JS)                               | Parse JSON and extract a list of courses        | Extract Server Data                 | Get Course Metadata, Get Recipe             |                                                                                                    |
| Get Course Metadata         | Set                                     | Extract and format recipe metadata              | Extract Available Courses           | Merge Course & Recipe                       |                                                                                                    |
| Get Recipe                 | HTTP Request                            | Fetch recipe page HTML                          | Extract Available Courses           | Extract Recipe Details                      |                                                                                                    |
| Extract Recipe Details      | HTML Extract                             | Extract recipe description, ingredients, etc.  | Get Recipe                        | Merge Course & Recipe                       |                                                                                                    |
| Merge Course & Recipe       | Merge                                   | Combine metadata and recipe details             | Get Course Metadata, Extract Recipe Details | Prepare Documents                       | Step 2. Create Recipe Documents For VectorStore                                                  |
| Prepare Documents           | Set                                     | Format recipe data into markdown document       | Merge Course & Recipe               | Qdrant Vector Store, Save Recipes to DB    |                                                                                                    |
| Recursive Character Text Splitter | Text Splitter (Langchain)          | Split document text into chunks                  | Prepare Documents                  | Default Data Loader                         | Step 3. Vectorise Recipes For Recommendation Engine                                              |
| Default Data Loader         | Document Loader (Langchain)               | Prepare documents with metadata for embedding   | Recursive Character Text Splitter  | Embeddings Mistral Cloud                    |                                                                                                    |
| Embeddings Mistral Cloud    | Mistral Embeddings (Langchain)            | Generate vector embeddings for documents         | Default Data Loader                | Qdrant Vector Store                         |                                                                                                    |
| Qdrant Vector Store         | Qdrant Vectorstore Insert                  | Insert embeddings into Qdrant collection          | Embeddings Mistral Cloud, Prepare Documents | (none)                                | Step 3. Vectorise Recipes For Recommendation Engine. Requires Qdrant collection 'hello_fresh' exists. |
| Save Recipes to DB          | Code (Python)                            | Save full recipe documents and metadata to SQLite | Prepare Documents                  | (none)                                    | Step 4. Save Original Document to Database                                                       |
| Chat Trigger               | Langchain Chat Trigger                    | Receive user chat input for recipe recommendations | (external webhook)                 | AI Agent                                   | Step 5. Chat with Our HelloFresh Recommendation AI Agent                                         |
| AI Agent                   | Langchain Agent                           | Conversational AI to recommend recipes           | Chat Trigger, Qdrant Recommend API | (chat response)                            |                                                                                                    |
| Qdrant Recommend API        | Tool Workflow                            | Tool interface to query Qdrant Recommend API     | AI Agent                         | AI Agent                                   | Step 5. Using Qdrant's Recommend API & Grouping Functionality                                    |
| Execute Workflow Trigger    | Execute Workflow Trigger                   | Triggers wait node for rate limiting              | AI Agent                         | Wait for Rate Limits                        |                                                                                                    |
| Wait for Rate Limits        | Wait                                    | Delays execution to prevent API rate limits      | Execute Workflow Trigger           | Get Mistral Embeddings                      |                                                                                                    |
| Get Mistral Embeddings      | HTTP Request                             | Generate embeddings for user query inputs         | Wait for Rate Limits              | Use Qdrant Recommend API                     |                                                                                                    |
| Use Qdrant Recommend API    | HTTP Request                             | Call Qdrant Recommend API with embeddings         | Get Mistral Embeddings            | Get Recipes From DB                         | Step 5. Using Qdrant's Recommend API & Grouping Functionality                                    |
| Get Recipes From DB         | Code (Python)                            | Retrieve full recipes from SQLite by recommended IDs | Use Qdrant Recommend API           | Get Tool Response                          |                                                                                                    |
| Get Tool Response           | Set                                     | Format retrieved recipes as JSON string response  | Get Recipes From DB               | AI Agent                                   |                                                                                                    |
| Mistral Cloud Chat Model    | Langchain Chat Model                      | Provide AI chat responses                          | AI Agent                         | AI Agent                                   |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Name: `When clicking "Test workflow"`
   - No special configuration.

2. **Create HTTP Request to Get Weekly Menu**
   - Type: HTTP Request
   - Name: `Get This Week's Menu`
   - URL: `https://www.hellofresh.co.uk/menus/{{ $now.year }}-W{{ $now.weekNumber }}`
   - Method: GET
   - Connect output of Manual Trigger to this node.

3. **Extract Server Data**
   - Type: HTML Extract
   - Name: `Extract Server Data`
   - Operation: Extract HTML Content
   - Extraction Values: Extract content of `script#__NEXT_DATA__`
   - Input: Connect from `Get This Week's Menu`

4. **Extract Available Courses**
   - Type: Code (JavaScript)
   - Name: `Extract Available Courses`
   - Code:
     ```javascript
     const pageData = JSON.parse($input.first().json.data);
     return pageData.props.pageProps.ssrPayload.courses.slice(0, 10);
     ```
   - Input: Connect from `Extract Server Data`

5. **Get Course Metadata**
   - Type: Set
   - Name: `Get Course Metadata`
   - Assign fields:
     - `name`: `={{ $json.recipe.name }}`
     - `cuisines`: `={{ $json.recipe.cuisines.map(item => item.name) }}`
     - `category`: `={{ $json.recipe.category.name }}`
     - `tags`: `={{ $json.recipe.tags.flatMap(tag => tag.preferences) }}`
     - `nutrition`: `={{ $json.recipe.nutrition.reduce((acc,item) => ({ ...acc, [item.name]: item.amount + item.unit }), {}) }}`
     - `url`: `={{ $json.recipe.websiteUrl }}`
     - `id`: `={{ $json.recipe.id }}`
   - Input: Connect from `Extract Available Courses`

6. **Get Recipe**
   - Type: HTTP Request
   - Name: `Get Recipe`
   - URL: `={{ $json.recipe.websiteUrl }}`
   - Method: GET
   - Input: Connect from `Extract Available Courses`

7. **Extract Recipe Details**
   - Type: HTML Extract
   - Name: `Extract Recipe Details`
   - Extraction:
     - Description: `[data-test-id="recipe-description"]`
     - Ingredients: `[data-test-id="ingredients-list"]`
     - Utensils: `[data-test-id="utensils"]`
     - Instructions: `[data-test-id="instructions"]` (exclude `img` and `a` tags)
   - Input: Connect from `Get Recipe`

8. **Merge Course & Recipe**
   - Type: Merge
   - Name: `Merge Course & Recipe`
   - Mode: Combine
   - Combination Mode: Merge by position
   - Input 1: Connect from `Get Course Metadata`
   - Input 2: Connect from `Extract Recipe Details`

9. **Prepare Documents**
   - Type: Set
   - Name: `Prepare Documents`
   - Assignments:
     - `data`: Template string with markdown format for name, description, website URL, ingredients, utensils, nutrition, instructions.
     - `cuisine`, `category`, `tag`, `week`, `id`, `name`: mapped from merged data.
   - Input: Connect from `Merge Course & Recipe`

10. **Recursive Character Text Splitter**
    - Type: Langchain Text Splitter (Recursive Character)
    - Name: `Recursive Character Text Splitter`
    - Input: Connect from `Prepare Documents`

11. **Default Data Loader**
    - Type: Langchain Document Default Data Loader
    - Name: `Default Data Loader`
    - Metadata fields: `week`, `cuisine`, `category`, `tag`, `recipe_id`
    - Input: Connect from `Recursive Character Text Splitter`

12. **Embeddings Mistral Cloud**
    - Type: Langchain Embeddings Mistral Cloud
    - Name: `Embeddings Mistral Cloud`
    - Credential: Configure Mistral Cloud API credentials
    - Input: Connect from `Default Data Loader`

13. **Qdrant Vector Store**
    - Type: Langchain Vectorstore Qdrant
    - Name: `Qdrant Vector Store`
    - Mode: Insert
    - Collection: `hello_fresh` (Must pre-exist in Qdrant with Cosine distance and vector size 1024)
    - Credential: Configure Qdrant API credentials
    - Inputs: Connect `Embeddings Mistral Cloud` (embedding input) and `Prepare Documents` (ai_document input)

14. **Save Recipes to DB**
    - Type: Code (Python)
    - Name: `Save Recipes to DB`
    - Code to create SQLite DB and insert/update recipe records with id, name, data, cuisine, category, tag, week.
    - Input: Connect from `Prepare Documents`

15. **Chat Trigger**
    - Type: Langchain Chat Trigger
    - Name: `Chat Trigger`
    - Configure webhook ID for interaction
    - Input: External user requests

16. **AI Agent**
    - Type: Langchain Agent
    - Name: `AI Agent`
    - System Message: Instruct to recommend only current week's HelloFresh recipes.
    - Input: Connect from `Chat Trigger`
    - Output: Chat responses

17. **Qdrant Recommend API (Tool Workflow)**
    - Type: Tool Workflow Node
    - Name: `Qdrant Recommend API`
    - Define input schema with `positive` and `negative` strings for user preferences.
    - Input: Invoked by AI Agent as a tool

18. **Execute Workflow Trigger**
    - Type: Execute Workflow Trigger
    - Name: `Execute Workflow Trigger`
    - Used to control flow before embeddings API call

19. **Wait for Rate Limits**
    - Type: Wait
    - Name: `Wait for Rate Limits`
    - Duration: 1.1 seconds
    - Input: Connect from `Execute Workflow Trigger`

20. **Get Mistral Embeddings**
    - Type: HTTP Request
    - Name: `Get Mistral Embeddings`
    - URL: `https://api.mistral.ai/v1/embeddings`
    - Method: POST
    - Body: JSON with model `mistral-embed`, encoding_format `float`, input array of positive and negative queries
    - Credential: Mistral Cloud API
    - Input: Connect from `Wait for Rate Limits`

21. **Use Qdrant Recommend API**
    - Type: HTTP Request
    - Name: `Use Qdrant Recommend API`
    - URL: `http://qdrant:6333/collections/hello_fresh/points/recommend/groups`
    - Method: POST
    - Body:
      - strategy: average_vector
      - limit: 3
      - positive: embedding vector from positive query
      - negative: embedding vector from negative query
      - filter: metadata.week matches current week from workflow context
      - with_payload: true
      - group_by: metadata.recipe_id
      - group_size: 3
    - Credential: Qdrant API
    - Input: Connect from `Get Mistral Embeddings`

22. **Get Recipes From DB**
    - Type: Code (Python)
    - Name: `Get Recipes From DB`
    - Query SQLite using recommended recipe IDs with score > 0.5
    - Input: Connect from `Use Qdrant Recommend API`

23. **Get Tool Response**
    - Type: Set
    - Name: `Get Tool Response`
    - Convert recipe results into JSON string response
    - Input: Connect from `Get Recipes From DB`

24. **Mistral Cloud Chat Model**
    - Type: Langchain Chat Model Node
    - Name: `Mistral Cloud Chat Model`
    - Model: `mistral-large-2402`
    - Credential: Mistral Cloud API
    - Input: Connect from `AI Agent`

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Step 1. Fetch Available Courses For the Current Week: Scrapes HelloFresh website’s weekly menu, which can be large. Please be patient.                                                                                                                            | Sticky Note on nodes: When clicking Test workflow, Get This Week's Menu, Extract Server Data, Extract Available Courses |
| Step 2. Create Recipe Documents For VectorStore: Combines metadata and recipe details into markdown-like formatted documents for vectorization.                                                                                                                   | Sticky Note on nodes: Merge Course & Recipe, Prepare Documents                                                |
| Step 3. Vectorise Recipes For Recommendation Engine: Uses Mistral embeddings and inserts documents into Qdrant vector store.                                                                                                                                       | Sticky Note on nodes: Recursive Character Text Splitter, Default Data Loader, Embeddings Mistral Cloud, Qdrant Vector Store |
| Step 4. Save Original Document to Database: Stores full recipes in SQLite database for full retrieval by agent.                                                                                                                                                      | Sticky Note on node: Save Recipes to DB                                                                       |
| Step 5. Chat with Our HelloFresh Recommendation AI Agent: AI Agent uses Qdrant Recommend API with positive/negative preferences to recommend current week’s recipes only.                                                                                           | Sticky Note on nodes: Chat Trigger, AI Agent, Qdrant Recommend API, Use Qdrant Recommend API                    |
| Step 5. Using Qdrant's Recommend API & Grouping Functionality: The Recommend API allows specifying both positive and negative queries, improving recommendation relevance. Grouping ensures unique recipes are returned.                                              | Sticky Note on node: Use Qdrant Recommend API                                                                  |
| Ensure Qdrant collection exists with appropriate vector size and distance metric before running workflow.                                                                                                                                                            | Sticky Note on node: Qdrant Vector Store                                                                       |
| Try it out! Workflow fetches and stores HelloFresh's weekly menu, builds recommendation engine with Qdrant and SQLite, and enables AI chat interface. Join n8n Discord or Community Forum for help.                                                                    | Sticky Note covering overall workflow at top-left corner                                                       |
| Configure your Qdrant connection carefully, including endpoint address.                                                                                                                                                                                             | Sticky Note near Use Qdrant Recommend API                                                                      |
| Qdrant collection creation command example: <br>```PUT collections/hello_fresh { "vectors": { "distance": "Cosine", "size": 1024 } }```                                                                                                                             | Sticky Note near Qdrant Vector Store                                                                            |
| Documentation Links:<br> - [Qdrant node docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreqdrant)<br> - [Code node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code)<br> - [AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br> - [Qdrant Recommend API](https://qdrant.tech/documentation/concepts/explore/?q=recommend) | Sticky Notes with helpful links on vectorstore, code node, AI agent, and Qdrant Recommend API                    |

---

This document provides a comprehensive, detailed overview and technical breakdown for understanding, modifying, or reproducing the "Recipe Recommendations with Qdrant and Mistral" n8n workflow.