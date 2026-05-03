Content Farming v4 - Blog Automation for Wordpress using ChatGPT 5 and Gemini

https://n8nworkflows.xyz/workflows/content-farming-v4---blog-automation-for-wordpress-using-chatgpt-5-and-gemini-10079


# Content Farming v4 - Blog Automation for Wordpress using ChatGPT 5 and Gemini

---

## 1. Workflow Overview

This workflow, titled **Content Farming v4 - Blog Automation for Wordpress using ChatGPT 5 and Gemini**, is designed for automated content generation, enrichment, quality assessment, and publication targeted at WordPress blogs. Its core purpose is to fetch news and data from various sources, perform AI-driven processing to generate and enhance blog articles, and then publish them on WordPress while maintaining content quality and metadata. It leverages advanced AI models (ChatGPT 5 variants and Gemini), MongoDB Atlas as vector store and document database, and integrates multiple external tools like RSS feeds, Google Sheets, Twitter, and dev.to.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Scheduled triggers, RSS feed reading, and Google Sheets data ingestion.
- **1.2 Data Preparation & Storage:** Filtering, normalization, splitting text, and storing data in MongoDB and vector stores.
- **1.3 AI-Driven Content Processing:** Multiple AI agents performing tasks such as article intelligence, task definition, content writing, quality checking, blogging title generation, metadata generation, content modification, and quality validation.
- **1.4 Content Publishing & Social Sharing:** Publishing posts to WordPress, updating posts, setting metadata and images, and creating tweets.
- **1.5 Auxiliary Tools & Utilities:** HTTP requests, crypto operations, iterative loops, and error handling.
- **1.6 Monitoring & Cleanup:** Deleting outdated news chunks, chat history, and managing data consistency.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception

**Overview:**  
This block initiates the workflow on schedule and ingests various inputs from RSS feeds and Google Sheets, which contain news articles, company profiles, online profiles, and categories. It triggers data ingestion and prepares it for further processing.

**Nodes Involved:**

- Get Articles Daily (Schedule Trigger)
- Get rss feed (Google Sheets)
- Split Out (split incoming data into items)
- Read RSS News Feeds (RSS Feed Read)
- Filter (Data filtering node)
- Get categories (Google Sheets)
- Get categories2 (Google Sheets)
- get company profile (Google Sheets)
- get online profiles (Google Sheets)
- Get Articles Daily1, 3, 5, 7, 8 (Schedule Triggers)

**Node Details:**

- **Get Articles Daily (Schedule Trigger):** Triggers workflow execution at defined intervals (default cron schedule).  
  - Inputs: None.  
  - Outputs: Triggers "Get rss feed".  
  - Edge cases: Missed trigger if n8n is down, delayed scheduling.

- **Get rss feed (Google Sheets):** Reads RSS feed URLs or news data from Google Sheets.  
  - Inputs: Trigger from schedule.  
  - Outputs: Splits data in "Split Out".  
  - Edge cases: Google Sheets API quota, authentication issues.

- **Split Out:** Splits the batch data into individual items for processing.  
  - Inputs: Bulk data from Google Sheets or RSS feed.  
  - Outputs: Individual news items to "Read RSS News Feeds".  
  - Edge cases: Empty inputs.

- **Read RSS News Feeds:** Fetches RSS feeds from URLs provided, extracting articles.  
  - Inputs: RSS feed URLs from "Split Out".  
  - Outputs: Articles data for filtering.  
  - Edge cases: Invalid RSS URLs, network failures.

- **Filter:** Filters incoming articles based on criteria (e.g., date, relevance).  
  - Inputs: Articles from RSS feeds.  
  - Outputs: Filtered articles for normalization.  
  - Edge cases: Incorrect filter expressions, malformed data.

- **Get categories, Get categories2 (Google Sheets):** Retrieve category data for articles, used later for metadata and classification.  
  - Inputs: Triggered from aggregates and AI agents.  
  - Outputs: Category info for processing.  
  - Edge cases: API limits, data consistency.

- **get company profile / get online profiles:** Fetches company and online profiles data from Sheets, for use in enriching articles.  
  - Inputs: From scheduled triggers.  
  - Outputs: Data to MongoDB9 and further processing.  
  - Edge cases: Missing profiles, API errors.

