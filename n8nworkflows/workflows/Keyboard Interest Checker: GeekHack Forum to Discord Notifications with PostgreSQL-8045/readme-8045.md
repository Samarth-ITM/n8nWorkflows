Keyboard Interest Checker: GeekHack Forum to Discord Notifications with PostgreSQL

https://n8nworkflows.xyz/workflows/keyboard-interest-checker--geekhack-forum-to-discord-notifications-with-postgresql-8045


# Keyboard Interest Checker: GeekHack Forum to Discord Notifications with PostgreSQL

### 1. Workflow Overview

This workflow automates monitoring of two specific subforums on the GeekHack keyboard enthusiast forum — Interest Checks and Group Buys — via their RSS feeds. Its primary goal is to detect new discussion threads (excluding replies), check if they have already been processed, extract key information including images and author messages from the thread page, and send formatted notifications to Discord through registered webhooks. Processed threads are tracked in a PostgreSQL database to avoid duplicate notifications. Additionally, the workflow provides a form-triggered endpoint for adding new Discord webhook URLs to the database.

**Logical blocks:**

- **1.1 Scheduled RSS Feed Reading:** Periodically fetches latest threads from Interest Checks and Group Buys forums.
- **1.2 Filtering & Deduplication:** Filters for new threads (not replies), extracts topic IDs, and checks if already processed in PostgreSQL.
- **1.3 Thread Content Extraction:** For each new thread, fetches the thread page, extracts author messages and images.
- **1.4 Discord Notification Construction and Sending:** Builds a Discord webhook payload with embeds including images and metadata, retrieves webhook URLs from PostgreSQL, and sends notifications.
- **1.5 Processed Threads Update:** Records processed threads in PostgreSQL to prevent re-processing.
- **1.6 Webhook Management via Form:** Exposes a form webhook in n8n to add new webhook URLs to PostgreSQL.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled RSS Feed Reading

- **Overview:**  
Triggers hourly, reads RSS feeds for new threads from two GeekHack boards: Interest Checks and Group Buys. Merges the feed items into a single stream for processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Interest Checks RSS  
  - Group Buys RSS  
  - Merge

- **Node Details:**

1. **Schedule Trigger**  
   - Type: Schedule Trigger  
   - Role: Initiates the workflow every hour.  
   - Configuration: Interval set to every 1 hour.  
   - Input: None (trigger node).  
   - Output: Triggers both RSS feed nodes.  
   - Edge cases: If n8n server is down or missed triggers, data can be outdated.

2. **Interest Checks RSS**  
   - Type: RSS Feed Read  
   - Role: Reads the RSS feed of the Interest Checks board (board id 132).  
   - Configuration: URL: `https://geekhack.org/index.php?action=.xml;type=rss2;board=132.0;limit=50`  
   - Input: Trigger from Schedule Trigger.  
   - Output: RSS items.  
   - Edge cases: RSS feed may be temporarily unavailable or malformed.

3. **Group Buys RSS**  
   - Type: RSS Feed Read  
   - Role: Reads the RSS feed of the Group Buys board (board id 70).  
   - Configuration: URL: `https://geekhack.org/index.php?action=.xml;type=rss2;board=70.0;limit=50`  
   - Input: Trigger from Schedule Trigger.  
   - Output: RSS items.  
   - Edge cases: Same as Interest Checks RSS.

4. **Merge**  
   - Type: Merge  
   - Role: Combines the two RSS feed outputs into one stream for unified processing.  
   - Configuration: Default (no special parameters)  
   - Input: Outputs from both RSS feed nodes.  
   - Output: Merged items array.  
   - Edge cases: If one feed is empty, still proceeds with available data.

---

#### 2.2 Filtering & Deduplication

- **Overview:**  
Filters out replies to retain only new threads, extracts topic IDs, and checks PostgreSQL table `processed_threads` to exclude threads already processed.

- **Nodes Involved:**  
  - Filter New Threads  
  - Loop Over Items  
  - If  
  - Check if Processed

