Notion to Pinecone Vector Store Integration

https://n8nworkflows.xyz/workflows/notion-to-pinecone-vector-store-integration-2797


# Notion to Pinecone Vector Store Integration

### 1. Workflow Overview

This workflow automates the extraction, processing, and storage of newly added Notion pages into a Pinecone vector store, enabling semantic search and AI-driven content retrieval. It is designed for teams using Notion as a knowledge base, converting page content into vector embeddings for enhanced search and analysis.

The workflow is logically divided into the following blocks:

- **1.1 Notion Page Monitoring and Content Retrieval:** Detects new pages in a specified Notion database and fetches their full content.
- **1.2 Content Filtering and Preparation:** Filters out non-text content, concatenates text blocks, and splits the text into token chunks.
- **1.3 Metadata Enrichment and Embedding Generation:** Adds metadata to the content and generates semantic embeddings using Google Gemini.
- **1.4 Vector Storage:** Inserts the embeddings and metadata into a Pinecone vector store for efficient retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 Notion Page Monitoring and Content Retrieval

**Overview:**  
This block triggers the workflow when a new page is added to a specific Notion database and retrieves the full content of that page, including all blocks.

**Nodes Involved:**  
- Notion - Page Added Trigger  
- Notion - Retrieve Page Content

**Node Details:**

- **Notion - Page Added Trigger**  
  - *Type:* Trigger node, Notion API trigger  
  - *Role:* Polls the specified Notion database every minute for newly added pages.  
  - *Configuration:*  
    - Poll mode: Every minute  
    - Database ID: Specific Notion database (ID: 17b11930-c10f-8000-a545-ece7cade03f9)  
    - Credentials: Uses Notion API credentials named "Auto: Notion"  
  - *Input:* None (trigger)  
  - *Output:* Emits metadata of newly added pages, including page ID, creation time, URL, and properties.  
  - *Edge Cases:*  
    - API rate limits or authentication failures may cause missed triggers.  
    - If the database ID changes or is invalid, no triggers will occur.  
  - *Version:* 1

- **Notion - Retrieve Page Content**  
  - *Type:* API node, Notion block retrieval  
  - *Role:* Fetches all blocks (text, images, videos, etc.) of the triggered page using the page URL.  
  - *Configuration:*  
    - Block ID: Extracted dynamically from the trigger node’s URL field.  
    - Operation: Get all blocks under the page.  
    - Return all: True (fetches all blocks without pagination)  
    - Credentials: Same Notion API credentials as above  
  - *Input:* Page URL from trigger node  
  - *Output:* Array of blocks with their types and content  
  - *Edge Cases:*  
    - Large pages may cause timeouts or partial data retrieval.  
    - API changes or permission issues may block content fetching.  
  - *Version:* 2.2

---

#### 1.2 Content Filtering and Preparation

**Overview:**  
Filters out non-text blocks (images, videos), concatenates the remaining text content into a single string, and splits it into token chunks suitable for embedding.

**Nodes Involved:**  
- Filter Non-Text Content  
- Summarize - Concatenate Notion's blocks content  
- Token Splitter

**Node Details:**

- **Filter Non-Text Content**  
  - *Type:* Filter node  
  - *Role:* Excludes blocks where the type is "image" or "video", allowing only textual content to pass.  
  - *Configuration:*  
    - Conditions: Block type not equal to "image" AND not equal to "video"  
  - *Input:* Blocks array from Notion content retrieval  
  - *Output:* Filtered blocks containing only text or other non-image/video types  
  - *Edge Cases:*  
    - If all blocks are images/videos, output will be empty, potentially halting downstream processing.  
  - *Version:* 2

- **Summarize - Concatenate Notion's blocks content**  
  - *Type:* Summarize node  
  - *Role:* Concatenates the "content" field of all filtered blocks into a single text string separated by new lines.  
  - *Configuration:*  
    - Output format: Separate items (each concatenated content as an item)  
    - Fields to summarize: "content" field, concatenated with newline separators  
  - *Input:* Filtered text blocks  
  - *Output:* Single concatenated text string representing the page content  
  - *Edge Cases:*  
    - If input is empty, output will be empty, affecting embedding generation.  
  - *Version:* 1

