🚀 Process YouTube Transcripts with Apify, OpenAI & Pinecone Database

https://n8nworkflows.xyz/workflows/---process-youtube-transcripts-with-apify--openai---pinecone-database-3184


# 🚀 Process YouTube Transcripts with Apify, OpenAI & Pinecone Database

### 1. Workflow Overview

This workflow is designed as a backend indexing pipeline that processes YouTube video transcripts and stores them as vector embeddings in a Pinecone database for semantic search. It is intended for use cases where a large collection of YouTube videos needs to be semantically searchable by their transcript content, enabling retrieval agents to find relevant video segments efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Fetch Video Records from Airtable:** Retrieves YouTube video URLs and metadata from Airtable.
- **1.2 Scrape YouTube Transcripts Using Apify:** For each video URL, triggers an Apify actor to scrape the transcript with timestamps.
- **1.3 Update Airtable with Transcript Data:** Stores the transcript JSON string back into Airtable, linked by video ID.
- **1.4 Process and Chunk Transcripts:** Parses the transcript JSON, converts timestamps, groups transcript entries into chunks enriched with metadata, and splits long texts for embedding.
- **1.5 Generate Embeddings and Index in Pinecone:** Creates vector embeddings for each chunk using OpenAI and indexes them in Pinecone for semantic search.

---

### 2. Block-by-Block Analysis

#### 1.1 Fetch Video Records from Airtable

- **Overview:**  
  Retrieves video records including URLs and metadata from Airtable to initiate processing.

- **Nodes Involved:**  
  - Airtable  
  - Loop Over Items (SplitInBatches)

- **Node Details:**

  - **Airtable**  
    - Type: Airtable node (v2.1)  
    - Role: Fetches video records from a configured Airtable base and table.  
    - Configuration: Likely set to read records with fields such as `url`, `title`, `description`, etc.  
    - Inputs: Manual trigger or upstream node (not explicitly connected here).  
    - Outputs: List of video records.  
    - Edge Cases: Airtable API rate limits, missing or malformed URLs.

  - **Loop Over Items (SplitInBatches)**  
    - Type: SplitInBatches node (v3)  
    - Role: Processes each video record individually to avoid overloading downstream nodes.  
    - Configuration: Batch size not explicitly shown, but typically 1 for sequential processing.  
    - Inputs: From Airtable node.  
    - Outputs: Single video record per batch.  
    - Edge Cases: Batch size too large causing timeouts or memory issues.

---

#### 1.2 Scrape YouTube Transcripts Using Apify

- **Overview:**  
  For each video URL, triggers an Apify actor to scrape the transcript with timestamps, waits for completion, then retrieves the transcript JSON.

- **Nodes Involved:**  
  - Apify NinjaPost (HTTP Request POST)  
  - Wait  
  - Get JSON TS (HTTP Request GET)

- **Node Details:**

  - **Apify NinjaPost**  
    - Type: HTTP Request (v4.2)  
    - Role: Sends a POST request to Apify’s YouTube transcript scraper actor to start scraping.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.apify.com/v2/acts/topaz_sharingan~youtube-transcript-scraper-1/runs?token=<YOUR_TOKEN>`  
      - Body: JSON including `"includeTimestamps": "Yes"` and `"startUrls": ["{{ $json.url }}"]` (dynamic video URL).  
    - Inputs: From Loop Over Items node (each video record).  
    - Outputs: Run initiation response.  
    - Edge Cases: Invalid or expired Apify token, network errors, invalid video URLs.

  - **Wait**  
    - Type: Wait node (v1.1)  
    - Role: Pauses workflow execution to allow Apify actor time to complete scraping.  
    - Configuration: Duration ~1 minute (exact duration not specified).  
    - Inputs: From Apify NinjaPost.  
    - Outputs: Passes data downstream after wait.  
    - Edge Cases: Insufficient wait time causing incomplete data retrieval.

  - **Get JSON TS**  
    - Type: HTTP Request (v4.2)  
    - Role: Retrieves the transcript JSON dataset from the last Apify run.  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.apify.com/v2/acts/topaz_sharingan~youtube-transcript-scraper-1/runs/last/dataset/items?token=<YOUR_TOKEN>`  
    - Inputs: From Wait node.  
    - Outputs: Transcript JSON data.  
    - Edge Cases: Token expiration, Apify actor failure, empty or malformed transcript data.

---

#### 1.3 Update Airtable with Transcript Data

- **Overview:**  
  Converts the transcript JSON to a formatted string, extracts the video ID from the URL, and updates the corresponding Airtable record with the transcript data.

- **Nodes Involved:**  
  - JSON Stringifier (Code node)  
  - Edit Fields (Set node)  
  - Airtable1 (Airtable Update node)

