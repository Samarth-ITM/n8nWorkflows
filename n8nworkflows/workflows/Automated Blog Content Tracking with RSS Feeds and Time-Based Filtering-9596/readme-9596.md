Automated Blog Content Tracking with RSS Feeds and Time-Based Filtering

https://n8nworkflows.xyz/workflows/automated-blog-content-tracking-with-rss-feeds-and-time-based-filtering-9596


# Automated Blog Content Tracking with RSS Feeds and Time-Based Filtering

### 1. Workflow Overview

This workflow automates the process of tracking and filtering blog content from multiple RSS feeds based on the publication date. It is designed to fetch blog posts from specified RSS feed URLs, standardize their publication dates, and filter out entries older than a configurable number of days (default 60 days). The workflow can be used by content curators, marketers, or developers who want to automate blog monitoring without API keys or paid services.

The workflow is divided into the following logical blocks:

- **1.1 Input Setup and Initialization**: Defines feed URLs and content age limits, triggered manually.
- **1.2 RSS Feed Splitting and Reading**: Splits the list of RSS feed URLs and reads blog items from each feed iteratively.
- **1.3 Data Enrichment and Merging**: Combines metadata and standardizes publication dates for all blog items.
- **1.4 Filtering Based on Content Age**: Filters out blog posts older than the configured maximum content age.
- **1.5 Output Handling (Placeholder)**: The workflow recommends that users add nodes here to process or store filtered blog items.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup and Initialization

- **Overview:**  
  This block sets the maximum blog age threshold and defines the list of RSS feed URLs to track. It is initiated through a manual trigger.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - max_content_age_days  
  - blogs to track  
  - Split Out

- **Node Details:**

1. **When clicking ‘Execute workflow’**  
   - *Type:* Manual Trigger  
   - *Role:* Starts the workflow manually on demand.  
   - *Config:* No parameters configured.  
   - *Connections:* Outputs to `max_content_age_days`.  
   - *Failure Types:* None typical; manual.

2. **max_content_age_days**  
   - *Type:* Set  
   - *Role:* Sets a numeric variable `max_content_age_days` to 60 by default, representing the max age in days for blog posts to keep.  
   - *Config:* Assigns `max_content_age_days = 60`. Includes other input fields unchanged.  
   - *Input/Output:* Receives from manual trigger, outputs to both `Client ID + Max Content Age + Blogs` and `blogs to track`.  
   - *Edge Cases:* User can change this number to filter differently. If set improperly (non-numeric), filtering may fail.

3. **blogs to track**  
   - *Type:* Set  
   - *Role:* Defines the array `source_identifier` containing RSS feed URLs to track.  
   - *Config:* Sets `source_identifier` to an array with two URLs:  
     - https://blog.n8n.io/rss  
     - https://zapier.com/blog/feeds/latest/  
   - *Input/Output:* Receives from `max_content_age_days`, outputs to `Split Out`.  
   - *Edge Cases:* Invalid or 403 Forbidden URLs will cause fetch failures downstream.

4. **Split Out**  
   - *Type:* Split Out  
   - *Role:* Splits the array field `source_identifier` into individual items, one per RSS feed URL, to process feeds one-by-one.  
   - *Config:* Field to split out: `source_identifier`.  
   - *Input/Output:* Receives from `blogs to track`, outputs to `Split RSS Feeds`.  
   - *Edge Cases:* Empty array results in no iterations.

---

#### 2.2 RSS Feed Splitting and Reading

- **Overview:**  
  This block iterates over each RSS feed URL, reads the feed items, and prepares them for merging.

- **Nodes Involved:**  
  - Split RSS Feeds  
  - Rss feed link  
  - RSS → Items  
  - rss feed links + blogs

- **Node Details:**