- **Token Splitter**  
  - *Type:* LangChain Text Splitter node  
  - *Role:* Splits the concatenated text into chunks of tokens for embedding processing.  
  - *Configuration:*  
    - Chunk size: 256 tokens  
    - Chunk overlap: 30 tokens (to preserve context between chunks)  
  - *Input:* Concatenated text from summarization  
  - *Output:* Array of token chunks  
  - *Edge Cases:*  
    - Very short text may produce a single chunk.  
    - Extremely long text may produce many chunks, potentially increasing processing time.  
  - *Version:* 1

---

#### 1.3 Metadata Enrichment and Embedding Generation

**Overview:**  
Adds relevant metadata to each text chunk and generates semantic embeddings using Google Gemini’s text embedding model.

**Nodes Involved:**  
- Create metadata and load content  
- Embeddings Google Gemini

**Node Details:**

- **Create metadata and load content**  
  - *Type:* LangChain Document Default Data Loader  
  - *Role:* Attaches metadata (page ID, creation time, page title) to the text chunks for traceability and context.  
  - *Configuration:*  
    - Metadata fields:  
      - pageId: from Notion trigger item JSON id  
      - createdTime: from Notion trigger item JSON created_time  
      - pageTitle: from Notion trigger item JSON properties.Name.title[0].text.content  
    - JSON data: Uses concatenated content expression from previous node  
  - *Input:* Token chunks from Token Splitter  
  - *Output:* Documents enriched with metadata ready for embedding  
  - *Edge Cases:*  
    - Missing metadata fields in Notion JSON may cause expression errors.  
  - *Version:* 1

- **Embeddings Google Gemini**  
  - *Type:* LangChain Embeddings node  
  - *Role:* Generates vector embeddings for each document chunk using Google Gemini’s "models/text-embedding-004".  
  - *Configuration:*  
    - Model name: "models/text-embedding-004"  
    - Credentials: Google Palm API credentials named "Google Gemini(PaLM) Api account"  
  - *Input:* Documents with metadata from previous node  
  - *Output:* Embeddings vectors corresponding to each text chunk  
  - *Edge Cases:*  
    - API rate limits or authentication failures may cause embedding generation errors.  
    - Model name must be valid and supported.  
  - *Version:* 1

---

#### 1.4 Vector Storage

**Overview:**  
Inserts the generated embeddings and associated metadata into the Pinecone vector store for persistent, searchable storage.

**Nodes Involved:**  
- Pinecone Vector Store

**Node Details:**

- **Pinecone Vector Store**  
  - *Type:* LangChain Vector Store node  
  - *Role:* Inserts embeddings into the Pinecone index named "notion-pages".  
  - *Configuration:*  
    - Mode: Insert (adds new vectors)  
    - Pinecone index: "notion-pages" (selected from list)  
    - Credentials: Pinecone API credentials named "Auto: PineconeApi"  
  - *Input:* Embeddings and documents with metadata from previous nodes  
  - *Output:* Confirmation of insertion or error messages  
  - *Edge Cases:*  
    - Index not found or misconfigured credentials will cause failures.  
    - Network issues may cause insertion timeouts.  
  - *Version:* 1

---

### 3. Summary Table

| Node Name                          | Node Type                                   | Functional Role                              | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                      |
|-----------------------------------|---------------------------------------------|----------------------------------------------|---------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| Notion - Page Added Trigger        | Notion Trigger                              | Detects new pages in Notion database          | None                            | Notion - Retrieve Page Content    |                                                                                                 |
| Notion - Retrieve Page Content     | Notion API                                  | Retrieves full content blocks of new page     | Notion - Page Added Trigger     | Filter Non-Text Content           |                                                                                                 |
| Filter Non-Text Content            | Filter                                      | Filters out images and videos                   | Notion - Retrieve Page Content  | Summarize - Concatenate Notion's blocks content |                                                                                                 |
| Summarize - Concatenate Notion's blocks content | Summarize                                  | Concatenates text blocks into single text      | Filter Non-Text Content          | Pinecone Vector Store (main), Token Splitter (implied) |                                                                                                 |
| Token Splitter                    | LangChain Text Splitter                      | Splits concatenated text into token chunks     | Summarize - Concatenate Notion's blocks content | Create metadata and load content |                                                                                                 |
| Create metadata and load content  | LangChain Document Data Loader               | Adds metadata to text chunks                    | Token Splitter                  | Pinecone Vector Store             |                                                                                                 |
| Embeddings Google Gemini          | LangChain Embeddings                         | Generates embeddings using Google Gemini model | Create metadata and load content | Pinecone Vector Store             |                                                                                                 |
| Pinecone Vector Store             | LangChain Vector Store                       | Inserts embeddings and metadata into Pinecone | Embeddings Google Gemini, Summarize - Concatenate Notion's blocks content, Create metadata and load content | None                              |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Notion - Page Added Trigger**  
   - Type: Notion Trigger  
   - Set to poll every minute.  
   - Configure database ID to your target Notion database (e.g., "17b11930-c10f-8000-a545-ece7cade03f9").  
   - Attach Notion API credentials with appropriate access.

