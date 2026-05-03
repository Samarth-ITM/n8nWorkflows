From Sitemap Crawling to Vector Storage: Creating an Efficient Workflow for RAG

https://n8nworkflows.xyz/workflows/from-sitemap-crawling-to-vector-storage--creating-an-efficient-workflow-for-rag-8707


# From Sitemap Crawling to Vector Storage: Creating an Efficient Workflow for RAG

---

## 1. Workflow Overview

This n8n workflow automates the process of crawling a website’s sitemap, extracting and cleaning web page content, performing quality and type detection on the content, generating embeddings with OpenAI, and storing the processed data efficiently in a Supabase vector database for Retrieval Augmented Generation (RAG) applications. It also manages task status and retries via integration with Crawl4AI and Supabase tables.

The workflow is logically divided into the following functional blocks:

- **1.1 Initialization and Sitemap Crawling**: Triggering the workflow manually, fetching the sitemap XML, parsing URLs, and initiating batch processing.
- **1.2 URL Deduplication and Queue Management**: Normalizing URLs, checking if URLs exist in the Supabase scrape queue, and inserting new URLs for crawling.
- **1.3 Web Page Crawling and Scraping via Crawl4AI**: Sending URLs to Crawl4AI for scraping, waiting for task completion, and handling retries and errors.
- **1.4 Content Cleaning and Quality Filtering**: Removing unwanted formatting/noise from scraped content, filtering low-quality texts.
- **1.5 Content Type Detection and Metadata Extraction**: Classifying content type (code, tutorial, FAQ, etc.), extracting metadata such as title, domain, language, and quality score.
- **1.6 Text Splitting, Embedding Generation, and Vector Storage**: Splitting text into chunks, generating embeddings with OpenAI, loading documents, and inserting them into Supabase vector store.
- **1.7 Status Updates and Retry Logic**: Updating the task status in Supabase, managing retry counts, and controlling workflow looping for pending tasks.

---

## 2. Block-by-Block Analysis

### 2.1 Initialization and Sitemap Crawling

- **Overview:**  
  This block starts the workflow manually, fetches the sitemap XML from a specified URL, parses it, and splits out the URLs for batch processing.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - HTTP Request  
  - XML  
  - Split Out  
  - Loop Over Items1 (Split In Batches)  

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts workflow execution on user demand  
    - Input: None  
    - Output: Initiates HTTP Request node  

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Fetch sitemap XML from https://www.kiekens.com/sitemap.xml  
    - Configuration: GET method, no authentication, default options  
    - Input: Manual trigger output  
    - Output: Raw XML content for parsing  

  - **XML**  
    - Type: XML Parser  
    - Role: Parses XML response into JSON object  
    - Configuration: Default options  
    - Input: HTTP Request node output  
    - Output: JSON object containing sitemap structure  

  - **Split Out**  
    - Type: Split Out (Array extraction)  
    - Role: Extracts and splits the array of URLs under `urlset.url` for individual processing  
    - Configuration: Field to split out: `urlset.url`  
    - Input: XML node output  
    - Output: Individual URL JSON items  

  - **Loop Over Items1**  
    - Type: Split In Batches  
    - Role: Processes the extracted URLs in batches to manage load and rate limits  
    - Configuration: Default batch size (implied)  
    - Input: Split Out node output  
    - Output: Batch chunks of URLs for further processing  

- **Potential Failures:**  
  - HTTP request failure (network, 404, timeout)  
  - XML parsing errors if sitemap malformed  
  - Empty or missing URLs in sitemap  

---

### 2.2 URL Deduplication and Queue Management

- **Overview:**  
  This block normalizes URLs, checks if they already exist in the Supabase `scrape_queue` table, and inserts new URLs to be scraped.

- **Nodes Involved:**  
  - Format the URL  
  - Check if the URL is in the Supabase Table  
  - Format the Output from the Supabase node (Code)  
  - If "shouldInsert" is true (If)  
  - URL in a new row (Supabase insert)  
  - Loop Over Items1 (batch looping)  
  - Sticky Note1 (comment)  