1. **Split RSS Feeds**  
   - *Type:* Split In Batches  
   - *Role:* Processes feed URLs in batches (default batch size 1) to avoid overload.  
   - *Config:* No special options; defaults used.  
   - *Input/Output:* Receives from `Split Out`, outputs two paths: one to `Client ID + Max Content Age + Blogs` (index 1) and one to `Rss feed link` (index 0).  
   - *Edge Cases:* Batch size not explicitly set; potential overload if large feed list.

2. **Rss feed link**  
   - *Type:* Set  
   - *Role:* Passes along the current feed URL as `source_identifier` for the RSS feed reader.  
   - *Config:* Sets `source_identifier` equal to current item's `source_identifier`.  
   - *Input/Output:* Receives from `Split RSS Feeds`, outputs to `RSS → Items` and `rss feed links + blogs` (second output).  
   - *Edge Cases:* None significant; simple pass-through.

3. **RSS → Items**  
   - *Type:* RSS Feed Read  
   - *Role:* Reads the RSS feed for the URL in `source_identifier`, returning blog items.  
   - *Config:*  
     - URL set dynamically from `{{$json.source_identifier}}`.  
     - SSL verification enabled.  
     - Custom fields empty.  
     - Retries twice on failure, continues output on error.  
   - *Input/Output:* Receives from `Rss feed link`, outputs to `rss feed links + blogs` (first output).  
   - *Edge Cases:* 403 or 404 errors if URL invalid; timeout possible; malformed RSS feeds may cause parsing errors.

4. **rss feed links + blogs**  
   - *Type:* Merge  
   - *Role:* Combines outputs from RSS items and the feed URL context into a single data stream.  
   - *Config:* Combine mode, combining all input data.  
   - *Input/Output:* Receives two inputs: from `RSS → Items` (main feed items) and from `Rss feed link` (feed URL context). Outputs to `Split RSS Feeds`.  
   - *Edge Cases:* Merge failure if inputs mismatch in length or structure.

---

#### 2.3 Data Enrichment and Merging

- **Overview:**  
  This block merges the blog data with client-specific data and standardizes blog post dates into a uniform format.

- **Nodes Involved:**  
  - Client ID + Max Content Age + Blogs  
  - Find Date & Time of Blogs  
  - Merge3

- **Node Details:**

1. **Client ID + Max Content Age + Blogs**  
   - *Type:* Merge  
   - *Role:* Combines the current blog items with the `max_content_age_days` and blog list metadata.  
   - *Config:* Mode combine all inputs.  
   - *Input/Output:* Receives from `Split RSS Feeds` (index 1) and `max_content_age_days`/`blogs to track`. Outputs to `Find Date & Time of Blogs` and `Merge3`.  
   - *Edge Cases:* Mismatched inputs could cause data loss.

2. **Find Date & Time of Blogs**  
   - *Type:* Date Time  
   - *Role:* Extracts and formats the publication date of each blog post into `standardizedPubDate` (format yyyy-MM-dd).  
   - *Config:*  
     - Date field uses either `pubDate` or `isoDate` from the blog item JSON.  
     - Operation: formatDate.  
     - Output field name: `standardizedPubDate`.  
   - *Input/Output:* Receives from `Client ID + Max Content Age + Blogs`, outputs to `Merge3`.  
   - *Edge Cases:* Missing or malformed date fields cause empty or invalid output; date parsing errors.

3. **Merge3**  
   - *Type:* Merge  
   - *Role:* Combines the blog data enriched with standardized publication dates into a single data stream for filtering.  
   - *Config:* Combine by position (pairwise).  
   - *Input/Output:* Receives from `Find Date & Time of Blogs` and `Client ID + Max Content Age + Blogs`. Outputs to `Filter Out Old Blogs`.  
   - *Edge Cases:* Misaligned data inputs could cause incorrect merges.

---

#### 2.4 Filtering Based on Content Age

- **Overview:**  
  This block filters blog posts to keep only those newer than the specified max age (default 60 days).

- **Nodes Involved:**  
  - Filter Out Old Blogs

- **Node Details:**

