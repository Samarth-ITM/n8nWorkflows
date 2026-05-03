Hybrid Search with Qdrant & n8n, Legal AI: Retrieval

https://n8nworkflows.xyz/workflows/hybrid-search-with-qdrant---n8n--legal-ai--retrieval-7946


# Hybrid Search with Qdrant & n8n, Legal AI: Retrieval

### 1. Workflow Overview

This workflow performs a hybrid search evaluation on a legal Q&A dataset using Qdrant and n8n automation. It targets legal AI practitioners and developers who want to assess the effectiveness of combined semantic and keyword retrieval over a structured dataset. The workflow is logically divided into two main blocks:

**1.1 Data Preparation and Filtering:**  
Starting from a manual trigger, it fetches dataset splits from Hugging Face, extracts the test split, retrieves test questions with answers, and filters them to keep only those suitable for evaluation.

**1.2 Hybrid Search and Evaluation:**  
For each filtered question, it performs a hybrid search query in Qdrant combining BM25 keyword and semantic embeddings with Reciprocal Rank Fusion (RRF), selects the top result, verifies if the retrieved chunk contains the correct answer, and aggregates hit rates to compute retrieval quality metrics.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Data Preparation and Filtering

**Overview:**  
This block prepares the evaluation dataset by fetching splits from Hugging Face, filtering to the test split, retrieving associated questions, and keeping only those with valid answers for fair evaluation.

**Nodes Involved:**  
- Index Dataset from HuggingFace (Manual Trigger)  
- Get Dataset Splits (HTTP Request)  
- Split Them All Out (Split Out)  
- Keep Test Split (Filter)  
- Get Test Queries (HTTP Request)  
- Divide Per Row (Split Out)  
- Keep Questions with Answers in the Dataset (Filter)  
- Keep Questions & IDs (Set)  

**Node Details:**

- **Index Dataset from HuggingFace**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually  
  - Configuration: No parameters; pinned input sets dataset to `"isaacus/LegalQAEval"`  
  - Inputs: None  
  - Outputs: Triggers "Get Dataset Splits"  

- **Get Dataset Splits**  
  - Type: HTTP Request  
  - Role: Fetch dataset splits metadata from Hugging Face Dataset Viewer API  
  - Configuration: Calls `https://datasets-server.huggingface.co/splits` with query param dataset = `={{ $json.dataset }}` (dynamic from trigger)  
  - Inputs: From "Index Dataset from HuggingFace"  
  - Outputs: JSON containing dataset splits, e.g. `train`, `test`, `validation`  

- **Split Them All Out**  
  - Type: Split Out  
  - Role: Splits array field `splits` into individual items for processing  
  - Configuration: Splits on field `splits` (the splits array from previous node)  
  - Inputs: From "Get Dataset Splits"  
  - Outputs: Each split item separately  

- **Keep Test Split**  
  - Type: Filter  
  - Role: Filters to keep only the split named `"test"`  
  - Configuration: Condition: `$json.split == 'test'` (strict, case-sensitive)  
  - Inputs: From "Split Them All Out"  
  - Outputs: Only the test split object  

- **Get Test Queries**  
  - Type: HTTP Request  
  - Role: Retrieves rows (questions) from the test split of the dataset  
  - Configuration: Calls `https://datasets-server.huggingface.co/rows` with query parameters:  
    - dataset = `={{ $json.dataset }}`  
    - config = `={{ $json.config }}` (from split)  
    - split = `={{ $json.split }}` (should be "test")  
    - length = 100 (fetches 100 rows)  
  - Inputs: From "Keep Test Split"  
  - Outputs: JSON rows with questions and answers  

- **Divide Per Row**  
  - Type: Split Out  
  - Role: Splits the `rows` array into individual questions for filtering  
  - Configuration: Split on field `rows`  
  - Inputs: From "Get Test Queries"  
  - Outputs: Individual question rows  

- **Keep Questions with Answers in the Dataset**  
  - Type: Filter  
  - Role: Only keep questions that have at least one answer (non-empty `answers` array)  
  - Configuration: Condition: `$json.row.answers.length > 0`  
  - Inputs: From "Divide Per Row"  
  - Outputs: Filtered question rows with answers  

- **Keep Questions & IDs**  
  - Type: Set  
  - Role: Reduces data to just the question text and `id` for evaluation  
  - Configuration: Sets two fields:  
    - `id_qa` = `={{ $json.row.id }}`  
    - `question` = `={{ $json.row.question }}`  
  - Inputs: From "Keep Questions with Answers in the Dataset"  
  - Outputs: Minimal objects with question and ID for search  

