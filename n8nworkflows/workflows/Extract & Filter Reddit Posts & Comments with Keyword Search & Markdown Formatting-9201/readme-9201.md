Extract & Filter Reddit Posts & Comments with Keyword Search & Markdown Formatting

https://n8nworkflows.xyz/workflows/extract---filter-reddit-posts---comments-with-keyword-search---markdown-formatting-9201


# Extract & Filter Reddit Posts & Comments with Keyword Search & Markdown Formatting

### 1. Workflow Overview

This workflow automates the extraction, filtering, and formatting of Reddit posts and comments based on specified keyword searches and subreddit inputs. It is designed to:

- Receive webhook requests specifying either keywords or subreddit names.
- Retrieve Reddit posts matching those criteria.
- Filter posts by posting date and upvotes.
- Remove duplicates and sort posts by popularity.
- For each filtered post, extract and format the top comments.
- Aggregate all results into a final Markdown-formatted report.
- Respond to the webhook caller with the compiled report.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Two webhook nodes listen for incoming requests—one for keyword-based searches, another for subreddit-based retrievals. They set relevant variables for downstream processing.
- **1.2 Reddit Posts Retrieval:** Nodes query Reddit’s API to fetch posts either by keyword search or from a specified subreddit.
- **1.3 Post Filtering & Sorting:** Posts are filtered by date and upvotes, duplicates are removed, and posts are sorted by descending upvotes.
- **1.4 Comments Extraction & Formatting:** For each filtered post, the top 20 comments are extracted and formatted into Markdown.
- **1.5 Aggregation & Response:** The formatted posts and comments are aggregated into a final report, which is then sent back as a webhook response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming webhook requests. Depending on the entry point, it sets keywords or subreddit input for the Reddit queries.
- **Nodes involved:**  
  - `reddit tool webhook` (webhook)  
  - `reddit tool webhook subreddit` (webhook)  
  - `Set keywords` (set)  
  - `Set subreddit` (set)

- **Node Details:**

  - **reddit tool webhook**  
    - Type: Webhook (REST API entry point)  
    - Role: Receives requests with keywords for searching Reddit posts.  
    - Configuration: Uses a static webhook ID; expects input data to contain keywords.  
    - Outputs to: `Set keywords`.  
    - Edge cases: Missing or malformed keyword inputs; webhook authentication not configured (open access).  
    - Version: 2 (current stable).

  - **reddit tool webhook subreddit**  
    - Type: Webhook  
    - Role: Receives requests with subreddit names to fetch posts from.  
    - Configuration: Shares webhook ID with above webhook (likely differentiated by path or parameters).  
    - Outputs to: `Set subreddit`.  
    - Edge cases: Missing or invalid subreddit names; rate limiting from Reddit API.

  - **Set keywords**  
    - Type: Set  
    - Role: Stores and prepares keywords for the Reddit API search node.  
    - Configuration: Likely sets an expression or static variable from webhook data.  
    - Outputs to: `Search for posts by keywords`.

  - **Set subreddit**  
    - Type: Set  
    - Role: Stores and prepares subreddit name for the Reddit API posts retrieval node.  
    - Configuration: Sets subreddit parameter from webhook input.  
    - Outputs to: `Get posts from subreddit`.

#### 2.2 Reddit Posts Retrieval

- **Overview:** Executes Reddit API queries to fetch posts either by keyword or from a specific subreddit.
- **Nodes involved:**  
  - `Search for posts by keywords` (reddit)  
  - `Get posts from subreddit` (reddit)  
  - `Set Reddit Posts` (set)

