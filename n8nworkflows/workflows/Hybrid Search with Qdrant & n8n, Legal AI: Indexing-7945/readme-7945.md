Hybrid Search with Qdrant & n8n, Legal AI: Indexing

https://n8nworkflows.xyz/workflows/hybrid-search-with-qdrant---n8n--legal-ai--indexing-7945


# Hybrid Search with Qdrant & n8n, Legal AI: Indexing

### 1. Workflow Overview

This workflow, **"Hybrid Search with Qdrant & n8n, Legal AI: Indexing"**, is designed to index a legal Q&A dataset into Qdrant for enabling hybrid search. It targets legal AI use cases where search combines semantic similarity (dense vectors) and keyword relevance (sparse vectors using BM25). The dataset originates from Hugging Face’s LegalQAEval corpus.

**Key Use Cases:**
- Indexing legal documents and Q&A pairs for hybrid retrieval.
- Supporting two embedding strategies:  
  a) Qdrant Cloud Inference with mxbai embeddings (1024-dim cosine similarity dense vectors).  
  b) External embedding provider (OpenAI text-embedding-3-small, 1536-dim cosine similarity dense vectors).
- Creating and managing Qdrant collections with appropriate vector configurations.
- Deduplicating dataset texts and aggregating related Q&A IDs.
- Batch processing with pagination for scalability.

**Logical Blocks:**

- **1.1 Dataset Retrieval and Preparation**  
  Fetches the legal dataset splits and rows from Hugging Face, deduplicates passages, aggregates Q&A IDs, and prepares data for batching.

- **1.2 Average Text Length Estimation**  
  Calculates average word counts per text chunk to support BM25 sparse vector configuration.

- **1.3 Qdrant Collection Management and Indexing Using Qdrant Cloud Inference**  
  Checks for collection existence, creates it if missing, and indexes text chunks using Qdrant’s internal embedding inference and sparse BM25 vectors.

- **1.4 Qdrant Collection Management and Indexing Using OpenAI Embeddings**  
  Manages a separate collection for OpenAI embeddings, performs external embedding calls, and indexes the data with BM25 sparse vectors.

- **Additional:** Includes sticky notes with documentation, explanations, and links to resources for maintainers and users.


---

### 2. Block-by-Block Analysis

---

#### 1.1 Dataset Retrieval and Preparation

**Overview:**  
This block retrieves the LegalQAEval dataset splits and rows using Hugging Face’s Dataset Viewer API, applies pagination to fetch the entire dataset, deduplicates text passages, aggregates relevant Q&A IDs linked to each passage, and formats data for batch processing.

**Nodes Involved:**  
- Index Dataset from HuggingFace (Manual Trigger)  
- Get Dataset Splits (HTTP Request)  
- Split Them All Out (Split Out)  
- Get Dataset Rows (Pagination) (HTTP Request with pagination)  
- Divide Per Row (Split Out)  
- Restructure for Deduplicating (Set)  
- Deduplicate Texts (Summarize)  
- Restructure for Batching (Set)  
- Merge (Merge)  

**Node Details:**

- **Index Dataset from HuggingFace**  
  - Type: Manual Trigger  
  - Role: Starts dataset fetching process.  
  - Outputs dataset parameter (`isaacus/LegalQAEval`) to next node.  
  - Edge cases: Manual start required; no direct failure points.

- **Get Dataset Splits**  
  - Type: HTTP Request  
  - Role: Calls Hugging Face API to retrieve dataset splits metadata.  
  - Config: Query parameter `dataset` set dynamically from trigger JSON.  
  - Output: List of splits for dataset.  
  - Edge cases: API failure, network errors.

- **Split Them All Out**  
  - Type: Split Out  
  - Role: Splits the array of splits into individual items for processing.  
  - Input: Dataset splits array.  
  - Output: Each split separately.  
  - Edge cases: Empty splits, malformed data.

- **Get Dataset Rows (Pagination)**  
  - Type: HTTP Request with Pagination  
  - Role: Fetches dataset rows in batches of 100 with pagination until all rows are retrieved.  
  - Config: Pagination by offset with 100 per page, waits 1000ms between requests.  
  - Input: Dataset, config, and split parameters from previous node.  
  - Output: Accumulated rows of dataset.  
  - Edge cases: Pagination errors, API rate limits, incomplete data retrieval.

