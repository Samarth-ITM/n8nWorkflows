Youtube RAG search with Frontend using Apify, Qdrant and AI

https://n8nworkflows.xyz/workflows/youtube-rag-search-with-frontend-using-apify--qdrant-and-ai-3288


# Youtube RAG search with Frontend using Apify, Qdrant and AI

### 1. Workflow Overview

This workflow implements a Retrieval-Augmented Generation (RAG) search engine for YouTube videos, specifically designed to index and search video transcripts from the official n8n YouTube channel or any other public channel. It leverages Apify for scraping YouTube video metadata and subtitles, Qdrant as a vector database for storing embeddings, and OpenAI’s LLM for semantic search and summarization. The workflow is divided into two main stages:

- **Stage 1: Data Collection and Vector Store Population**
  - Fetch latest YouTube videos using Apify’s YouTube scraper.
  - Download video subtitles (transcripts).
  - Chunk transcripts into manageable pieces.
  - Generate embeddings for chunks using OpenAI.
  - Store embeddings and metadata in Qdrant vector store.

- **Stage 2: Web Frontend and Search API**
  - Expose a webhook serving a simple web UI for user queries.
  - Accept search queries via API webhook.
  - Generate query embeddings.
  - Perform advanced grouped search in Qdrant to retrieve relevant transcript chunks grouped by video.
  - Use AI to extract and summarize relevant transcript parts.
  - Format results into an HTML template.
  - Return combined AI-generated answer and search results to the frontend.
  - Embedded YouTube player allows users to watch videos at relevant timestamps.

Logical blocks are:

- 1.1 Fetch Latest Videos (Apify)
- 1.2 Process Video Transcripts and Populate Vector Store (Subworkflow)
- 1.3 Search API with Rate Limiting (Redis)
- 1.4 Qdrant Grouped Search and AI Extraction
- 1.5 Results Formatting and Answer Generation
- 1.6 Web Frontend Serving and Video Playback

---

### 2. Block-by-Block Analysis

#### 1.1 Fetch Latest Videos (Apify)

**Overview:**  
This block fetches the latest videos from a specified YouTube channel using Apify’s YouTube channel scraper API, then removes duplicates to avoid reprocessing.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking ‘Test workflow’  
- Get Latest Youtube Videos  
- Ignore Already Seen  
- For Each Video  
- Vectorise Subworkflow1 (trigger node)  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Runs weekly on Saturday at 6 AM to automate fetching new videos.  
  - Input: None  
  - Output: Triggers "Get Latest Youtube Videos".  
  - Edge cases: Missed schedules if n8n is down.

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Allows manual triggering for testing.  
  - Input: None  
  - Output: Triggers "Get Latest Youtube Videos".

- **Get Latest Youtube Videos**  
  - Type: HTTP Request  
  - Calls Apify’s YouTube channel scraper API to fetch up to 10 latest videos from the n8n official channel.  
  - Configured to exclude shorts, live, and other video types by default.  
  - Uses HTTP Header Authentication with Apify personal token.  
  - Input: Trigger node output.  
  - Output: JSON array of video metadata.  
  - Edge cases: API rate limits, network errors, invalid credentials.

- **Ignore Already Seen**  
  - Type: Remove Duplicates  
  - Removes videos already processed based on video ID to prevent duplicate processing.  
  - Input: Video list from previous node.  
  - Output: Filtered video list.  
  - Edge cases: If dedupe value missing or malformed, may fail.

- **For Each Video**  
  - Type: Split In Batches  
  - Iterates over each new video to process transcripts.  
  - Input: Filtered video list.  
  - Output: Single video item per iteration.  

- **Vectorise Subworkflow1**  
  - Type: Execute Workflow Trigger (Subworkflow)  
  - Invokes the subworkflow to process each video’s transcript and vectorize it.  
  - Input: Single video JSON.  
  - Output: None (runs asynchronously).  
  - Edge cases: Subworkflow failures, timeouts.

---

#### 1.2 Process Video Transcripts and Populate Vector Store (Subworkflow)

**Overview:**  
This subworkflow downloads video subtitles, chunks them, generates embeddings, and inserts them into the Qdrant vector store with metadata.

