Monitor Reddit keyword trends and email reports with Apify

https://n8nworkflows.xyz/workflows/monitor-reddit-keyword-trends-and-email-reports-with-apify-14298


# Monitor Reddit keyword trends and email reports with Apify

# 1. Workflow Overview

This workflow monitors Reddit keyword trends on a schedule, retrieves Reddit posts for each keyword using Apify, ranks the posts by a custom engagement score, emails a formatted report, and stores the ranked results in an n8n Data Table.

Primary use cases:
- Recurring monitoring of Reddit discussions for a list of keywords
- Generating email reports of top-performing Reddit content
- Persisting ranked trend snapshots for later analysis

Logical blocks:

## 1.1 Scheduled Start and Query Retrieval
The workflow starts on a schedule and reads all keyword rows from an n8n Data Table. Each row is expected to contain a `Query` field used as the Reddit search term.

## 1.2 Query Looping and Reddit Scraping
Each query is processed in a loop. For every keyword, the workflow sends a POST request to an Apify Reddit scraper actor and retrieves raw Reddit posts.

## 1.3 Reddit Scoring and Ranking
Two Code nodes transform the raw Apify output:
1. compute a custom Reddit engagement score
2. normalize results into a reporting format with rank, source, author, score, level, and URL

## 1.4 Reporting and Storage
The ranked records are sent to two destinations:
- a Gmail HTML email report
- a Data Table for persistence

## 1.5 Loop Continuation
After the email is sent, the loop returns to the batch node so the next keyword can be processed.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Start and Query Retrieval

### Overview
This block launches the workflow automatically and loads the list of Reddit queries to monitor. It provides the input dataset that drives all later processing.

### Nodes Involved
- Reddit Schedule Trigger
- Get Reddit Query Rows

### Node Details

#### Reddit Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow on a recurring schedule.
- **Configuration choices:**  
  Uses an interval-based rule. The JSON shows a minimal interval configuration, so the actual cadence should be reviewed in the editor.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input; outputs to **Get Reddit Query Rows**.
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Misconfigured schedule frequency
  - Workflow not activated
  - Unexpected trigger intervals if the blank interval object is left as-is
- **Sub-workflow reference:**  
  None.

#### Get Reddit Query Rows
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Reads rows from an n8n Data Table containing Reddit search terms.
- **Configuration choices:**  
  - Operation: `get`
  - Return all rows: enabled
  - Data table: `Query social media`
- **Key expressions or variables used:**  
  No expressions in the node itself. Downstream logic expects a field named `Query`.
- **Input and output connections:**  
  Input from **Reddit Schedule Trigger**; output to **Loop Over Reddit Queries**.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Missing or inaccessible Data Table
  - Empty table returns no work items
  - Rows missing the `Query` field will break downstream expressions or produce empty searches
- **Sub-workflow reference:**  
  None.

---

## 2.2 Query Looping and Reddit Scraping

### Overview
This block iterates over each query row and requests Reddit posts from Apify. It is the acquisition layer of the workflow.

### Nodes Involved
- Loop Over Reddit Queries
- Fetch Reddit Posts via Apify1

### Node Details

#### Loop Over Reddit Queries
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates over the query rows one batch at a time.
- **Configuration choices:**  
  Default options are used; no custom batch size is shown.
- **Key expressions or variables used:**  
  Downstream nodes use the current item’s `Query` field.
- **Input and output connections:**  
  Input from **Get Reddit Query Rows**.  
  Batch output goes to **Fetch Reddit Posts via Apify1**.  
  The loop continuation input is fed back from **Send Reddit Report Email**.
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - Empty input results in no downstream processing
  - If the loop-back connection is broken, only the first batch may run
  - If downstream nodes fail, remaining queries will not process unless error handling is added
- **Sub-workflow reference:**  
  None.