1. **Filter Out Old Blogs**  
   - *Type:* If  
   - *Role:* Filters blog posts where `pubDate` is after or equal to the date computed as today minus `max_content_age_days`.  
   - *Config:*  
     - Condition: `pubDate >= (today - max_content_age_days)`  
     - Uses expression with `DateTime.now().minus({ days: $json.max_content_age_days }).toFormat('yyyy-MM-dd')`.  
     - True path: new/recent blogs.  
     - False path: old blogs.  
   - *Input/Output:* Receives from `Merge3`, outputs true to `Merge3` (for further processing), false path unused in this workflow.  
   - *Edge Cases:* Incorrect or missing `pubDate` causes filtering errors; timezone differences may impact filtering accuracy.

---

#### 2.5 Output Handling (User Extension Suggested)

- **Overview:**  
  The workflow currently ends after filtering; the author recommends users add nodes to process or store filtered blog posts (e.g., batch processing into Google Sheets).

- **Nodes Involved:**  
  - None explicitly; recommended to add downstream of `Filter Out Old Blogs` true output.

- **Node Details:**  
  - No nodes exist here in the current workflow.  
  - User should add nodes to store, notify, or further process new blog posts.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                           | Input Node(s)                         | Output Node(s)                    | Sticky Note                                                                                                  |
|-------------------------------|------------------------|-----------------------------------------|-------------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Starts workflow manually                 | -                                   | max_content_age_days              |                                                                                                              |
| max_content_age_days           | Set                    | Sets max blog age (default 60 days)     | When clicking ‘Execute workflow’    | Client ID + Max Content Age + Blogs, blogs to track | ## Choose # of Days Filter out blogs older than **your number of days** old The default is 60, so it won't save blogs posted over 60 days ago. |
| blogs to track                | Set                    | Sets RSS feed URLs to track              | max_content_age_days                | Split Out                        | ## Find RSS FEED URLS Add the rss feed urls of the blogs that you want to track. I highly recommend using an LLM like perplexity to help you find the most up to date rss feeds. Search for something like: What is the rss feed url for the n8n blog in [today's date] |
| Split Out                    | Split Out               | Splits feed URLs into individual items  | blogs to track                     | Split RSS Feeds                  | ## Splits all of the websites into their own items so that they go into the loop one at a time                 |
| Split RSS Feeds              | Split In Batches        | Processes feed URLs in batches           | Split Out                         | Client ID + Max Content Age + Blogs (index 1), Rss feed link (index 0) | ## This loops over the websites one at a time to find all blogs.                                               |
| Rss feed link                | Set                    | Prepares current feed URL for RSS reader | Split RSS Feeds                   | RSS → Items, rss feed links + blogs (second output) |                                                                                                              |
| RSS → Items                  | RSS Feed Read          | Reads RSS feed items from URL            | Rss feed link                    | rss feed links + blogs (first output) |                                                                                                              |
| rss feed links + blogs       | Merge                  | Merges RSS items with feed URL context   | RSS → Items, Rss feed link         | Split RSS Feeds                  |                                                                                                              |
| Client ID + Max Content Age + Blogs | Merge                  | Combines blog items with metadata        | Split RSS Feeds, max_content_age_days, blogs to track | Find Date & Time of Blogs, Merge3 |                                                                                                              |
| Find Date & Time of Blogs    | Date Time              | Standardizes blog publication dates      | Client ID + Max Content Age + Blogs | Merge3                          | ## Adds a 'published date' in an easy to read format                                                          |
| Merge3                      | Merge                  | Combines enriched blog data               | Find Date & Time of Blogs, Client ID + Max Content Age + Blogs | Filter Out Old Blogs            |                                                                                                              |
| Filter Out Old Blogs         | If                     | Filters blogs older than `max_content_age_days` | Merge3                          | Merge3 (true path)               | ## Filters out blogs that are older than the date you specified                                              |
| Sticky Note                 | Sticky Note            | Introductory explanation and usage notes | -                                 | -                                | ## Welcome My Friend - This is how you download blogs and filter out old ones - all for free and with no api keys Necessary setup: 1. The node called 'max_content_age_days' is for you to set how recent you want your blogs to be. The default is 60. That means it will save blogs that were published within the last 60 days. 2. The 'blogs to track' node is a list of the blogs you are interested in tracking. The trick is that you must search for the websites' rss feed url. I have started you off with the rss feed url of n8n and zapier. If this workflow gives you a 403 error that means one of your links is wrong. It's best to find the most up to date rss feed url using something like perplexity (most up to date information). 3. The 'filter out old blogs' node is set up to send OLD blogs down the false path, and NEW blogs down the true path. 4. I highly recommend adding a node of your choice to the TRUE path of the 'filter out old blogs' node. Think about where you want to store these newly saved blogs. For example: - a loop over items node and a google sheets node. This will send one blog into the google sheet at a time. Check my page for more templates, I'm posting more variations of this workflow. Send me questions if you need help! I got you! |
| Sticky Note1                | Sticky Note            | Explains max content age days setting    | -                                 | -                                | ## Choose # of Days Filter out blogs older than **your number of days** old The default is 60, so it won't save blogs posted over 60 days ago. |
| Sticky Note2                | Sticky Note            | Advice on finding RSS feed URLs           | -                                 | -                                | ## Find RSS FEED URLS Add the rss feed urls of the blogs that you want to track. I highly recommend using an LLM like perplexity to help you find the most up to date rss feeds. Search for something like: What is the rss feed url for the n8n blog in [today's date] |
| Sticky Note3                | Sticky Note            | Explains batch processing of feeds        | -                                 | -                                | ## This loops over the websites one at a time to find all blogs.                                              |
| Sticky Note4                | Sticky Note            | Explains date formatting                   | -                                 | -                                | ## Adds a 'published date' in an easy to read format                                                           |
| Sticky Note5                | Sticky Note            | Explains filtering of old blogs            | -                                 | -                                | ## Filters out blogs that are older than the date you specified                                               |
| Sticky Note6                | Sticky Note            | Explains splitting of feed URLs            | -                                 | -                                | ## Splits all of the websites into their own items so that they go into the loop one at a time                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow.