- **Node Details:**

  - **Format the URL**  
    - Type: Set  
    - Role: Cleans and normalizes URL strings by trimming and converting to lowercase  
    - Assigns `loc` = trimmed, lowercase URL string  
    - Input: Loop Over Items1 output (individual URL JSON)  
    - Output: Normalized URL JSON  

  - **Check if the URL is in the Supabase Table**  
    - Type: Supabase  
    - Role: Query Supabase `scrape_queue` table for existing URL entries  
    - Configuration: Filter where `url` equals normalized `loc`  
    - Input: Format the URL output  
    - Output: Query result (existing row or empty)  
    - Error Handling: Continues on error, retries on failure with wait  

  - **Format the Output from the Supabase node**  
    - Type: Code  
    - Role: Determines whether URL should be inserted (if not found in table)  
    - Logic: Checks if Supabase query result is empty, sets `shouldInsert` boolean flag  
    - Input: Supabase query result  
    - Output: JSON with `url` and `shouldInsert`  

  - **If "shouldInsert" is true**  
    - Type: If  
    - Role: Branches workflow to insert new URL if it doesn’t exist  
    - Condition: `shouldInsert == true`  
    - Input: Format Output node  
    - Output: True branch triggers insert, false branch loops back  

  - **URL in a new row**  
    - Type: Supabase  
    - Role: Inserts new URL row into `scrape_queue` table with default status `pending`  
    - Input: If node true branch  
    - Output: Triggers Loop Over Items1 to continue processing batches  

  - **Loop Over Items1**  
    - Type: Split In Batches  
    - Role: Continues batching URLs for processing (looping)  

  - **Sticky Note1**  
    - Content: "## Put all Website`s URLs in Supabase Table - scrape_queue"  
    - Visual aid for context  

- **Potential Failures:**  
  - Supabase connection/auth errors  
  - Duplicate URL insertion errors (handled by unique constraint)  
  - Expression errors if URLs missing or malformed  

---

### 2.3 Web Page Crawling and Scraping via Crawl4AI

- **Overview:**  
  This block handles sending URLs to Crawl4AI for scraping, polls for task status, and handles errors or retries based on task results.

- **Nodes Involved:**  
  - Get a row - scrape_queue Table (Supabase get)  
  - If2 (Check task status)  
  - Crawl4ai Web Page Scrape (HTTP POST to Crawl4AI)  
  - Wait (Delay)  
  - Crawl4AI_Task Status (HTTP GET for status)  
  - If (Check if task completed)  
  - Update a row in scrape_queue Table (update status and task_id)  
  - Update a row in scrape_queue Table1 (update on error)  
  - Loop Over Items (batch processing)  
  - If1 (Check attempt count and task_id existence)  
  - Task_id Counter (Code for counting attempts)  
  - Update a row in scrape_queue Table2 (mark error after max attempts)  
  - Wait1 (Delay)  
  - Sticky Note3 ("## Crawl4AI URL Scraping")  
  - Sticky Note ("## Get the URL from Supabase and check if it is completed or not\n\n**Only the NOT completed URLs will be passed**")  