- **Node Details:**

  - **Search for posts by keywords**  
    - Type: Reddit node (API)  
    - Role: Queries Reddit for posts matching the keywords.  
    - Configuration: Uses keywords from previous `Set keywords` node; likely sets search parameters such as limit, sort order.  
    - Outputs to: `Set Reddit Posts`.  
    - Edge cases: API rate limits, no results found, invalid keywords.

  - **Get posts from subreddit**  
    - Type: Reddit node (API)  
    - Role: Retrieves recent or top posts from the specified subreddit.  
    - Configuration: Uses subreddit name from `Set subreddit` node; may specify sorting or time filters.  
    - Outputs to: `Set Reddit Posts`.  
    - Edge cases: Private or banned subreddits, API errors, empty results.

  - **Set Reddit Posts**  
    - Type: Set  
    - Role: Prepares raw posts data for filtering.  
    - Configuration: Possibly extracts relevant fields from Reddit API response, normalizes data structure.  
    - Outputs to: `Posted in Last x days`.

#### 2.3 Post Filtering & Sorting

- **Overview:** Filters posts based on recency and minimum upvotes, removes duplicates, then sorts by popularity.
- **Nodes involved:**  
  - `Posted in Last x days` (code)  
  - `Upvotes Requirement Filtering` (if)  
  - `Remove Duplicates` (code)  
  - `Sort by upvotes (descending)` (sort)  
  - `Limit` (limit)

- **Node Details:**

  - **Posted in Last x days**  
    - Type: Code (JavaScript)  
    - Role: Filters posts to only those posted within a configurable recent timeframe.  
    - Configuration: Uses input post timestamps; compares to current date minus X days.  
    - Outputs to: `Upvotes Requirement Filtering`.  
    - Edge cases: Posts with missing or malformed timestamps; timezone issues.

  - **Upvotes Requirement Filtering**  
    - Type: If  
    - Role: Filters posts based on a minimum upvote threshold.  
    - Configuration: Checks if the post’s upvote count exceeds a preset minimum.  
    - Outputs to: `Remove Duplicates` if true; else filtered out.  
    - Edge cases: Posts with missing upvote data.

  - **Remove Duplicates**  
    - Type: Code  
    - Role: Ensures no duplicate posts remain by comparing unique post IDs.  
    - Outputs to: `Sort by upvotes (descending)`.

  - **Sort by upvotes (descending)**  
    - Type: Sort  
    - Role: Orders posts descending by upvote count to prioritize popular content.  
    - Outputs to: `Limit`.

  - **Limit**  
    - Type: Limit  
    - Role: Restricts the number of posts processed further to a manageable batch size.  
    - Outputs to: `Loop Over Items1`.

#### 2.4 Comments Extraction & Formatting

- **Overview:** For each post, fetches top comments, extracts the top 20, and formats them into Markdown.
- **Nodes involved:**  
  - `Loop Over Items1` (splitInBatches)  
  - `Aggregate` (aggregate)  
  - `Set post for Loop` (set)  
  - `Get Comments` (reddit)  
  - `Extract Top 20 Comments` (code)  
  - `Format Comments` (code)  
  - `Set Final Report` (set)

- **Node Details:**

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Processes posts in batches to manage API load and memory.  
    - Outputs to: Two paths—`Aggregate` and `Set post for Loop`.

  - **Set post for Loop**  
    - Type: Set  
    - Role: Prepares individual post data for comment retrieval.  
    - Outputs to: `Get Comments`.

  - **Get Comments**  
    - Type: Reddit node  
    - Role: Retrieves comments for the current post.  
    - Configuration: Uses post ID or permalink from `Set post for Loop`.  
    - Edge cases: Posts with no comments, deleted comments, API failures.  
    - Outputs to: `Extract Top 20 Comments`.

  - **Extract Top 20 Comments**  
    - Type: Code  
    - Role: Filters and extracts the top 20 comments by some criteria (e.g., upvotes).  
    - Configuration: Runs once per post.  
    - Outputs to: `Format Comments`.

  - **Format Comments**  
    - Type: Code  
    - Role: Converts comment data into Markdown format, suitable for reports.  
    - Outputs to: `Set Final Report`.

  - **Set Final Report**  
    - Type: Set  
    - Role: Aggregates formatted post and comment data into a final structure.  
    - Outputs to: `Loop Over Items1` (for next batch).

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects all processed batches into a single dataset for final output.  
    - Outputs to: `Combine Posts`.

