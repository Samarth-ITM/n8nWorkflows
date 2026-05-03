Monitor Instagram hashtag trends and email reports with Apify and Gmail

https://n8nworkflows.xyz/workflows/monitor-instagram-hashtag-trends-and-email-reports-with-apify-and-gmail-14302


# Monitor Instagram hashtag trends and email reports with Apify and Gmail

# 1. Workflow Overview

This workflow monitors Instagram hashtag performance on a schedule, retrieves recent Instagram posts for each configured query via Apify, ranks the posts using a custom engagement formula, stores the ranked results in an n8n Data Table, and emails a formatted HTML report through Gmail.

Its main use case is recurring social media monitoring for one or more hashtags stored in a table. The workflow is structured around iterative processing: each query is handled one at a time, then the loop proceeds to the next query after storage and email delivery.

## 1.1 Scheduled Input Reception

The workflow starts on a recurring schedule and loads all search queries from a Data Table.

## 1.2 Query Iteration and Instagram Data Retrieval

Each query is processed sequentially through a batch loop. For each query, an HTTP request is sent to the Apify Instagram scraping actor to retrieve hashtag-based post data.

## 1.3 Instagram Post Filtering and Scoring

The raw Apify dataset is filtered to keep only main Instagram post URLs, then engagement scores are calculated using likes and comments.

## 1.4 Result Ranking and Output Preparation

The ranked Instagram posts are normalized into a reporting format with source, keyword, author, score, content preview, and qualitative level.

## 1.5 Persistence and Notification

The final ranked results are written to a Data Table and also sent as an HTML email through Gmail. After the email finishes, the loop continues with the next query.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

### Overview
This block triggers the workflow automatically and fetches all configured hashtag queries from an n8n Data Table. It is the single entry point of the workflow.

### Nodes Involved
- Scheduled Run Trigger
- Read Queries from Table

### Node Details

#### Scheduled Run Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow on a time-based schedule.
- **Configuration choices:**  
  The trigger uses an interval rule with default interval settings shown in the JSON. This means the exact schedule should be verified in the n8n UI, because the exported structure does not clearly indicate a custom cadence beyond “interval”.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input
  - Outputs to **Read Queries from Table**
- **Version-specific requirements:**  
  Type version `1.2`
- **Edge cases or potential failure types:**  
  - Misconfigured schedule may cause unexpected frequency
  - Workflow may appear inactive if not activated
- **Sub-workflow reference:**  
  None

#### Read Queries from Table
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Reads all rows from an n8n Data Table containing the hashtag/query list.
- **Configuration choices:**  
  - Operation: `get`
  - Return all rows: enabled
  - Data Table: `Query social media`
- **Key expressions or variables used:**  
  None in parameters; downstream nodes expect a field named `Query`.
- **Input and output connections:**  
  - Input from **Scheduled Run Trigger**
  - Output to **Loop Over Queries1**
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Data Table missing or inaccessible
  - Returned rows may not contain the expected `Query` column
  - Empty table causes the loop to process nothing
- **Sub-workflow reference:**  
  None

---

## 2.2 Query Iteration and Instagram Data Retrieval

### Overview
This block iterates over each query one by one and sends the current query to Apify’s Instagram scraping actor. The actor returns recent hashtag-related Instagram post data.

### Nodes Involved
- Loop Over Queries1
- Fetch Instagram Posts via Apify1

### Node Details

#### Loop Over Queries1
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates over incoming query rows in batches; here it is effectively used as a one-by-one loop controller.
- **Configuration choices:**  
  No custom batch options are defined. It relies on the standard loop behavior of Split in Batches.
- **Key expressions or variables used:**  
  Downstream nodes refer to the current item’s `Query` field.
- **Input and output connections:**  
  - Input from **Read Queries from Table**
  - Loop branch output to **Fetch Instagram Posts via Apify1**
  - Continue-loop input receives control back from **Send Instagram Report Email**
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - If rows do not contain `Query`, the fetch node will build invalid requests
  - If the loop is not properly closed, only the first batch would run; in this workflow it is closed through the email node
