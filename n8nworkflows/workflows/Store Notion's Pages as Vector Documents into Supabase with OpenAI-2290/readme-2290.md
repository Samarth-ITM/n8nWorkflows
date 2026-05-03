Store Notion's Pages as Vector Documents into Supabase with OpenAI

https://n8nworkflows.xyz/workflows/store-notion-s-pages-as-vector-documents-into-supabase-with-openai-2290


# Store Notion's Pages as Vector Documents into Supabase with OpenAI

### 1. Workflow Overview

This n8n workflow automates the ingestion of newly added Notion pages into a Supabase database that supports vector embeddings. It is designed to efficiently transform textual content from Notion pages into vector documents stored in Supabase, enabling AI-driven search or retrieval. The workflow is divided into these logical blocks:

**1.1 Notion Page Monitoring and Content Retrieval**  
Detects newly added pages in a specified Notion database and fetches their full block content.

**1.2 Content Filtering and Summarization**  
Filters out non-text media blocks (images, videos) and concatenates textual content blocks into a single summarized text for embedding.

**1.3 Content Processing for Vector Storage**  
Splits the summarized text into chunks, generates embeddings via OpenAI, attaches metadata, and stores the resulting vector documents in Supabase.

---

### 2. Block-by-Block Analysis

#### 2.1 Notion Page Monitoring and Content Retrieval

- **Overview:**  
  This block triggers the workflow each time a new page is added to a specified Notion database, then retrieves all the block-level content of that page for further processing.

- **Nodes Involved:**  
  - `Notion - Page Added Trigger`  
  - `Notion - Retrieve Page Content`

- **Node Details:**

  - **Notion - Page Added Trigger**  
    - **Type & Role:** Notion Trigger node; monitors a Notion database for new pages.  
    - **Configuration:** Set to poll every minute (`pollTimes`), monitoring a specific Notion database ID (user-defined).  
    - **Key Expressions:** Triggers based on new page entries; outputs page metadata including page ID and URL.  
    - **Connections:** Outputs to `Notion - Retrieve Page Content`.  
    - **Edge Cases:** Polling frequency could miss rapid updates; requires proper Notion API credentials and access to the database.  
    - **Version:** v1

  - **Notion - Retrieve Page Content**  
    - **Type & Role:** Notion node; fetches all blocks of content from the triggered page.  
    - **Configuration:** Uses the page URL from trigger (`={{ $json.url }}`) to get all blocks (`getAll: true`).  
    - **Connections:** Outputs to `Filter Non-Text Content`.  
    - **Edge Cases:** Large pages may cause timeouts or incomplete fetches; API rate limits apply.  
    - **Version:** v2.2

#### 2.2 Content Filtering and Summarization

- **Overview:**  
  This block filters out non-text content blocks such as images and videos and then summarizes (concatenates) the remaining textual content into a single text string for embedding.

- **Nodes Involved:**  
  - `Filter Non-Text Content`  
  - `Summarize - Concatenate Notion's blocks content`

- **Node Details:**

  - **Filter Non-Text Content**  
    - **Type & Role:** Filter node; excludes blocks of type `image` and `video`.  
    - **Configuration:** Filters where `$json.type` is not equal to `"image"` and not equal to `"video"`.  
    - **Connections:** Outputs filtered blocks to summarization node.  
    - **Edge Cases:** If all blocks are media, the output would be empty; the workflow should handle empty summaries gracefully.  
    - **Version:** v2

  - **Summarize - Concatenate Notion's blocks content**  
    - **Type & Role:** Summarize node; concatenates the `content` field of each block into a single text with new line separators.  
    - **Configuration:** Output format is separate items, concatenating the `content` field separated by newline characters.  
    - **Connections:** Outputs concatenated content to both `Supabase Vector Store` and `Create metadata and load content` nodes.  
    - **Edge Cases:** If blocks have missing or empty `content` fields, concatenation may produce incomplete text.  
    - **Version:** v1

#### 2.3 Content Processing for Vector Storage

- **Overview:**  
  This block prepares the summarized text for vector storage by splitting it into chunks, generating embeddings with OpenAI, attaching metadata, and inserting the vector documents into Supabase.

- **Nodes Involved:**  
  - `Token Splitter`  
  - `Create metadata and load content`  
  - `Embeddings OpenAI`  
  - `Supabase Vector Store`

- **Node Details:**

  - **Token Splitter**  
    - **Type & Role:** Text splitter node; splits text into chunks for embedding generation.  
    - **Configuration:** Chunk size set to 256 tokens, with 30 tokens overlap between chunks to maintain context.  
    - **Connections:** Outputs chunks to `Create metadata and load content`.  
    - **Edge Cases:** Incorrect chunk sizes can lead to inefficient embeddings or truncated context.  
    - **Version:** v1

  - **Create metadata and load content**  
    - **Type & Role:** Document loader node; attaches metadata and prepares documents for embedding.  
    - **Configuration:** Attaches metadata fields: `pageId`, `createdTime`, and `pageTitle` extracted from the Notion trigger node. Loads JSON data from the summarized concatenated content.  
    - **Connections:** Outputs documents to `Supabase Vector Store`.  
    - **Edge Cases:** Missing metadata fields or empty content may cause failures during storage.  
    - **Version:** v1

  - **Embeddings OpenAI**  
    - **Type & Role:** Embedding generator; uses OpenAI to produce vector embeddings of text chunks.  
    - **Configuration:** Default OpenAI embedding options; requires OpenAI credentials configured in n8n.  
    - **Connections:** Outputs embeddings to `Supabase Vector Store`.  
    - **Edge Cases:** API quota limits, network errors, or malformed inputs can cause failures.  
    - **Version:** v1

  - **Supabase Vector Store**  
    - **Type & Role:** Vector store node; inserts vector documents into a Supabase table with a vector column.  
    - **Configuration:** Insert mode enabled; target table configured by user (must have vector column).  
    - **Connections:** Receives data from both `Summarize - Concatenate Notion's blocks content` (for main data) and embedding/document loaders.  
    - **Edge Cases:** Database connection failures, incorrect table schema, or vector column misconfiguration can cause errors.  
    - **Version:** v1