- **Node Details:**

  - **Get a row - scrape_queue Table**  
    - Type: Supabase  
    - Role: Fetches a specific URL row from `scrape_queue` to check its status  
    - Input: From Loop Over Items batch output  
    - Output: Row data including status and task_id  

  - **If2**  
    - Type: If  
    - Role: Checks if the task status is "pending" to decide if scraping should proceed  
    - Condition: status contains "pending"  
    - Input: Supabase row data  
    - Output: True branch triggers Crawl4ai Web Page Scrape node  

  - **Crawl4ai Web Page Scrape**  
    - Type: HTTP Request (POST)  
    - Role: Sends URL to Crawl4AI API for scraping with priority 10  
    - Authentication: HTTP Header Auth (credential configured)  
    - Input: URL JSON from If2 true branch  
    - Output: Task creation response including task_id  
    - Retry: Enabled on failure with 5s wait intervals  
    - Error Handling: Continues on error output  

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 30 seconds to allow scraping completion  
    - Input: Crawl4ai Web Page Scrape output  
    - Output: Triggers Crawl4AI_Task Status node  

  - **Crawl4AI_Task Status**  
    - Type: HTTP Request (GET)  
    - Role: Polls Crawl4AI API to check the scraping task status by task_id  
    - Authentication: HTTP Header Auth  
    - Retry: Enabled, wait 5s between tries  
    - Output: Returns task status (e.g., “completed”, “pending”, “error”)  

  - **If**  
    - Type: If  
    - Role: Checks if the task status is “completed” to continue processing content  
    - Condition: status == “completed”  
    - Output: True branch to content cleaning, False branch to update error status  

  - **Update a row in scrape_queue Table**  
    - Type: Supabase update  
    - Role: Updates task status and task_id in Supabase for URLs with completed tasks  
    - Input: Crawl4AI_Task Status output  
    - Output: Triggers next processing block  

  - **Update a row in scrape_queue Table1**  
    - Type: Supabase update  
    - Role: Updates status to error and task_id on failed or errored tasks  
    - Input: Crawl4AI_Task Status false branch  
    - Output: Loops over batch  

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes batch of URLs in scraping queue  

  - **If1**  
    - Type: If  
    - Role: Checks if task_id exists and if attempt_count >= 10 to stop retries  
    - Condition: task_id exists AND attempt_count >= 10  
    - Output: True branch updates status to error, False branch waits then retries  

  - **Task_id Counter**  
    - Type: Code  
    - Role: Maintains a counter of attempts per task_id for retry logic  
    - Uses global variables to reset count for new task_ids  

  - **Update a row in scrape_queue Table2**  
    - Type: Supabase update  
    - Role: Marks URL status as error after max attempts exceeded  

  - **Wait1**  
    - Type: Wait  
    - Role: Waits some unspecified minutes before retrying Crawl4AI Web Page Scrape  

- **Potential Failures:**  
  - Crawl4AI API timeout or failure  
  - Authentication errors with HTTP Header Auth  
  - Supabase update failures or conflicts  
  - Infinite loops if retry logic malfunctions  

---

### 2.4 Content Cleaning and Quality Filtering

- **Overview:**  
  This block cleans raw HTML/markdown content from scraping results, removes noise and formatting, and filters out low-quality or insufficient content.

- **Nodes Involved:**  
  - Remove redundant data from the scraping (Code)  
  - Quality Filter Node (Code)  
  - Sticky Note4 ("## Clean te HTML code")  

- **Node Details:**

  - **Remove redundant data from the scraping**  
    - Type: Code  
    - Role: Applies extensive regex and string replacements to clean markdown, HTML remnants, social media noise, redundant punctuation, and short lines  
    - Also extracts metadata such as word count, meaningful content boolean, extracted title, domain, and quality score  
    - Input: Crawl4AI_Task Status completed output with raw content  
    - Output: Cleaned text and metadata for filtering  

  - **Quality Filter Node**  
    - Type: Code  
    - Role: Filters out items with qualityScore < 0.5, wordCount < 50, or cleanedText length < 200  
    - Input: Cleaned data from previous node  
    - Output: Only high-quality content proceeds  

- **Potential Failures:**  
  - Regex errors if unexpected input format  
  - Empty or null content causing filtering out all data  

---

### 2.5 Content Type Detection and Metadata Extraction

- **Overview:**  
  This block detects the type of content (e.g., code, tutorial, FAQ), extracts enhanced metadata for each document, including titles, domains, language, and quality scores.

- **Nodes Involved:**  
  - Content Type Detection (Code)  
  - Better Metadata Extraction (Code)  

- **Node Details:**

  - **Content Type Detection**  
    - Type: Code  
    - Role: Uses keyword heuristics to classify content into types such as code, tutorial, FAQ, documentation, blog, product, or article  
    - Input: Filtered content from Quality Filter Node  
    - Output: Adds `contentType` field to JSON  

  - **Better Metadata Extraction**  
    - Type: Code  
    - Role: Extracts title from content (first meaningful line), domain from URL, word count, quality score, and performs a simple English/Dutch language detection  
    - Constructs a detailed metadata object for each document including page URL, title, domain, contentType, wordCount, scrape date, language, qualityScore, and content length  
    - Input: Content Type Detection output  
    - Output: Enhanced JSON including `metadata` object and individual metadata fields  