- **Sub-workflow reference:**  
  None

#### Fetch Instagram Posts via Apify1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Apify actor endpoint to run the Instagram scraper and immediately retrieve dataset items.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.apify.com/v2/acts/reGe1ST3OBgYZSsZJ/run-sync-get-dataset-items`
  - Authentication: predefined credential type using `apifyApi`
  - Content-Type header: `application/json`
  - JSON body is dynamically built from the current query
  - Retry on fail: enabled
  - Execute once: disabled
- **Key expressions or variables used:**  
  - Hashtag expression:
    `{{ $json.Query.replace(/\s+/g, '') }}`
  - Request body:
    - `hashtags`: array containing sanitized query
    - `keywordSearch`: `false`
    - `resultsLimit`: `50`
    - `resultsType`: `"posts"`
- **Input and output connections:**  
  - Input from **Loop Over Queries1**
  - Output to **Sort Instagram Results**
- **Version-specific requirements:**  
  Type version `4.2`
- **Edge cases or potential failure types:**  
  - Invalid/missing Apify credentials
  - Apify actor unavailable or rate-limited
  - Query contains unsupported characters
  - API timeout or slow actor execution
  - Returned payload shape may vary over time
- **Sub-workflow reference:**  
  None

---

## 2.3 Instagram Post Filtering and Scoring

### Overview
This block takes the raw Apify result set, excludes unwanted entries such as carousel items, computes a weighted engagement score, sorts posts by score, and limits the output to the top 10.

### Nodes Involved
- Sort Instagram Results

### Node Details

#### Sort Instagram Results
- **Type and technical role:** `n8n-nodes-base.code`  
  Processes all fetched Instagram items in JavaScript and returns a simplified ranked structure.
- **Configuration choices:**  
  The code:
  - Reads all input items with `$input.all()`
  - Keeps only posts where:
    - `productType !== 'carousel_item'`
    - `url` exists
    - `url` contains `/p/`
  - Converts hidden like counts (`-1` or invalid values) to `0`
  - Calculates score as:
    `score = likes + (comments * 2)`
  - Sorts descending by score
  - Keeps top 10
  - Adds `rank`
- **Key expressions or variables used:**  
  Internal code variables:
  - `items`
  - `posts`
  - `likes`
  - `comments`
  - `totalScore`
  - `rankedPosts`
- **Input and output connections:**  
  - Input from **Fetch Instagram Posts via Apify1**
  - Output to **Rank Instagram Posts1**
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - Unexpected Apify schema may omit `likesCount`, `commentsCount`, `caption`, or `url`
  - Filtering may discard all records if URL patterns differ
  - Console logging is useful for debugging but not part of returned data
- **Sub-workflow reference:**  
  None

**Important implementation note:**  
This node is functionally valid, but the next node does not reference it by the correct name. The downstream ranking node expects a node named `Sort Instagram`, while the actual node name is `Sort Instagram Results`.

---

## 2.4 Result Ranking and Output Preparation

### Overview
This block converts the simplified Instagram results into the final reporting schema used for storage and email. It adds the source, keyword, author, content preview, qualitative level, and final rank.

### Nodes Involved
- Rank Instagram Posts1

### Node Details

#### Rank Instagram Posts1
- **Type and technical role:** `n8n-nodes-base.code`  
  Transforms scored Instagram results into the final reporting format and limits the output to the top 15.
- **Configuration choices:**  
  The code intends to:
  - Read the current keyword from the loop node
  - Read all posts from the previous sorting node
  - Assign `source = 'Instagram'`
  - Derive author from the Instagram URL
  - Create a shortened content preview
  - Compute a qualitative level:
    - `High` if score >= 10000
    - `Medium` if score >= 1000
    - `Low` otherwise
  - Re-sort by score descending
  - Add final `rank`
  - Add `date`
  - Return the top 15