#### Fetch Reddit Posts via Apify1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the Apify Reddit Scraper Lite actor and retrieves dataset items synchronously.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.apify.com/v2/acts/trudax~reddit-scraper-lite/run-sync-get-dataset-items`
  - Body sent as JSON
  - Authentication: predefined credential type using `apifyApi`
  - Header: `Content-Type: application/json`
  - Important request body settings:
    - `searches`: array containing `{{ $json.Query }}`
    - `searchPosts`: `true`
    - `searchComments`: `false`
    - `searchCommunities`: `false`
    - `searchUsers`: `false`
    - `sort`: `top`
    - `time`: `month`
    - `maxItems`: `50`
    - `maxPostCount`: `50`
    - `maxComments`: `1`
    - `scrollTimeout`: `40`
    - Apify residential proxy enabled
- **Key expressions or variables used:**  
  - `{{ $json.Query }}` inside the JSON body
- **Input and output connections:**  
  Input from **Loop Over Reddit Queries**; output to **Sort Reddit Posts by Score**.
- **Version-specific requirements:**  
  Type version `4.2`.  
  Requires a valid Apify credential configured in n8n.
- **Edge cases or potential failure types:**  
  - Invalid Apify credentials
  - Actor endpoint changes or actor deprecation
  - Rate limiting or quota exhaustion on Apify
  - Empty results for niche keywords
  - Timeout due to actor runtime
  - Expression failure if `Query` is missing
  - Response shape changes could break downstream code expectations
- **Sub-workflow reference:**  
  None.

---

## 2.3 Reddit Scoring and Ranking

### Overview
This block converts raw Apify Reddit records into ranked outputs. The first Code node calculates a custom score from upvotes and comment count, while the second reformats and classifies posts for reporting and storage.

### Nodes Involved
- Sort Reddit Posts by Score
- Rank Reddit Posts1

### Node Details

#### Sort Reddit Posts by Score
- **Type and technical role:** `n8n-nodes-base.code`  
  Filters raw Reddit items, computes a score, sorts posts, and returns the top 10 with simplified fields.
- **Configuration choices:**  
  Uses JavaScript code in the Code node.
- **Key expressions or variables used:**  
  - `$input.all()` to retrieve all incoming items
  - Expected Apify fields:
    - `dataType`
    - `upVotes`
    - `numberOfComments`
    - `url`
    - `title`
    - `body`
    - `communityName`
    - `createdAt`
- **Processing logic:**  
  - Keeps only items where `dataType === 'post'`
  - Computes `score = upVotes * 1 + numberOfComments * 2`
  - Sorts descending by score
  - Keeps only top 10
  - Outputs:
    - `rank`
    - `url`
    - `title`
    - `body`
    - `score`
- **Input and output connections:**  
  Input from **Fetch Reddit Posts via Apify1**; output to **Rank Reddit Posts1**.
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Missing fields from Apify response
  - Unexpected `dataType` values leading to zero posts
  - Large body/title values are allowed but may affect logging or later email rendering
  - Console logs are useful for debug but not persisted externally
- **Sub-workflow reference:**  
  None.

#### Rank Reddit Posts1
- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes Reddit items into a reporting schema and applies a score-level classification.
- **Configuration choices:**  
  Uses JavaScript code with explicit references to prior nodes.
- **Key expressions or variables used:**  
  - `$('Loop Over Reddit Queries').first().json.Query`
  - `$('Sort Reddit Posts by Score').all()`
- **Processing logic:**  
  - Reads the current keyword from the loop node
  - Reads scored Reddit posts from the previous code node
  - Builds normalized items with:
    - `source = 'Reddit'`
    - `keyword`
    - `author` extracted from `/r/...` in the URL
    - `content` = truncated title
    - `score`
    - `level` based on thresholds:
      - `High` if `score >= 10000`
      - `Medium` if `score >= 1000`
      - otherwise `Low`
    - `url`
    - `originalRank`
  - Re-sorts descending by score
  - Adds final `rank`
  - Adds `date`
  - Keeps top 15, though practically the previous node only provides 10
- **Input and output connections:**  
  Input from **Sort Reddit Posts by Score**; outputs to:
  - **Send Reddit Report Email**
  - **Push Results to Data Table**
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - `.first()` on the loop node may become ambiguous in some execution contexts; `$json.Query` from the current branch would be safer
  - If URL does not match `/r/<subreddit>/`, author becomes `user_inconnu`
  - Title truncation always appends `...`, even on short titles
  - Thresholds may classify almost all posts as `Low` if scores are modest
  - Top 15 request is redundant because only top 10 enter from previous node