- **Potential Failures:**  
  - Errors in parsing or string operations on missing fields  
  - Inaccurate language detection due to simplistic heuristics  

---

### 2.6 Text Splitting, Embedding Generation, and Vector Storage

- **Overview:**  
  This block splits cleaned text into chunks, creates OpenAI embeddings, and stores the documents along with embeddings into a Supabase vector store for efficient retrieval.

- **Nodes Involved:**  
  - Character Text Splitter  
  - Default Data Loader  
  - Embeddings OpenAI  
  - Supabase Vector Store_documents  
  - Update a row in scrape_queue Table  

- **Node Details:**

  - **Character Text Splitter**  
    - Type: LangChain Text Splitter Character Text Splitter  
    - Role: Splits text into chunks of max 5000 characters for embedding  
    - Input: Better Metadata Extraction output  
    - Output: Text chunks for embedding  

  - **Default Data Loader**  
    - Type: LangChain Document Default Data Loader  
    - Role: Loads documents for vector store insertion, attaching metadata with page URL  
    - Input: Character Text Splitter output  
    - Output: Documents with metadata for embedding  

  - **Embeddings OpenAI**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates vector embeddings using OpenAI’s "text-embedding-ada-002" model  
    - Credentials: OpenAI API key configured  
    - Input: Default Data Loader output  
    - Output: Embeddings for documents  

  - **Supabase Vector Store_documents**  
    - Type: LangChain Vector Store Supabase  
    - Role: Inserts vectors and documents into Supabase `documents` table  
    - Table schema includes id, content, metadata (JSONB), and embedding vector(1536)  
    - Credentials: Supabase API configured  
    - Input: Embeddings OpenAI output  
    - Output: Insert confirmation  

  - **Update a row in scrape_queue Table**  
    - Type: Supabase update  
    - Role: Marks URL status as "completed" and updates task_id after successful vector insertion  
    - Input: Supabase Vector Store_documents output  

- **Potential Failures:**  
  - OpenAI API quota or connectivity issues  
  - Supabase insert conflicts or vector dimension mismatch  
  - Text chunking too large or too small affecting embeddings  

---

### 2.7 Status Updates and Retry Logic

- **Overview:**  
  This block manages status updates in the Supabase `scrape_queue` table, counting task attempts and enforcing retry limits to avoid infinite loops.

- **Nodes Involved:**  
  - Task_id Counter (Code)  
  - If1 (Check attempt count and task_id)  
  - Update a row in scrape_queue Table2 (mark error after max attempts)  
  - Update a row in scrape_queue Table1 (update on error)  
  - Wait1 (Wait for retry)  

- **Node Details:**

  - **Task_id Counter**  
    - Type: Code  
    - Role: Maintains a global counter of attempts per unique `task_id` to track retries  
    - Resets counter for a new task_id, increments for repeated task_id  
    - Input: Edit Fields output  
    - Output: Adds `attempt_count` to JSON  

  - **If1**  
    - Type: If  
    - Role: Checks if attempt_count >= 10 and task_id exists to stop retrying  
    - True branch updates status to "error"  
    - False branch waits then retries crawling  

  - **Update a row in scrape_queue Table2**  
    - Type: Supabase update  
    - Role: Marks row status as "error" after too many failed attempts  
    - Input: If1 true branch  

  - **Update a row in scrape_queue Table1**  
    - Type: Supabase update  
    - Role: Updates status to error on task failure detected in Crawl4AI_Task Status node  

  - **Wait1**  
    - Type: Wait  
    - Role: Delays workflow before retrying Crawl4ai Web Page Scrape  

- **Potential Failures:**  
  - Global variable misuse causing incorrect counting  
  - Supabase update failures causing retry logic stuck  
  - Infinite retry loops if status not updated  

---

## 3. Summary Table

