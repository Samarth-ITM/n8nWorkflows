Flow from Reddit to Gmail with Key Features and GPT-4o Mini Usage

https://n8nworkflows.xyz/workflows/flow-from-reddit-to-gmail-with-key-features-and-gpt-4o-mini-usage-7162


# Flow from Reddit to Gmail with Key Features and GPT-4o Mini Usage

### 1. Workflow Overview

This workflow automates the process of generating a daily email digest of hot Reddit posts from multiple subreddits, enhanced by AI-generated summaries. It targets users interested in curated Reddit content summaries delivered directly to their Gmail inbox. The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Configuration:** Defines when the workflow runs daily and sets the list of subreddits, email recipient, and digest topic.
- **1.2 Data Retrieval:** Fetches hot posts from each specified subreddit and normalizes their dates.
- **1.3 Filtering & Sorting:** Filters posts from the last 24 hours with a score above a threshold and sorts them descending by score.
- **1.4 Post and Comments Processing:** Retrieves detailed post data and top-level comments, flattens comment threads, and constructs a text representation of each Reddit thread.
- **1.5 AI Summarization & Formatting:** Uses GPT-4o Mini to summarize each thread, formats summaries into Gmail-compatible HTML.
- **1.6 Email Delivery:** Sends the formatted digest email to the configured Gmail address.
- **1.7 Sub-workflow Invocation:** For each Reddit thread, invokes a sub-workflow that handles summarization and emailing.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Configuration

- **Overview:** Sets up the daily trigger for the workflow, and configures the subreddits to track, the digest topic, and the recipient email.
- **Nodes Involved:** 
  - Schedule: Daily run  
  - Set Topic, Subreddits and Email Address  
  - Split Subreddit List  
  - Sticky Notes (for user guidance)
  
- **Node Details:**

  - **Schedule: Daily run**  
    - *Type:* Schedule Trigger  
    - *Config:* Runs daily at 06:00 (configured via triggerAtHour=6), timezone Asia/Singapore  
    - *Input/Output:* No input; outputs to "Set Topic..." node  
    - *Edge Cases:* Timezone mismatch can cause unexpected run times; users must verify workflow timezone settings.
  
  - **Set Topic, Subreddits and Email Address**  
    - *Type:* Set  
    - *Config:* Defines three variables:  
      - `topic`: short label for digest (e.g., "Investing")  
      - `subreddits`: array of subreddit names (e.g., ["investing", "stocks"])  
      - `email`: recipient Gmail address  
    - *Input:* Trigger from Schedule  
    - *Output:* Feeds into Split Subreddit List  
    - *Edge Cases:* Invalid email or empty subreddit list will break downstream logic.
  
  - **Split Subreddit List**  
    - *Type:* SplitOut  
    - *Config:* Splits the array in `subreddits` into individual items in `subreddit` field for iteration  
    - *Input:* From Set Topic node  
    - *Output:* Feeds into List Hot Posts node  
    - *Edge Cases:* Empty or malformed subreddit names cause API errors.
  
  - **Sticky Notes**  
    - Provide user instructions on configuration, scheduling, and workflow purpose.

#### 2.2 Data Retrieval

- **Overview:** Fetches hot posts from each subreddit; normalizes post dates to ISO format; merges raw posts with normalized data.
- **Nodes Involved:**  
  - List Hot Posts  
  - Normalize Post Date (created_utc → ISO)  
  - Attach ISO Date to Posts1 (Merge node)  
  - Select Fields (id, score, url, date, subreddit)  
  - Sticky Note (labeling this block)
  
