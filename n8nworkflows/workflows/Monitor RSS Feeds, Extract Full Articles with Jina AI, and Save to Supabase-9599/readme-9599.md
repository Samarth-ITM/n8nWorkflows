Monitor RSS Feeds, Extract Full Articles with Jina AI, and Save to Supabase

https://n8nworkflows.xyz/workflows/monitor-rss-feeds--extract-full-articles-with-jina-ai--and-save-to-supabase-9599


# Monitor RSS Feeds, Extract Full Articles with Jina AI, and Save to Supabase

### 1. Workflow Overview

This workflow is designed to monitor multiple RSS feeds from specified blogs, extract the full articles using Jina AI's HTTP extraction service, filter out older content based on a configurable maximum age, and save the processed blog data into a Supabase database. It is intended for content aggregators, researchers, or marketing professionals who want to automate the collection and storage of fresh blog content from multiple sources.

The workflow’s logic is grouped into the following blocks:

- **1.1 Scheduled Trigger & Configuration:** Defines when the workflow runs and sets parameters such as maximum content age and the target RSS feed URLs.

- **1.2 RSS Feed Processing:** Retrieves RSS feed items, splits them into individual blog entries, and formats publication dates.

- **1.3 Blog Filtering:** Filters out blogs older than the specified maximum age.

- **1.4 Full Content Extraction:** Uses Jina AI's HTTP extraction API to obtain the complete blog content beyond the RSS snippet.

- **1.5 Data Formatting & Storage:** Prepares the final data structure and saves it into Supabase, then waits before the next iteration.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration

- **Overview:**  
  This block schedules the workflow to run daily at noon and sets initial parameters for the maximum allowed blog age and the list of RSS feed URLs to monitor.

- **Nodes Involved:**  
  - Schedule Trigger  
  - max_content_age_days  
  - blogs to track  
  - Split Out

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow daily at 12:00 PM.  
    - Key Config: Interval set to trigger at hour 12 daily.  
    - Input: None  
    - Output: Triggers `max_content_age_days`.  
    - Failures: Possible schedule misconfiguration or disabled workflow.

  - **max_content_age_days**  
    - Type: Set  
    - Role: Defines how many days back blogs should be considered "fresh". Default is 60 days.  
    - Key Config: Sets an integer field `max_content_age_days` to 60.  
    - Input: From Schedule Trigger  
    - Output: To `Client ID + Max Content Age + Blogs` and `blogs to track`.  
    - Failure Modes: Incorrect data type or missing value can cause filtering issues.

  - **blogs to track**  
    - Type: Set  
    - Role: Stores an array of RSS feed URLs to monitor.  
    - Key Config: Hardcoded array with two RSS feeds: n8n blog and Zapier blog feeds.  
    - Input: From `max_content_age_days`  
    - Output: To `Split Out` node.  
    - Edge Cases: RSS feed URLs must be current; 403 errors indicate invalid URLs.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the array of RSS feed URLs into individual items for sequential processing.  
    - Key Config: Splits on `source_identifier` field.  
    - Input: From `blogs to track`  
    - Output: To `Split RSS Feeds`  
    - Failures: Empty array input will halt downstream processing.

---

#### 2.2 RSS Feed Processing

- **Overview:**  
  This block reads RSS feeds one by one, extracts individual blog items, and formats publication dates into a standardized format.

- **Nodes Involved:**  
  - Split RSS Feeds  
  - Rss feed link  
  - RSS → Items  
  - rss feed links + blogs  
  - Client ID + Max Content Age + Blogs  
  - All Data  
  - Find Date & Time of Blogs  
  - Merge3