- **Key expressions or variables used:**  
  In code:
  - `$('Loop Over Query').first().json.Query`
  - `$('Sort Instagram').all()`
  - `getLevel(score)`
  - `getAuthor(item)`
- **Input and output connections:**  
  - Connected input from **Sort Instagram Results**
  - Outputs to:
    - **Send Instagram Report Email**
    - **Save Results to Data Table**
- **Version-specific requirements:**  
  Type version `2`
- **Edge cases or potential failure types:**  
  - **Critical expression/name mismatch:**  
    The code references:
    - `$('Loop Over Query')` but actual node name is **Loop Over Queries1**
    - `$('Sort Instagram')` but actual node name is **Sort Instagram Results**
    This will likely fail at runtime unless the node names are corrected.
  - Author extraction uses URL regex `instagram\.com\/([^\/]+)` which may return `@p` for post URLs like `/p/...` rather than the account username. This logic is not reliable for Instagram post URLs.
  - If captions are missing, fallback text is used.
  - If fewer than 15 posts are available, it returns fewer items without error.
- **Sub-workflow reference:**  
  None

**Important implementation note:**  
This node is the workflow’s main structural weakness. As exported, it is likely broken due to incorrect node references inside the JavaScript code.

---

## 2.5 Persistence and Notification

### Overview
This block writes the final ranked results into a Data Table and sends a styled HTML report by email. The email node also serves as the control point that routes execution back into the query loop.

### Nodes Involved
- Save Results to Data Table
- Send Instagram Report Email

### Node Details

#### Save Results to Data Table
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Persists ranked result rows into an n8n Data Table for historical tracking or dashboard use.
- **Configuration choices:**  
  - Writes mapped columns into Data Table `Trend Social media`
  - Fields stored:
    - `url`
    - `Date`
    - `rank`
    - `level`
    - `score`
    - `author`
    - `source`
    - `content`
    - `keyword`
  - `Date` is set using `{{$now.toISO()}}`
- **Key expressions or variables used:**  
  - `={{ $json.url }}`
  - `={{ $now.toISO() }}`
  - `={{ $json.rank }}`
  - `={{ $json.level }}`
  - `={{ $json.score }}`
  - `={{ $json.author }}`
  - `={{ $json.source }}`
  - `={{ $json.content }}`
  - `={{ $json.keyword }}`
- **Input and output connections:**  
  - Input from **Rank Instagram Posts1**
  - No downstream connection
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Data type mismatch if fields are not in expected formats
  - Data Table schema changes can break insert mapping
  - Missing values from upstream ranking node
- **Sub-workflow reference:**  
  None

#### Send Instagram Report Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a formatted HTML email report using Gmail OAuth2.
- **Configuration choices:**  
  - Recipient: `user@example.com`
  - Subject: `Social media monitoring`
  - Body: large HTML template with CSS styling
  - Execute once: enabled
  - Gmail OAuth2 credential: `Allan Growth AI`
- **Key expressions or variables used:**  
  The HTML template references:
  - `$('Loop Over Queries1').first().json.Query`
  - `$('Rank Instagram Posts1').all()...`
  It calculates:
  - Total scores by source
  - A row list for ranked posts
  - Generation date with `new Date().toLocaleDateString('fr-FR', ...)`
- **Input and output connections:**  
  - Input from **Rank Instagram Posts1**
  - Output back to **Loop Over Queries1** to continue iteration
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - Gmail OAuth2 authentication errors
  - Recipient misconfiguration
  - HTML rendering differences across email clients
  - If **Rank Instagram Posts1** fails or returns no items, the email content may be incomplete
  - `executeOnce: true` means only one email execution per workflow execution branch; behavior should be verified in context of looping
- **Sub-workflow reference:**  
  None

**Important implementation note:**  
The email template expects `Rank Instagram Posts1` items to contain multiple sources (`TikTok`, `Reddit`, `Instagram`), but this workflow only produces `Instagram`. As a result:
- TikTok total will always be `0`
- Reddit total will always be `0`
- Only Instagram totals will contain values