2. **Create Node: Notion - Retrieve Page Content**  
   - Type: Notion API node  
   - Operation: Get all blocks under a block (resource: block, operation: getAll).  
   - Set block ID dynamically from trigger node’s URL field (`={{ $json.url }}`).  
   - Use same Notion API credentials.

3. **Create Node: Filter Non-Text Content**  
   - Type: Filter node  
   - Configure conditions to exclude blocks where `type` equals "image" or "video".  
   - Connect input from Notion content retrieval node.

4. **Create Node: Summarize - Concatenate Notion's blocks content**  
   - Type: Summarize node  
   - Set output format to "separate items".  
   - Configure to concatenate the "content" field of all input items, separated by newline characters.  
   - Connect input from Filter node.

5. **Create Node: Token Splitter**  
   - Type: LangChain Text Splitter node  
   - Set chunk size to 256 tokens and chunk overlap to 30 tokens.  
   - Connect input from Summarize node.

6. **Create Node: Create metadata and load content**  
   - Type: LangChain Document Default Data Loader  
   - Configure metadata fields:  
     - `pageId` = `={{ $('Notion - Page Added Trigger').item.json.id }}`  
     - `createdTime` = `={{ $('Notion - Page Added Trigger').item.json.created_time }}`  
     - `pageTitle` = `={{ $('Notion - Page Added Trigger').item.json.properties.Name.title[0].text.content }}`  
   - Set JSON data to the concatenated content expression (`={{ $json.concatenated_content }}`).  
   - Connect input from Token Splitter node.

7. **Create Node: Embeddings Google Gemini**  
   - Type: LangChain Embeddings node  
   - Set model name to "models/text-embedding-004".  
   - Attach Google Palm API credentials with access to Google Gemini.  
   - Connect input from Create metadata and load content node.

8. **Create Node: Pinecone Vector Store**  
   - Type: LangChain Vector Store node  
   - Set mode to "insert".  
   - Select Pinecone index named "notion-pages" or your own index.  
   - Attach Pinecone API credentials.  
   - Connect inputs from:  
     - Embeddings Google Gemini (embedding input)  
     - Create metadata and load content (document input)  
     - Summarize node (main input) — as per workflow connections, ensure proper mapping.

9. **Connect all nodes in the order described:**  
   - Notion Trigger → Notion Retrieve Content → Filter → Summarize → Token Splitter → Create Metadata → Embeddings → Pinecone Vector Store.

10. **Activate the workflow.**  
    - Ensure all credentials are valid and have required permissions.  
    - Test with new pages added to the Notion database to verify end-to-end processing.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow runs every minute to ensure near real-time ingestion of new Notion pages.             | Timing configuration in Notion Trigger node.                                                   |
| Google Gemini (PaLM) API is used for embedding generation, requiring valid Google Cloud credentials.| Google Cloud Console and API setup for PaLM API.                                               |
| Pinecone vector store enables semantic search and fast retrieval of document embeddings.            | Pinecone documentation: https://www.pinecone.io/docs/                                           |
| The workflow is tagged as "Production" indicating readiness for live environments.                  | Tag metadata in workflow configuration.                                                        |
| For best results, ensure Notion database schema includes a "Name" property for page titles.         | Required for metadata extraction in the loader node.                                           |

---

This structured documentation provides a complete understanding of the "Notion to Pinecone Vector Store Integration" workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents alike.