- **Node Details:**

  - **Split RSS Feeds**  
    - Type: Split In Batches  
    - Role: Processes RSS feed URLs batch-wise for efficient handling.  
    - Input: From `rss feed links + blogs` (combined feed URLs and blog metadata)  
    - Output (two branches):  
      - To `Client ID + Max Content Age + Blogs` (merging data)  
      - To `Rss feed link` (for RSS reading)  
    - Failures: Batch size default; large feeds may cause timeouts.

  - **Rss feed link**  
    - Type: Set  
    - Role: Prepares feed URL field (`source_identifier`) for the RSS feed reader.  
    - Input: From `Split RSS Feeds`  
    - Output: To `RSS → Items`  
    - Failures: Missing or malformed URLs cause HTTP errors.

  - **RSS → Items**  
    - Type: RSS Feed Read  
    - Role: Reads items from the RSS feed URL provided.  
    - Config: Ignores SSL errors disabled, retries twice on failure.  
    - Input: From `Rss feed link`  
    - Output (two branches):  
      - To `rss feed links + blogs` (merge with blog metadata)  
      - Empty branch for errors or empty feeds  
    - Failures: 403 or 404 HTTP errors reflect invalid or inaccessible feeds.

  - **rss feed links + blogs**  
    - Type: Merge  
    - Role: Combines the RSS feed items with blog metadata.  
    - Input: From `RSS → Items` and `Rss feed link`  
    - Output: To `Split RSS Feeds` (forming batches)  
    - Failures: Merging empty inputs can cause data loss.

  - **Client ID + Max Content Age + Blogs**  
    - Type: Merge  
    - Role: Combines client config (max age) with blog data.  
    - Input: From `max_content_age_days` and `Split RSS Feeds`  
    - Output: To `All Data`  
    - Failures: Missing inputs may break downstream logic.

  - **All Data**  
    - Type: Set  
    - Role: Passes through all fields for downstream nodes.  
    - Input: From `Client ID + Max Content Age + Blogs`  
    - Output: To `Find Date & Time of Blogs`  
    - Failures: None significant.

  - **Find Date & Time of Blogs**  
    - Type: DateTime  
    - Role: Normalizes the blog publication date to `yyyy-MM-dd` format in `standardizedPubDate` field.  
    - Input: From `All Data`  
    - Output: To `Merge3`  
    - Failures: Missing or malformed date fields cause expression failures.

  - **Merge3**  
    - Type: Merge  
    - Role: Combines the date-formatted data with filtered blog data for further processing.  
    - Input: From `Find Date & Time of Blogs` and `Filter Out Old Blogs` (next block)  
    - Output: To `Filter Out Old Blogs` (first input)  
    - Failures: Sync issues if inputs are out of order.

---

#### 2.3 Blog Filtering

- **Overview:**  
  This block filters out blogs older than the configured maximum content age, ensuring only recent blogs proceed for full content extraction.

- **Nodes Involved:**  
  - Filter Out Old Blogs  
  - Loop Over Items

- **Node Details:**

  - **Filter Out Old Blogs**  
    - Type: If  
    - Role: Compares each blog’s publication date against the current date minus max content age (in days).  
    - Condition: Checks if `pubDate` is after or equal to `DateTime.now() - max_content_age_days`.  
    - Input: From `Merge3`  
    - Output:  
      - True path (new blogs): To `Loop Over Items`  
      - False path (old blogs): Discarded or processed differently (no output connected in this workflow).  
    - Failures: Date field missing or misformatted causes false negatives.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes the filtered new blog items one at a time with a delay to avoid rate limits.  
    - Input: From `Filter Out Old Blogs` (true path)  
    - Output:  
      - Main (true): To `Some data` for data preparation  
      - Secondary (false): Empty, no further processing  
    - Config: Wait between tries is 3000ms (3 seconds) for retry.  
    - Failures: Large batches could cause execution timeout.

---

#### 2.4 Full Content Extraction

- **Overview:**  
  This block prepares the data for extraction, calls Jina AI to extract the full blog content from the URL, and merges the extracted content with original metadata.

- **Nodes Involved:**  
  - Some data  
  - Extract the full blog  
  - Some data + Full Blog