This is not fatal, but it indicates the email template was likely reused from a broader cross-platform workflow.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scheduled Run Trigger | Schedule Trigger | Starts the workflow on a recurring interval |  | Read Queries from Table | ## Schedule trigger and query fetch\n\nA scheduled trigger fires the workflow and retrieves the list of Instagram search queries stored in a data table. |
| Read Queries from Table | Data Table | Loads all configured queries from the source table | Scheduled Run Trigger | Loop Over Queries1 | ## Schedule trigger and query fetch\n\nA scheduled trigger fires the workflow and retrieves the list of Instagram search queries stored in a data table. |
| Loop Over Queries1 | Split In Batches | Iterates over queries one at a time and controls loop progression | Read Queries from Table; Send Instagram Report Email | Fetch Instagram Posts via Apify1 | ## Loop and fetch Instagram posts\n\nIterates over each query in batches, calls the Apify Instagram API for each one, and collects the raw post data. |
| Fetch Instagram Posts via Apify1 | HTTP Request | Calls Apify to scrape Instagram hashtag posts | Loop Over Queries1 | Sort Instagram Results | ## Loop and fetch Instagram posts\n\nIterates over each query in batches, calls the Apify Instagram API for each one, and collects the raw post data. |
| Sort Instagram Results | Code | Filters and scores Instagram posts, returns top 10 | Fetch Instagram Posts via Apify1 | Rank Instagram Posts1 | ## Loop and fetch Instagram posts\n\nIterates over each query in batches, calls the Apify Instagram API for each one, and collects the raw post data. |
| Rank Instagram Posts1 | Code | Converts scored posts into final reporting schema | Sort Instagram Results | Send Instagram Report Email; Save Results to Data Table | ## Rank results and send output\n\nRanks the sorted Instagram posts using custom code, then simultaneously saves the results to a data table and sends a Gmail notification before looping back to the next query. |
| Save Results to Data Table | Data Table | Stores final ranked results for later analysis | Rank Instagram Posts1 |  | ## Rank results and send output\n\nRanks the sorted Instagram posts using custom code, then simultaneously saves the results to a data table and sends a Gmail notification before looping back to the next query. |
| Send Instagram Report Email | Gmail | Sends a styled HTML email report and returns control to the loop | Rank Instagram Posts1 | Loop Over Queries1 | ## Rank results and send output\n\nRanks the sorted Instagram posts using custom code, then simultaneously saves the results to a data table and sends a Gmail notification before looping back to the next query. |
| Sticky Note5 | Sticky Note | Documentation block on the canvas |  |  | ## Untitled workflow\n\n### How it works\n\n1. A scheduled trigger fires periodically and fetches a list of search queries from a data table.\n2. The workflow loops over each query one at a time using a batch iterator.\n3. For each query, Instagram posts are fetched via the Apify API.\n4. The raw post data is sorted and then ranked using custom code.\n5. The ranked results are stored back into a data table and sent as a Gmail notification.\n6. The loop continues with the next query until all queries are processed.\n\n### Setup steps\n\n- - [ ] Configure the **Reddit Schedule Trigger** with the desired run frequency (e.g., daily, hourly).\n- - [ ] Set up the **Reddit Get row(s)** data table with the list of Instagram search queries to process.\n- - [ ] Add your **Apify API key** to the `Insta get posts solo` HTTP Request node (URL/auth configuration).\n- - [ ] Connect your **Gmail account** credentials to the `Reddit Send a message1` node and set the recipient email address.\n- - [ ] Configure the **Reddot Push to data table** node to point to the correct destination data table for storing results.\n\n### Customization\n\nYou can adjust the ranking and sorting logic inside the `Sort Instagram solo` and `Classement Instagram` code nodes to change how posts are scored or filtered. The batch size in `Reddit Loop Over Query` can also be tuned depending on API rate limits. |
| Sticky Note10 | Sticky Note | Canvas documentation for trigger/query block |  |  |  |
| Sticky Note11 | Sticky Note | Canvas documentation for loop/fetch block |  |  |  |
| Sticky Note12 | Sticky Note | Canvas documentation for ranking/output block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a clean rebuild path, including the corrections needed so the workflow works reliably.