- **Node Details:**

  - **List Hot Posts**  
    - *Type:* Reddit  
    - *Config:* Gets all hot posts from subreddit field; uses Reddit OAuth2 credentials  
    - *Edge Cases:* Rate limits or auth failures if Reddit tokens expire.
  
  - **Normalize Post Date (created_utc → ISO)**  
    - *Type:* Set  
    - *Config:* Converts `created_utc` (epoch seconds) into ISO 8601 date string `created_iso`  
    - *Key Expression:* JavaScript expression to handle seconds vs milliseconds  
    - *Edge Cases:* Missing or malformed dates are ignored safely (ignoreConversionErrors=true).
  
  - **Attach ISO Date to Posts1**  
    - *Type:* Merge (combine by position)  
    - *Config:* Combines raw post data with normalized date data for each post.
  
  - **Select Fields (id, score, url, date, subreddit)**  
    - *Type:* Set  
    - *Config:* Selects and renames fields for downstream filtering:  
      - `id`, `score`, `url`, `date` (normalized), and `subreddit`  
    - *Edge Cases:* Missing fields can cause filter errors later.

#### 2.3 Filtering & Sorting

- **Overview:** Filters posts published within the last 24 hours and with score greater than 30, then sorts by score descending.
- **Nodes Involved:**  
  - Filter: Last 24h & Score > 30  
  - Sort by Score (desc)  
  - Sticky Note (labeling this block)
  
- **Node Details:**

  - **Filter: Last 24h & Score > 30**  
    - *Type:* Filter  
    - *Config:*  
      - `date` >= (now - 24 hours)  
      - `score` > 30  
    - *Edge Cases:* Posts with missing dates or scores may be excluded; score threshold can be customized.
  
  - **Sort by Score (desc)**  
    - *Type:* Sort  
    - *Config:* Sorts posts descending by `score` field.

#### 2.4 Post and Comments Processing

- **Overview:** Processes each post individually, fetches detailed post info and top-level comments, flattens comment threads, filters valid comments, and builds a text representation of the Reddit conversation thread.
- **Nodes Involved:**  
  - Iterate Posts (single batch)  
  - Get Post by ID  
  - Deduplicate by ID  
  - Build Post Body (root)  
  - List Top-level Comments  
  - Flatten Comment Threads  
  - Filter: Valid Comments  
  - Append Posts + Comments (merge)  
  - Build Thread Text (code)  
  - Merge: Thread + Post Details  
  - Set (id, thread_text, link)  
  - Sticky Note (labeling this block)
  
- **Node Details:**

  - **Iterate Posts (single batch)**  
    - *Type:* SplitInBatches  
    - *Config:* Batch size set to all items (process all posts in one batch)  
    - *Output:* Branches to Get Post by ID and Merge node
  
  - **Get Post by ID**  
    - *Type:* Reddit  
    - *Config:* Fetches detailed post info by ID and subreddit  
    - *Edge Cases:* API errors or missing posts.
  
  - **Deduplicate by ID**  
    - *Type:* RemoveDuplicates  
    - *Config:* Removes duplicate items based on `id` field.
  
  - **Build Post Body (root)**  
    - *Type:* Code  
    - *Config:* Maps each post to a structured object including post metadata and body (title + selftext)  
    - *Edge Cases:* Missing fields gracefully handled.
  
  - **List Top-level Comments**  
    - *Type:* Reddit  
    - *Config:* Fetches all top-level comments for the post by postId and subreddit.
  
  - **Flatten Comment Threads**  
    - *Type:* Code  
    - *Config:* Recursively walks comment replies to flatten threaded comments into a list with depth info.
  
  - **Filter: Valid Comments**  
    - *Type:* Filter  
    - *Config:* Ensures comments have a non-empty `post_id` field.
  
  - **Append Posts + Comments**  
    - *Type:* Merge  
    - *Config:* Combines post and comment items into one list.
  
  - **Build Thread Text**  
    - *Type:* Code  
    - *Config:* Builds a Reddit-style indented plain text thread for each post, preserving hierarchy and timestamps.
  
  - **Merge: Thread + Post Details**  
    - *Type:* Merge  
    - *Config:* Combines thread text with post metadata.
  
  - **Set (id, thread_text, link)**  
    - *Type:* Set  
    - *Config:* Prepares simplified output with just `id`, `thread_text`, and `link` for downstream summarization.
  