- **Node Details:**

  - **Some data**  
    - Type: Set  
    - Role: Prepares a cleaned dataset including URL, max content age, publication date, title, and creator for extraction.  
    - Key Fields Set: `url` (from `link`), `max_content_age_days`, `pubDate`, `title`, `creator`.  
    - Input: From false path of `Loop Over Items` (processing individual blog data)  
    - Output: To `Extract the full blog`  
    - Failures: Missing URL or critical fields cause extraction failure.

  - **Extract the full blog**  
    - Type: HTTP Request  
    - Role: Calls Jina AI’s HTTP extraction service to retrieve the full blog article content by passing the blog URL.  
    - Config:  
      - URL is constructed by removing protocol from blog URL and prefixing with "https://r.jina.ai/http://".  
      - Response format set to text.  
      - Retries enabled with 5000ms delay between tries.  
    - Input: From `Some data`  
    - Output: To `Some data + Full Blog`  
    - Failures:  
      - HTTP errors (403, 404) if URL is invalid or blocked.  
      - Timeout or connectivity issues.  
      - Response parsing errors.

  - **Some data + Full Blog**  
    - Type: Merge  
    - Role: Combines original blog metadata with the extracted full blog content.  
    - Input: From `Some data` and `Extract the full blog`  
    - Output: To `final data`  
    - Failures: Missing extraction data results in incomplete records.

---

#### 2.5 Data Formatting & Storage

- **Overview:**  
  This block formats the combined data into the final schema and saves it into the Supabase database. After saving, the workflow waits for 2 minutes before allowing the next trigger.

- **Nodes Involved:**  
  - final data  
  - Save Blog Data to Database  
  - Chill for a sec

- **Node Details:**

  - **final data**  
    - Type: Set  
    - Role: Formats the blog data fields for storage, including URL, max content age, publication date, title, creator, and the extracted content snippet.  
    - Fields set explicitly.  
    - Input: From `Some data + Full Blog`  
    - Output: To `Save Blog Data to Database`  
    - Failures: Missing or malformed fields can cause failure in downstream DB insertion.

  - **Save Blog Data to Database**  
    - Type: Supabase  
    - Role: Inserts the formatted blog record into the Supabase table `content_queue_1`.  
    - Config:  
      - Fields mapped: content_type (fixed as "blog"), source_url, content_snippet, status ("new"), published_date, creator, title.  
      - Credentials: Uses configured Supabase API credentials.  
      - Retries on failure enabled.  
      - On error, continues regular output to avoid workflow stopping.  
    - Input: From `final data`  
    - Output: To `Chill for a sec`  
    - Failures:  
      - Auth errors with Supabase.  
      - Schema mismatches or permission issues.  
      - Network failures.

  - **Chill for a sec**  
    - Type: Wait  
    - Role: Pauses the workflow for 120 seconds after saving data before allowing re-triggering or next batch.  
    - Input: From `Save Blog Data to Database`  
    - Output: To `Loop Over Items` (restarting the loop if applicable)  
    - Failures: None significant.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                                       | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                                                       |