- **Divide Per Row**  
  - Type: Split Out  
  - Role: Splits all rows into individual items for processing.  
  - Input: Accumulated rows array.  
  - Output: Single row per item.  
  - Edge cases: Empty rows, malformed data.

- **Restructure for Deduplicating**  
  - Type: Set  
  - Role: Prepares each row by extracting `id` and `text` fields for deduplication.  
  - Assigns fields: `id_qa` from row `id`, `text` from row `text`.  
  - Edge cases: Missing fields.

- **Deduplicate Texts**  
  - Type: Summarize  
  - Role: Deduplicates dataset by `text` field, aggregates all corresponding `id_qa` values into an array.  
  - Output: Unique text entries with aggregated question-answer IDs.  
  - Edge cases: Large aggregation sizes, memory issues.

- **Restructure for Batching**  
  - Type: Set  
  - Role: Prepares deduplicated data for batching by adding an index (`idx`), `text`, and aggregated `ids_qa`.  
  - Edge cases: Indexing correctness.

- **Merge**  
  - Type: Merge  
  - Role: Combines multiple streams; here used to merge processed batches.  
  - Edge cases: Merging errors if streams misaligned.

---

#### 1.2 Average Text Length Estimation

**Overview:**  
Calculates the average number of words per text chunk in a sample of the dataset. This average length (`avg_len`) is a parameter for BM25 sparse vector configuration.

**Nodes Involved:**  
- Limit (Limit)  
- Calculate #words in Each Text (Set)  
- Sum them Up (Summarize)  
- Get the Average Text Length (Set)  
- Merge (Merge)  

**Node Details:**

- **Limit**  
  - Type: Limit  
  - Role: Restricts the dataset size to 500 items for manageable sampling.  
  - Config: `maxItems` = 500.  
  - Edge cases: If dataset smaller than 500, processes all.

- **Calculate #words in Each Text**  
  - Type: Set  
  - Role: Counts words in each text chunk using whitespace splitting regex.  
  - Stores result in `words_in_text`.  
  - Edge cases: Empty or malformed text.

- **Sum them Up**  
  - Type: Summarize  
  - Role: Sums all `words_in_text` values to get total words count.  
  - Edge cases: Integer overflow unlikely but possible with huge data.

- **Get the Average Text Length**  
  - Type: Set  
  - Role: Calculates average length by dividing total words by number of items (500).  
  - Sets `avg_len` field for BM25 use.  
  - Edge cases: Division by zero if dataset empty (unlikely due to prior limit).

- **Merge**  
  - Role: Combines streams for downstream processing.

---

#### 1.3 Qdrant Collection Management and Indexing Using Qdrant Cloud Inference

**Overview:**  
This block manages Qdrant collection creation and indexing of text chunks using Qdrant's built-in Cloud Inference for embeddings. It creates a collection configured with dense vectors (mxbai embeddings) and sparse vectors (BM25), indexes data in batches, and supports hybrid retrieval.

**Nodes Involved:**  
- Check Collection Exists (Qdrant)  
- If (Conditional)  
- Create Collection (Qdrant)  
- Loop Over Batches (Split In Batches)  
- Aggregate a Batch (Aggregate)  
- Upsert Points (Qdrant)  

**Node Details:**

- **Check Collection Exists**  
  - Type: Qdrant node  
  - Role: Checks if `legalQA_test` collection exists in Qdrant.  
  - Credentials: Qdrant API configured.  
  - Edge cases: Auth failures, network errors.

- **If**  
  - Type: Conditional  
  - Role: Branches the flow based on collection existence.  
  - Condition: `$json.result.exists === true`  
  - True branch: Skips creation.  
  - False branch: Creates collection.  
  - Edge cases: Expression evaluation failure.