| Node Name                          | Node Type                            | Functional Role                                | Input Node(s)                          | Output Node(s)                             | Sticky Note                                                                              |
|-----------------------------------|------------------------------------|------------------------------------------------|--------------------------------------|--------------------------------------------|------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger                     | Workflow start trigger                          | None                                 | HTTP Request                               |                                                                                          |
| HTTP Request                      | HTTP Request                       | Fetch sitemap XML                              | When clicking ‘Test workflow’         | XML                                        |                                                                                          |
| XML                              | XML Parser                        | Parse sitemap XML                              | HTTP Request                         | Split Out                                  |                                                                                          |
| Split Out                        | Split Out                        | Extract URLs array                             | XML                                  | Loop Over Items1                            |                                                                                          |
| Loop Over Items1                  | Split In Batches                  | Batch processing of URLs                        | Split Out                            | Format the URL                              | ## Put all Website`s URLs in Supabase Table - scrape_queue                               |
| Format the URL                   | Set                              | Normalize URL strings                           | Loop Over Items1                     | Check if the URL is in Supabase Table      |                                                                                          |
| Check if the URL is in the Supabase Table | Supabase                         | Check URL existence in Supabase                 | Format the URL                      | Format the Output from the Supabase node   |                                                                                          |
| Format the Output from the Supabase node | Code                             | Decide if URL should be inserted                | Check if the URL is in Supabase Table | If "shouldInsert" is true                  |                                                                                          |
| If "shouldInsert" is true        | If                               | Branch for new URLs                             | Format Output node                  | URL in a new row, Loop Over Items1          |                                                                                          |
| URL in a new row                 | Supabase                         | Insert new URL in scrape_queue                  | If "shouldInsert" is true           | Loop Over Items1                            |                                                                                          |
| Get a row - scrape_queue Table    | Supabase                         | Retrieve URL row for status check                | Loop Over Items                     | If2                                         | ## Get the URL from Supabase and check if it is completed or not - Only NOT completed URLs will be passed |
| If2                             | If                               | Check if URL status is pending                   | Get a row - scrape_queue Table      | Crawl4ai Web Page Scrape, Loop Over Items   | ## Crawl4AI URL Scraping                                                                |
| Crawl4ai Web Page Scrape          | HTTP Request                     | Send URL to Crawl4AI for scraping                | If2                               | Wait, Wait1                                 |                                                                                          |
| Wait                            | Wait                             | Delay for Crawl4AI task completion               | Crawl4ai Web Page Scrape            | Crawl4AI_Task Status                        |                                                                                          |
| Wait1                           | Wait                             | Delay before retrying Crawl4AI scraping          | Crawl4ai Web Page Scrape            | Crawl4ai Web Page Scrape                    |                                                                                          |
| Crawl4AI_Task Status             | HTTP Request                     | Poll status of Crawl4AI task                      | Wait                              | If, Update a row in scrape_queue Table1, Update a row in scrape_queue Table |                                                                                          |
| If                              | If                               | Check if Crawl4AI task is completed               | Crawl4AI_Task Status               | Remove redundant data from the scraping, Edit Fields |                                                                                          |
| Remove redundant data from the scraping | Code                             | Clean scraped content                             | If                               | Quality Filter Node                         | ## Clean te HTML code                                                                    |
| Edit Fields                     | Set                              | Add task_id field from Crawl4ai Web Page Scrape  | If                               | Task_id Counter                            |                                                                                          |
| Task_id Counter                 | Code                             | Count retry attempts per task_id                  | Edit Fields                      | If1                                        |                                                                                          |
| If1                             | If                               | Check if max retry attempts reached                | Task_id Counter                  | Update a row in scrape_queue Table2, Wait |                                                                                          |
| Update a row in scrape_queue Table2 | Supabase                         | Mark URL as error after max retries                | If1 true branch                 | Loop Over Items                             |                                                                                          |
| Update a row in scrape_queue Table1 | Supabase                         | Update URL status to error on Crawl4AI failure     | Crawl4AI_Task Status false branch | Loop Over Items                             |                                                                                          |
| Update a row in scrape_queue Table | Supabase                         | Update URL status and task_id on task completion   | Supabase Vector Store_documents    | Loop Over Items                             |                                                                                          |
| Quality Filter Node             | Code                             | Filter out low-quality or short content            | Remove redundant data from scraping | Content Type Detection                      |                                                                                          |
| Content Type Detection          | Code                             | Detect content type by keyword heuristics           | Quality Filter Node              | Better Metadata Extraction                   |                                                                                          |
| Better Metadata Extraction      | Code                             | Extract enhanced metadata and language detection     | Content Type Detection           | Character Text Splitter                      |                                                                                          |
| Character Text Splitter         | LangChain Text Splitter          | Split text into manageable chunks                    | Better Metadata Extraction       | Default Data Loader                          |                                                                                          |
| Default Data Loader             | LangChain Document Loader        | Prepare documents with metadata for embeddings       | Character Text Splitter          | Embeddings OpenAI                            |                                                                                          |
| Embeddings OpenAI              | LangChain Embeddings OpenAI      | Generate embeddings using OpenAI’s ada-002 model     | Default Data Loader             | Supabase Vector Store_documents              |                                                                                          |
| Supabase Vector Store_documents | LangChain Vector Store Supabase | Insert documents and embeddings into Supabase vector store | Embeddings OpenAI              | Update a row in scrape_queue Table           |                                                                                          |
| CREATE TABLE scrape_queue in Supabase | Postgres                        | (Executed manually once) Creates scrape_queue table | None                           | None                                       | ## Execute Once                                                                         |
| CREATE TABLE scrape_queue in Supabase1 | Postgres                        | (Executed manually once) Creates documents table     | None                           | None                                       | ## Execute Once                                                                         |
| Sticky Note1                   | Sticky Note                     | Comment "Put all Website`s URLs in Supabase Table - scrape_queue" | None                           | None                                       | ## Put all Website`s URLs in Supabase Table - scrape_queue                               |
| Sticky Note2                   | Sticky Note                     | Comment "Execute Once"                               | None                           | None                                       | ## Execute Once                                                                         |
| Sticky Note3                   | Sticky Note                     | Comment "Crawl4AI URL Scraping"                      | None                           | None                                       | ## Crawl4AI URL Scraping                                                                |
| Sticky Note4                   | Sticky Note                     | Comment "Clean te HTML code"                          | None                           | None                                       | ## Clean te HTML code                                                                    |
| Sticky Note                    | Sticky Note                     | Comment "Get the URL from Supabase and check if it is completed or not\n\n**Only the NOT completed URLs will be passed**" | None                           | None                                       | ## Get the URL from Supabase and check if it is completed or not - Only NOT completed URLs will be passed |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Supabase Tables (Execute Once):**

   - Create `scrape_queue` table with columns:  
     - `id` UUID primary key (default generated)  
     - `url` text unique  
     - `status` text with default 'pending'  
     - `task_id` text nullable  
     - `result` text nullable  
     - `created_at` and `updated_at` timestamps with triggers to auto-update `updated_at`  
     Use the provided SQL in "CREATE TABLE scrape_queue in Supabase" node.

   - Create `documents` table with columns:  
     - `id` SERIAL primary key  
     - `content` TEXT  
     - `metadata` JSONB  
     - `embedding` VECTOR(1536) for OpenAI embeddings  
     Use the SQL in "CREATE TABLE scrape_queue in Supabase1" node.