|-------------------------------|---------------------|-----------------------------------------------------|------------------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger    | Triggers workflow daily at 12:00 PM                  | None                                     | max_content_age_days                    | ## Find blogs from your chosen websites, filter out old ones, then extract the full blog page ... See Sticky Note for details.  |
| max_content_age_days          | Set                 | Sets max blog age (default 60 days)                   | Schedule Trigger                         | Client ID + Max Content Age + Blogs, blogs to track | ## Choose # of Days Filter out blogs older than **your number of days** old ...                                                  |
| blogs to track                | Set                 | Provides list of RSS feed URLs to track               | max_content_age_days                     | Split Out                              | ## Find RSS FEED URLS Add the rss feed urls of the blogs that you want to track ...                                              |
| Split Out                    | Split Out            | Splits RSS feed URLs array into individual items      | blogs to track                          | Split RSS Feeds                        | ## Splits all of the websites into their own items so that they go into the loop one at a time                                   |
| Split RSS Feeds              | Split In Batches     | Processes each RSS feed URL in batches                 | rss feed links + blogs                   | Client ID + Max Content Age + Blogs, Rss feed link | ## This loops over the websites one at a time to find all blogs.                                                                |
| Rss feed link                 | Set                 | Prepares feed URL for RSS reading                      | Split RSS Feeds                         | RSS → Items                           |                                                                                                                                  |
| RSS → Items                  | RSS Feed Read        | Reads items from RSS feed                               | Rss feed link                          | rss feed links + blogs                 |                                                                                                                                  |
| rss feed links + blogs        | Merge                | Combines RSS feed items with blog metadata             | RSS → Items, Rss feed link               | Split RSS Feeds                       |                                                                                                                                  |
| Client ID + Max Content Age + Blogs | Merge          | Combines client config and blog data                    | max_content_age_days, Split RSS Feeds   | All Data                             |                                                                                                                                  |
| All Data                     | Set                  | Passes all data fields through                          | Client ID + Max Content Age + Blogs      | Find Date & Time of Blogs             |                                                                                                                                  |
| Find Date & Time of Blogs     | DateTime             | Formats blog publication date                           | All Data                               | Merge3                              | ## Adds a 'published date' in an easy to read format                                                                             |
| Merge3                      | Merge                | Combines date-formatted data with filtered blogs         | Find Date & Time of Blogs, Filter Out Old Blogs | Filter Out Old Blogs               |                                                                                                                                  |
| Filter Out Old Blogs          | If                   | Filters blogs older than max_content_age_days           | Merge3                                | Loop Over Items                       | ## Filters out blogs that are older than the date you specified                                                                  |
| Loop Over Items               | Split In Batches     | Processes filtered blog items individually with delay  | Filter Out Old Blogs                    | Some data                            |                                                                                                                                  |
| Some data                    | Set                  | Prepares data for full content extraction               | Loop Over Items (false path)            | Extract the full blog                 | ## Gathers the necessary data to extract the FULL blog                                                                           |
| Extract the full blog         | HTTP Request         | Calls Jina AI API to extract full blog content          | Some data                             | Some data + Full Blog                 | ## Extract Full Blog Sometimes the rss feed only captures a snippet of the blogs...                                              |
| Some data + Full Blog         | Merge                | Merges metadata with extracted full blog content        | Some data, Extract the full blog        | final data                          |                                                                                                                                  |
| final data                   | Set                  | Formats final data for storage                           | Some data + Full Blog                   | Save Blog Data to Database           | ## Format the final data to be uploaded to your storage location                                                                 |
| Save Blog Data to Database    | Supabase             | Saves blog data into Supabase database                   | final data                           | Chill for a sec                    | ## Save your blog!!! Switch this node out for one that your prefer, if you don't like Supabase...                                |
| Chill for a sec              | Wait                 | Pauses after saving before next iteration                | Save Blog Data to Database              | Loop Over Items                      |                                                                                                                                  |
| Sticky Note                  | Sticky Note          | General workflow instructions and setup notes            | None                                 | None                                | ## Find blogs from your chosen websites, filter out old ones, then extract the full blog page... See content for details.        |
| Sticky Note1                 | Sticky Note          | Explains max_content_age_days purpose                     | None                                 | None                                | ## Choose # of Days Filter out blogs older than **your number of days** old                                                       |
| Sticky Note2                 | Sticky Note          | Advises on finding correct RSS feed URLs                  | None                                 | None                                | ## Find RSS FEED URLS Add the rss feed urls of the blogs that you want to track...                                                |
| Sticky Note3                 | Sticky Note          | Describes looping over websites one at a time             | None                                 | None                                | ## This loops over the websites one at a time to find all blogs.                                                                 |
| Sticky Note4                 | Sticky Note          | Describes adding a formatted published date                | None                                 | None                                | ## Adds a 'published date' in an easy to read format                                                                             |
| Sticky Note5                 | Sticky Note          | Explains filtering out old blogs                            | None                                 | None                                | ## Filters out blogs that are older than the date you specified                                                                   |
| Sticky Note6                 | Sticky Note          | Describes splitting websites for processing                | None                                 | None                                | ## Splits all of the websites into their own items so that they go into the loop one at a time                                    |
| Sticky Note7                 | Sticky Note          | Explains data gathering before full blog extraction        | None                                 | None                                | ## Gathers the necessary data to extract the FULL blog                                                                           |
| Sticky Note8                 | Sticky Note          | Explains how full blog extraction works                     | None                                 | None                                | ## Extract Full Blog Sometimes the rss feed only captures a snippet of the blogs. This is a completely free - zero setup method. |
| Sticky Note9                 | Sticky Note          | Explains final data formatting before storage               | None                                 | None                                | ## Format the final data to be uploaded to your storage location                                                                  |
| Sticky Note10                | Sticky Note          | Suggests alternatives for blog data storage                  | None                                 | None                                | ## Save your blog!!! Switch this node out for one that your prefer, if you don't like Supabase...                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger once daily at 12:00 PM.