- **Node Details:**

  - **JSON Stringifier**  
    - Type: Code node (v2)  
    - Role: Converts the transcript JSON object into a pretty-printed JSON string for storage.  
    - Configuration:  
      - JavaScript code:  
        ```javascript
        const jsonObject = items[0].json;
        const jsonString = JSON.stringify(jsonObject, null, 2);
        return { json: { stringifiedJson: jsonString } };
        ```  
    - Inputs: From Get JSON TS node.  
    - Outputs: JSON stringified transcript.  
    - Edge Cases: Empty or invalid JSON input.

  - **Edit Fields**  
    - Type: Set node (v3.4)  
    - Role: Extracts the YouTube video ID from the URL for record matching.  
    - Configuration:  
      - Expression: `{{$json.url.split('v=')[1].split('&')[0]}}`  
      - Sets field `videoid` for Airtable update.  
    - Inputs: From JSON Stringifier.  
    - Outputs: Data with video ID field.  
    - Edge Cases: URLs not containing `v=` parameter, malformed URLs.

  - **Airtable1**  
    - Type: Airtable node (v2.1)  
    - Role: Updates Airtable records with the transcript string and video ID.  
    - Configuration:  
      - Update mode matching on `videoid`.  
      - Fields updated: `ts` (transcript string), `videoid`.  
    - Inputs: From Edit Fields.  
    - Outputs: Updated Airtable record confirmation.  
    - Edge Cases: Airtable API errors, record not found, partial updates.

---

#### 1.4 Process and Chunk Transcripts

- **Overview:**  
  Fetches updated Airtable records containing transcripts, parses and chunks the transcript JSON into semantic segments enriched with metadata, and splits long text chunks for embedding.

- **Nodes Involved:**  
  - Airtable2 (Airtable Search)  
  - Transcript Processor (Code node)  
  - Recursive Character Text Splitter1 (Text Splitter)  
  - Default Data Loader (Document Loader)

- **Node Details:**

  - **Airtable2**  
    - Type: Airtable node (v2.1)  
    - Role: Retrieves records with transcript data for processing.  
    - Configuration: Likely filters records where transcript field (`ts`) is not empty.  
    - Inputs: Manual Trigger node or upstream.  
    - Outputs: Records with transcript JSON strings.  
    - Edge Cases: Airtable API limits, missing transcripts.

  - **Transcript Processor**  
    - Type: Code node (v2)  
    - Role:  
      - Parses transcript JSON string back into objects.  
      - Converts timestamps from "mm:ss" to seconds.  
      - Groups transcript entries into chunks based on a 3-second gap threshold.  
      - Constructs chunk objects containing:  
        - Text segment  
        - Video metadata (ID, title, description, published date, thumbnail)  
        - Start and end timestamps  
        - Direct YouTube URL linking to the timestamp (e.g., `https://youtube.com/watch?v=VIDEOID&t=XXs`)  
    - Inputs: From Airtable2.  
    - Outputs: Array of enriched transcript chunks.  
    - Edge Cases: Malformed JSON, missing timestamps, inconsistent data.

  - **Recursive Character Text Splitter1**  
    - Type: Recursive Character Text Splitter (v1)  
    - Role: Splits long transcript chunks into smaller segments (e.g., 500 characters with 50-character overlap) for better embedding quality.  
    - Configuration: Character limit and overlap settings.  
    - Inputs: From Default Data Loader.  
    - Outputs: Smaller text chunks.  
    - Edge Cases: Very short texts, splitting logic errors.

  - **Default Data Loader**  
    - Type: Document Default Data Loader (v1)  
    - Role: Attaches metadata to each chunk and prepares documents for embedding.  
    - Inputs: From Recursive Character Text Splitter1.  
    - Outputs: Documents enriched with metadata for embedding.  
    - Edge Cases: Missing metadata fields.

---

#### 1.5 Generate Embeddings and Index in Pinecone

- **Overview:**  
  Converts each transcript chunk into vector embeddings using OpenAI, then indexes these embeddings in Pinecone under a specified index and namespace.

- **Nodes Involved:**  
  - Embeddings OpenAI  
  - Pinecone Vector Store