- **Create Collection**  
  - Type: Qdrant node  
  - Role: Creates `legalQA_test` collection with vectors:  
    - Dense vector `mxbai_large`: size 1024, cosine similarity metric.  
    - Sparse vector `bm25`: uses IDF modifier.  
  - Edge cases: API errors, cluster limits.

- **Loop Over Batches**  
  - Type: Split In Batches  
  - Role: Processes data in batches of 8 for indexing.  
  - Config: `batchSize=8`.  
  - Edge cases: Batch size too large/small.

- **Aggregate a Batch**  
  - Type: Aggregate  
  - Role: Aggregates batch items into array for bulk indexing.  
  - Edge cases: Large batch aggregation memory.

- **Upsert Points**  
  - Type: Qdrant node  
  - Role: Upserts points into `legalQA_test` collection with:  
    - Dense vector generated by Qdrant Cloud Inference on text using `mixedbread-ai/mxbai-embed-large-v1` model.  
    - Sparse vector BM25 with average length parameter.  
  - Edge cases: API errors, batch size limits.

---

#### 1.4 Qdrant Collection Management and Indexing Using OpenAI Embeddings

**Overview:**  
Manages a separate Qdrant collection configured for OpenAI embeddings. Retrieves OpenAI embeddings externally, merges with sparse BM25 vectors, and indexes data in batches.

**Nodes Involved:**  
- Check Collection Exists1 (Qdrant)  
- If1 (Conditional)  
- Create Collection1 (Qdrant)  
- Loop Over Batches1 (Split In Batches)  
- Edit Fields (Set)  
- Merge1 (Merge)  
- Aggregate a Batch to Embed (Aggregate)  
- Get OpenAI embeddings (HTTP Request)  
- Split Out (Split Out)  
- Aggregate a Batch to Upsert (Aggregate)  
- Upsert Points1 (Qdrant)  

**Node Details:**

- **Check Collection Exists1**  
  - Checks if `legalQA_openAI_test` collection exists.  
  - Credentials: Same Qdrant API.  
  - Edge cases: Auth, network errors.

- **If1**  
  - Branches creation of collection.  
  - Condition: `$json.result.exists === true`

- **Create Collection1**  
  - Creates collection `legalQA_openAI_test` with vectors:  
    - Dense vector `open_ai_small`: size 1536, cosine similarity metric (OpenAI embedding size).  
    - Sparse vector `bm25` with IDF modifier.  
  - Edge cases: API errors.

- **Loop Over Batches1**  
  - Processes data in batches of 8 for OpenAI embedding and indexing.

- **Edit Fields**  
  - Prepares data for embedding call or indexing.

- **Merge1**  
  - Combines streams for embedding and indexing.

- **Aggregate a Batch to Embed**  
  - Aggregates batch for embedding request.

- **Get OpenAI embeddings**  
  - Calls OpenAI API with `text-embedding-3-small` model.  
  - Input: Batch of text strings.  
  - Output: Embeddings.  
  - Credentials: OpenAI API key.  
  - Edge cases: API rate limits, auth errors, input size limits.

- **Split Out**  
  - Splits returned embeddings for processing.

- **Aggregate a Batch to Upsert**  
  - Aggregates batch with embedding data for upsert.

- **Upsert Points1**  
  - Upserts points to `legalQA_openAI_test` collection with:  
    - Dense vector from OpenAI embeddings.  
    - Sparse BM25 vector with average length parameter.  
  - Edge cases: API errors, batch size limits.