2. **Add a Set node named `max_content_age_days`:**  
   - Input: Connect from Schedule Trigger.  
   - Parameters: Create a number field `max_content_age_days` with value 60 (default).  
   - Purpose: Define max age of blogs to keep.

3. **Add a Set node named `blogs to track`:**  
   - Input: Connect from `max_content_age_days`.  
   - Parameters: Create an array field `source_identifier` containing RSS feed URLs to monitor, e.g.  
     ```
     ['https://blog.n8n.io/rss', 'https://zapier.com/blog/feeds/latest/']
     ```  
   - Purpose: List of RSS feeds to monitor.

4. **Add a Split Out node named `Split Out`:**  
   - Input: Connect from `blogs to track`.  
   - Parameters: Split on field `source_identifier`.  
   - Purpose: Split the feed URLs array into individual items.

5. **Add a Merge node named `rss feed links + blogs`:**  
   - Mode: Combine All.  
   - Purpose: Will combine RSS feed items with metadata later.

6. **Add a Split In Batches node named `Split RSS Feeds`:**  
   - Input: Connect from `rss feed links + blogs`.  
   - Parameters: Default batch options.  
   - Purpose: Process feeds batch-wise.

7. **Add a Set node named `Rss feed link`:**  
   - Input: Connect from `Split RSS Feeds` (one branch).  
   - Parameters: Set single string field `source_identifier` with the feed URL.  
   - Purpose: Prepare feed URL for RSS reading.

8. **Add an RSS Feed Read node named `RSS → Items`:**  
   - Input: Connect from `Rss feed link`.  
   - Parameters:  
     - URL: `={{ $json.source_identifier }}`  
     - Ignore SSL errors: false  
     - Retry on fail: true, max tries 2  
   - Purpose: Read RSS feed items.

9. **Connect `RSS → Items` output to `rss feed links + blogs` input (one branch).**

10. **Add a Merge node named `Client ID + Max Content Age + Blogs`:**  
    - Mode: Combine All.  
    - Inputs: From `max_content_age_days` and from `Split RSS Feeds` (other branch).  
    - Purpose: Combine max age config with blogs data.

11. **Add a Set node named `All Data`:**  
    - Input: Connect from `Client ID + Max Content Age + Blogs`.  
    - Parameters: Pass all fields through.  
    - Purpose: Pass combined data downstream.

12. **Add a DateTime node named `Find Date & Time of Blogs`:**  
    - Input: Connect from `All Data`.  
    - Parameters:  
      - Date field: `={{ $json.pubDate || $json.isoDate }}`  
      - Format: `yyyy-MM-dd`  
      - Operation: Format Date  
      - Output field: `standardizedPubDate`  
    - Purpose: Normalize publication date.

13. **Add a Merge node named `Merge3`:**  
    - Mode: Combine By Position.  
    - Inputs: From `Find Date & Time of Blogs` and `Filter Out Old Blogs` (to be added).  
    - Purpose: Combine data for filtering.