1. **Create a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name it: `Scheduled Run Trigger`
   - Configure the interval you want, for example hourly or daily.
   - Activate the workflow only after all credentials and tables are configured.

2. **Create the source Data Table**
   - In n8n, create a Data Table named something like `Query social media`.
   - Add at least one column named `Query`.
   - Populate it with hashtags or search terms, such as:
     - `fitness`
     - `ai`
     - `travel`
   - Queries are later sanitized by removing whitespace.

3. **Add a Data Table node to read queries**
   - Node type: **Data Table**
   - Name it: `Read Queries from Table`
   - Operation: **Get**
   - Return All: **true**
   - Select the source table: `Query social media`
   - Connect:
     - `Scheduled Run Trigger` → `Read Queries from Table`

4. **Add the loop node**
   - Node type: **Split In Batches**
   - Name it: `Loop Over Queries1`
   - Default settings are sufficient for this design.
   - Connect:
     - `Read Queries from Table` → `Loop Over Queries1`

5. **Configure Apify credentials**
   - In n8n credentials, create an **Apify API** credential.
   - Use your Apify token.
   - Make sure the HTTP Request node can authenticate with it.

6. **Add the HTTP Request node for Instagram scraping**
   - Node type: **HTTP Request**
   - Name it: `Fetch Instagram Posts via Apify1`
   - Method: **POST**
   - URL:
     `https://api.apify.com/v2/acts/reGe1ST3OBgYZSsZJ/run-sync-get-dataset-items`
   - Authentication: **Predefined Credential Type**
   - Credential type: **Apify API**
   - Header:
     - `Content-Type: application/json`
   - Body type: **JSON**
   - JSON body:
     ```json
     {
       "hashtags": ["{{ $json.Query.replace(/\\s+/g, '') }}"],
       "keywordSearch": false,
       "resultsLimit": 50,
       "resultsType": "posts"
     }
     ```
   - Enable **Retry On Fail**
   - Connect:
     - `Loop Over Queries1` → `Fetch Instagram Posts via Apify1`

7. **Add the first Code node for filtering and scoring**
   - Node type: **Code**
   - Name it: `Sort Instagram Results`
   - Paste logic equivalent to:
     - Read all input items
     - Exclude `productType = carousel_item`
     - Keep only URLs containing `/p/`
     - Convert hidden/invalid likes to `0`
     - Compute `score = likes + comments * 2`
     - Sort descending
     - Keep top 10
     - Add `rank`
   - Connect:
     - `Fetch Instagram Posts via Apify1` → `Sort Instagram Results`

8. **Add the second Code node for final ranking/output**
   - Node type: **Code**
   - Name it: `Rank Instagram Posts1`
   - Use corrected references. The exported workflow contains broken node names; fix them.
   - Recommended corrected logic:
     - Get keyword from `$('Loop Over Queries1').first().json.Query`
     - Get Instagram items from `$('Sort Instagram Results').all()`
     - Set `source = 'Instagram'`
     - Use a safer author field from upstream data if available; if not, fallback to `@unknown`
     - Shorten caption/content
     - Set level thresholds:
       - High: `>= 10000`
       - Medium: `>= 1000`
       - Low: otherwise
     - Sort descending by score
     - Add final rank and date
     - Return top 15 or fewer
   - Connect:
     - `Sort Instagram Results` → `Rank Instagram Posts1`

9. **Create the destination Data Table**
   - Create another Data Table, for example `Trend Social media`
   - Add columns:
     - `rank` number
     - `keyword` string
     - `source` string
     - `author` string
     - `content` string
     - `score` number
     - `level` string
     - `url` string
     - `Date` datetime