#### 2.5 AI Summarization & Formatting

- **Overview:** For each thread, calls a sub-workflow to summarize and email the digest. Within the sub-workflow, uses GPT-4o Mini for summarization, merges summaries and links, formats the content into Gmail-compatible HTML.
- **Nodes Involved:**  
  - Call Sub-workflow: Summarize & Email  
  - Sub-workflow Trigger (within sub-workflow)  
  - Summarize Threads (LLM chain summarization)  
  - GPT-4o mini (open router LLM)  
  - Combine Summaries + Links  
  - Aggregate Summaries & Links  
  - Format HTML for Gmail (LLM chain with prompt)  
  - Send Digest (Gmail node)  
  - Extract Link  
  - Extract Thread Text  
  - Sticky Note (labeling this block)
  
- **Node Details:**

  - **Call Sub-workflow: Summarize & Email**  
    - *Type:* ExecuteWorkflow  
    - *Config:* Passes `id`, `link`, `email`, `topic`, and `thread_text` to sub-workflow with the same workflow ID (nested execution)  
    - *Edge Cases:* Recursive workflow calls can cause execution depth issues if not handled carefully.
  
  - **Sub-workflow Trigger**  
    - *Type:* ExecuteWorkflowTrigger  
    - *Config:* Receives inputs from parent workflow.
  
  - **Summarize Threads**  
    - *Type:* Langchain Chain Summarization  
    - *Config:* Summarizes thread text using configured LLM  
    - *Edge Cases:* LLM API failures or rate limits.
  
  - **GPT-4o mini**  
    - *Type:* Langchain LM Chat Open Router  
    - *Config:* Uses OpenRouter API with GPT-4o mini model for summarization steps.
  
  - **Combine Summaries + Links**  
    - *Type:* Merge  
    - *Config:* Combines summary texts with corresponding Reddit links.
  
  - **Aggregate Summaries & Links**  
    - *Type:* Aggregate  
    - *Config:* Aggregates all summary texts and links into arrays.
  
  - **Format HTML for Gmail**  
    - *Type:* Langchain Chain LLM  
    - *Config:* Custom prompt formats summaries and links into Gmail-safe HTML email content with clean hyperlinks and spacing.
  
  - **Send Digest**  
    - *Type:* Gmail  
    - *Config:* Sends email to recipient using OAuth2 credentials, subject includes topic and "Daily Digest".
  
  - **Extract Link / Extract Thread Text**  
    - *Type:* Set  
    - *Config:* Extracts the `link` and `thread_text` fields from workflow inputs for summarization.

#### 2.6 Sticky Notes for User Guidance

Multiple sticky notes provide instructions and context for users, including:

- Required settings and configuration steps  
- Workflow purpose and overview  
- Customization tips  
- Link to a YouTube tutorial on workflow setup

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                                 | Input Node(s)                              | Output Node(s)                                   | Sticky Note                                                                                              |
|-----------------------------------|----------------------------------|------------------------------------------------|-------------------------------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule: Daily run                | Schedule Trigger                 | Daily workflow scheduling                       | -                                         | Set Topic, Subreddits and Email Address          | **Start here (required settings)**: Configure schedule and topic/subreddits/email                        |
| Set Topic, Subreddits and Email Address | Set                                | Define subreddits, email, and digest topic      | Schedule: Daily run                        | Split Subreddit List                              | **Start here (required settings)**: Set `topic`, `subreddits`, and `email`                              |
| Split Subreddit List              | SplitOut                        | Split subreddit array into individual subreddits | Set Topic, Subreddits and Email Address   | List Hot Posts                                    | ## Set subreddits you wish to track                                                                      |
| List Hot Posts                   | Reddit                          | Fetch all hot posts per subreddit                | Split Subreddit List                      | Normalize Post Date (created_utc → ISO), Attach ISO Date to Posts1 | ## Retrieve daily hot subreddit posts                                                                   |
| Normalize Post Date (created_utc → ISO) | Set                                | Convert created_utc epoch seconds to ISO date    | List Hot Posts                           | Attach ISO Date to Posts1                          | ## Retrieve daily hot subreddit posts                                                                   |
| Attach ISO Date to Posts1        | Merge                          | Combine raw post and normalized date data        | Normalize Post Date, List Hot Posts      | Select Fields (id, score, url, date, subreddit)  | ## Retrieve daily hot subreddit posts                                                                   |
| Select Fields (id, score, url, date, subreddit) | Set                                | Select relevant fields for filtering              | Attach ISO Date to Posts1                 | Filter: Last 24h & Score > 30                      | ## Retrieve daily hot subreddit posts                                                                   |
| Filter: Last 24h & Score > 30    | Filter                         | Filter posts within last 24h and score > 30      | Select Fields                            | Sort by Score (desc)                              | ## Process each post                                                                                      |
| Sort by Score (desc)             | Sort                           | Sort posts descending by score                    | Filter: Last 24h & Score > 30             | Iterate Posts (single batch)                       | ## Process each post                                                                                      |
| Iterate Posts (single batch)     | SplitInBatches                 | Process all posts in a single batch               | Sort by Score (desc)                     | Call Sub-workflow: Summarize & Email, Merge: Thread + Post Details, Get Post by ID | ## Process each post                                                                                      |
| Get Post by ID                  | Reddit                         | Retrieve detailed post data by ID                  | Iterate Posts (single batch)              | Deduplicate by ID                                 | ## Process each post                                                                                      |
| Deduplicate by ID               | RemoveDuplicates               | Remove duplicate posts by ID                        | Get Post by ID                          | Build Post Body (root)                            | ## Process each post                                                                                      |
| Build Post Body (root)          | Code                          | Build structured post object with metadata         | Deduplicate by ID                       | Append Posts + Comments, List Top-level Comments | ## Process each post                                                                                      |
| List Top-level Comments          | Reddit                         | Fetch top-level comments for a post                 | Build Post Body (root)                   | Flatten Comment Threads                           | ## Process each post                                                                                      |
| Flatten Comment Threads          | Code                          | Flatten comment replies recursively into list       | List Top-level Comments                  | Filter: Valid Comments                            | ## Process each post                                                                                      |
| Filter: Valid Comments           | Filter                         | Filter comments ensuring post_id exists             | Flatten Comment Threads                  | Append Posts + Comments                           | ## Process each post                                                                                      |
| Append Posts + Comments          | Merge                          | Combine posts and comments into single list           | Build Post Body (root), Filter: Valid Comments | Build Thread Text                               | ## Process each post                                                                                      |
| Build Thread Text               | Code                          | Build Reddit-style indented thread text               | Append Posts + Comments                  | Merge: Thread + Post Details                      | ## Process each post                                                                                      |
| Merge: Thread + Post Details     | Merge                          | Combine thread text with post metadata                 | Build Thread Text, Iterate Posts (single batch) | Set (id, thread_text, link)                      | ## Process each post                                                                                      |
| Set (id, thread_text, link)      | Set                            | Prepare minimal data for summarization and email        | Merge: Thread + Post Details             | Iterate Posts (single batch)                       | ## Process each post                                                                                      |
| Call Sub-workflow: Summarize & Email | ExecuteWorkflow               | Call sub-workflow to summarize threads and send email | Iterate Posts (single batch)              | -                                               | ## Summarizes each post and sends to your inbox                                                        |
| Sub-workflow Trigger            | ExecuteWorkflowTrigger         | Entry point for sub-workflow receiving post data       | Call Sub-workflow: Summarize & Email    | Extract Link, Extract Thread Text                 | ## Summarizes each post and sends to your inbox                                                        |
| Extract Link                   | Set                            | Extract `link` field from input for formatting          | Sub-workflow Trigger                    | Combine Summaries + Links                          | ## Summarizes each post and sends to your inbox                                                        |
| Extract Thread Text            | Set                            | Extract `thread_text` field from input for summarizing    | Sub-workflow Trigger                    | Summarize Threads                                  | ## Summarizes each post and sends to your inbox                                                        |
| Summarize Threads              | Langchain Chain Summarization | Generate summary text from Reddit thread text            | Extract Thread Text                     | Combine Summaries + Links                          | ## Summarizes each post and sends to your inbox                                                        |
| GPT-4o mini                   | Langchain LM Chat Open Router | Use GPT-4o Mini model for summarization                   | Summarize Threads                      | Format HTML for Gmail                              | ## Summarizes each post and sends to your inbox                                                        |
| Combine Summaries + Links       | Merge                          | Combine summarized texts with corresponding links         | Extract Link, Summarize Threads         | Aggregate Summaries & Links                        | ## Summarizes each post and sends to your inbox                                                        |
| Aggregate Summaries & Links     | Aggregate                      | Aggregate summaries and links into arrays                  | Combine Summaries + Links                | Format HTML for Gmail                              | ## Summarizes each post and sends to your inbox                                                        |
| Format HTML for Gmail           | Langchain Chain LLM            | Format summaries and links into Gmail-safe HTML             | Aggregate Summaries & Links             | Send Digest                                       | ## Summarizes each post and sends to your inbox                                                        |
| Send Digest                    | Gmail                          | Send the final digest email to configured Gmail address     | Format HTML for Gmail                   | -                                               | ## Summarizes each post and sends to your inbox                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: Schedule: Daily run  
   - Set the rule to trigger daily at 06:00 (adjust timezone in workflow settings as needed).  