---

### 1.2 Data Preparation & Storage

**Overview:**  
Processes filtered input data by normalizing fields, splitting text for processing, and storing in MongoDB databases and vector stores to enable fast retrieval and embedding-based search.

**Nodes Involved:**

- Set and Normalize Fields
- Loop Over Items2
- Recursive Character Text Splitter
- Default Data Loader
- MongoDB Atlas Vector Store
- MongoDB Atlas Vector Store1
- Insert documents / Insert documents1
- Update documents
- delete news chunks
- delete news articles
- delete chat history

**Node Details:**

- **Set and Normalize Fields:** Standardizes article fields (title, date, URL, author) for consistency.  
  - Inputs: Filtered articles.  
  - Outputs: Normalized items for batching.  
  - Edge cases: Missing fields, null values.

- **Loop Over Items2:** Processes data in batches for scalable database operations.  
  - Inputs: Normalized articles.  
  - Outputs: Batched articles to HTTP Request and MongoDB insertion.  
  - Edge cases: Large batch sizes may cause timeouts.

- **Recursive Character Text Splitter:** Splits large text content recursively into smaller chunks for AI processing.  
  - Inputs: Article content from MongoDB or feeds.  
  - Outputs: Text chunks for embedding or AI agents.  
  - Edge cases: Very large texts, malformed content.

- **Default Data Loader:** Loads data into Langchain-compatible format from MongoDB.  
  - Inputs: Text chunks.  
  - Outputs: Structured data for vector store insertion.  
  - Edge cases: Data format issues.

- **MongoDB Atlas Vector Store / MongoDB Atlas Vector Store1:** Store text embeddings and document metadata in MongoDB Atlas using vector search capabilities.  
  - Inputs: Embeddings from OpenAI.  
  - Outputs: Indexed data for retrieval.  
  - Edge cases: Connection issues, quota limits.

- **Insert documents / Insert documents1:** Insert processed data/documents into MongoDB collections.  
  - Inputs: Processed articles or chains.  
  - Outputs: Confirmation of insert.  
  - Edge cases: Duplicate keys, insertion errors.

- **Update documents:** Updates existing MongoDB documents with new data or status flags.  
  - Inputs: Article or post updates.  
  - Outputs: Updated records in DB.  
  - Edge cases: Document not found.

- **delete news chunks / delete news articles / delete chat history:** Cleanup nodes removing outdated chunks, articles, and chat history from MongoDB to maintain database hygiene.  
  - Inputs: Criteria for deletion.  
  - Outputs: Confirmation of deletions.  
  - Edge cases: Incorrect deletion filters causing data loss.

---

### 1.3 AI-Driven Content Processing

**Overview:**  
This is the core AI logic block where multiple Langchain agents and OpenAI models are used to analyze articles, generate titles, write content, check quality, and modify content iteratively using structured output parsers and custom code nodes for decision making and ranking.

**Nodes Involved:**

- AI Agent, AI Agent1, AI Agent12
- Article Intelligence Agent
- Task Definition Agent
- Content Writer Agent
- Quality Check Agent
- Blog title generator
- Meta data generator
- Content modifier
- OpenAI Chat Model1, 2, 3, gpt-5-mini, gpt-5-nano, chat gpt 4o
- Structured Output Parser, Structured Output Parser1-8, Structured Output Parser16
- Code, Code1-Code27, Code33, Code40, Code41
- Aggregate, Aggregate1-4
- If, If1, If2, If3, If4, If content meets threshold
- Loop Over Items, Loop Over Items1, Loop Over Items3
- Limit1
- Memory

**Node Details:**

- **AI Agents:** Serve as orchestrators for Langchain AI chains; each agent specializes in a task:
  - **Article Intelligence Agent:** Analyzes articles and extracts insights.
  - **Task Definition Agent:** Defines content generation tasks.
  - **Content Writer Agent:** Generates article content using LLMs.
  - **Quality Check Agent:** Assesses content quality against thresholds.
  - **Blog title generator:** Produces blog post title ideas.
  - **Meta data generator:** Generates metadata for articles.
  - **Content modifier:** Edits or improves content based on feedback.
  - **AI Agent & AI Agent1:** General AI processing with custom logic.