---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                                        | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                        |
|-----------------------------|-------------------------|-------------------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Index Dataset from HuggingFace | Manual Trigger          | Starts dataset retrieval process                       |                              | Get Dataset Splits            | ## Get Dataset from Hugging Face: fetching LegalQAEval dataset and deduplicating texts before indexing.                           |
| Get Dataset Splits           | HTTP Request            | Retrieves dataset splits metadata                      | Index Dataset from HuggingFace | Split Them All Out            | See above                                                                                                                         |
| Split Them All Out           | Split Out               | Splits dataset splits array into individual items     | Get Dataset Splits            | Get Dataset Rows (Pagination) | See above                                                                                                                         |
| Get Dataset Rows (Pagination) | HTTP Request (paginated) | Fetches dataset rows with pagination                   | Split Them All Out            | Divide Per Row                | See above                                                                                                                         |
| Divide Per Row               | Split Out               | Splits rows array into individual rows                 | Get Dataset Rows (Pagination) | Restructure for Deduplicating | See above                                                                                                                         |
| Restructure for Deduplicating | Set                     | Extracts id and text fields for deduplication          | Divide Per Row                | Deduplicate Texts             | See above                                                                                                                         |
| Deduplicate Texts            | Summarize               | Deduplicates texts and aggregates Q&A IDs             | Restructure for Deduplicating | Restructure for Batching      | See above                                                                                                                         |
| Restructure for Batching     | Set                     | Prepares data for batching with idx, text, and ids_qa | Deduplicate Texts             | Limit                        | See above                                                                                                                         |
| Limit                       | Limit                   | Limits data size to 500 for average length estimation  | Restructure for Batching      | Calculate #words in Each Text | ## Estimate Average Length of Text Chunks for BM25 weighting                                                                    |
| Calculate #words in Each Text | Set                     | Counts words per text chunk                             | Limit                        | Sum them Up                  | See above                                                                                                                         |
| Sum them Up                 | Summarize               | Sums word counts                                       | Calculate #words in Each Text | Get the Average Text Length  | See above                                                                                                                         |
| Get the Average Text Length  | Set                     | Calculates average text length for BM25                | Sum them Up                  | Merge                       | See above                                                                                                                         |
| Merge                       | Merge                   | Merges streams for further processing                   | Get the Average Text Length, Restructure for Batching | Loop Over Batches, Loop Over Batches1 |                                                                                                           |
| Check Collection Exists      | Qdrant                  | Checks if collection `legalQA_test` exists             | Index Dataset from HuggingFace | If                          | ## Create Qdrant Collection for Hybrid Search with mxbai embeddings and BM25 sparse vectors                                        |
| If                          | If                      | Branches on collection existence                        | Check Collection Exists      | Create Collection (false) or Loop Over Batches (true) |                                                                                                           |
| Create Collection            | Qdrant                  | Creates collection with dense mxbai and sparse BM25 vectors | If (false branch)            |                              | See above                                                                                                                         |
| Loop Over Batches            | Split In Batches        | Splits data into batches of 8 for indexing              | Merge                       | Aggregate a Batch            | ## Index using Qdrant Cloud Inference (dense embeddings + BM25 sparse vectors)                                                    |
| Aggregate a Batch            | Aggregate               | Aggregates batch items for bulk indexing                | Loop Over Batches            | Upsert Points                | See above                                                                                                                         |
| Upsert Points               | Qdrant                  | Upserts points with dense mxbai embeddings and BM25    | Aggregate a Batch            | Loop Over Batches (loop)     | See above                                                                                                                         |
| Check Collection Exists1     | Qdrant                  | Checks if collection `legalQA_openAI_test` exists      | Merge                       | If1                         | ## Create separate Qdrant collection for OpenAI embeddings (text-embedding-3-small) and BM25 sparse vectors                      |
| If1                         | If                      | Branches on collection existence                        | Check Collection Exists1     | Create Collection1 (false) or Loop Over Batches1 (true) |                                                                                                           |
| Create Collection1           | Qdrant                  | Creates collection with dense OpenAI and sparse BM25 vectors | If1 (false branch)           |                              | See above                                                                                                                         |
| Loop Over Batches1           | Split In Batches        | Splits data into batches of 8 for OpenAI embedding and indexing | Merge                       | Edit Fields, Aggregate a Batch to Embed |                                                                                                           |
| Edit Fields                 | Set                     | Prepares data fields for OpenAI embedding call         | Loop Over Batches1           | Merge1                       |                                                                                                                                    |
| Merge1                      | Merge                   | Merges streams before embedding                         | Edit Fields, Loop Over Batches1 output | Aggregate a Batch to Upsert, Aggregate a Batch to Embed |                                                                                                           |
| Aggregate a Batch to Embed   | Aggregate               | Aggregates batch items for OpenAI embedding request    | Merge1                      | Get OpenAI embeddings        |                                                                                                                                    |
| Get OpenAI embeddings        | HTTP Request            | Calls OpenAI API to get embeddings                      | Aggregate a Batch to Embed   | Split Out                   |                                                                                                                                    |
| Split Out                   | Split Out               | Splits OpenAI embeddings array for processing          | Get OpenAI embeddings        | Aggregate a Batch to Upsert  |                                                                                                                                    |
| Aggregate a Batch to Upsert  | Aggregate               | Aggregates batch for indexing with OpenAI embeddings   | Split Out                   | Upsert Points1               |                                                                                                                                    |
| Upsert Points1              | Qdrant                  | Upserts points with OpenAI embeddings and BM25 sparse vectors | Aggregate a Batch to Upsert  | Loop Over Batches1 (loop)    |                                                                                                                                    |
| Sticky Note                 | Sticky Note             | Documentation and usage instructions                    |                              |                              | Multiple sticky notes throughout workflow provide detailed documentation and resource links.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "Index Dataset from HuggingFace" to initiate the workflow.