#### 2.5 Aggregation & Response

- **Overview:** Combines all formatted posts and comments and responds to the webhook caller.
- **Nodes involved:**  
  - `Combine Posts` (code)  
  - `Respond to Webhook` (respondToWebhook)

- **Node Details:**

  - **Combine Posts**  
    - Type: Code  
    - Role: Merges all processed posts and comments into a coherent Markdown report.  
    - Outputs to: `Respond to Webhook`.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Returns the final report as the HTTP response to the webhook request.  
    - Configuration: Sends the combined Markdown report as the response body.  
    - Edge cases: Large payloads causing timeouts; malformed response data.

---

### 3. Summary Table

| Node Name                      | Node Type         | Functional Role                            | Input Node(s)                    | Output Node(s)                   | Sticky Note                            |
|--------------------------------|-------------------|-------------------------------------------|---------------------------------|---------------------------------|--------------------------------------|
| reddit tool webhook             | Webhook           | Receives keyword-based webhook request    | —                               | Set keywords                    |                                      |
| reddit tool webhook subreddit   | Webhook           | Receives subreddit-based webhook request  | —                               | Set subreddit                  |                                      |
| Set keywords                   | Set               | Sets keywords for Reddit search            | reddit tool webhook              | Search for posts by keywords    |                                      |
| Set subreddit                  | Set               | Sets subreddit for Reddit posts retrieval  | reddit tool webhook subreddit    | Get posts from subreddit        |                                      |
| Search for posts by keywords   | Reddit            | Searches Reddit posts by keywords          | Set keywords                    | Set Reddit Posts               |                                      |
| Get posts from subreddit       | Reddit            | Retrieves posts from a subreddit            | Set subreddit                   | Set Reddit Posts               |                                      |
| Set Reddit Posts              | Set               | Prepares Reddit posts for filtering          | Search for posts by keywords, Get posts from subreddit | Posted in Last x days          |                                      |
| Posted in Last x days          | Code              | Filters posts by recency                     | Set Reddit Posts                | Upvotes Requirement Filtering  |                                      |
| Upvotes Requirement Filtering | If                | Filters posts by minimum upvote count       | Posted in Last x days           | Remove Duplicates              |                                      |
| Remove Duplicates             | Code              | Removes duplicate posts                      | Upvotes Requirement Filtering  | Sort by upvotes (descending)  |                                      |
| Sort by upvotes (descending) | Sort              | Sorts posts by descending upvotes            | Remove Duplicates              | Limit                        |                                      |
| Limit                        | Limit             | Limits number of posts to process             | Sort by upvotes (descending)  | Loop Over Items1              |                                      |
| Loop Over Items1             | SplitInBatches    | Processes posts in batches                     | Limit                         | Aggregate, Set post for Loop   |                                      |
| Set post for Loop            | Set               | Prepares single post data for comment retrieval | Loop Over Items1              | Get Comments                  |                                      |
| Get Comments                 | Reddit            | Retrieves comments for a post                  | Set post for Loop             | Extract Top 20 Comments       |                                      |
| Extract Top 20 Comments      | Code              | Extracts top 20 comments per post               | Get Comments                  | Format Comments               |                                      |
| Format Comments             | Code              | Formats comments into Markdown                   | Extract Top 20 Comments       | Set Final Report              |                                      |
| Set Final Report            | Set               | Aggregates formatted post & comment data         | Format Comments               | Loop Over Items1              |                                      |
| Aggregate                  | Aggregate         | Aggregates all batches of processed posts         | Loop Over Items1              | Combine Posts                |                                      |
| Combine Posts              | Code              | Combines all posts and comments into final report | Aggregate                   | Respond to Webhook           |                                      |
| Respond to Webhook         | RespondToWebhook  | Sends final Markdown report as HTTP response      | Combine Posts                | —                           |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes:**  
   - Create `reddit tool webhook` (Webhook node) with a unique webhook URL to receive keyword search requests.  
   - Create `reddit tool webhook subreddit` (Webhook node) with a webhook URL to receive subreddit requests.  
   - Both webhooks should be configured with HTTP POST. No authentication is strictly necessary but recommended for security.