- **OpenAI Chat Models:** Various versions of ChatGPT 5 and GPT-4 models used as language models for generation and processing tasks.

- **Structured Output Parsers:** Parse AI agent responses into structured JSON or data objects for further processing.

- **Code Nodes:** Custom JavaScript code snippets handling ranking, decision-making, data transformation, and error handling.

- **Aggregate Nodes:** Combine data for batch processing or summarization.

- **If Nodes:** Conditional branching based on data or AI output, e.g., checking if content meets quality thresholds.

- **Loop Over Items Nodes:** Batch processing and iteration over items for scalable processing.

- **Memory Node:** MongoDB-backed chat memory for context preservation across AI agent calls.

- **Limit1:** Limits batch size or execution concurrency to avoid resource exhaustion.

**Potential Failure Modes:**  
- AI API rate limits or timeouts.  
- Parsing errors if AI response format deviates.  
- Code node exceptions due to unexpected data.  
- Conditional branches failing due to missing variables.  
- Memory node connectivity issues.

---

### 1.4 Content Publishing & Social Sharing

**Overview:**  
This block manages final content publication on WordPress, updating posts, adding images and metadata, and sharing on social media (Twitter) and developer platforms (dev.to).

**Nodes Involved:**

- Create a post1 (WordPress)
- Update a post (WordPress)
- Set Image (HTTP Request)
- Set metatag (HTTP Request)
- Create Tweet (Twitter)
- dev.to (HTTP Request)
- Edit Fields1
- If2
- Code21, Code23, Code26, Code27

**Node Details:**

- **Create a post1:** Publishes new blog posts to WordPress.  
  - Inputs: Finalized content, metadata, images.  
  - Outputs: Post ID and confirmation.  
  - Edge cases: WordPress API auth failures, content validation errors.

- **Update a post:** Updates existing WordPress posts with new content or metadata.  
  - Inputs: Post ID and updated fields.  
  - Outputs: Updated post confirmation.  
  - Edge cases: Post not found, permission errors.

- **Set Image:** Requests or uploads images related to the post, possibly through an external API.  
  - Inputs: Image URLs or generation requests.  
  - Outputs: Image metadata for setting on WordPress.  
  - Edge cases: Image generation failures.

- **Set metatag:** Sets SEO metadata on posts via HTTP API calls.  
  - Inputs: SEO data from metadata generation.  
  - Outputs: Confirmation response.  
  - Edge cases: Metadata format errors.

- **Create Tweet:** Posts tweets linking to the blog post for social promotion.  
  - Inputs: Tweet text and URLs.  
  - Outputs: Tweet confirmation.  
  - Edge cases: Twitter API limits or authentication errors.

- **dev.to:** Posts or updates articles on dev.to platform via API.  
  - Inputs: Article data.  
  - Outputs: Confirmation or error.  
  - Edge cases: API quota or content policy violations.

- **Edit Fields1:** Adjusts data fields before final publishing steps.  
  - Inputs: Metadata and content fields.  
  - Outputs: Modified fields passed downstream.  
  - Edge cases: Missing fields.

- **If2:** Conditional logic to decide whether to proceed with post creation.  
  - Inputs: Quality or content flags.  
  - Outputs: Branches to create post or halt.  
  - Edge cases: Incorrect conditions leading to skipped publishing.

---

### 1.5 Auxiliary Tools & Utilities

**Overview:**  
Supporting nodes providing utilities such as HTTP requests, cryptographic operations, link checking, and external workflow calls.

**Nodes Involved:**

- HTTP Request4
- Crypto
- Code17
- Code2
- check link status (Tool Workflow)
- search the web (Tool Workflow)
- Generate image (Tool Workflow)
- Generate Graph (Tool Workflow)
- Call 'gemini' (Execute Workflow)

**Node Details:**

- **HTTP Request4:** Generic HTTP request node used for external API calls, with error continuation enabled.  
  - Inputs: API endpoints and payloads.  
  - Outputs: API responses.  
  - Edge cases: API timeouts, invalid URLs.