2. **Add HTTP Request node** "Get Dataset Splits":  
   - Method: GET  
   - URL: `https://datasets-server.huggingface.co/splits`  
   - Query parameter: `dataset` set to `={{ $json.dataset }}` (from trigger input)  
   - Connect manual trigger output to this node.

3. **Add Split Out node** "Split Them All Out" to split the splits array.

4. **Add HTTP Request node** "Get Dataset Rows (Pagination)":  
   - Method: GET  
   - URL: `https://datasets-server.huggingface.co/rows`  
   - Query parameters:  
     - dataset, config, split dynamically taken from input JSON  
     - length = 100  
   - Enable pagination with offset, 100 items per page, 1000ms delay between requests, and completion condition based on total rows.

5. **Add Split Out node** "Divide Per Row" to split rows into individual items.

6. **Add Set node** "Restructure for Deduplicating":  
   - Assign `id_qa` = `{{$json.row.id}}`  
   - Assign `text` = `{{$json.row.text}}`

7. **Add Summarize node** "Deduplicate Texts":  
   - Split by `text`  
   - Summarize `id_qa` by appending all IDs.

8. **Add Set node** "Restructure for Batching":  
   - Assign `idx` = `{{$itemIndex}}` (index of item)  
   - Assign `text` = `{{$json.text}}`  
   - Assign `ids_qa` = `{{$json.appended_id_qa}}`

9. **Add Limit node** "Limit":  
   - Set max items to 500 to limit sample size for average length calculation.

10. **Add Set node** "Calculate #words in Each Text":  
    - Assign `words_in_text` = `{{$json.text.trim().split(/\s+/).length}}`

11. **Add Summarize node** "Sum them Up":  
    - Sum `words_in_text`

12. **Add Set node** "Get the Average Text Length":  
    - Assign `avg_len` = `{{$json.sum_words_in_text / 500}}`

13. **Add Merge node** "Merge" to combine streams from:  
    - "Get the Average Text Length" output  
    - "Restructure for Batching" output

14. **Qdrant Collection Management & Indexing with Qdrant Cloud Inference:**

    a. **Add Qdrant node** "Check Collection Exists":  
       - Operation: `collectionExists`  
       - Collection Name: `legalQA_test`  
       - Use Qdrant API credentials.

    b. **Add If node** "If":  
       - Condition: `$json.result.exists === true`  
       - True: bypass collection creation  
       - False: proceed to create collection

    c. **Add Qdrant node** "Create Collection":  
       - Operation: `createCollection`  
       - Collection Name: `legalQA_test`  
       - Vectors config:  
         - Dense vector `mxbai_large` size 1024, distance metric `Cosine`  
         - Sparse vector `bm25` with modifier `idf`  
       - Use same Qdrant credentials.

    d. **Add Split In Batches node** "Loop Over Batches":  
       - Batch size: 8

    e. **Add Aggregate node** "Aggregate a Batch":  
       - Aggregate all item data into `batch`

    f. **Add Qdrant node** "Upsert Points":  
       - Operation: `upsertPoints`  
       - Collection Name: `legalQA_test`  
       - Points payload for each batch item:  
         - `id`: `idx`  
         - `payload`: includes `text` and `ids_qa`  
         - `vector`: dense vector generated by Qdrant using model `mixedbread-ai/mxbai-embed-large-v1` on text  
         - Sparse vector BM25 configured with `avg_len`  
       - Use Qdrant credentials.