10. **Add the Data Table node to store results**
    - Node type: **Data Table**
    - Name it: `Save Results to Data Table`
    - Operation: insert/create rows according to the node UI for your n8n version
    - Select the destination table: `Trend Social media`
    - Map fields:
      - `url` → `{{ $json.url }}`
      - `Date` → `{{ $now.toISO() }}`
      - `rank` → `{{ $json.rank }}`
      - `level` → `{{ $json.level }}`
      - `score` → `{{ $json.score }}`
      - `author` → `{{ $json.author }}`
      - `source` → `{{ $json.source }}`
      - `content` → `{{ $json.content }}`
      - `keyword` → `{{ $json.keyword }}`
    - Connect:
      - `Rank Instagram Posts1` → `Save Results to Data Table`

11. **Configure Gmail OAuth2 credentials**
    - Create a **Gmail OAuth2** credential in n8n.
    - Authenticate the Google account that will send the email.
    - Confirm Gmail API access and consent configuration are valid.

12. **Add the Gmail node**
    - Node type: **Gmail**
    - Name it: `Send Instagram Report Email`
    - Operation: **Send**
    - To: set your destination email, replacing `user@example.com`
    - Subject: `Social media monitoring`
    - Message type: HTML
    - Paste the HTML template if you want the same visual output.
    - Connect Gmail OAuth2 credentials.
    - Connect:
      - `Rank Instagram Posts1` → `Send Instagram Report Email`

13. **Loop the process**
    - Connect:
      - `Send Instagram Report Email` → `Loop Over Queries1`
    - This closes the batch loop and lets n8n continue with the next query.

14. **Correct the email template if needed**
    - The provided HTML template references `Rank Instagram Posts1` correctly.
    - However, it assumes there may be TikTok and Reddit rows. In this workflow there are only Instagram rows.
    - You can:
      - keep it as-is, with TikTok and Reddit totals staying `0`, or
      - simplify it so only Instagram totals are shown.

15. **Test with one query first**
    - Add a single simple hashtag in the source Data Table.
    - Run manually.
    - Confirm:
      - Apify returns data
      - `Sort Instagram Results` outputs ranked rows
      - `Rank Instagram Posts1` no longer errors on node references
      - Data rows are inserted
      - Email is delivered

16. **Recommended fixes before production**
    - In `Rank Instagram Posts1`, replace:
      - `$('Loop Over Query')` with `$('Loop Over Queries1')`
      - `$('Sort Instagram')` with `$('Sort Instagram Results')`
    - Improve author extraction by carrying forward `ownerUsername` from the first code node instead of trying to parse it from the post URL.
    - Consider whether `executeOnce` on the Gmail node is desired in a looping design.

### Suggested corrected data flow
1. Schedule Trigger  
2. Read query rows  
3. Split in batches over queries  
4. Call Apify Instagram actor  
5. Filter and score posts  
6. Normalize and rank report items  
7. Save results  
8. Send email  
9. Return to loop

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The canvas documentation contains several legacy node names such as “Reddit Schedule Trigger”, “Reddit Get row(s)”, “Insta get posts solo”, and “Reddot Push to data table”. These do not match the actual workflow and appear to be leftovers from a reused template. | Workflow sticky note content |
| The workflow’s first code node defines the Instagram engagement formula as: likes × 1 plus comments × 2. | Sort Instagram Results note |
| The workflow is designed around Apify’s actor endpoint `run-sync-get-dataset-items`, which waits for actor completion and returns the dataset in one request. | Apify integration behavior |
| The HTML email is written in French, even though the workflow title is English. | Gmail node message template |
| The report template is cross-platform in style, but this specific workflow only generates Instagram results. | Gmail node message template |
| The biggest technical issue in the exported version is broken node-name references inside `Rank Instagram Posts1`. These must be corrected before reliable execution. | Code node implementation note |