- **Crypto:** Performs cryptographic operations (hashing, encryption) on data for security or ID generation.  
  - Inputs: Text or data to encrypt/hash.  
  - Outputs: Hashed or encrypted value.  
  - Edge cases: Unsupported algorithms.

- **Tool Workflows:**  
  - **check link status:** Validates if URLs are reachable and valid.  
  - **search the web:** Performs web searches to augment content.  
  - **Generate image:** Generates images via AI or external services.  
  - **Generate Graph:** Creates graphs or visual data representations.  
  - These nodes are integrated as Langchain tools, invoked by AI agents.

- **Call 'gemini':** Executes a sub-workflow named 'gemini' for advanced AI processing or external tasks.  
  - Inputs: Data from AI Agent1.  
  - Outputs: Data back to main workflow.  
  - Edge cases: Sub-workflow failures or timeout.

---

### 1.6 Monitoring & Cleanup

**Overview:**  
Manages cleanup of obsolete data and maintenance of the MongoDB database to keep the system efficient and up-to-date.

**Nodes Involved:**

- delete news chunks
- delete news articles
- delete chat history
- Limit1

**Node Details:**

- **delete news chunks / delete news articles:** Removes processed or outdated content chunks and articles from MongoDB collections.  
  - Inputs: Criteria to identify stale data.  
  - Outputs: Deletion confirmations.  
  - Edge cases: Accidental deletions if filters are misconfigured.

- **delete chat history:** Clears stored chat memory history to maintain privacy and reduce storage.  
  - Inputs: Chat session identifiers or timestamps.  
  - Outputs: Confirmation of deletion.  
  - Edge cases: Loss of conversation context.

- **Limit1:** Throttles execution to limit concurrency or batch size during cleanup or processing.  
  - Inputs: Stream of items.  
  - Outputs: Limited set for processing.  
  - Edge cases: Bottlenecks or delays if limit too low.

---

## 3. Summary Table