- **Sub-workflow reference:**  
  None.

---

## 2.4 Reporting and Storage

### Overview
This block sends the ranked results via Gmail and writes them into a Data Table. It is the workflow’s delivery and persistence layer.

### Nodes Involved
- Send Reddit Report Email
- Push Results to Data Table

### Node Details

#### Send Reddit Report Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an HTML email report summarizing the ranked trend results.
- **Configuration choices:**  
  - Recipient: `user@example.com`
  - Subject: `Social media monitoring`
  - Execute once: enabled
  - Uses Gmail OAuth2 credentials named `Allan Growth AI`
  - HTML body contains a styled dashboard
- **Key expressions or variables used:**  
  - `$('Loop Over Reddit Queries').first().json.Query`
  - `$('Rank Reddit Posts1').all()...`
  - `new Date().toLocaleDateString('fr-FR', ...)`
- **Important observation:**  
  The email template is not aligned with the actual workflow output. It references TikTok and Instagram totals even though this workflow only generates Reddit items. It also assumes fields from `Rank Reddit Posts1` exist for multi-source reporting.
- **Input and output connections:**  
  Input from **Rank Reddit Posts1**; output back to **Loop Over Reddit Queries** to continue iteration.
- **Version-specific requirements:**  
  Type version `2.1`.  
  Requires Gmail OAuth2 credentials with send permission.
- **Edge cases or potential failure types:**  
  - Invalid/expired Gmail OAuth token
  - Recipient address left as placeholder
  - HTML expressions referencing unavailable nodes or fields if the node is renamed
  - Email may aggregate all items once because `executeOnce` is enabled
  - Since the template references only `Rank Reddit Posts1`, it works structurally, but non-Reddit source totals will remain zero
  - `.first()` may not always reflect the current loop item in all scenarios
- **Sub-workflow reference:**  
  None.

#### Push Results to Data Table
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Inserts ranked result items into a Data Table for history/storage.
- **Configuration choices:**  
  - Target table: `Trend Social media`
  - Mapping mode: define below
  - Fields mapped:
    - `url = $json.url`
    - `Date = $now.toISO()`
    - `rank = $json.rank`
    - `level = $json.level`
    - `score = $json.score`
    - `author = $json.author`
    - `source = $json.source`
    - `content = $json.content`
    - `keyword = $json.keyword`
- **Key expressions or variables used:**  
  - `{{ $json... }}`
  - `{{ $now.toISO() }}`
- **Input and output connections:**  
  Input from **Rank Reddit Posts1**; no downstream node.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Missing Data Table permissions/access
  - Schema mismatch if table columns differ
  - Type mismatch if columns expect strict types
  - Duplicate runs create duplicate historical entries because no deduplication/upsert logic exists
- **Sub-workflow reference:**  
  None.

---

## 2.5 Documentation / In-Canvas Notes

### Overview
These nodes are non-executable sticky notes embedded in the workflow canvas. They provide setup, customization, and block descriptions.

### Nodes Involved
- Sticky Note13
- Sticky Note24
- Sticky Note25
- Sticky Note26
- Sticky Note27

### Node Details

#### Sticky Note13
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  General workflow documentation.
- **Configuration choices:**  
  Large note covering the overall workflow.
- **Key content:**  
  Describes process, setup steps, and customization options.
- **Connections:**  
  None.
- **Edge cases:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note24
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Describes trigger and query retrieval block.
- **Connections:**  
  None.

#### Sticky Note25
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Describes loop and scraping block.
- **Connections:**  
  None.

#### Sticky Note26
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Describes ranking block.
- **Connections:**  
  None.