**Edge Cases & Failure Modes:**  
- HTTP request failures (timeouts, 4xx/5xx responses) when querying Hugging Face API  
- Empty or missing fields in dataset splits or rows  
- No test split found or test split empty  
- Questions without answers filtered out correctly  

---

#### Block 1.2: Hybrid Search and Evaluation

**Overview:**  
Processes each question individually by performing a hybrid search on Qdrant using BM25 and semantic embeddings, applying Reciprocal Rank Fusion (RRF), selecting the top result, checking if it contains the correct answer, and aggregating hits to compute a hits@1 metric.

**Nodes Involved:**  
- Loop Over Items (Split In Batches)  
- Query Points (Qdrant node)  
- Merge (Merge)  
- isHit = If we Found the Correct Answer (Set)  
- Aggregate Evals (Aggregate)  
- Percentage of isHits in Evals (Set)  

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each question individually in batches (default batch size 1) to perform queries sequentially  
  - Configuration: Default reset = false (does not reset state)  
  - Inputs: From "Keep Questions & IDs"  
  - Outputs: Each question item forwarded to “Query Points” and parallel to “Aggregate Evals”  

- **Query Points**  
  - Type: Qdrant  
  - Role: Performs a hybrid search query on Qdrant collection `legalQA_test` using Reciprocal Rank Fusion of BM25 and semantic search results  
  - Configuration:  
    - Operation: `queryPoints`  
    - Collection: `legalQA_test`  
    - Limit: 1 (top result only)  
    - Query: Fusion query type `"fusion": "rrf"`  
    - Prefetch: Two queries fused:  
      1. Semantic query with `"mxbai-embed-large-v1"` model embedding the question text  
      2. BM25 keyword-based query with `"qdrant/bm25"` model embedding the question text  
    - Using Qdrant Cloud Inference for semantic embeddings  
  - Credentials: Uses stored Qdrant API credentials  
  - Inputs: From "Loop Over Items" (questions)  
  - Outputs: Search results with candidate points  

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from "Query Points" and "isHit" evaluation to synchronise data for aggregation  
  - Configuration: Mode `combineAll` (combine all items from both inputs)  
  - Inputs: From "Query Points" and "isHit = If we Found the Correct Answer"  
  - Outputs: Merged data forwarded for aggregation  

- **isHit = If we Found the Correct Answer**  
  - Type: Set  
  - Role: Adds boolean flag `isHit` indicating if the top retrieved text chunk contains the correct answer  
  - Configuration: Sets `isHit` to boolean expression:  
    `{{ $json.result.points[0].payload.ids_qa.includes($json.id_qa) }}`  
    This checks if the question ID is included in the payload’s `ids_qa` list of the top point  
  - Inputs: From "Merge" (receives question and search result)  
  - Outputs: Adds `isHit` flag for evaluation  

- **Aggregate Evals**  
  - Type: Aggregate  
  - Role: Aggregates all evaluation items into a single collection for summary statistics  
  - Configuration: Aggregate all item data into field named `eval`  
  - Inputs: From "Loop Over Items" (after processing each item)  
  - Outputs: Aggregated evaluation data  

- **Percentage of isHits in Evals**  
  - Type: Set  
  - Role: Calculates the hits@1 metric: percentage of queries where top result contained correct answer  
  - Configuration: Sets field `"Hits percentage"` with expression:  
    `{{ ($json.eval.filter(item => item.isHit).length * 100) / $json.eval.length }}`  
  - Inputs: From "Aggregate Evals"  
  - Outputs: Final metric output  