| Node Name                | Node Type                         | Functional Role                          | Input Node(s)                    | Output Node(s)                    | Sticky Note                                       |
|--------------------------|----------------------------------|----------------------------------------|---------------------------------|----------------------------------|--------------------------------------------------|
| Get Articles Daily        | Schedule Trigger                 | Initiates workflow on schedule         | None                            | Get rss feed                     |                                                  |
| Get rss feed             | Google Sheets                   | Reads RSS feed URLs/news data          | Get Articles Daily              | Split Out                       |                                                  |
| Split Out                | Split Out                       | Splits batch data into individual items| Get rss feed                   | Read RSS News Feeds             |                                                  |
| Read RSS News Feeds      | RSS Feed Read                   | Retrieves articles from RSS URLs       | Split Out                      | Filter                         |                                                  |
| Filter                   | Filter                         | Filters articles based on criteria     | Read RSS News Feeds             | Set and Normalize Fields        |                                                  |
| Set and Normalize Fields | Set                            | Normalizes fields for consistency      | Filter                         | Loop Over Items2                |                                                  |
| Loop Over Items2         | Split In Batches                | Batches data for scalable processing   | Set and Normalize Fields        | HTTP Request4, Aggregate        |                                                  |
| HTTP Request4            | HTTP Request                   | External API calls                      | Loop Over Items2                | Code17, Loop Over Items2        |                                                  |
| Code17                   | Code                           | Custom processing                       | HTTP Request4                  | Loop Over Items2                |                                                  |
| Recursive Character Text Splitter | Text Splitter (Recursive)      | Splits large text into smaller chunks  | Default Data Loader            | Default Data Loader output      |                                                  |
| Default Data Loader      | Document Data Loader           | Loads data for Langchain processing    | Recursive Character Text Splitter | MongoDB Atlas Vector Store     |                                                  |
| MongoDB Atlas Vector Store | Vector Store MongoDB Atlas     | Stores embeddings and documents         | Insert documents, Insert documents1 | Loop Over Items2              |                                                  |
| Insert documents         | MongoDB Insert                 | Inserts documents into MongoDB          | Code2                         | MongoDB Atlas Vector Store      |                                                  |
| Insert documents1        | MongoDB Insert                 | Inserts documents into MongoDB          | Code                         | Loop Over Items                 |                                                  |
| Update documents         | MongoDB Update                 | Updates existing MongoDB documents      | Code1, Code5                  | Loop Over Items                 |                                                  |
| delete news chunks       | MongoDB Delete                 | Deletes old news chunks                  | Loop Over Items                | delete news articles            |                                                  |
| delete news articles     | MongoDB Delete                 | Deletes old news articles                | delete news chunks             |                              |                                                  |
| delete chat history      | MongoDB Delete                 | Deletes old chat history                 | Code22                       |                              |                                                  |
| Get categories           | Google Sheets                 | Retrieves category data                  | Aggregate                     |                              |                                                  |
| Get categories2          | Google Sheets                 | Retrieves additional category data      | Aggregate2                    | Edit Fields1                   |                                                  |
| get company profile      | Google Sheets                 | Fetches company profile data             | Get Articles Daily5            | get online profiles            |                                                  |
| get online profiles      | Google Sheets                 | Fetches online profiles                   | get company profile            | MongoDB9                      |                                                  |
| AI Agent                 | Langchain Agent               | General AI processing                    | Split Out4                   | Aggregate2                    |                                                  |
| AI Agent1                | Langchain Agent               | AI processing with Gemini call          | MongoDB8                     | Call 'gemini'                 |                                                  |
| AI Agent12               | Langchain Agent               | AI agent for content generation          | Aggregate4                   | Split Out2                   |                                                  |
| Article Intelligence Agent | Langchain Agent               | Analyzes article content                 | Aggregate3                   | Get many posts               |                                                  |
| Task Definition Agent    | Langchain Agent               | Defines content generation tasks         | Code10                       | Split Out3                   |                                                  |
| Content Writer Agent     | Langchain Agent               | Writes and modifies article content      | Structured Output Parser      | Code13                      |                                                  |
| Quality Check Agent      | Langchain Agent               | Checks article quality                    | MongoDB14                    | If content meets threshold    |                                                  |
| Blog title generator     | Langchain Agent               | Generates blog post titles                | MongoDB3                     | Code7                       |                                                  |
| Meta data generator      | Langchain Agent               | Generates metadata                        | MongoDB5                     | If3                         |                                                  |
| Content modifier         | Langchain Agent               | Modifies and improves content             | Loop Over Items3             | Code4                       |                                                  |
| OpenAI Chat Model1       | LLM Chat OpenAI               | Language model for content writing        | Content Writer Agent         |                              |                                                  |
| OpenAI Chat Model2       | LLM Chat OpenAI               | Language model for task definition         | Task Definition Agent        |                              |                                                  |
| OpenAI Chat Model3       | LLM Chat OpenAI               | Language model for AI Agent processing     | AI Agent                    |                              |                                                  |
| gpt-5-mini               | LLM Chat OpenAI               | Mini version of GPT-5                      | Multiple AI Agents           |                              |                                                  |
| gpt-5-nano               | LLM Chat OpenAI               | Nano version of GPT-5                      | Multiple AI Agents           |                              |                                                  |
| chat gpt 4o              | LLM Chat OpenAI               | GPT-4 model for content modification       | Content modifier            |                              |                                                  |
| Structured Output Parser | Output Parser Langchain       | Parses AI outputs into structured format   | Multiple agents              | Multiple nodes                |                                                  |
| Code                     | Code Node                    | Custom JS logic for ranking, decisions     | Multiple inputs              | MongoDB / AI Agents          | Multiple notes: "rank and choose the best one"   |
| Aggregate                | Aggregate Node               | Aggregates data for batch processing        | Loop outputs                | Google Sheets / AI Agents    |                                                  |
| If                       | If Node                     | Conditional branching                       | Various                     | Multiple nodes               |                                                  |
| Limit1                   | Limit Node                  | Limits batch size or concurrency            | Loop outputs                | MongoDB29                   |                                                  |
| Memory                   | MongoDB-backed Chat Memory  | Stores AI chat context                       | Content Writer Agent        |                              |                                                  |
| Create a post1           | WordPress                   | Publishes new posts                          | If2                        | Set Image                   |                                                  |
| Update a post            | WordPress                   | Updates existing posts                        | MongoDB23                  | Code14                      |                                                  |
| Set Image                | HTTP Request                | Sets images for posts                         | Create a post1              | Set metatag                 |                                                  |
| Set metatag              | HTTP Request                | Adds metadata to posts                        | Set Image                  | Code23                      |                                                  |
| Create Tweet             | Twitter                    | Publishes tweet about new posts              | Basic LLM Chain             | Code26                      |                                                  |
| dev.to                   | HTTP Request                | Publishes articles to dev.to platform         | Code21                     | Code27                      |                                                  |
| check link status        | Tool Workflow              | Checks URL validity                           | search the web, check link status | Content Writer Agent, Content modifier |                                                  |
| search the web           | Tool Workflow              | Web search for content augmentation            | Content Writer Agent, Content modifier |                              |                                                  |
| Generate image           | Tool Workflow              | Generates images for content                    | Content Writer Agent        |                              |                                                  |
| Generate Graph           | Tool Workflow              | Generates visual graphs for content             | Content Writer Agent        |                              |                                                  |
| Call 'gemini'            | Execute Workflow           | Invokes sub-workflow 'gemini' for AI processing | AI Agent1                 | Code12                      |                                                  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Scheduled Triggers:**  
   - Create nodes of type `Schedule Trigger` named "Get Articles Daily", "Get Articles Daily1", "Get Articles Daily3", "Get Articles Daily5", "Get Articles Daily7", and "Get Articles Daily8". Configure the schedule according to your desired frequency.