#### Sticky Note27
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Describes email and storage block.
- **Connections:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Reddit Schedule Trigger | Schedule Trigger | Starts the workflow on a recurring schedule |  | Get Reddit Query Rows | ## Trigger and fetch queries<br>A schedule trigger fires the workflow and retrieves the list of Reddit search queries from a data table to process. |
| Get Reddit Query Rows | Data Table | Retrieves all query rows from the source Data Table | Reddit Schedule Trigger | Loop Over Reddit Queries | ## Trigger and fetch queries<br>A schedule trigger fires the workflow and retrieves the list of Reddit search queries from a data table to process. |
| Loop Over Reddit Queries | Split In Batches | Iterates through Reddit query rows one by one | Get Reddit Query Rows; Send Reddit Report Email | Fetch Reddit Posts via Apify1 | ## Loop and scrape Reddit posts<br>Iterates over each query in a batch loop and calls the Apify Reddit scraper API to fetch raw posts for that query. |
| Fetch Reddit Posts via Apify1 | HTTP Request | Calls the Apify Reddit scraper actor for the current query | Loop Over Reddit Queries | Sort Reddit Posts by Score | ## Loop and scrape Reddit posts<br>Iterates over each query in a batch loop and calls the Apify Reddit scraper API to fetch raw posts for that query. |
| Sort Reddit Posts by Score | Code | Filters Reddit posts, computes score, sorts, returns top 10 | Fetch Reddit Posts via Apify1 | Rank Reddit Posts1 | ## Sort and rank results<br>Two code nodes process the raw scraped data — first sorting posts into a simplified structure, then applying a full ranking algorithm. |
| Rank Reddit Posts1 | Code | Normalizes and classifies Reddit posts for reporting/storage | Sort Reddit Posts by Score | Send Reddit Report Email; Push Results to Data Table | ## Sort and rank results<br>Two code nodes process the raw scraped data — first sorting posts into a simplified structure, then applying a full ranking algorithm. |
| Send Reddit Report Email | Gmail | Sends the HTML email report | Rank Reddit Posts1 | Loop Over Reddit Queries | ## Send email and store results<br>Dispatches the ranked Reddit results via Gmail and pushes the data into a storage table for historical tracking. |
| Push Results to Data Table | Data Table | Stores ranked results in the output Data Table | Rank Reddit Posts1 |  | ## Send email and store results<br>Dispatches the ranked Reddit results via Gmail and pushes the data into a storage table for historical tracking. |
| Sticky Note13 | Sticky Note | General in-canvas workflow documentation |  |  | ## Trend monitoring Reddit<br><br>### How it works<br><br>1. A schedule trigger fires periodically and retrieves a list of Reddit search queries from a data table.<br>2. The workflow loops over each query and sends it to the Apify Reddit scraper API to fetch posts.<br>3. Two code nodes sort and rank the scraped Reddit posts based on engagement or relevance.<br>4. The ranked results are sent as a Gmail email notification and simultaneously stored back into a data table for record-keeping.<br><br>### Setup steps<br><br>- - [ ] Configure the **Reddit Schedule Trigger1** with the desired run frequency (e.g. daily, hourly).<br>- - [ ] Populate the **Reddit Get row(s)1** data table with the Reddit search queries you want to monitor.<br>- - [ ] Add your **Apify API key** to the `Reddit solo get post` HTTP Request node (URL contains the Apify actor endpoint).<br>- - [ ] Connect your **Gmail account** credentials to the `Reddit Send a message1` node and set the recipient email address.<br>- - [ ] Configure the **Reddot Push to data table1** node to point to the correct data table for storing results.<br><br>### Customization<br><br>You can adjust the ranking/sorting logic inside the two code nodes (`Sort Reddit solo` and `Classement Reddit1`) to weight posts by upvotes, comments, recency, or other criteria. The Apify actor parameters in `Reddit solo get post` can be tuned to change the number of posts fetched per query or subreddit scope. |
| Sticky Note24 | Sticky Note | In-canvas note for trigger/query block |  |  |  |
| Sticky Note25 | Sticky Note | In-canvas note for scrape block |  |  |  |
| Sticky Note26 | Sticky Note | In-canvas note for ranking block |  |  |  |
| Sticky Note27 | Sticky Note | In-canvas note for delivery/storage block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Monitor Reddit keyword trends and email reports with Apify`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Reddit Schedule Trigger`
   - Configure the execution frequency you want, such as:
     - every hour
     - every day
     - every week
   - Activate the workflow only after all credentials and tables are configured.