**Edge Cases & Failure Modes:**  
- Qdrant API errors (authentication, network failures, invalid queries)  
- Empty or missing `result.points` causing undefined errors in expression  
- No matching points in Qdrant response  
- Batch processing errors or lost data on merge operations  
- Division by zero if no eval items  

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                                  | Input Node(s)                        | Output Node(s)                               | Sticky Note                                                                                                   |
|----------------------------------|-----------------------|-------------------------------------------------|------------------------------------|----------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Index Dataset from HuggingFace    | Manual Trigger        | Entry point, trigger the workflow                | -                                  | Get Dataset Splits                           |                                                                                                              |
| Get Dataset Splits                | HTTP Request          | Fetch dataset splits from Hugging Face API       | Index Dataset from HuggingFace      | Split Them All Out                           |                                                                                                              |
| Split Them All Out                | Split Out             | Split splits array to individual split items     | Get Dataset Splits                  | Keep Test Split                             |                                                                                                              |
| Keep Test Split                  | Filter                | Keep only the "test" split                        | Split Them All Out                  | Get Test Queries                            |                                                                                                              |
| Get Test Queries                 | HTTP Request          | Get test split questions (rows)                   | Keep Test Split                    | Divide Per Row                              |                                                                                                              |
| Divide Per Row                   | Split Out             | Split rows array into individual question rows   | Get Test Queries                   | Keep Questions with Answers in the Dataset |                                                                                                              |
| Keep Questions with Answers in the Dataset | Filter        | Keep only questions that have answers             | Divide Per Row                     | Keep Questions & IDs                        |                                                                                                              |
| Keep Questions & IDs             | Set                   | Extract minimal question and ID fields            | Keep Questions with Answers in the Dataset | Loop Over Items                         |                                                                                                              |
| Loop Over Items                 | Split In Batches       | Process each question individually in batches    | Keep Questions & IDs               | Query Points, Aggregate Evals               |                                                                                                              |
| Query Points                   | Qdrant                 | Perform hybrid semantic + BM25 search with RRF   | Loop Over Items                   | Merge                                       |                                                                                                              |
| Merge                         | Merge                  | Combine query results and evaluation flags       | Query Points, isHit = If we Found the Correct Answer | isHit = If we Found the Correct Answer   |                                                                                                              |
| isHit = If we Found the Correct Answer | Set             | Add boolean flag if top result contains correct answer | Merge                            | Loop Over Items                             |                                                                                                              |
| Aggregate Evals                | Aggregate              | Aggregate all evaluation items into one collection | Loop Over Items                   | Percentage of isHits in Evals               |                                                                                                              |
| Percentage of isHits in Evals  | Set                    | Calculate hits@1 metric as percentage             | Aggregate Evals                   | -                                           |                                                                                                              |
| Sticky Note1                   | Sticky Note            | Overview and detailed explanation of workflow    | -                                | -                                           | ## Evaluate Hybrid Search on Legal Dataset ... See full note content in Section 5                             |
| Sticky Note2                   | Sticky Note            | Explanation of dataset preparation steps          | -                                | -                                           | ## Get Questions to Eval Retrieval from Hugging Face Dataset ... See full note content in Section 5           |
| Sticky Note4                   | Sticky Note            | Explanation of hybrid search evaluation steps     | -                                | -                                           | ## Check Quality of Simple Hybrid Search on Legal Q&A Dataset ... See full note content in Section 5          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `Index Dataset from HuggingFace`  
   - Purpose: Start the workflow manually  
   - No parameters needed  
   - Assign fixed dataset input: `dataset = "isaacus/LegalQAEval"` (pin or set in initial data)  

2. **Create HTTP Request Node:**  
   - Name: `Get Dataset Splits`  
   - URL: `https://datasets-server.huggingface.co/splits`  
   - Method: GET  
   - Query Parameters: `dataset = {{$json.dataset}}` (use expression)  
   - Connect output of Manual Trigger to this node  

3. **Create Split Out Node:**  
   - Name: `Split Them All Out`  
   - Field to split: `splits`  
   - Connect from `Get Dataset Splits`  

4. **Create Filter Node:**  
   - Name: `Keep Test Split`  
   - Condition: `$json.split == 'test'` (strict string equals)  
   - Connect from `Split Them All Out`  

5. **Create HTTP Request Node:**  
   - Name: `Get Test Queries`  
   - URL: `https://datasets-server.huggingface.co/rows`  
   - Method: GET  
   - Query Parameters:  
     - `dataset = {{$json.dataset}}`  
     - `config = {{$json.config}}`  
     - `split = {{$json.split}}`  
     - `length = 100`  
   - Connect from `Keep Test Split`  

6. **Create Split Out Node:**  
   - Name: `Divide Per Row`  
   - Field to split: `rows`  
   - Connect from `Get Test Queries`  

7. **Create Filter Node:**  
   - Name: `Keep Questions with Answers in the Dataset`  
   - Condition: `$json.row.answers.length > 0` (number greater than 0)  
   - Connect from `Divide Per Row`  

8. **Create Set Node:**  
   - Name: `Keep Questions & IDs`  
   - Assign fields:  
     - `id_qa` = `{{$json.row.id}}`  
     - `question` = `{{$json.row.question}}`  
   - Connect from `Keep Questions with Answers in the Dataset`  

9. **Create Split In Batches Node:**  
   - Name: `Loop Over Items`  
   - Default batch size (1)  
   - Connect from `Keep Questions & IDs`  