**Nodes Involved:**  
- Video Ref (NoOp)  
- Get Video Subtitles (HTTP Request)  
- Chunk Subtitles (Set)  
- Chunks to Items (Split Out)  
- For Each Chunk (Split In Batches)  
- Default Data Loader (Langchain Document Loader)  
- Text Splitter (Langchain Text Splitter)  
- Embeddings (OpenAI Embeddings)  
- Qdrant Vector Store (Insert Mode)  
- Wait (Wait Node)  

**Node Details:**  

- **Video Ref**  
  - Type: NoOp  
  - Holds the current video metadata for reference in expressions.  
  - Input: Video JSON from parent workflow.  
  - Output: Passes video JSON downstream.

- **Get Video Subtitles**  
  - Type: HTTP Request  
  - Calls Apify’s YouTube scraper API to download subtitles (VTT format) for the current video URL.  
  - Uses HTTP Header Authentication with Apify token.  
  - Input: Video URL from Video Ref.  
  - Output: JSON with subtitles array.  
  - Edge cases: No subtitles available, API errors.

- **Chunk Subtitles**  
  - Type: Set  
  - Splits the subtitle text into chunks of max 30,000 characters to avoid size limits.  
  - Uses JavaScript expression to slice the VTT string.  
  - Input: Subtitle text.  
  - Output: Array of subtitle chunks.

- **Chunks to Items**  
  - Type: Split Out  
  - Converts the array of chunks into individual items for batch processing.  
  - Input: Array of chunks.  
  - Output: Single chunk per item.

- **For Each Chunk**  
  - Type: Split In Batches  
  - Processes each chunk individually for embedding and insertion.  
  - Input: Single chunk.  
  - Output: Single chunk item.

- **Default Data Loader**  
  - Type: Langchain Document Default Data Loader  
  - Prepares chunk text and attaches metadata (videoId, title, channelId, url, type) for embedding.  
  - Cleans subtitle text by replacing double newlines.  
  - Input: Chunk text and video metadata.  
  - Output: Document object for embedding.

- **Text Splitter**  
  - Type: Langchain Recursive Character Text Splitter  
  - Further splits text into smaller chunks of 3000 characters if needed.  
  - Input: Document text.  
  - Output: Smaller text chunks.

- **Embeddings**  
  - Type: Langchain OpenAI Embeddings  
  - Generates vector embeddings for text chunks using OpenAI API.  
  - Input: Text chunks.  
  - Output: Embeddings array.  
  - Edge cases: API rate limits, invalid API key.

- **Qdrant Vector Store**  
  - Type: Langchain Qdrant Vector Store (Insert Mode)  
  - Inserts embeddings and metadata into Qdrant collection "n8n_videos".  
  - Uses Qdrant API credentials.  
  - Input: Embeddings and metadata.  
  - Output: Confirmation of insertion.  
  - Edge cases: Qdrant connection issues, collection not existing (requires manual creation).

- **Wait**  
  - Type: Wait  
  - Waits 1 second between chunk insertions to avoid rate limits or overload.  
  - Input: After Qdrant insertion.  
  - Output: Triggers next chunk processing.

---

#### 1.3 Search API with Rate Limiting (Redis)

**Overview:**  
This block exposes a webhook API endpoint for search queries, implements rate limiting using Redis, and controls access to the search functionality.

**Nodes Involved:**  
- SEARCH API (Webhook)  
- Incr Rate Limit (Redis)  
- 10req/min (If)  
- Get Query (Set)  
- 429 Response (Set)  
- Respond to Webhook2 (Respond to Webhook)  

**Node Details:**  

- **SEARCH API**  
  - Type: Webhook  
  - Path: `/n8n_videos/api/search`  
  - Accepts HTTP GET requests with query parameters.  
  - Input: User search query and optional type filter.  
  - Output: Passes request data downstream.  
  - Edge cases: Bot detection, malformed requests.

- **Incr Rate Limit**  
  - Type: Redis  
  - Increments a Redis key based on client IP (`x-forwarded-for` header) to count requests.  
  - Key expires automatically to reset count.  
  - Input: Request headers.  
  - Output: Current count.

- **10req/min**  
  - Type: If  
  - Checks if request count is less than 11 (limit 10 requests per minute).  
  - Input: Redis counter.  
  - Output: Routes to search or rate limit response.

