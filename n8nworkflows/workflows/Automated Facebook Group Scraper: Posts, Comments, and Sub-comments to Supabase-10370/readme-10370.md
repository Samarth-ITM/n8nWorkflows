Automated Facebook Group Scraper: Posts, Comments, and Sub-comments to Supabase

https://n8nworkflows.xyz/workflows/automated-facebook-group-scraper--posts--comments--and-sub-comments-to-supabase-10370


# Automated Facebook Group Scraper: Posts, Comments, and Sub-comments to Supabase

### 1. Workflow Overview

This workflow automates the scraping of Facebook group data—posts, comments on those posts, and sub-comments (replies to comments)—and stores the structured information in a Supabase database. It is designed for users who want to continuously gather and archive Facebook group discussions for analysis, monitoring, or archival purposes.

The workflow is logically divided into three main functional blocks:

- **1.1 Input Reception:** Captures inputs (Facebook group URL and number of posts to scrape) via a form trigger.
- **1.2 Facebook Group Posts Scraping and Storage:** Scrapes group posts using the Apify Facebook Group Posts actor, formats the data, and inserts it into Supabase’s `posts` table.
- **1.3 Comments and Sub-comments Scraping and Storage:** For each post, scrapes comments and stores them in a `comments` table, then conditionally scrapes sub-comments (replies to comments) and stores them in a `replies` table in Supabase.

Each block involves multiple nodes that process data step-by-step, ensuring data integrity, conditional flows, and seamless integration between Apify scraping and database insertion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives user inputs via a form submission, specifically the URL of the Facebook group and optionally the number of posts to scrape. The input triggers the workflow.

- **Nodes Involved:**  
  - On form submission  
  - Edit Fields1

- **Node Details:**  

  1. **On form submission**  
     - Type: `formTrigger`  
     - Role: Entry point that waits for form data submission with fields `url` (string, required) and `number of posts` (number, optional).  
     - Configuration: Form titled "URL" with two fields: `url` (Facebook group URL) and `number of posts` (to limit scraping).  
     - Inputs: External user input (form)  
     - Outputs: JSON with `url` and `post` (number of posts)  
     - Edge cases: Missing URL will prevent trigger; number of posts missing will default to scraping all posts.  
     
  2. **Edit Fields1**  
     - Type: `set`  
     - Role: Formats and sets fields for downstream use; essentially passes input values while setting default or structured fields.  
     - Configuration:  
       - Sets `url` from form input  
       - Sets `post` number to 11 (default number of posts to scrape if not provided)  
     - Inputs: From "On form submission" node  
     - Outputs: JSON with set fields for next node  
     - Edge cases: If `post` is missing, defaults to 11.

---

#### 2.2 Facebook Group Posts Scraping and Storage

- **Overview:**  
  This block uses the Apify Facebook Group Posts actor to scrape posts from the specified Facebook group URL. The scraped posts are then formatted and stored in the Supabase `posts` table.

- **Nodes Involved:**  
  - Run an Actor  
  - Get dataset items  
  - Add A Post  
  - Sticky Note (explanatory)

- **Node Details:**  

  1. **Run an Actor**  
     - Type: Apify actor node  
     - Role: Executes the Apify actor "AtBpiepuIUNs2k2ku" (Facebook Group Posts Scraper) with parameters from the input.  
     - Configuration:  
       - Memory: 16 GB  
       - Actor ID: fixed to Facebook Group Posts Scraper  
       - Custom body includes:  
         - `count`: number of posts to scrape (from input JSON)  
         - `minDelay` and `maxDelay` for pacing requests  
         - `proxy` enabled via Apify proxy  
         - `scrapeGroupPosts.groupUrl`: URL of Facebook group to scrape  
         - `sortType`: "new_posts" (sort posts by newest first)  
     - Inputs: From "Edit Fields1" node  
     - Outputs: Initiates dataset creation on Apify for scraped posts  
     - Edge cases: API errors, proxy failures, actor timeout, invalid group URL, missing permissions.  
     - Credentials: Apify API credentials required.  
     
  2. **Get dataset items**  
     - Type: Apify dataset retrieval node  
     - Role: Retrieves scraped posts data from Apify dataset created by the actor.  
     - Configuration: Uses dataset ID from the output of "Run an Actor" node.  
     - Inputs: From "Run an Actor" node output  
     - Outputs: JSON array of posts data  
     - Edge cases: Dataset not ready, empty dataset, API errors.  
     - Credentials: Apify API credentials required.  
     
  3. **Add A Post**  
     - Type: Supabase insert node  
     - Role: Inserts each post into Supabase database table `posts`.  
     - Configuration: Maps fields from scraped data to Supabase columns:  
       - `createdat`: formatted date from UNIX timestamp (day, month name, year)  
       - `url`: post URL  
       - `user_name`: poster’s name  
       - `text`: post text content  
       - `reactioncount`, `sharecount`, `commentcount`: respective counts  
       - `attachments`: array of attachment URLs extracted from post data  
     - Inputs: From "Get dataset items" node  
     - Outputs: Confirmation of inserted rows  
     - Edge cases: Invalid data format, insertion errors, missing fields.  
     - Credentials: Supabase API credentials required.

  4. **Sticky Note**  
     - Content explains this block automates Facebook group post scraping and storage into Supabase posts table.  
     - Provides high-level summary and purpose.