2. **Set Up Data Input Nodes:**  
   - Create `Google Sheets` nodes named "Get rss feed", "Get categories", "Get categories2", "get company profile", and "get online profiles" configured to read respective sheets containing RSS URLs, categories, company profiles, and online profiles.  
   - Connect "Get Articles Daily" to "Get rss feed".  
   - Connect "Get Articles Daily5" to "get company profile", and link "get company profile" to "get online profiles".

3. **Create RSS Feed Reading Pipeline:**  
   - Add a `Split Out` node ("Split Out") after "Get rss feed" to split batch data.  
   - Add `RSS Feed Read` node ("Read RSS News Feeds") connected to "Split Out" to fetch articles.  
   - Add `Filter` node ("Filter") connected to "Read RSS News Feeds" to filter articles based on criteria.  
   - Connect "Filter" to `Set` node ("Set and Normalize Fields") to normalize fields.

4. **Batch Processing Setup:**  
   - Add `Split In Batches` node ("Loop Over Items2") connected to "Set and Normalize Fields" for batch processing.  
   - Connect "Loop Over Items2" to `HTTP Request` node ("HTTP Request4") and to aggregate nodes as needed.

5. **Text Splitting and Data Loading:**  
   - Add `Recursive Character Text Splitter` connected to a `Default Data Loader` node for splitting and loading text documents.  
   - Connect loader to MongoDB Atlas vector store nodes ("MongoDB Atlas Vector Store" and "MongoDB Atlas Vector Store1").  
   - Connect embedding nodes (OpenAI embedding nodes) to feed embeddings to vector stores.

6. **MongoDB Storage Setup:**  
   - Create `MongoDB` nodes for inserts ("Insert documents", "Insert documents1"), updates ("Update documents"), and deletes ("delete news chunks", "delete news articles", "delete chat history"). Configure all with correct collection and credentials.  
   - Connect these nodes accordingly after processing steps and batch loops.

7. **AI Agent Setup:**  
   - Create Langchain `Agent` nodes for "AI Agent", "AI Agent1", "AI Agent12", "Article Intelligence Agent", "Task Definition Agent", "Content Writer Agent", "Quality Check Agent", "Blog title generator", "Meta data generator", and "Content modifier".  
   - Configure each agent with appropriate prompts or chains targeting their roles.  
   - Connect structured output parser nodes to each agent output for parsing AI responses.

8. **OpenAI Chat Models:**  
   - Add OpenAI Chat nodes named "OpenAI Chat Model1", "OpenAI Chat Model2", "OpenAI Chat Model3", "gpt-5-mini", "gpt-5-nano", and "chat gpt 4o".  
   - Configure them with OpenAI credentials and select model versions accordingly.