- **Node Details:**

  - **Embeddings OpenAI**  
    - Type: OpenAI Embeddings node (v1.2)  
    - Role: Generates vector embeddings for each transcript chunk text.  
    - Configuration:  
      - Model: OpenAI embedding model (e.g., `text-embedding-ada-002`)  
      - Batch size: Adjustable (e.g., 512) for performance tuning.  
    - Inputs: From Default Data Loader.  
    - Outputs: Embeddings vectors.  
    - Edge Cases: API rate limits, invalid API key, large input size.

  - **Pinecone Vector Store**  
    - Type: Pinecone Vector Store node (v1)  
    - Role: Indexes embeddings into Pinecone vector database for semantic search.  
    - Configuration:  
      - Pinecone index name (e.g., `"videos"`)  
      - Namespace (e.g., `"transcripts"`)  
      - Metadata fields included for retrieval context.  
    - Inputs: From Embeddings OpenAI and Transcript Processor (for metadata).  
    - Outputs: Confirmation of indexed vectors.  
    - Edge Cases: Pinecone API errors, quota limits, network issues.

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                          |
|-----------------------------|--------------------------------------------|----------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| Airtable                    | Airtable (v2.1)                            | Fetch video records from Airtable      | When clicking ‘Test workflow’ | Loop Over Items             |                                                                                                    |
| Loop Over Items             | SplitInBatches (v3)                        | Process each video record individually | Airtable                    | Apify NinjaPost (main), (error) |                                                                                                    |
| Apify NinjaPost             | HTTP Request (POST) (v4.2)                 | Trigger Apify actor to scrape transcript | Loop Over Items             | Wait                       |                                                                                                    |
| Wait                       | Wait (v1.1)                                | Pause to allow Apify processing         | Apify NinjaPost             | Get JSON TS                 |                                                                                                    |
| Get JSON TS                 | HTTP Request (GET) (v4.2)                  | Retrieve transcript JSON from Apify    | Wait                       | JSON Stringifier           |                                                                                                    |
| JSON Stringifier            | Code (v2)                                  | Convert transcript JSON to string      | Get JSON TS                 | Edit Fields                |                                                                                                    |
| Edit Fields                 | Set (v3.4)                                 | Extract video ID from URL               | JSON Stringifier            | Airtable1                  |                                                                                                    |
| Airtable1                  | Airtable Update (v2.1)                      | Update Airtable with transcript string | Edit Fields                 | Loop Over Items (error), Loop Over Items (main) |                                                                                                    |
| Airtable2                  | Airtable (v2.1)                            | Fetch updated records with transcripts | When clicking ‘Test workflow’ | Transcript Processor       |                                                                                                    |
| Transcript Processor        | Code (v2)                                  | Parse and chunk transcripts with metadata | Airtable2                  | Pinecone Vector Store      |                                                                                                    |
| Recursive Character Text Splitter1 | Recursive Character Text Splitter (v1) | Split long transcript chunks into smaller segments | Default Data Loader         | Default Data Loader        |                                                                                                    |
| Default Data Loader         | Document Default Data Loader (v1)          | Attach metadata and prepare documents  | Recursive Character Text Splitter1 | Pinecone Vector Store      |                                                                                                    |
| Embeddings OpenAI           | OpenAI Embeddings (v1.2)                   | Generate vector embeddings              | Default Data Loader         | Pinecone Vector Store      |                                                                                                    |
| Pinecone Vector Store       | Pinecone Vector Store (v1)                  | Index embeddings in Pinecone            | Embeddings OpenAI, Transcript Processor |                            |                                                                                                    |
| When clicking ‘Test workflow’ | Manual Trigger (v1)                        | Start workflow manually                  |                             | Airtable2                  |                                                                                                    |
| Installation Tutorial       | Sticky Note                                |                                        |                             |                            |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Airtable Node (Airtable)**  
   - Type: Airtable (v2.1)  
   - Configure credentials for your Airtable base.  
   - Set operation to "Read Records" from the table containing YouTube video records.  
   - Select fields including `url`, `title`, `description`, etc.

3. **Create SplitInBatches Node (Loop Over Items)**  
   - Type: SplitInBatches (v3)  
   - Connect input from Airtable node.  
   - Set batch size to 1 (process one video record at a time).

4. **Create HTTP Request Node (Apify NinjaPost)**  
   - Type: HTTP Request (v4.2)  
   - Connect input from Loop Over Items node (main output).  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/topaz_sharingan~youtube-transcript-scraper-1/runs?token=<YOUR_TOKEN>`  
   - Body Content-Type: JSON  
   - Body:  
     ```json
     {
       "includeTimestamps": "Yes",
       "startUrls": ["{{ $json.url }}"]
     }
     ```  
   - Replace `<YOUR_TOKEN>` with your Apify API token.

5. **Create Wait Node**  
   - Type: Wait (v1.1)  
   - Connect input from Apify NinjaPost node.  
   - Set wait duration to approximately 60 seconds.

6. **Create HTTP Request Node (Get JSON TS)**  
   - Type: HTTP Request (v4.2)  
   - Connect input from Wait node.  
   - Method: GET  
   - URL: `https://api.apify.com/v2/acts/topaz_sharingan~youtube-transcript-scraper-1/runs/last/dataset/items?token=<YOUR_TOKEN>`  
   - Replace `<YOUR_TOKEN>` with your Apify API token.