---

#### 2.3 Comments and Sub-comments Scraping and Storage

- **Overview:**  
  This block scrapes comments for each post stored earlier, adds them into Supabase `comments` table, then conditionally scrapes sub-comments (replies to comments) and stores those in a separate `replies` table.

- **Nodes Involved:**  
  - ScrapeComments  
  - Get dataset items1  
  - Add A Comment  
  - If1  
  - Edit Fields  
  - Split Out  
  - Create a row  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**  

  1. **ScrapeComments**  
     - Type: Apify actor node  
     - Role: Executes Apify actor "us5srxAYnsrkgUv2v" that scrapes comments from a Facebook post URL.  
     - Configuration:  
       - Memory: 8 GB  
       - Actor ID: fixed to Facebook Comments Scraper  
       - Custom body includes:  
         - `includeNestedComments`: false (only first-level comments here)  
         - `startUrls`: post URL passed from previous node  
         - `viewOption`: "RANKED_UNFILTERED" (all comments unfiltered)  
     - Inputs: From "Add A Post" (post URL)  
     - Outputs: Dataset created on Apify with comments data  
     - Edge cases: Actor failures, missing post URL, rate limits, comments disabled on post.  
     - Credentials: Apify API credentials required.  
     
  2. **Get dataset items1**  
     - Type: Apify dataset retrieval node  
     - Role: Retrieves scraped comments for a post from Apify dataset.  
     - Configuration: Uses dataset ID from "ScrapeComments" node output.  
     - Inputs: From "ScrapeComments"  
     - Outputs: JSON array of comments  
     - Edge cases: Dataset empty, retrieval errors.  
     - Credentials: Apify API credentials required.  
     
  3. **Add A Comment**  
     - Type: Supabase insert node  
     - Role: Inserts comments into Supabase `comments` table, linking them to group title, post title, and original post text.  
     - Configuration:  
       - Maps fields such as:  
         - `group_title`: extracted from URL path segments  
         - `post_title`: post title from previous data  
         - `facebookurl`: original input URL  
         - `commenturl`, `text`, `profilename`, `likescount`, `commentscount` from comment data  
         - `Attachments`: array of attachment URLs extracted safely from nested objects  
     - Inputs: From "Get dataset items1"  
     - Outputs: Confirmation of comment insertion  
     - Edge cases: Missing fields, attachment mapping issues, insertion errors.  
     - Credentials: Supabase API credentials required.  
     
  4. **If1**  
     - Type: Conditional (if) node  
     - Role: Checks if a comment has more than 0 nested sub-comments (commentsCount > 0) to decide if sub-comments scraping is needed.  
     - Configuration: Condition: `commentsCount` from comment data > 0  
     - Inputs: From "Add A Comment" node output  
     - Outputs: True branch proceeds to scrape sub-comments, false ends branch  
     - Edge cases: Missing commentsCount, null values causing false negatives.  
     
  5. **Edit Fields**  
     - Type: `set`  
     - Role: Prepares the `comments` array from the JSON data retrieved to enable splitting into individual comment items.  
     - Configuration: Assigns `comments` field with the array from `Get dataset items1` JSON  
     - Inputs: From "If1" true branch  
     - Outputs: Structured JSON to be split out  
     - Edge cases: Empty or missing comments array.  
     
  6. **Split Out**  
     - Type: `splitOut`  
     - Role: Splits the `comments` array into individual messages to process sub-comments separately.  
     - Configuration: Splits on `comments` field  
     - Inputs: From "Edit Fields"  
     - Outputs: Individual comment JSON items  
     - Edge cases: Empty array results in no output.  
     
  7. **Create a row**  
     - Type: Supabase insert node  
     - Role: Inserts sub-comments (replies) into Supabase `replies` table, linking back to parent comment and post.  
     - Configuration: Maps fields including:  
       - `parent_comment`, `parent_text` from parent comment data  
       - `commenturl`, `text`, `profileurl`, `profilename` from sub-comment data  
       - `post_text` from original post text  
     - Inputs: From "Split Out"  
     - Outputs: Confirmation of sub-comment insertion  
     - Edge cases: Missing parent data, insertion errors.  
     - Credentials: Supabase API credentials required.  
     
  8. **Sticky Note1**  
     - Explains that this block scrapes comments for previously scraped posts and stores them in Supabase comments table.  
     - Highlights automated syncing and processing features.  
     
  9. **Sticky Note2**  
     - Explains the sub-comments scraping step and its separate storage in Supabase replies table.  
     - Emphasizes automation and real-time syncing.