15. **Qdrant Collection Management & Indexing with OpenAI Embeddings:**

    a. **Add Merge node** "Merge" (reuse or new) to merge input data stream for OpenAI processing.

    b. **Add Qdrant node** "Check Collection Exists1":  
       - Operation: `collectionExists`  
       - Collection Name: `legalQA_openAI_test`  
       - Use Qdrant API credentials.

    c. **Add If node** "If1":  
       - Condition: `$json.result.exists === true`

    d. **Add Qdrant node** "Create Collection1":  
       - Operation: `createCollection`  
       - Collection Name: `legalQA_openAI_test`  
       - Vectors config:  
         - Dense vector `open_ai_small` size 1536, distance metric `Cosine`  
         - Sparse vector `bm25` with modifier `idf`  
       - Use Qdrant credentials.

    e. **Add Split In Batches node** "Loop Over Batches1":  
       - Batch size: 8

    f. **Add Set node** "Edit Fields":  
       - Prepare or adjust fields if necessary before embedding.

    g. **Add Merge node** "Merge1" to combine streams before embedding.

    h. **Add Aggregate node** "Aggregate a Batch to Embed":  
       - Aggregate all batch items into `batch`

    i. **Add HTTP Request node** "Get OpenAI embeddings":  
       - Method: POST  
       - URL: `https://api.openai.com/v1/embeddings`  
       - Body parameters:  
         - `input`: array of text strings from batch  
         - `model`: `text-embedding-3-small`  
       - Authentication: OpenAI API credentials.

    j. **Add Split Out node** "Split Out":  
       - Splits embeddings array per item.

    k. **Add Aggregate node** "Aggregate a Batch to Upsert":  
       - Aggregate batch items with embeddings.

    l. **Add Qdrant node** "Upsert Points1":  
       - Operation: `upsertPoints`  
       - Collection Name: `legalQA_openAI_test`  
       - Points payload:  
         - `id`: `idx`  
         - `payload`: `text` and `ids_qa`  
         - Dense vector from OpenAI embedding  
         - Sparse vector BM25 with `avg_len`  
       - Use Qdrant credentials.

16. **Connect all nodes according to the dependencies detailed above to form the complete workflow.**

17. **Configure Credentials:**  
    - Add Qdrant API credentials with URL and API_KEY for Qdrant Cloud.  
    - Add OpenAI API credentials with valid API key.

18. **Test the workflow manually triggering the first node and verify successful dataset retrieval, embedding, and indexing.**

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|----------------|
| This workflow is the first part ("Indexing") of a two-part pipeline. The second part covers retrieval and evaluation. | Workflow description |
| Hybrid Search combines dense semantic vectors and sparse keyword vectors (BM25) for improved search accuracy. | https://qdrant.tech/articles/hybrid-search/ |
| Dataset used: LegalQAEval from Hugging Face by user isaacus. | https://huggingface.co/datasets/isaacus/LegalQAEval |
| Embedding options: Qdrant Cloud Inference with mxbai-embed-large-v1 (1024 dim), or OpenAI text-embedding-3-small (1536 dim). | https://qdrant.tech/documentation/cloud/inference/, https://platform.openai.com/docs/models/text-embedding-3-small |
| Qdrant collections must be configured with vector size and distance metric matching embedding models. | https://qdrant.tech/documentation/concepts/collections/ |
| BM25 sparse vectors use IDF modifier for keyword-based retrieval on Qdrant. | https://qdrant.tech/documentation/concepts/indexing/#idf-modifier |
| To discuss Qdrant integration or get support, join Qdrant Discord: https://discord.gg/ArVgNHV6 | Qdrant Community |
| Star Qdrant n8n community node repo on GitHub: https://github.com/qdrant/n8n-nodes-qdrant | Community resource |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.