- **Get Query**  
  - Type: Set  
  - Extracts and sanitizes the user query and type filter from the webhook request.  
  - Limits query length to 128 characters.  
  - Input: Webhook JSON.  
  - Output: Cleaned query object.

- **429 Response**  
  - Type: Set  
  - Prepares a response indicating the search limit has been reached.  
  - Input: None.  
  - Output: Message for rate limit exceeded.

- **Respond to Webhook2**  
  - Type: Respond to Webhook  
  - Returns the 429 response with HTTP 200 and HTML content.  
  - Input: 429 Response node.  
  - Output: HTTP response to client.

---

#### 1.4 Qdrant Grouped Search and AI Extraction

**Overview:**  
This block performs semantic search using Qdrant’s advanced grouped search API, extracts relevant transcript parts using AI, and prepares the results for formatting.

**Nodes Involved:**  
- Get Embeddings (HTTP Request)  
- Qdrant Groups Search (HTTP Request)  
- Has Results?1 (If)  
- Groups to Items1 (Split Out)  
- For Each Group (Split In Batches)  
- Group Ref (NoOp)  
- Extract Results (Langchain Information Extractor)  
- OpenAI Chat Model (Langchain Chat LLM)  

**Node Details:**  

- **Get Embeddings**  
  - Type: HTTP Request  
  - Calls OpenAI embeddings API to generate embedding vector for user query.  
  - Model: `text-embedding-3-small`.  
  - Input: Query text.  
  - Output: Embedding vector.  
  - Edge cases: API errors, invalid input.

- **Qdrant Groups Search**  
  - Type: HTTP Request  
  - Calls Qdrant’s `/points/search/groups` endpoint to perform grouped similarity search.  
  - Parameters:  
    - Limit: 4 groups (videos)  
    - Group size: 3 points (chunks) per video  
    - Filter by type if specified (video or stream)  
    - Vector: query embedding  
  - Uses Qdrant API credentials.  
  - Input: Query embedding.  
  - Output: Search results grouped by videoId.  
  - Edge cases: Qdrant API errors, no results.

- **Has Results?1**  
  - Type: If  
  - Checks if search returned any groups.  
  - Input: Qdrant search results.  
  - Output: Routes to processing or empty response.

- **Groups to Items1**  
  - Type: Split Out  
  - Splits groups array into individual group items for processing.  
  - Input: Search groups.  
  - Output: Single group per item.

- **For Each Group**  
  - Type: Split In Batches  
  - Iterates over each group to extract relevant transcript parts.  
  - Input: Single group.  
  - Output: Single group item.

- **Group Ref**  
  - Type: NoOp  
  - Holds current group data for reference in expressions.  
  - Input: Group item.  
  - Output: Passes group downstream.

- **Extract Results**  
  - Type: Langchain Information Extractor  
  - Uses AI to analyze transcript chunks and extract relevant parts matching user query.  
  - Input: Transcript chunks in XML-like format with metadata.  
  - System prompt instructs to return 3-10 relevant extracts with timestamps, video title, and URL, removing VTT tags.  
  - Output: Extracted relevant transcript parts.  
  - Edge cases: LLM errors, incomplete extraction.

- **OpenAI Chat Model**  
  - Type: Langchain Chat LLM (gpt-4o-mini)  
  - Optional node for further AI processing or summarization (used downstream).  
  - Input: Extracted results.  
  - Output: AI-generated text.

---

#### 1.5 Results Formatting and Answer Generation

**Overview:**  
This block formats the extracted transcript parts into an HTML results list, generates a concise AI answer summarizing the results, and prepares the final response.

**Nodes Involved:**  
- Combine Results (Set)  
- Has Results? (If)  
- Transcripts to Items (Split Out)  
- Clean Up Output (Set)  
- Sort By Video ID (Sort)  
- Generate Template (Set)  
- Answer Query (Langchain Chain LLM)  
- Markdown (Markdown to HTML)  
- Map Fields (Set)  

**Node Details:**  

- **Combine Results**  
  - Type: Set  
  - Combines all extracted outputs from groups into a single array.  
  - Input: Outputs from For Each Group.  
  - Output: Combined array.