---

### 3. Summary Table

| Node Name        | Node Type                      | Functional Role                                         | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                       |
|------------------|--------------------------------|--------------------------------------------------------|-----------------------|----------------------|-------------------------------------------------------------------------------------------------|
| On form submission | formTrigger                   | Entry point; receives Facebook group URL and post count | -                     | Edit Fields1         |                                                                                                 |
| Edit Fields1     | set                            | Sets and formats input fields for scraping              | On form submission    | Run an Actor         |                                                                                                 |
| Run an Actor     | Apify actor node               | Scrapes Facebook group posts using Apify actor          | Edit Fields1          | Get dataset items    |                                                                                                 |
| Get dataset items | Apify dataset retrieval        | Retrieves scraped posts dataset                          | Run an Actor          | Add A Post           |                                                                                                 |
| Add A Post       | Supabase insert                | Inserts posts into Supabase `posts` table                | Get dataset items     | ScrapeComments       |                                                                                                 |
| ScrapeComments   | Apify actor node               | Scrapes comments for each post                           | Add A Post            | Get dataset items1   |                                                                                                 |
| Get dataset items1| Apify dataset retrieval        | Retrieves scraped comments dataset                       | ScrapeComments        | Add A Comment        |                                                                                                 |
| Add A Comment    | Supabase insert                | Inserts comments into Supabase `comments` table          | Get dataset items1    | If1                  |                                                                                                 |
| If1              | if                            | Checks if comments have sub-comments to scrape           | Add A Comment         | Edit Fields (true)   |                                                                                                 |
| Edit Fields      | set                            | Prepares comments array for splitting                    | If1 (true branch)     | Split Out            |                                                                                                 |
| Split Out        | splitOut                      | Splits comments array into individual sub-comments       | Edit Fields           | Create a row         |                                                                                                 |
| Create a row     | Supabase insert                | Inserts sub-comments into Supabase `replies` table       | Split Out             |                      |                                                                                                 |
| Sticky Note      | stickyNote                    | Overview of Facebook posts scraping block                 | -                     | -                    | ## Facebook Scraping Automation  \nAutomation: This workflow scrapes posts from a Facebook group and inserts them into a **Supabase** database.  \nStep-by-Step:  \n1. Facebook Group Scraping\n2. Data Formatting\n\nFeatures: Automated scraping of posts and real-time syncing with Supabase. |
| Sticky Note1     | stickyNote                    | Overview of comments scraping block                        | -                     | -                    | ## Scraping Comments for Previous Posts  \nAutomation: Scrapes comments for posts and stores them in Supabase comments table.  \nFeatures: Automated scraping of comments and syncing with Supabase. |
| Sticky Note2     | stickyNote                    | Overview of sub-comments scraping block                    | -                     | -                    | ## Scraping Comments on Comments (Sub-comments)  \nAutomation: Scrapes sub-comments and stores them in a separate Supabase table.  \nFeatures: Automated scraping and syncing with Supabase. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a form trigger node ("On form submission")**  
   - Type: `formTrigger`  
   - Configure webhook with fields:  
     - `url` (string, required, placeholder: "Enter URL")  
     - `post` (number, optional, placeholder: "Enter number of posts...")  
   - Title the form "URL".

2. **Add a Set node ("Edit Fields1")**  
   - Type: `set`  
   - Assign:  
     - `url` = `{{$json["url"]}}` (from form)  
     - `post` = 11 (default number of posts if not provided)