2. **Configure Credentials:**

   - Set up OpenAI API key credentials for the Embeddings OpenAI node.  
   - Configure Supabase API credentials for Supabase nodes.  
   - Configure HTTP Header Auth credential for Crawl4AI API.

3. **Build Workflow Nodes:**

   - **Manual Trigger:** Add a Manual Trigger node named "When clicking ‘Test workflow’" with default settings.

   - **HTTP Request (Fetch Sitemap):** Add HTTP Request node to GET `https://www.kiekens.com/sitemap.xml`.

   - **XML Parser:** Add XML node to parse the HTTP Request output.

   - **Split Out URLs:** Add Split Out node to extract `urlset.url` array from parsed XML.

   - **Batch Processing URLs:** Add Split In Batches node (Loop Over Items1) to iterate over URLs.

   - **Normalize URLs:** Add Set node (Format the URL) to trim and lowercase URLs.

   - **Check URL Existence:** Add Supabase node to query `scrape_queue` filtering by normalized URL.

   - **Decide Insert:** Add Code node to check if query result is empty and set `shouldInsert`.

   - **Conditional Insert:** Add If node to test `shouldInsert == true`.

   - **Insert URL:** Add Supabase insert node to add URL to `scrape_queue` table with default status.

   - **Batch Loop:** Connect insert back to batch loop to continue processing URLs.