- **Has Results?**  
  - Type: If  
  - Checks if combined results array is not empty.  
  - Input: Combined results.  
  - Output: Routes to processing or empty response.

- **Transcripts to Items**  
  - Type: Split Out  
  - Splits combined results array into individual transcript items.  
  - Input: Combined array.  
  - Output: Single transcript item.

- **Clean Up Output**  
  - Type: Set  
  - Cleans and formats transcript extracts:  
    - Removes newlines, formats timestamps to MM:SS, calculates video timestamp offset with buffer.  
    - Prepares fields: title, url, extract, timestamp, videoId, video_ts.  
  - Input: Single transcript item.  
  - Output: Cleaned transcript item.

- **Sort By Video ID**  
  - Type: Sort  
  - Sorts cleaned transcript items by videoId ascending.  
  - Input: Cleaned items.  
  - Output: Sorted items.

- **Generate Template**  
  - Type: Set  
  - Generates HTML results list grouped by videoId using JavaScript expression.  
  - Uses htmx framework-compatible markup for frontend interaction.  
  - Input: Sorted transcript items.  
  - Output: HTML string with results.

- **Answer Query**  
  - Type: Langchain Chain LLM  
  - Generates a 1-2 sentence AI summary answer based on the results HTML and user query.  
  - Model: gpt-4o-mini.  
  - Input: Results HTML and user query.  
  - Output: AI answer text.

- **Markdown**  
  - Type: Markdown to HTML  
  - Converts AI answer markdown text to HTML.  
  - Input: AI answer text.  
  - Output: HTML formatted answer.

- **Map Fields**  
  - Type: Set  
  - Maps final response fields: `text` (AI answer HTML) and `results` (results HTML).  
  - Input: Markdown and Generate Template outputs.  
  - Output: Final combined response.

---

#### 1.6 Web Frontend Serving and Video Playback

**Overview:**  
This block serves the web UI frontend for users to enter queries and view results with embedded video playback.

**Nodes Involved:**  
- WEB UI (Webhook)  
- Generate Webpage (HTML)  
- Render Page (Respond to Webhook)  

**Node Details:**  

- **WEB UI**  
  - Type: Webhook  
  - Path: `/n8n_videos/`  
  - Serves the initial HTML page for the video search frontend.  
  - Input: HTTP GET request from browser.  
  - Output: Triggers Generate Webpage.

- **Generate Webpage**  
  - Type: HTML  
  - Contains full HTML, CSS, and JavaScript for the frontend UI.  
  - Features:  
    - Search form with query and type filter.  
    - Results panel dynamically updated via htmx AJAX calls to search API.  
    - Embedded YouTube player that loads videos at specified timestamps.  
    - About section with credits and links.  
  - Input: Trigger from WEB UI.  
  - Output: HTML content.

- **Render Page**  
  - Type: Respond to Webhook  
  - Returns the generated HTML page with HTTP 200 and content-type text/html.  
  - Input: HTML node output.  
  - Output: HTTP response to client browser.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                          |