3. **Add an Apify actor node ("Run an Actor")**  
   - Type: Apify node  
   - Credentials: Apify API with valid key  
   - Configure:  
     - Memory: 16384 MB  
     - Actor ID: `AtBpiepuIUNs2k2ku` (Facebook Group Posts Scraper)  
     - Custom Body (use expression):  
       ```
       {
         "count": {{$json["post"]}},
         "maxDelay": 10,
         "minDelay": 1,
         "proxy": { "useApifyProxy": true },
         "scrapeGroupPosts.groupUrl": "{{$json["url"]}}",
         "sortType": "new_posts"
       }
       ```

4. **Add Apify dataset retrieval node ("Get dataset items")**  
   - Type: Apify node  
   - Credentials: Apify API  
   - Configure to get dataset items from the dataset ID returned by "Run an Actor".

5. **Add Supabase insert node ("Add A Post")**  
   - Credentials: Supabase API  
   - Table: `posts`  
   - Map fields from dataset items:  
     - `createdat`: format UNIX timestamp `createdAt` to string "day, month name, year"  
     - `url`: post URL or null  
     - `user_name`: poster’s name or null  
     - `text`: post text or null  
     - `reactioncount`, `sharecount`, `commentcount`: respective counts or null  
     - `attachments`: extract array of URLs from attachments or null

6. **Add Apify actor node ("ScrapeComments")**  
   - Credentials: Apify API  
   - Memory: 8192 MB  
   - Actor ID: `us5srxAYnsrkgUv2v` (Facebook Comments Scraper)  
   - Custom Body:  
     ```
     {
       "includeNestedComments": false,
       "startUrls": [{"url": "{{$json["url"]}}"}],
       "viewOption": "RANKED_UNFILTERED"
     }
     ```

7. **Add Apify dataset node ("Get dataset items1")**  
   - Credentials: Apify API  
   - Retrieve the dataset for comments from "ScrapeComments".

8. **Add Supabase insert node ("Add A Comment")**  
   - Credentials: Supabase API  
   - Table: `comments`  
   - Map fields:  
     - `group_title`: extract last segment of URL path from prior step  
     - `post_title`: from input or prior post data  
     - `facebookurl`: original URL input  
     - `commenturl`, `text`, `profilename`, `likescount`, `commentscount` from comment data  
     - `Attachments`: map URLs from attachments safely

9. **Add If node ("If1")**  
   - Condition: `commentsCount` > 0  
   - Input: from "Add A Comment"

10. **Add Set node ("Edit Fields")** (on If true branch)  
    - Assign `comments` = `$('Get dataset items1').item.json.comments`

11. **Add Split Out node ("Split Out")**  
    - Split field: `comments`

12. **Add Supabase insert node ("Create a row")**  
    - Credentials: Supabase API  
    - Table: `replies`  
    - Map fields:  
      - `parent_comment`, `parent_text` from parent comment data  
      - `commenturl`, `text`, `profileurl`, `profilename` from sub-comment  
      - `post_text` from original post text

13. **Connect nodes in the order described:**  
    - On form submission → Edit Fields1 → Run an Actor → Get dataset items → Add A Post → ScrapeComments → Get dataset items1 → Add A Comment → If1 → (true) Edit Fields → Split Out → Create a row

14. **Add sticky notes for documentation and clarity at relevant points**, summarizing each block’s purpose.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                |
|-------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow requires Apify API credentials with access to Facebook scraping actors.           | Credential setup in Apify portal               |
| Supabase database must have three tables: `posts`, `comments`, and `replies` with matching schema fields as mapped. | Supabase schema design                          |
| Facebook scraping actors rely on proxy usage enabled in Apify to avoid rate limiting and blocks. | Apify Proxy usage recommended                   |
| Date formatting in "Add A Post" node uses JavaScript Date functions on UNIX timestamp fields.   | See node expressions for details                |
| Sub-comments scraping is conditional on comments having replies (`commentsCount > 0`).          | Prevents unnecessary scraping                    |
| Workflow handles null or missing data gracefully by defaulting to null in database inserts.     | Reduces insertion errors                          |
| For more info on Apify Facebook actors: https://apify.com/store/actors                            | Apify actor marketplace                          |

---

**Disclaimer:** The provided text originates solely from an automated workflow built with n8n, an integration and automation tool. All processing complies fully with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.