- **Node Details:**

1. **Filter New Threads**  
   - Type: Code  
   - Role: Filters out RSS items that are replies (title starts with "Re:") and extracts `topicId` from URL.  
   - Configuration: Custom JavaScript filtering and regex extraction.  
   - Key expression:  
     - `!title.startsWith('Re:')`  
     - Extract topic id from link via regex `/topic=(\d+)/`.  
   - Input: Merged RSS feed items.  
   - Output: Filtered list of new thread items with topicId added.  
   - Edge cases: Threads without `topicId` in URL are ignored; malformed titles or links may cause misses.

2. **Loop Over Items**  
   - Type: Split In Batches  
   - Role: Processes each filtered thread individually for downstream conditional checks.  
   - Configuration: Default, processes one item at a time (batch size = 1 implied).  
   - Input: Filtered new threads.  
   - Output: Single thread item per iteration.  
   - Edge cases: Large batch sizes could cause performance issues; batch processing allows controlled concurrency.

3. **If**  
   - Type: If  
   - Role: Checks if the output of the next node "Check if Processed" is empty (meaning thread not yet processed).  
   - Configuration: Condition checks `isNotEmpty()` of previous node's output and negates it (`false` operator).  
   - Input: From Loop Over Items (to the left: Check if Processed output).  
   - Output: Branches:  
     - True (thread not found in DB): proceeds to fetch and process the thread.  
     - False (already processed): loops back or stops processing for this item.  
   - Edge cases: Expression failures if `$json` structure unexpected.

4. **Check if Processed**  
   - Type: PostgreSQL Select  
   - Role: Queries `processed_threads` table for an existing record with the current thread's topicId.  
   - Configuration:  
     - Table: `processed_threads`  
     - Schema: `public`  
     - Where clause: `topic_id = current topicId`  
     - Return all matches (should be 0 or 1)  
   - Credentials: PostgreSQL connection setup.  
   - Input: Single thread item from Loop Over Items.  
   - Output: Query results (empty if not processed).  
   - Edge cases: DB connection failure, query errors, or missing table.

---

#### 2.3 Thread Content Extraction

- **Overview:**  
For each unprocessed thread, fetches its webpage, extracts the author message HTML, then extracts all images with attributes from the message content.

- **Nodes Involved:**  
  - Get geekhack page  
  - Extract author message  
  - Extract images

- **Node Details:**

1. **Get geekhack page**  
   - Type: HTTP Request  
   - Role: Retrieves the HTML content of the thread page URL.  
   - Configuration:  
     - URL: dynamically set to thread's link from "Filter New Threads" node.  
     - Headers: User-Agent set to `"Mozilla/5.0 (compatible; bot)"` to mimic a bot client.  
   - Input: From "If" node's true branch for unprocessed threads.  
   - Output: Raw HTML content in response.  
   - Edge cases: HTTP errors, timeouts, 404 if thread deleted.

2. **Extract author message**  
   - Type: HTML Extract  
   - Role: Extracts the HTML content of the author’s message from the thread page.  
   - Configuration:  
     - CSS selector constructed dynamically to match element with id `#msg_<messageId>`, where `messageId` is extracted from thread link fragment (`#msg...`).  
     - Returns inner HTML content.  
   - Input: HTML from HTTP request node.  
   - Output: Extracted HTML snippet containing author message.  
   - Edge cases: Selector mismatch if page layout changes; missing message ID fragment.

3. **Extract images**  
   - Type: Code  
   - Role: Parses extracted author message HTML to find all `<img>` tags and retrieves attributes: src, alt, class, width, height, full tag.  
   - Configuration:  
     - Uses regex to match `<img>` tags and capture attribute values.  
     - Filters out images with empty src.  
     - Returns total image count and array of image objects.  
   - Input: Extracted author message HTML.  
   - Output: JSON with `totalImages` and array `images`.  
   - Edge cases: Malformed HTML, missing attributes, no images found.

---

#### 2.4 Discord Notification Construction and Sending