|----------------------------|----------------------------------|----------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Manual start trigger                          | -                             | Get Latest Youtube Videos       |                                                                                                    |
| Schedule Trigger           | Schedule Trigger                  | Scheduled start trigger (weekly)              | -                             | Get Latest Youtube Videos       |                                                                                                    |
| Get Latest Youtube Videos  | HTTP Request                     | Fetch latest videos from Apify scraper        | When clicking ‘Test workflow’, Schedule Trigger | Ignore Already Seen             | ## 1. Fetch Latest Videos with [Apify.com](https://www.apify.com?fpr=414q6)                        |
| Ignore Already Seen        | Remove Duplicates                | Remove already processed videos               | Get Latest Youtube Videos      | For Each Video                 |                                                                                                    |
| For Each Video             | Split In Batches                 | Iterate over each new video                    | Ignore Already Seen            | Vectorise Subworkflow           |                                                                                                    |
| Vectorise Subworkflow1     | Execute Workflow Trigger         | Process each video transcript and vectorize  | For Each Video                 | Video Ref                      | ## 2. Get Video Transcript with [Apify.com](https://www.apify.com?fpr=414q6)                       |
| Video Ref                  | NoOp                            | Holds current video metadata                   | Vectorise Subworkflow1         | Get Video Subtitles             |                                                                                                    |
| Get Video Subtitles        | HTTP Request                    | Download subtitles for video from Apify       | Video Ref                     | Chunk Subtitles                |                                                                                                    |
| Chunk Subtitles            | Set                             | Chunk subtitle text into 30,000 char pieces  | Get Video Subtitles            | Chunks to Items                |                                                                                                    |
| Chunks to Items            | Split Out                      | Split chunks array into individual items      | Chunk Subtitles                | For Each Chunk                 |                                                                                                    |
| For Each Chunk             | Split In Batches                 | Iterate over each subtitle chunk               | Chunks to Items               | Default Data Loader, Qdrant Vector Store (via Wait) |                                                                                                    |
| Default Data Loader        | Langchain Document Loader        | Prepare chunk text and metadata for embedding | For Each Chunk                 | Embeddings                    |                                                                                                    |
| Text Splitter              | Langchain Text Splitter          | Further split text chunks to 3000 chars       | Default Data Loader            | Embeddings                    |                                                                                                    |
| Embeddings                 | Langchain OpenAI Embeddings      | Generate embeddings for text chunks            | Text Splitter                 | Qdrant Vector Store           |                                                                                                    |
| Qdrant Vector Store        | Langchain Vector Store Qdrant    | Insert embeddings and metadata into Qdrant    | Embeddings                   | Wait                         | ## 3. Populate Qdrant Vector Store to Build a Search Index                                         |
| Wait                      | Wait                            | Pause between chunk insertions                  | Qdrant Vector Store           | For Each Chunk                |                                                                                                    |
| SEARCH API                | Webhook                         | API endpoint for search queries                 | -                             | Incr Rate Limit               | ## 4. Search API with Rate Limiting                                                                |
| Incr Rate Limit           | Redis                           | Increment Redis counter for rate limiting      | SEARCH API                   | 10req/min                    |                                                                                                    |
| 10req/min                 | If                              | Check if request count is within limit          | Incr Rate Limit              | Get Query, 429 Response       |                                                                                                    |
| Get Query                 | Set                             | Extract and sanitize user query and type       | 10req/min                   | Get Embeddings               |                                                                                                    |
| 429 Response              | Set                             | Prepare rate limit exceeded response            | 10req/min                   | Respond to Webhook2           |                                                                                                    |
| Respond to Webhook2       | Respond to Webhook              | Return rate limit exceeded message              | 429 Response                | -                           |                                                                                                    |
| Get Embeddings            | HTTP Request                   | Generate embedding for user query               | Get Query                   | Qdrant Groups Search          |                                                                                                    |
| Qdrant Groups Search      | HTTP Request                   | Perform grouped similarity search in Qdrant    | Get Embeddings              | Has Results?1                | ## 5. Qdrant Advanced Search - Point Groups                                                        |
| Has Results?1             | If                              | Check if search returned any groups             | Qdrant Groups Search        | Groups to Items1, Generate Empty Response1 |                                                                                                    |
| Groups to Items1          | Split Out                      | Split groups array into individual group items | Has Results?1               | For Each Group               |                                                                                                    |
| For Each Group            | Split In Batches                 | Iterate over each group                          | Groups to Items1            | Group Ref                    |                                                                                                    |
| Group Ref                 | NoOp                            | Holds current group data                         | For Each Group              | Extract Results              |                                                                                                    |
| Extract Results           | Langchain Information Extractor | Extract relevant transcript parts using AI     | Group Ref                   | OpenAI Chat Model            | ## 6. Contextually Understanding Transcripts with AI                                              |
| OpenAI Chat Model         | Langchain Chat LLM              | Optional AI processing of extracted results    | Extract Results             | -                           |                                                                                                    |
| Combine Results           | Set                             | Combine all extracted results into one array   | For Each Group              | Has Results?                 |                                                                                                    |
| Has Results?              | If                              | Check if combined results array is not empty   | Combine Results             | Transcripts to Items, Generate Empty Response |                                                                                                    |
| Transcripts to Items      | Split Out                      | Split combined results into individual items   | Has Results?                | Clean Up Output             |                                                                                                    |
| Clean Up Output           | Set                             | Clean and format transcript extracts            | Transcripts to Items        | Sort By Video ID            |                                                                                                    |
| Sort By Video ID          | Sort                            | Sort transcript items by videoId                 | Clean Up Output             | Generate Template           |                                                                                                    |
| Generate Template         | Set                             | Generate HTML results list grouped by videoId  | Sort By Video ID            | Answer Query                | ## 7. Generate Results HTML Template                                                               |
| Answer Query              | Langchain Chain LLM             | Generate AI summary answer from results         | Generate Template           | Markdown                    | ## 8. Summarise Results to Generate Answer                                                        |
| Markdown                  | Markdown                        | Convert AI answer markdown to HTML               | Answer Query                | Map Fields                  |                                                                                                    |
| Map Fields                | Set                             | Map final response fields (answer + results)    | Markdown, Generate Template | Respond to Webhook           | ## 9. Return Answer & Search Results                                                               |
| Respond to Webhook        | Respond to Webhook              | Return combined AI answer and search results     | Map Fields                  | -                           |                                                                                                    |
| WEB UI                    | Webhook                        | Serve web frontend HTML page                      | -                           | Generate Webpage            | ## 10. N8N Video Search Frontend using Web UI                                                     |
| Generate Webpage          | HTML                           | Generate full HTML, CSS, JS frontend page        | WEB UI                      | Render Page                 |                                                                                                    |
| Render Page               | Respond to Webhook              | Return HTML page to client browser                | Generate Webpage            | -                           |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking ‘Test workflow’" for manual testing.