2. **Create a Set node**  
   - Name: `max_content_age_days`  
   - Purpose: To define the maximum age of blog posts to keep.  
   - Configuration: Add a number field named `max_content_age_days` with value `60`. Enable "Include Other Fields" to pass through all data. Connect input from the manual trigger.

3. **Create a Set node**  
   - Name: `blogs to track`  
   - Purpose: To define the RSS feed URLs to monitor.  
   - Configuration: Add an array field named `source_identifier` with the following values:  
     - `https://blog.n8n.io/rss`  
     - `https://zapier.com/blog/feeds/latest/`  
   - Connect input from `max_content_age_days`.

4. **Create a Split Out node**  
   - Name: `Split Out`  
   - Purpose: To split the array of RSS URLs into individual items.  
   - Configuration: Set `Field to Split Out` to `source_identifier`.  
   - Connect input from `blogs to track`.

5. **Create a Split In Batches node**  
   - Name: `Split RSS Feeds`  
   - Purpose: To process feed URLs one at a time (batch size defaults to 1).  
   - Configuration: Default options (no batch size override).  
   - Connect input from `Split Out`.

6. **Create a Set node**  
   - Name: `Rss feed link`  
   - Purpose: To pass current feed URL to the RSS reader.  
   - Configuration: Set field `source_identifier` to `{{$json.source_identifier}}`.  
   - Connect input from `Split RSS Feeds` (first output).

7. **Create an RSS Feed Read node**  
   - Name: `RSS → Items`  
   - Purpose: To fetch blog posts from the feed URL.  
   - Configuration:  
     - URL: `{{$json.source_identifier}}` (expression)  
     - Ignore SSL: false  
     - Custom Fields: empty  
     - Retry on fail: enabled with 2 max tries  
     - On error: continue regular output  
   - Connect input from `Rss feed link`.