3. **Create the source Data Table for queries**
   - In n8n, create a Data Table named something like `Query social media`
   - Add at least one column:
     - `Query` as string/text
   - Populate it with example values such as:
     - `n8n`
     - `openai`
     - `langchain`

4. **Add a Data Table node to read the queries**
   - Node type: **Data Table**
   - Name: `Get Reddit Query Rows`
   - Operation: `Get`
   - Enable `Return All`
   - Select the source Data Table created above
   - Connect `Reddit Schedule Trigger -> Get Reddit Query Rows`

5. **Add a Split In Batches node**
   - Node type: **Split In Batches**
   - Name: `Loop Over Reddit Queries`
   - Leave default options unless you want a custom batch size
   - Connect `Get Reddit Query Rows -> Loop Over Reddit Queries`

6. **Create Apify credentials**
   - In n8n credentials, create an **Apify API** credential
   - Use your Apify token
   - Ensure the token has access to run the actor endpoint

7. **Add an HTTP Request node for Reddit scraping**
   - Node type: **HTTP Request**
   - Name: `Fetch Reddit Posts via Apify1`
   - Method: `POST`
   - URL:  
     `https://api.apify.com/v2/acts/trudax~reddit-scraper-lite/run-sync-get-dataset-items`
   - Authentication: **Predefined Credential Type**
   - Credential type: **Apify API**
   - Add header:
     - `Content-Type: application/json`
   - Enable body sending
   - Body type: **JSON**
   - JSON body:
     ```json
     {
       "searches": ["{{ $json.Query }}"],
       "searchPosts": true,
       "searchComments": false,
       "searchCommunities": false,
       "searchUsers": false,
       "sort": "top",
       "time": "month",
       "maxItems": 50,
       "maxPostCount": 50,
       "maxComments": 1,
       "scrollTimeout": 40,
       "proxy": {
         "useApifyProxy": true,
         "apifyProxyGroups": ["RESIDENTIAL"]
       }
     }
     ```
   - Connect `Loop Over Reddit Queries -> Fetch Reddit Posts via Apify1`

8. **Add the first Code node to score Reddit posts**
   - Node type: **Code**
   - Name: `Sort Reddit Posts by Score`
   - Language: JavaScript
   - Paste logic equivalent to:
     - read all input items
     - keep only `dataType === 'post'`
     - compute `score = upVotes + (numberOfComments * 2)`
     - sort descending by score
     - keep top 10
     - output fields:
       - `rank`
       - `url`
       - `title`
       - `body`
       - `score`
   - Connect `Fetch Reddit Posts via Apify1 -> Sort Reddit Posts by Score`

9. **Use this scoring behavior in the first Code node**
   - Default values:
     - missing `upVotes` => `0`
     - missing `numberOfComments` => `0`
     - missing `title` => fallback string
     - missing `body` => fallback string
   - Ensure the node returns one item per ranked post

10. **Add the second Code node to normalize results**
    - Node type: **Code**
    - Name: `Rank Reddit Posts1`
    - Language: JavaScript
    - Configure logic to:
      - read the current keyword from the loop context
      - read all items from `Sort Reddit Posts by Score`
      - map each item to:
        - `source = 'Reddit'`
        - `keyword`
        - `author`
        - `content`
        - `score`
        - `level`
        - `url`
      - classify `level`:
        - `High` if score >= 10000
        - `Medium` if score >= 1000
        - `Low` otherwise
      - extract `author` from the URL path `/r/<subreddit>/`
      - sort descending by score
      - assign final `rank`
      - optionally add current date
      - keep top 15
    - Connect `Sort Reddit Posts by Score -> Rank Reddit Posts1`

11. **Recommended improvement while rebuilding**
    - Instead of using `$('Loop Over Reddit Queries').first().json.Query`, use the current item context where possible to avoid ambiguity in loops.
    - For example, preserve the keyword through the stream or merge it before ranking.