14. **Add an If node named `Filter Out Old Blogs`:**  
    - Input: Connect from `Merge3`.  
    - Parameters:  
      - Condition: Check if `pubDate` is after or equal to current date minus `max_content_age_days` (expressed as:  
        ```  
        $json.pubDate >= DateTime.now().minus({days: $json.max_content_age_days}).toFormat('yyyy-MM-dd')  
        ```  
      )  
    - Output: True to `Loop Over Items`.

15. **Add a Split In Batches node named `Loop Over Items`:**  
    - Input: Connect from True output of `Filter Out Old Blogs`.  
    - Parameters: Default batch size, waitBetweenTries 3000 ms.  
    - Output: False path to `Some data`.

16. **Add a Set node named `Some data`:**  
    - Input: Connect from False output of `Loop Over Items`.  
    - Parameters: Set fields:  
      - `url` = `{{$json.link}}`  
      - `max_content_age_days`, `pubDate`, `title`, `creator` from input JSON.  
    - Purpose: Prepare data for extraction.

17. **Add an HTTP Request node named `Extract the full blog`:**  
    - Input: Connect from `Some data`.  
    - Parameters:  
      - URL Expression:  
        ```
        = "https://r.jina.ai/http://" + $json.url.replace('https://','').replace('http://','')
        ```  
      - Response Format: Text  
      - Retry on Fail: Enabled, wait 5000 ms between tries  
    - Purpose: Retrieve full blog content.

18. **Add a Merge node named `Some data + Full Blog`:**  
    - Inputs: From `Some data` and `Extract the full blog`.  
    - Mode: Combine By Position.  
    - Purpose: Merge original data with extracted content.

19. **Add a Set node named `final data`:**  
    - Input: Connect from `Some data + Full Blog`.  
    - Parameters: Set fields explicitly for storage:  
      - `url`, `max_content_age_days`, `pubDate`, `title`, `creator`, and `content_snippet` (extracted content).  
    - Purpose: Format final data for DB.

20. **Add a Supabase node named `Save Blog Data to Database`:**  
    - Input: Connect from `final data`.  
    - Parameters:  
      - Table: `content_queue_1`  
      - Fields mapping:  
        - content_type: "blog" (static)  
        - source_url: `{{$json.url}}`  
        - content_snippet: `{{$json.content_snippet}}`  
        - status: "new" (static)  
        - published_date: `{{$json.pubDate}}`  
        - creator: `{{$json.creator}}`  
        - title: `{{$json.title}}`  
      - Credentials: Configure with valid Supabase API credentials.  
      - Retry on fail: Enabled  
      - On error: Continue regular output.  
    - Purpose: Store blog data.

21. **Add a Wait node named `Chill for a sec`:**  
    - Input: Connect from `Save Blog Data to Database`.  
    - Parameters: Wait 120 seconds.  
    - Output: Connect back to `Loop Over Items` to continue processing if needed.  
    - Purpose: Rate limiting and pacing.

22. **Connect all nodes respecting the workflow flow as described above.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| The node called 'max_content_age_days' allows you to set how recent you want your blogs. Default is 60 days.                                                                                                                                                      | Sticky Note1                                            |
| Use an LLM or web search tool like Perplexity to find the most up-to-date RSS feed URLs for your target blogs to avoid 403 errors.                                                                                                                                 | Sticky Note2                                            |
| The Jina AI HTTP extraction service is a free, zero-setup method to extract full blog content beyond RSS snippets. It captures everything on the blog page, including title and full text.                                                                         | Sticky Note8                                            |
| You can replace the Supabase node with other storage options such as Google Sheets, Airtable, or n8n's Data Table node for flexibility.                                                                                                                             | Sticky Note10                                           |
| This workflow waits 2 minutes after saving each blog to avoid rate limits and allow smooth operation.                                                                                                                                                              | Chill for a sec node description                         |
| For advanced use, consider replacing the hardcoded 'blogs to track' node with a dynamic database or Google Sheet source to manage feed URLs.                                                                                                                      | Sticky Note (general)                                   |
| The workflow is designed to continue processing even if some nodes fail (e.g., RSS read or database save) to ensure robustness.                                                                                                                                    | Node error handling configurations                       |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. It strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.