2. **Create a Set node:**  
   - Name: Set Topic, Subreddits and Email Address  
   - Add three variables:  
     - `topic` (string): e.g., "Investing"  
     - `subreddits` (array): e.g., ["investing", "stocks"]  
     - `email` (string): recipient Gmail address e.g., "johndoe@gmail.com"  
   - Connect Schedule Trigger output to this node.  

3. **Create a SplitOut node:**  
   - Name: Split Subreddit List  
   - Field to split: `subreddits`  
   - Destination field name: `subreddit`  
   - Connect from Set node.  

4. **Create a Reddit node:**  
   - Name: List Hot Posts  
   - Operation: getAll  
   - Filters: category = hot  
   - Subreddit: `={{ $json.subreddit }}`  
   - Use Reddit OAuth2 credentials (configure with your Reddit app credentials).  
   - Connect from Split Subreddit List.  

5. **Create a Set node:**  
   - Name: Normalize Post Date (created_utc → ISO)  
   - Use JavaScript expression to convert `created_utc` epoch seconds to ISO string assigned to `created_iso`.  
   - Set option to ignore conversion errors.  
   - Connect from List Hot Posts.  

6. **Create a Merge node:**  
   - Name: Attach ISO Date to Posts1  
   - Mode: Combine by position  
   - Connect main input from List Hot Posts, second input from Normalize Post Date node.  

7. **Create a Set node:**  
   - Name: Select Fields (id, score, url, date, subreddit)  
   - Assign fields:  
     - `subreddit` = `{{ $json.subreddit }}`  
     - `id` = `{{ $json.id }}`  
     - `date` = `{{ $json.created_iso }}`  
     - `score` = `{{ $json.score }}`  
     - `url` = `{{ $json.url }}`  
   - Connect from Attach ISO Date node.  

8. **Create a Filter node:**  
   - Name: Filter: Last 24h & Score > 30  
   - Conditions:  
     - `date` after or equal to `now - 24h` (use expression for dynamic date)  
     - `score` greater than 30 (customize threshold as needed)  
   - Connect from Select Fields.