- **Overview:**  
Creates a Discord webhook payload with embeds containing thread metadata and images (up to 4), retrieves all registered webhook URLs from the database, sends the payload to each, and finally logs the processing in the DB.

- **Nodes Involved:**  
  - Create Payload  
  - Select rows from a table  
  - Merge SQL outputs  
  - Send to discord  
  - Update entry  
  - DebugHelper

- **Node Details:**

1. **Create Payload**  
   - Type: Code  
   - Role: Constructs Discord webhook JSON payload with embeds: first embed contains thread title, URL, footer category, timestamp, and first image; up to 3 additional embeds with further images.  
   - Configuration:  
     - Limits to max 4 images.  
     - Includes username "Geekhack Updates".  
   - Input: Extracted images JSON and filtered thread metadata.  
   - Output: JSON payload for Discord webhook.  
   - Edge cases: No images results in single embed without image; malformed image URLs.

2. **Select rows from a table**  
   - Type: PostgreSQL Select  
   - Role: Fetches all stored webhook URLs from `webhooks` table to send notifications.  
   - Configuration:  
     - Table: `webhooks`  
     - Schema: `public`  
     - Return all rows.  
   - Input: From Create Payload.  
   - Output: List of webhook URLs.  
   - Edge cases: DB connection failure, empty table.

3. **Merge SQL outputs**  
   - Type: Merge  
   - Role: Combines the outputs from webhook URLs and signals to send notifications and update DB.  
   - Configuration: Default.  
   - Input: From Select rows and Update entry nodes.  
   - Output: Combined signal for sending Discord messages.  
   - Edge cases: Merging errors if data structures mismatch.

4. **Send to discord**  
   - Type: HTTP Request  
   - Role: Posts the constructed payload to each Discord webhook URL.  
   - Configuration:  
     - URL: dynamically set to each webhook URL from DB.  
     - Method: POST  
     - Body: JSON payload from Create Payload.  
     - Error handling: On error, continues workflow without stopping.  
   - Input: From Merge SQL outputs.  
   - Output: Discord API response (logged or ignored).  
   - Edge cases: Webhook URL invalid or revoked, network errors.

5. **Update entry**  
   - Type: PostgreSQL Insert/Update  
   - Role: Inserts or updates the `processed_threads` table with topic ID, title, and timestamp to mark the thread as processed.  
   - Configuration:  
     - Table: `processed_threads`  
     - Fields: `topic_id`, `title`, `processed_at` (current timestamp)  
   - Input: From Create Payload node (thread info).  
   - Output: DB update confirmation.  
   - Edge cases: DB write failure, duplicate keys.

6. **DebugHelper**  
   - Type: Debug Helper  
   - Role: Logs or inspects the output of Send to discord node for troubleshooting.  
   - Configuration: Set to category "doNothing" (pass-through).  
   - Input: From Send to discord.  
   - Output: None; for debugging only.

---

#### 2.5 Webhook Management via Form

- **Overview:**  
Provides an HTTP webhook endpoint with a form to allow adding new Discord webhook URLs into the `webhooks` PostgreSQL table.

- **Nodes Involved:**  
  - On form submission  
  - Insert rows in a table

- **Node Details:**

1. **On form submission**  
   - Type: Form Trigger  
   - Role: Listens for HTTP form submissions to add webhook URLs.  
   - Configuration:  
     - Webhook ID: fixed UUID  
     - Form title: "Add new webhooks"  
     - Form fields: single field labeled `URL`  
   - Input: External HTTP requests with form data.  
   - Output: Form data JSON with `URL` and submission timestamp.  
   - Edge cases: Invalid URL input, unauthenticated submissions.

2. **Insert rows in a table**  
   - Type: PostgreSQL Insert  
   - Role: Inserts submitted webhook URL and timestamp into `webhooks` table.  
   - Configuration:  
     - Table: `webhooks`  
     - Schema: `public`  
     - Columns: `url` (from form), `created_at` (submission time)  
   - Input: From form submission node.  
   - Output: DB insertion confirmation.  
   - Edge cases: DB write failure, duplicate URLs.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                               | Input Node(s)                | Output Node(s)                  | Sticky Note                                                |