---

### 3. Summary Table

| Node Name                                 | Node Type                                             | Functional Role                              | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                                                              |
|-------------------------------------------|-------------------------------------------------------|----------------------------------------------|-----------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                               | stickyNote                                            | Workflow description and instructions       |                                   |                                       | ## Store Notion's Pages as Vector Documents into Supabase. [Supabase Vector Columns Guide](https://supabase.com/docs/guides/ai/vector-columns) |
| Notion - Page Added Trigger               | notionTrigger                                         | Trigger on new Notion page                    |                                   | Notion - Retrieve Page Content        |                                                                                                                                          |
| Notion - Retrieve Page Content            | notion                                                | Retrieve all blocks content from Notion page| Notion - Page Added Trigger        | Filter Non-Text Content                |                                                                                                                                          |
| Filter Non-Text Content                   | filter                                                | Exclude images and videos                      | Notion - Retrieve Page Content     | Summarize - Concatenate Notion's blocks content |                                                                                                                                          |
| Summarize - Concatenate Notion's blocks content | summarize                                            | Concatenate text blocks into one summary     | Filter Non-Text Content            | Supabase Vector Store, Create metadata and load content |                                                                                                                                          |
| Token Splitter                           | textSplitter                                         | Split text into chunks for embedding          | (Not directly connected in main flow; used downstream) | Create metadata and load content       |                                                                                                                                          |
| Create metadata and load content          | documentDefaultDataLoader                             | Attach metadata and prepare documents         | Token Splitter                    | Supabase Vector Store                  |                                                                                                                                          |
| Embeddings OpenAI                        | embeddingsOpenAi                                     | Generate vector embeddings using OpenAI       | (Presumed from metadata loader)    | Supabase Vector Store                  |                                                                                                                                          |
| Supabase Vector Store                    | vectorStoreSupabase                                  | Insert vector documents into Supabase         | Summarize - Concatenate Notion's blocks content, Create metadata and load content, Embeddings OpenAI |                                       |                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Notion Trigger Node:**  
   - Type: `Notion Trigger`  
   - Configure to poll every minute.  
   - Set the database ID to the target Notion database you want to monitor for new pages.  
   - Ensure Notion credentials with required scopes are configured.

2. **Add a Notion Node to Retrieve Page Content:**  
   - Type: `Notion`  
   - Operation: `Get All` blocks from a page.  
   - Set the `blockId` parameter dynamically to the URL output from the trigger (`={{ $json.url }}`).  
   - Connect output of the trigger node to this node.

3. **Add a Filter Node to Exclude Media Content:**  
   - Type: `Filter`  
   - Configure conditions to exclude blocks where `type` equals `"image"` or `"video"`.  
   - Connect output of Notion content retrieval to this filter node.

4. **Add a Summarize Node to Concatenate Text Blocks:**  
   - Type: `Summarize`  
   - Fields to summarize: `content` field of each block.  
   - Separator: New line (`\n`).  
   - Output format: Separate items.  
   - Connect filter node output here.

5. **Add a Token Splitter Node:**  
   - Type: `Text Splitter (Token Splitter)`  
   - Configure chunk size: 256 tokens.  
   - Configure chunk overlap: 30 tokens.  
   - Connect the summarized content output here (if needed as per design, or connect downstream).

6. **Add a Document Loader Node to Create Metadata and Load Content:**  
   - Type: `Document Default Data Loader`  
   - Metadata fields:  
     - `pageId` = `={{ $('Notion - Page Added Trigger').item.json.id }}`  
     - `createdTime` = `={{ $('Notion - Page Added Trigger').item.json.created_time }}`  
     - `pageTitle` = `={{ $('Notion - Page Added Trigger').item.json.properties.Page.title[0].text.content }}`  
   - JSON Data: Use the concatenated content from the summarize node (`={{ $('Summarize - Concatenate Notion's blocks content').item.json.concatenated_content }}`).  
   - Connect Token Splitter output to this node.

7. **Add an OpenAI Embeddings Node:**  
   - Type: `Embeddings OpenAI`  
   - Configure with your OpenAI API credentials.  
   - Connect the document loader node output here.

8. **Add a Supabase Vector Store Node:**  
   - Type: `Vector Store Supabase`  
   - Mode: Insert.  
   - Configure the table name where vector documents will be stored (table must have a vector column).  
   - Connect outputs from:  
     - Summarize node (main content)  
     - Document loader node (documents with metadata)  
     - OpenAI Embeddings node (embeddings)  
   - Configure Supabase credentials with access to the appropriate project and table.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow assumes a Supabase project with a table containing a vector column. If absent, follow the guide.  | [Supabase Langchain Guide](https://supabase.com/docs/guides/ai/langchain?queryGroups=database-method&database-method=sql) |
| Updated on 17/06/2024: Added summarization to avoid creating a row per Notion block.                            | Workflow change log                                                                                            |
| Supabase vector column setup instructions are crucial for this workflow to function correctly.                  | [Supabase Vector Columns Guide](https://supabase.com/docs/guides/ai/vector-columns)                             |

---

This structured document fully describes the workflow for both human users and AI agents to understand, reproduce, and troubleshoot the integration of Notion pages with Supabase vector storage using OpenAI embeddings.