2. **Create a Schedule Trigger node** named "Schedule Trigger" to run weekly on Saturdays at 6 AM.

3. **Create an HTTP Request node** named "Get Latest Youtube Videos":  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-channel-scraper/run-sync-get-dataset-items`  
   - Authentication: HTTP Header Auth with Apify personal token credential.  
   - Body (JSON): Configure to fetch up to 10 latest videos from channel `https://www.youtube.com/@n8n-io`.  
   - Connect both triggers to this node.

4. **Add a Remove Duplicates node** named "Ignore Already Seen":  
   - Operation: Remove items seen in previous executions.  
   - Dedupe value: video ID from JSON.  
   - Connect from "Get Latest Youtube Videos".

5. **Add a Split In Batches node** named "For Each Video":  
   - Connect from "Ignore Already Seen".

6. **Create a Subworkflow** (call it "Vectorise Subworkflow") to process each video transcript:  
   - Input: Single video JSON.  
   - Output: None (runs asynchronously).

7. **In the main workflow**, add an Execute Workflow Trigger node named "Vectorise Subworkflow1":  
   - Set to run the "Vectorise Subworkflow" for each video.  
   - Connect from "For Each Video".

8. **Inside the "Vectorise Subworkflow":**  
   - Add a NoOp node "Video Ref" to hold video metadata.  
   - Add HTTP Request node "Get Video Subtitles":  
     - POST to Apify YouTube scraper API for subtitles.  
     - Use video URL from "Video Ref".  
     - Auth: Apify token.  
   - Add Set node "Chunk Subtitles":  
     - Use JavaScript to split subtitles VTT text into 30,000 char chunks.  
   - Add Split Out node "Chunks to Items" to split chunks array.  
   - Add Split In Batches node "For Each Chunk".  
   - Add Langchain Document Default Data Loader node "Default Data Loader":  
     - Input chunk text and video metadata.  
   - Add Langchain Recursive Character Text Splitter node "Text Splitter":  
     - Chunk size 3000 chars.  
   - Add Langchain OpenAI Embeddings node "Embeddings":  
     - Use OpenAI credentials.  
   - Add Langchain Qdrant Vector Store node "Qdrant Vector Store":  
     - Mode: Insert  
     - Collection: "n8n_videos"  
     - Use Qdrant credentials.  
   - Add Wait node "Wait" with 1 second delay.  
   - Connect nodes in order: Video Ref → Get Video Subtitles → Chunk Subtitles → Chunks to Items → For Each Chunk → Default Data Loader → Text Splitter → Embeddings → Qdrant Vector Store → Wait → For Each Chunk (loop).