12. **Create the destination Data Table**
    - Create another Data Table named something like `Trend Social media`
    - Add columns:
      - `rank` number
      - `keyword` string
      - `source` string
      - `author` string
      - `content` string
      - `score` number
      - `level` string
      - `url` string
      - `Date` dateTime

13. **Add a Data Table node to store results**
    - Node type: **Data Table**
    - Name: `Push Results to Data Table`
    - Choose the destination Data Table
    - Set mapping mode to manual/define-below
    - Map:
      - `url` = `{{ $json.url }}`
      - `Date` = `{{ $now.toISO() }}`
      - `rank` = `{{ $json.rank }}`
      - `level` = `{{ $json.level }}`
      - `score` = `{{ $json.score }}`
      - `author` = `{{ $json.author }}`
      - `source` = `{{ $json.source }}`
      - `content` = `{{ $json.content }}`
      - `keyword` = `{{ $json.keyword }}`
    - Connect `Rank Reddit Posts1 -> Push Results to Data Table`

14. **Create Gmail OAuth2 credentials**
    - In n8n credentials, create **Gmail OAuth2**
    - Authenticate the Google account allowed to send messages
    - Confirm the scopes support sending email

15. **Add a Gmail node**
    - Node type: **Gmail**
    - Name: `Send Reddit Report Email`
    - Operation: send email
    - Set recipient address
    - Set subject, for example:
      `Social media monitoring`
    - Enable HTML message body
    - Paste your HTML report template
    - The current workflow uses expressions referencing:
      - `$('Loop Over Reddit Queries').first().json.Query`
      - `$('Rank Reddit Posts1').all()`
    - Connect `Rank Reddit Posts1 -> Send Reddit Report Email`

16. **Be careful with the email template**
    - The provided template includes TikTok and Instagram sections, but this workflow only produces Reddit data.
    - If rebuilding faithfully, keep it as-is.
    - If rebuilding cleanly, simplify the template so it only shows Reddit totals and Reddit rows.

17. **Enable execute-once behavior if desired**
    - In the Gmail node, enable `Execute Once` if you want one email per query batch result rather than one per item.
    - Validate how n8n behaves with your version and item flow.

18. **Close the loop**
    - Connect `Send Reddit Report Email -> Loop Over Reddit Queries`
    - This connection is essential for iterating through all query rows.

19. **Optionally add sticky notes**
    - Add sticky notes to describe:
      - trigger/query retrieval
      - looping/scraping
      - ranking
      - sending/storage
    - These do not affect execution but improve maintainability.

20. **Test with one query first**
    - Add a single row in the source Data Table
    - Run the workflow manually
    - Verify:
      - Apify returns Reddit posts
      - scoring node outputs ranked items
      - normalization node outputs expected fields
      - Data Table receives rows
      - Gmail email renders correctly

21. **Test with multiple queries**
    - Add several query rows
    - Run again and confirm the loop progresses through each query
    - Check that each query produces a separate report and stored records

22. **Activate the workflow**
    - Once all tests pass, activate the workflow so the Schedule Trigger can run automatically.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Trend monitoring Reddit — A schedule trigger retrieves query rows, loops through them, calls Apify for Reddit posts, ranks the posts in two Code nodes, emails the results, and stores them in a Data Table. | General workflow note from canvas |
| Setup reminders: configure the schedule, populate the query Data Table, add Apify API credentials to the HTTP Request node, connect Gmail credentials, and point the storage node to the correct Data Table. | General workflow note from canvas |
| Customization note: adjust ranking logic in the Code nodes to weight upvotes, comments, recency, or other factors; tune the Apify actor parameters to change result volume or scraping scope. | General workflow note from canvas |

## Additional implementation notes
- The current workflow has **one real entry point**: `Reddit Schedule Trigger`.
- There are **no sub-workflows** or workflow-call nodes.
- The current email template appears to be adapted from a broader social media dashboard and is only partially aligned with the actual Reddit-only data produced here.
- The loop relies on the Gmail node to continue batch execution. If the email step fails, the workflow may stop before processing remaining queries.
- The second ranking node asks for top 15 results, but the prior node only emits top 10, so the effective maximum is 10.