7. **Create Code Node (JSON Stringifier)**  
   - Type: Code (v2)  
   - Connect input from Get JSON TS node.  
   - Code:  
     ```javascript
     const jsonObject = items[0].json;
     const jsonString = JSON.stringify(jsonObject, null, 2);
     return [{ json: { stringifiedJson: jsonString, url: $json.url } }];
     ```  
   - Ensure to pass along the original URL if needed.

8. **Create Set Node (Edit Fields)**  
   - Type: Set (v3.4)  
   - Connect input from JSON Stringifier node.  
   - Add a field `videoid` with expression:  
     ```javascript
     {{$json.url.split('v=')[1].split('&')[0]}}
     ```  
   - Also include `ts` field mapped to `stringifiedJson` for Airtable update.

9. **Create Airtable Node (Airtable1)**  
   - Type: Airtable (v2.1)  
   - Connect input from Edit Fields node.  
   - Configure credentials.  
   - Set operation to "Update Record" matching on `videoid`.  
   - Update fields: `ts` (transcript string), `videoid`.  
   - Set "Continue On Fail" to true to avoid workflow interruption.

10. **Connect Airtable1 node’s output back to Loop Over Items node’s error and main outputs**  
    - This allows the loop to continue processing next items even if one update fails.

11. **Create Airtable Node (Airtable2)**  
    - Type: Airtable (v2.1)  
    - Connect input from Manual Trigger node.  
    - Configure to read records where `ts` (transcript) field is present.

12. **Create Code Node (Transcript Processor)**  
    - Type: Code (v2)  
    - Connect input from Airtable2.  
    - Implement logic to:  
      - Parse `ts` JSON string back to object.  
      - Convert timestamps from "mm:ss" to seconds.  
      - Group transcript entries into chunks based on 3-second gaps.  
      - For each chunk, create an object with text, video metadata, start/end timestamps, and a direct YouTube URL with timestamp.

13. **Create Recursive Character Text Splitter Node**  
    - Type: Recursive Character Text Splitter (v1)  
    - Connect input from Default Data Loader node (see next step).  
    - Configure chunk size (e.g., 500 characters) and overlap (e.g., 50 characters).

14. **Create Default Data Loader Node**  
    - Type: Document Default Data Loader (v1)  
    - Connect input from Recursive Character Text Splitter node.  
    - This node enriches chunks with metadata for embedding.

15. **Create Embeddings OpenAI Node**  
    - Type: OpenAI Embeddings (v1.2)  
    - Connect input from Default Data Loader node.  
    - Configure OpenAI credentials.  
    - Set model (e.g., `text-embedding-ada-002`).  
    - Adjust batch size as needed.

16. **Create Pinecone Vector Store Node**  
    - Type: Pinecone Vector Store (v1)  
    - Connect input from Embeddings OpenAI node (embedding output) and Transcript Processor node (metadata).  
    - Configure Pinecone credentials.  
    - Set index name (e.g., `"videos"`).  
    - Set namespace (e.g., `"transcripts"`).

17. **Verify all connections and test the workflow**  
    - Run manual trigger to start processing.  
    - Monitor logs for errors and adjust wait times or batch sizes as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow only handles transcript processing and indexing; retrieval/search is implemented separately. | Workflow description and design rationale.                                                         |
| Apify YouTube Transcript Scraper actor used: `topaz_sharingan~youtube-transcript-scraper-1`      | https://apify.com/topaz_sharingan/youtube-transcript-scraper                                        |
| Pinecone vector database is used for semantic indexing and retrieval.                            | https://www.pinecone.io/                                                                            |
| OpenAI embeddings model recommended: `text-embedding-ada-002`                                   | https://platform.openai.com/docs/models/embeddings                                                 |
| Airtable API rate limits and token expiration should be monitored to avoid workflow failures.   | https://airtable.com/api                                                                           |
| Adjust wait duration after Apify actor trigger based on average transcript scraping time.        | To avoid incomplete transcript retrieval.                                                          |
| The direct YouTube URL with timestamp enables retrieval agents to jump to exact video moments.  | Format: `https://youtube.com/watch?v=VIDEOID&t=XXs`                                                |
| For large datasets, consider increasing batch size or parallelizing with caution to avoid rate limits. | Performance tuning advice.                                                                          |

---

This document provides a complete and detailed reference for understanding, reproducing, and maintaining the YouTube transcript processing and indexing workflow in n8n.