2. **Set Input Variables:**  
   - After `reddit tool webhook`, add a `Set` node named `Set keywords` to extract and define keywords from the webhook payload.  
   - After `reddit tool webhook subreddit`, add a `Set` node named `Set subreddit` to extract and define the subreddit parameter.

3. **Reddit Post Retrieval:**  
   - Add a `Reddit` node named `Search for posts by keywords`. Link it to `Set keywords`. Configure to search posts by keyword with appropriate filters (e.g., limit, sort).  
   - Add a `Reddit` node named `Get posts from subreddit`. Link it to `Set subreddit`. Configure to get posts from the specified subreddit.  
   - Both Reddit nodes require Reddit API credentials configured in n8n.

4. **Normalize Posts Data:**  
   - Add a `Set` node named `Set Reddit Posts` connected from both `Search for posts by keywords` and `Get posts from subreddit`. Configure it to extract and normalize necessary post fields (e.g., ID, title, upvotes, creation date).

5. **Filtering Posts:**  
   - Add a `Code` node named `Posted in Last x days`. Configure a JavaScript function that filters posts based on creation date within the last X days (X configurable).  
   - Add an `If` node named `Upvotes Requirement Filtering`, configure condition to check if post upvotes exceed a minimum threshold.  
   - Add a `Code` node named `Remove Duplicates` to remove duplicate posts by unique ID.  
   - Add a `Sort` node named `Sort by upvotes (descending)` to sort posts descending by upvotes.  
   - Add a `Limit` node named `Limit` to restrict the number of posts processed further.

6. **Process Posts in Batches:**  
   - Add a `SplitInBatches` node named `Loop Over Items1` connected from `Limit`. Configure batch size as needed (e.g., 5 or 10).  
   - Connect two outputs from `Loop Over Items1`: one to an `Aggregate` node and another to a `Set` node named `Set post for Loop`.

7. **Comments Extraction:**  
   - Connect `Set post for Loop` to a `Reddit` node named `Get Comments`. Configure to fetch comments for the current post using post ID or permalink.  
   - Add a `Code` node named `Extract Top 20 Comments` that extracts the top 20 comments based on criteria like upvotes or relevance.  
   - Add a `Code` node named `Format Comments` that converts comments into Markdown format.

8. **Aggregate and Finalize:**  
   - Add a `Set` node named `Set Final Report` to prepare the formatted data for aggregation.  
   - Connect back to `Loop Over Items1` to continue processing batches.  
   - Connect `Loop Over Items1` (aggregate output) to an `Aggregate` node to collect all batches.  
   - Add a `Code` node named `Combine Posts` to merge all aggregated data into one Markdown report.

9. **Respond to Webhook:**  
   - Add a `Respond to Webhook` node connected to `Combine Posts`. Configure it to send the final Markdown report as the HTTP response.

10. **Credentials Setup:**  
    - Configure Reddit API credentials in n8n for all Reddit nodes.  
    - Ensure webhook URLs are exposed and accessible.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                             |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| The workflow uses batch processing to handle Reddit API rate limits and large data volumes efficiently. | General workflow design practice.                            |
| Markdown formatting enables easy integration with reporting tools or messaging platforms that support Markdown. | Output formatting rationale.                                 |
| Reddit API credentials must have sufficient scope to read posts and comments.                         | Reddit API documentation: https://www.reddit.com/dev/api/   |
| Webhook nodes share the same webhook ID but handle different input parameters (keywords vs subreddit). | Ensure to differentiate requests or use separate webhook paths for clarity. |
| Edge cases include handling private subreddits, deleted comments, and API rate limits.                | Plan for retry or error handling in production.             |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.