4. **Scrape Management:**

   - Add Supabase node to get a row from `scrape_queue` for current URL.

   - Add If node (If2) to check if status contains "pending".

   - Add HTTP Request node (Crawl4ai Web Page Scrape) to POST URL to Crawl4AI with priority.

   - Add Wait node (30 sec) to delay for scraping.

   - Add HTTP Request node (Crawl4AI_Task Status) to GET task status by task_id.

   - Add If node to check if status == "completed".

   - Add Supabase update node to update `scrape_queue` row with completed status and task_id.

   - Add Supabase update node to update row with error status if task fails.

   - Add Split In Batches node (Loop Over Items) to handle batch processing.

5. **Retry Logic:**

   - Add Code node (Task_id Counter) to count attempts per task_id globally.

   - Add If node (If1) to check if attempts >= 10.

   - Add Supabase update node to mark error status after max attempts.

   - Add Wait node (Wait1) for delay before retrying Crawl4AI scraping.

6. **Content Cleaning and Filtering:**

   - Add Code node (Remove redundant data from the scraping) to clean markdown/HTML noise and extract metadata.

   - Add Code node (Quality Filter Node) to filter out low-quality content.

7. **Content Classification and Metadata:**

   - Add Code node (Content Type Detection) to classify content type heuristically.

   - Add Code node (Better Metadata Extraction) to extract title, domain, language, quality metrics.

8. **Embedding and Vector Storage:**

   - Add LangChain Character Text Splitter node with chunk size 5000.

   - Add LangChain Default Data Loader node using cleaned text and metadata.

   - Add LangChain Embeddings OpenAI node configured to model `text-embedding-ada-002` with OpenAI credentials.

   - Add LangChain Supabase Vector Store node to insert documents and embeddings into `documents` table.

   - Add Supabase update node to update `scrape_queue` status to "completed" and update task_id.

9. **Workflow Connections:**

   - Connect nodes in the order described in the block-by-block analysis, ensuring batch loops for URLs and retry loops for pending tasks.

10. **Sticky Notes:**

    - Add sticky notes as comments for clarity at relevant points, e.g., for Supabase table purpose, Crawl4AI scraping, cleaning steps, and execution instructions.

---

## 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                        |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| The workflow requires initial manual execution of table creation SQL nodes only once per Supabase instance.     | Sticky Note2 and SQL nodes                                             |
| Crawl4AI API requires HTTP Header Auth credential setup with valid API key.                                    | Crawl4ai Web Page Scrape and Crawl4AI_Task Status nodes               |
| OpenAI API key must be securely stored and configured for embedding generation.                                | Embeddings OpenAI node credential configuration                       |
| Supabase vector storage uses a vector dimension of 1536 matching OpenAI ada-002 embedding size.                | CREATE TABLE documents SQL and Supabase Vector Store_documents node   |
| Extensive cleaning logic applies regex to remove common web noise, markdown, code blocks, and social media text. | Remove redundant data from the scraping node                           |
| Content type detection uses heuristics and keyword matching; may require tuning for specific use cases.        | Content Type Detection node                                           |
| Retry logic is capped at 10 attempts per task_id to prevent infinite loops; adjust as needed.                   | Task_id Counter and If1 nodes                                          |
| Links and documentation for Crawl4AI API and n8n LangChain nodes should be referenced for advanced configuration. | N8N official docs and Crawl4AI API documentation                      |

---

**Disclaimer:**  
This document is derived exclusively from an n8n automated workflow JSON export. It complies with all current content policies and contains no illegal or protected content. All data processed is legal and public.

---