9. **Custom Code Nodes:**  
   - Add all `Code` nodes (Code, Code1…Code27, Code33, Code40, Code41), implementing necessary JavaScript logic for ranking, decision-making, and data transformation.  
   - Connect these nodes as per the flow, often after AI agents and database nodes.

10. **Conditional Logic:**  
    - Add `If` nodes (If, If1, If2, If3, If4, If content meets threshold) to handle branching based on content quality, data presence, or process outcomes.

11. **Publishing Nodes:**  
    - Create WordPress nodes "Create a post1" and "Update a post" configured with WordPress OAuth2 credentials.  
    - Add HTTP Request nodes "Set Image" and "Set metatag" for managing post images and metadata.  
    - Connect "Create a post1" to "Set Image" and then to "Set metatag".

12. **Social Media Integration:**  
    - Create `Twitter` node "Create Tweet" configured with Twitter API credentials.  
    - Connect it downstream from content publishing or quality checks.

13. **External Platform Posting:**  
    - Add HTTP Request node "dev.to" configured to post articles to the dev.to API.  
    - Connect it to appropriate code nodes to prepare content.

14. **Auxiliary Tools:**  
    - Setup Langchain tool workflows: "check link status", "search the web", "Generate image", and "Generate Graph".  
    - Connect these as AI agent tools to augment content generation and validation.

15. **Sub-Workflow:**  
    - Import or create the sub-workflow named "gemini".  
    - Connect "AI Agent1" to "Call 'gemini'" execute workflow node.

16. **Memory Node:**  
    - Configure the MongoDB-backed "Memory" node to store AI chat context, connected to "Content Writer Agent" for contextual awareness.

17. **Loop and Batch Controls:**  
    - Add `Split Out` and `Split In Batches` nodes to handle iterative processing over datasets.  
    - Use `Limit` nodes to control concurrency and batch sizes.

18. **Credentials:**  
    - Configure all credentials:
      - OpenAI (for all AI-related nodes)
      - MongoDB Atlas (for all MongoDB nodes)
      - Google Sheets OAuth2 (for all Sheets nodes)
      - WordPress OAuth2 (for WordPress nodes)
      - Twitter API (for Twitter node)
      - HTTP API keys as required (for image and metadata services)

19. **Error Handling & Retries:**  
    - Enable retry on fail where appropriate (e.g., AI agents, HTTP requests). Use "continueErrorOutput" in code nodes where partial failure is acceptable.

20. **Validation & Testing:**  
    - After construction, execute the workflow with test data to verify correct flow, data transformations, AI responses, and publishing.

---

## 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow utilizes advanced Langchain AI agents linked with OpenAI GPT-5 and Gemini engines.   | Core AI processing nodes.                                                                                |
| MongoDB Atlas is used both as vector store and document database for efficient retrieval.      | Requires MongoDB Atlas credentials and proper indexing setup.                                          |
| Integration with WordPress uses OAuth2 for secure content publishing and post management.     | WordPress API documentation: https://developer.wordpress.org/rest-api/                                 |
| Twitter node requires API keys with posting permissions for social promotion.                  | Twitter API docs: https://developer.twitter.com/en/docs/twitter-api                                  |
| Google Sheets nodes require OAuth2 credentials with access to specific sheets for input data.  | Google Sheets API docs: https://developers.google.com/sheets/api                                       |
| Sub-workflow 'gemini' is essential for advanced AI processing; ensure it is imported correctly.| n8n sub-workflows enable modular, reusable logic.                                                      |
| Code nodes include ranking logic to select best AI-generated content; carefully review scripts.| Custom JS logic critical for quality control and decision-making.                                      |
| The workflow maintains chat context using MongoDB-backed memory for better AI interactions.    | Langchain memory integration documentation: https://js.langchain.com/docs/modules/memory              |
| Several nodes have sticky notes indicating "rank and choose the best one" logic in code nodes. | Important for understanding content selection and quality filtering steps.                            |
| RSS feed reading may require periodic update of feed URLs in Google Sheets to remain current. | RSS feed validation and maintenance needed for continuous operation.                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---