|---------------------|----------------------|-----------------------------------------------|-----------------------------|--------------------------------|------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger     | Initiates workflow hourly                       | -                           | Interest Checks RSS, Group Buys RSS |                                                            |
| Interest Checks RSS | RSS Feed Read        | Reads Interest Checks RSS feed                  | Schedule Trigger             | Merge                          |                                                            |
| Group Buys RSS      | RSS Feed Read        | Reads Group Buys RSS feed                        | Schedule Trigger             | Merge                          |                                                            |
| Merge               | Merge                | Combines two RSS feed outputs                    | Interest Checks RSS, Group Buys RSS | Filter New Threads             |                                                            |
| Filter New Threads  | Code                 | Filters new threads, extracts topic IDs         | Merge                       | Loop Over Items                |                                                            |
| Loop Over Items     | Split In Batches     | Processes each thread item individually          | Filter New Threads           | If, Check if Processed         |                                                            |
| If                  | If                   | Checks if thread already processed               | Loop Over Items              | Get geekhack page (true branch), Check if Processed (false branch) |                                                            |
| Check if Processed  | PostgreSQL Select    | Queries DB to detect processed threads           | Loop Over Items              | Loop Over Items (if processed), If (if not) |                                                            |
| Get geekhack page   | HTTP Request         | Fetches thread HTML page                          | If (true branch)             | Extract author message         |                                                            |
| Extract author message | HTML Extract        | Extracts author's message HTML from thread page  | Get geekhack page            | Extract images                 |                                                            |
| Extract images      | Code                 | Extracts img tags and attributes from message HTML| Extract author message       | Create Payload                 |                                                            |
| Create Payload      | Code                 | Builds Discord webhook JSON payload with embeds | Extract images               | Select rows from a table, Update entry |                                                            |
| Select rows from a table | PostgreSQL Select | Retrieves registered webhook URLs                 | Create Payload              | Merge SQL outputs              |                                                            |
| Merge SQL outputs   | Merge                | Combines DB outputs for sending notifications    | Select rows from a table, Update entry | Send to discord            |                                                            |
| Send to discord     | HTTP Request         | Sends webhook payload to Discord URLs             | Merge SQL outputs            | DebugHelper                   | On error, continues without stopping.                      |
| Update entry        | PostgreSQL Insert/Update | Records thread as processed in DB                | Create Payload               | Merge SQL outputs             |                                                            |
| DebugHelper         | Debug Helper         | Logs output for troubleshooting                   | Send to discord              | -                            |                                                            |
| On form submission  | Form Trigger         | HTTP endpoint to submit new webhook URLs          | -                           | Insert rows in a table         |                                                            |
| Insert rows in a table | PostgreSQL Insert   | Inserts new webhook URL into DB                    | On form submission           | -                            |                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to every 1 hour to trigger workflow regularly.

2. **Add two RSS Feed Read nodes:**  
   - **Interest Checks RSS:** URL: `https://geekhack.org/index.php?action=.xml;type=rss2;board=132.0;limit=50`  
   - **Group Buys RSS:** URL: `https://geekhack.org/index.php?action=.xml;type=rss2;board=70.0;limit=50`

3. **Create a Merge node**  
   - Connect outputs of both RSS nodes to Merge inputs to unify RSS items.

4. **Add a Code node named "Filter New Threads"**  
   - Paste JavaScript code to:  
     - Filter out RSS items whose title starts with "Re:"  
     - Extract `topicId` from the thread link URL using regex `/topic=(\d+)/`  
   - Connect Merge output to this node.

5. **Add a SplitInBatches node named "Loop Over Items"**  
   - Connect Filter New Threads output to this node.  
   - Default batch size (1) is suitable.