10. **Create Qdrant Node:**  
    - Name: `Query Points`  
    - Operation: `queryPoints`  
    - Collection: `legalQA_test`  
    - Limit: 1  
    - Query JSON:  
      ```json
      {
        "fusion": "rrf"
      }
      ```  
    - Prefetch Array (JSON) with two queries fused via RRF:  
      - Semantic query:  
        ```json
        {
          "query": {
            "text": "{{ $json.question }}",
            "model": "mixedbread-ai/mxbai-embed-large-v1"
          },
          "using": "mxbai_large",
          "limit": 25
        }
        ```  
      - BM25 query:  
        ```json
        {
          "query": {
            "text": "{{ $json.question }}",
            "model": "qdrant/bm25"
          },
          "using": "bm25",
          "limit": 25
        }
        ```  
    - Credentials: Qdrant API credentials configured with valid API key and endpoint  
    - Connect from `Loop Over Items`  

11. **Create Merge Node:**  
    - Name: `Merge`  
    - Mode: `combineAll` to merge all incoming items  
    - Connect output of `Query Points` to first input  
    - Connect output of `isHit = If we Found the Correct Answer` node (step 13) to second input  

12. **Create Set Node:**  
    - Name: `isHit = If we Found the Correct Answer`  
    - Assign field:  
      - `isHit` (boolean) = `{{$json.result.points[0].payload.ids_qa.includes($json.id_qa)}}`  
    - Include fields: `id_qa`, `question` and all other fields  
    - Connect from `Merge` node  

13. **Connect `isHit = If we Found the Correct Answer` back to `Loop Over Items`** as per workflow connections for iterative processing.

14. **Create Aggregate Node:**  
    - Name: `Aggregate Evals`  
    - Aggregate: `aggregateAllItemData`  
    - Destination Field: `eval`  
    - Connect from `Loop Over Items` (second output or after last iteration)  

15. **Create Set Node:**  
    - Name: `Percentage of isHits in Evals`  
    - Assign field `"Hits percentage"` with expression:  
      `{{ ($json.eval.filter(item => item.isHit).length * 100) / $json.eval.length }}`  
    - Connect from `Aggregate Evals`  

16. **Add Sticky Notes:**  
    - Add informational sticky notes referencing:  
      - Hugging Face Dataset Viewer API docs  
      - Explanation of evaluation logic and hybrid search rationale  
      - Links to Qdrant docs on hybrid queries and embedding inference  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                                                                                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ## Evaluate Hybrid Search on Legal Dataset  This is the second part of "Hybrid Search with Qdrant & n8n, Legal AI." The first part, "Indexing," covers preparing and uploading the dataset to Qdrant. This pipeline demonstrates hybrid search using BM25 + semantic search fused by Reciprocal Rank Fusion (RRF) on a subset of questions. It calculates hits@1, the percentage of questions for which the top retrieved chunk contains the correct answer. Embedding inference uses Qdrant Cloud Inference by default, but external providers like OpenAI can be configured. Prerequisites: completed indexing pipeline, Qdrant collection created. For Qdrant-related questions join their Discord and star their GitHub repo. | https://qdrant.tech/documentation/concepts/collections/#collections  https://discord.gg/ArVgNHV6  https://github.com/qdrant/n8n-nodes-qdrant                                                                                                                  |
| ## Get Questions to Eval Retrieval from Hugging Face Dataset (Already Indexed to Qdrant) Fetches questions from LegalQAEval dataset using Dataset Viewer API. Steps include retrieving dataset splits, fetching a small subsample of test split questions (with pagination available), and filtering to only questions with paired answers. Dataset link: [LegalQAEval (isaacus)](https://huggingface.co/datasets/isaacus/LegalQAEval). Useful links: Dataset Viewer API docs, HTTP node pagination guide.                                                                                                                                                                                                                                                                              | https://huggingface.co/docs/dataset-viewer/quick_start  https://docs.n8n.io/code/cookbook/http-node/pagination/#enable-pagination  https://huggingface.co/datasets/isaacus/LegalQAEval                                                                        |
| ## Check Quality of Simple Hybrid Search on Legal Q&A Dataset For each evaluation question, performs hybrid search in Qdrant by fusing BM25 keyword search and mxbai-embed-large semantic search with RRF. Selects top-1 result and checks if it contains correct answer by matching question ID. Aggregates results to compute hits@1 metric. If results are good, reuse Qdrant query node to build legal RAG chatbot; otherwise, improve with reranking, score boosting etc. Links to hybrid queries docs. Encouragement to experiment.                                                                                                                                                                                                                                               | https://qdrant.tech/documentation/concepts/hybrid-queries/                                                                                                                                                                                                          |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.