8. **Create a Merge node**  
   - Name: `rss feed links + blogs`  
   - Purpose: To combine RSS items with feed URL context.  
   - Configuration: Mode `combineAll`.  
   - Connect inputs from `RSS → Items` (main output) and `Rss feed link` (secondary output).

9. **Connect output of `rss feed links + blogs` back to `Split RSS Feeds` (second output)**  
   - This closes the loop to process all feeds.

10. **Create a Merge node**  
    - Name: `Client ID + Max Content Age + Blogs`  
    - Purpose: To join blog items with max age and feed list metadata.  
    - Configuration: Mode `combineAll`.  
    - Connect inputs from `Split RSS Feeds` (second output) and from `max_content_age_days` & `blogs to track`.

11. **Create a Date Time node**  
    - Name: `Find Date & Time of Blogs`  
    - Purpose: To standardize blog post dates.  
    - Configuration:  
      - Date: `{{$json.pubDate || $json.isoDate}}` (expression)  
      - Operation: Format Date  
      - Format: `yyyy-MM-dd`  
      - Output field: `standardizedPubDate`  
    - Connect input from `Client ID + Max Content Age + Blogs`.

12. **Create a Merge node**  
    - Name: `Merge3`  
    - Purpose: To merge enriched blog data.  
    - Configuration: Combine by position.  
    - Connect inputs from `Find Date & Time of Blogs` and `Client ID + Max Content Age + Blogs`.

13. **Create an If node**  
    - Name: `Filter Out Old Blogs`  
    - Purpose: To filter blogs newer than max age.  
    - Configuration:  
      - Condition:  
        - Left value: `{{$json.pubDate}}`  
        - Right value: `{{DateTime.now().minus({ days: $json.max_content_age_days }).toFormat('yyyy-MM-dd')}}`  
        - Operator: DateTime, afterOrEquals  
    - Connect input from `Merge3`.

14. **Connect true output of `Filter Out Old Blogs` to `Merge3`**  
    - This allows further processing of recent blogs.

15. **Add user-defined nodes here for processing filtered blogs**  
    - For example, loop over items and send to Google Sheets, email, or database.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Welcome My Friend - This is how you download blogs and filter out old ones - all for free and with no api keys. Necessary setup: 1. The node called 'max_content_age_days' is for you to set how recent you want your blogs to be. The default is 60. That means it will save blogs that were published within the last 60 days. 2. The 'blogs to track' node is a list of the blogs you are interested in tracking. The trick is that you must search for the websites' rss feed url. I have started you off with the rss feed url of n8n and zapier. If this workflow gives you a 403 error that means one of your links is wrong. It's best to find the most up to date rss feed url using something like perplexity (most up to date information). 3. The 'filter out old blogs' node is set up to send OLD blogs down the false path, and NEW blogs down the true path. 4. I highly recommend adding a node of your choice to the TRUE path of the 'filter out old blogs' node. Think about where you want to store these newly saved blogs. For example: - a loop over items node and a google sheets node. This will send one blog into the google sheet at a time. Check my page for more templates, I'm posting more variations of this workflow. Send me questions if you need help! I got you! | Inline Sticky Note in workflow introduction                                                                 |
| Choose # of Days. Filter out blogs older than **your number of days** old. The default is 60, so it won't save blogs posted over 60 days ago.                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note1                                                                                        |
| Find RSS FEED URLS. Add the rss feed urls of the blogs that you want to track. I highly recommend using an LLM like perplexity to help you find the most up to date rss feeds. Search for something like: What is the rss feed url for the n8n blog in [today's date]                                                                                                                                                                                                                                                                 | Sticky Note2                                                                                        |
| This loops over the websites one at a time to find all blogs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note3                                                                                        |
| Adds a 'published date' in an easy to read format                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note4                                                                                        |
| Filters out blogs that are older than the date you specified                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note5                                                                                        |
| Splits all of the websites into their own items so that they go into the loop one at a time                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note6                                                                                        |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.