6. **Add a PostgreSQL Select node "Check if Processed"**  
   - Configure credentials for PostgreSQL.  
   - Set operation to select from table `processed_threads` in schema `public`.  
   - Add where condition: `topic_id` equals current item's `topicId` expression `={{ $json.topicId }}`.  
   - Connect Loop Over Items output to this node.

7. **Add an If node**  
   - Connect Loop Over Items output to If node.  
   - Condition: check if output from "Check if Processed" is empty (use expression `isNotEmpty()` and negate).  
   - True branch proceeds to next step, False branch ends or loops back.

8. **Add an HTTP Request node "Get geekhack page"**  
   - URL set dynamically to thread link: `={{ $('Filter New Threads').item.json.link }}`  
   - Set header `User-Agent` to `Mozilla/5.0 (compatible; bot)`  
   - Connect If node true branch here.

9. **Add an HTML Extract node "Extract author message"**  
   - Operation: Extract HTML content.  
   - CSS selector: dynamically constructed as `=#msg_{{ $('Filter New Threads').item.json.link.split('#msg')[1] }}`  
   - Connect Get geekhack page output here.

10. **Add a Code node "Extract images"**  
    - Paste JavaScript code to parse HTML input and extract all `<img>` tags with attributes (src, alt, class, width, height).  
    - Connect Extract author message output here.

11. **Add a Code node "Create Payload"**  
    - JavaScript code builds a Discord webhook payload with embeds containing thread title, URL, footer, timestamp, and up to 4 images.  
    - Connect Extract images output here.

12. **Add a PostgreSQL Select node "Select rows from a table"**  
    - Set to select all rows from table `webhooks` in schema `public`.  
    - Connect Create Payload output here.

13. **Add a PostgreSQL Insert/Update node "Update entry"**  
    - Inserts or updates `processed_threads` table with `topic_id`, `title`, `processed_at` (current timestamp).  
    - Input values use expressions from Filter New Threads node and current time.  
    - Connect Create Payload output here.

14. **Add a Merge node "Merge SQL outputs"**  
    - Connect outputs of Select rows from a table and Update entry nodes to this Merge node.

15. **Add an HTTP Request node "Send to discord"**  
    - Method: POST  
    - URL: dynamic, set to each webhook URL from Select rows node output.  
    - Body: JSON, set to the payload created in Create Payload node.  
    - On error: continue without stopping.  
    - Connect Merge SQL outputs output to this node.

16. **Add a Debug Helper node "DebugHelper"**  
    - Connect Send to discord output here for logging and debugging.

17. **Create a Form Trigger node "On form submission"**  
    - Configure with webhook ID, form title "Add new webhooks".  
    - Add a single form field labeled `URL`.  
    - No input connection.

18. **Add a PostgreSQL Insert node "Insert rows in a table"**  
    - Insert into table `webhooks` the submitted URL and current timestamp.  
    - Connect On form submission output here.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow relies on PostgreSQL tables named `processed_threads` and `webhooks` with fields suitable to store thread IDs, titles, timestamps, and webhook URLs. | Database schema must be prepared before running the workflow.                                       |
| User-Agent header in HTTP requests is set to mimic a bot to avoid HTTP 403 or blocks from GeekHack. | Helps ensure reliable access to thread pages.                                                       |
| The workflow ignores replies by filtering titles starting with "Re:". This heuristic may fail if GeekHack changes its reply formatting. | Possible need to update filtering logic if forum changes.                                          |
| Discord webhook payload limits should be respected; this workflow limits to 4 images per embed to avoid exceeding size limits. | Discord API documentation: https://discord.com/developers/docs/resources/webhook                      |
| The form webhook allows adding new webhook URLs dynamically without code changes. | Useful for managing notification targets without workflow redeployment.                             |
| Error handling on sending Discord messages continues on error to prevent workflow stoppage. | Ensures robustness if individual webhooks are invalid or revoked.                                  |

---

**Disclaimer:** The text provided originates exclusively from an n8n automated workflow. All data processed is legal and public. The workflow respects content policies and contains no illegal or offensive elements.