9. **Back in main workflow, create Webhook node "SEARCH API":**  
   - Path: `/n8n_videos/api/search`  
   - Method: GET  
   - Connect to Redis node "Incr Rate Limit":  
     - Key: `n8n_videos_session_{{ $json.headers['x-forwarded-for'] }}`  
     - Operation: INCR with expiry.  
   - Add If node "10req/min" to check if count < 11.  
   - True branch:  
     - Set node "Get Query": sanitize and extract query and type.  
     - HTTP Request node "Get Embeddings": call OpenAI embeddings API with query text.  
     - HTTP Request node "Qdrant Groups Search": POST to Qdrant `/collections/n8n_videos/points/search/groups` with parameters: limit=4, group_size=3, filter by type if specified, vector from embeddings.  
     - If node "Has Results?1": check if groups exist.  
     - True branch: Split Out node "Groups to Items1" → Split In Batches "For Each Group" → NoOp "Group Ref" → Langchain Information Extractor "Extract Results" → Langchain Chat LLM "OpenAI Chat Model".  
     - Set node "Combine Results": combine extracted outputs.  
     - If node "Has Results?": check if combined results not empty.  
     - True branch: Split Out "Transcripts to Items" → Set "Clean Up Output" → Sort "Sort By Video ID" → Set "Generate Template" → Langchain Chain LLM "Answer Query" → Markdown node → Set "Map Fields" → Respond to Webhook node.  
     - False branch: Set "Generate Empty Response1" → Respond to Webhook4.  
   - False branch (rate limit exceeded): Set "429 Response" → Respond to Webhook2.

10. **Create Webhook node "WEB UI":**  
    - Path: `/n8n_videos/`  
    - Method: GET  
    - Connect to HTML node "Generate Webpage": full HTML, CSS, JS for frontend UI with htmx and embedded YouTube player.  
    - Connect to Respond to Webhook node "Render Page".

11. **Create all required credentials:**  
    - Apify HTTP Header Auth with personal token.  
    - OpenAI API key.  
    - Qdrant API key and endpoint.  
    - Redis connection for rate limiting.

12. **Manually create Qdrant collection "n8n_videos"** with vector size 1536 and cosine distance:  
    ```
    PUT collections/n8n_videos
    {
      "vectors": {
          "distance": "Cosine",
          "size": 1536
      }
    }
    ```

13. **Activate the workflow.**  
    - Run the manual trigger or wait for schedule to populate vector store.  
    - Access the frontend at `/webhook/n8n_videos/` to search videos.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow uses Apify’s YouTube scrapers to bypass official API limits and get subtitles and video metadata.                         | [Apify.com](https://www.apify.com?fpr=414q6)                                                         |
| Qdrant’s advanced Search Groups API is used to group search results by video, improving result diversity.                                | [Qdrant Search Groups](https://qdrant.tech/documentation/concepts/search/#search-groups)              |
| The frontend uses htmx (htmx.org) for dynamic single-page application behavior.                                                         | [htmx.org](https://htmx.org)                                                                          |
| Rate limiting is implemented with Redis to prevent abuse of the search API. Remove or adjust for public deployments.                    | [Redis Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.redis)             |
| The AI extraction uses Langchain’s Information Extractor node to parse transcripts and extract relevant parts with timestamps.          | [Information Extractor Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor) |
| The AI answer generation uses Langchain Chain LLM node with OpenAI GPT-4o-mini model for concise user query summarization.              | [Langchain Chain LLM Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) |
| The frontend HTML includes embedded YouTube player API for video playback at specific timestamps.                                       | [YouTube IFrame API](https://developers.google.com/youtube/iframe_api_reference)                       |
| Demo URL: [https://jimleuk.app.n8n.cloud/webhook/n8n_videos](https://jimleuk.app.n8n.cloud/webhook/n8n_videos)                          |                                                                                                       |
| For help or discussion, join the n8n Discord or community forum.                                                                          | [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)                   |
| To customize, swap the YouTube channel URL in the Apify scraper nodes to target other public channels.                                   |                                                                                                       |

---

This documentation fully describes the workflow’s structure, node configurations, data flow, and integration points to enable reproduction, modification, and troubleshooting.