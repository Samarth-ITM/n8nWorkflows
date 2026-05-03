Daily News Aggregation with Perplexity AI & MongoDB Storage

https://n8nworkflows.xyz/workflows/daily-news-aggregation-with-perplexity-ai---mongodb-storage-7483


# Daily News Aggregation with Perplexity AI & MongoDB Storage

---

### 1. Workflow Overview

This workflow automates the daily aggregation of global news articles using the Perplexity AI API and stores them into a MongoDB database. It then sends a notification email confirming successful storage. The process runs once per day at 8:00 AM UTC.

Logical blocks in the workflow are:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specified hour.
- **1.2 News Fetching via Perplexity AI:** Queries Perplexity AI to fetch recent major global news articles formatted as JSON.
- **1.3 Data Parsing and Formatting:** Processes the raw JSON string response from Perplexity into discrete news items.
- **1.4 Batch Loop and MongoDB Storage:** Iterates over each news item and inserts it into a MongoDB collection.
- **1.5 Aggregation and Notification Preparation:** Aggregates all processed news items for summary and notification.
- **1.6 Notification Email:** Sends a confirmation email via Gmail indicating that news registration succeeded.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow once daily at 8:00 AM UTC.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: scheduleTrigger  
    - Configuration: Set to trigger at hour 8 (8:00 AM UTC) daily.  
    - Input: None (it's the entry trigger).  
    - Output: Initiates the workflow to the Perplexity node.  
    - Version-specific: Using version 1.2, standard schedule node functionality.  
    - Possible failures: Scheduler misconfiguration, workflow disabled, or system downtime.  
    - Notes: Acts as the starting point of the workflow.

#### 2.2 News Fetching via Perplexity AI

- **Overview:**  
  This block queries Perplexity AI using a detailed prompt to fetch high-impact global news in a well-defined JSON format.

- **Nodes Involved:**  
  - Perplexity  
  - Response Perplexity (code node)  
  - Sticky Note3 (comment)

- **Node Details:**

  - **Perplexity**  
    - Type: perplexity  
    - Configuration:  
      - Model: sonar-pro (advanced model for news retrieval).  
      - Options: searchRecency set to "day" to limit to recent news.  
      - Message prompt: A precise prompt requesting the latest breaking global news focusing on major topics and reputable sources. Requires return as a JSON array with structured fields (headline, timestamp, source, summary, url, category, language, metadata).  
      - Simplify: true to get a simplified response.  
    - Credentials: Perplexity API key linked via "Perplexity account".  
    - Input: Trigger from Schedule Trigger.  
    - Output: Raw JSON string response with news data.  
    - Possible failures: API authentication errors, rate limits, request timeout, malformed response.

  - **Response Perplexity**  
    - Type: code (JavaScript)  
    - Configuration:  
      - Parses the raw string from Perplexity (removes markdown backticks and "json" tags).  
      - Converts the cleaned string into an array of news article objects.  
      - Outputs each news article as a separate item for looping.  
    - Input: Raw JSON string from Perplexity node.  
    - Output: Array of discrete news items.  
    - Possible failures: JSON parsing errors if the response is malformed or unexpected.

  - **Sticky Note3**  
    - Content: "News Perplexity" - indicates that this block is dedicated to fetching news from Perplexity AI.

#### 2.3 Data Parsing and Formatting

- **Overview:**  
  Aggregates all news items back into a single JSON array to prepare for notification and final summary.

- **Nodes Involved:**  
  - News (code node)

- **Node Details:**

  - **News**  
    - Type: code (JavaScript)  
    - Configuration:  
      - Collects all individual news items from the loop and bundles them into a single JSON object under the key `noticias`.  
      - This prepares the data for final notification or further processing.  
    - Input: Multiple news items from loop iteration.  
    - Output: One aggregated JSON containing all news articles.  
    - Possible failures: Logic errors if input is empty or null.

#### 2.4 Batch Loop and MongoDB Storage

- **Overview:**  
  Loops over each news article individually and inserts it into the MongoDB database collection.

- **Nodes Involved:**  
  - Loop Over News (splitInBatches)  
  - DB News (MongoDB insert node)  
  - Sticky Note (comment: “Official Database”)

- **Node Details:**

  - **Loop Over News**  
    - Type: splitInBatches  
    - Configuration:  
      - Processes news items one by one (default batch size is 1 implicitly).  
      - Reset option disabled to maintain state across executions.  
    - Input: Output from Response Perplexity node (individual news items).  
    - Output: Single news item per batch to MongoDB insert node.  
    - Possible failures: Batch processing errors, memory issues with large data sets.

  - **DB News**  
    - Type: mongoDb  
    - Configuration:  
      - Operation: Insert to add new documents.  
      - Collection: "news" (MongoDB collection name).  
      - Fields to insert: headline, timestamp, source, summary, url, category, language, metadata.  
    - Credentials: MongoDB account credentials linked.  
    - Input: Single news item from Loop Over News node.  
    - Output: Confirmation of insertion, passed back to loop node.  
    - Possible failures: MongoDB authentication errors, network issues, duplicate key errors if indexes exist, data schema mismatch.  
    - Notes: Marked with sticky note “Official Database”.

#### 2.5 Aggregation and Notification Preparation

- **Overview:**  
  After all news items are processed and inserted, this block prepares a summary notification for email dispatch.

- **Nodes Involved:**  
  - News (code node, already described above)  
  - Sticky Note4 (comment: “Final Notification”)

- **Node Details:**

  - **News**  
    - As described in block 2.3, aggregates news items into a single JSON output for notification.  
    - Output flows into the Send Message email node.  

  - **Sticky Note4**  
    - Content: "Final Notification" indicating this is the final output stage before notifying the user.

#### 2.6 Notification Email

- **Overview:**  
  Sends an HTML-formatted email via Gmail confirming that the news articles were successfully stored.

- **Nodes Involved:**  
  - Send Message (Gmail node)  
  - Sticky Note (comment: “News registration confirmation”)

- **Node Details:**

  - **Send Message**  
    - Type: gmail  
    - Configuration:  
      - Recipient: User must replace "<YOUR EMAIL>" with the target email address.  
      - Subject: "[ News ] Record of all news items"  
      - Message: A full HTML email template that thanks the user and confirms successful news registration.  
      - Options: Attribution disabled to keep email clean.  
    - Credentials: Uses Gmail OAuth2 credentials linked via "Gmail account 2".  
    - Input: Aggregated news JSON from the News node.  
    - Output: Confirmation of email sent.  
    - Possible failures: Gmail API authentication errors, quota limits, invalid recipient email, network issues.  
    - Notes: Sticky note marks this as news registration confirmation.

---

### 3. Summary Table

| Node Name          | Node Type          | Functional Role                  | Input Node(s)         | Output Node(s)         | Sticky Note                           |
|--------------------|--------------------|--------------------------------|-----------------------|-----------------------|-------------------------------------|
| Schedule Trigger    | scheduleTrigger    | Workflow daily trigger          | None                  | Perplexity            |                                     |
| Perplexity         | perplexity          | Fetches news from Perplexity AI| Schedule Trigger      | Response Perplexity    |                                     |
| Response Perplexity | code                | Parses Perplexity raw JSON      | Perplexity            | Loop Over News         |                                     |
| Loop Over News      | splitInBatches      | Loops over news articles        | Response Perplexity    | DB News, News          |                                     |
| DB News             | mongoDb             | Inserts news into MongoDB       | Loop Over News         | Loop Over News         | Official Database                   |
| News                | code                | Aggregates news for notification| Loop Over News         | Send Message           |                                     |
| Send Message        | gmail               | Sends confirmation email       | News                   | None                  | News registration confirmation     |
| Sticky Note         | stickyNote          | Documentation & comments       | None                   | None                  | See detailed notes for each sticky note |
| Sticky Note3        | stickyNote          | "News Perplexity" comment      | None                   | None                  | News Perplexity                     |
| Sticky Note4        | stickyNote          | "Final Notification" comment   | None                   | None                  | Final Notification                  |
| Sticky Note (large) | stickyNote          | Workflow overview and explanation| None                  | None                  | Detailed workflow description       |
| Sticky Note1        | stickyNote          | Empty (reserved space)          | None                   | None                  |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: `scheduleTrigger`  
   - Set to trigger daily at 8:00 AM UTC using the interval configuration.  
   - No credentials needed.

2. **Create Perplexity Node**  
   - Type: `perplexity`  
   - Configure with:  
     - Model: `sonar-pro`  
     - Options: set `searchRecency` to `"day"`  
     - Message prompt: Use the detailed prompt text specifying requirements for global breaking news with structured JSON output (see node 1a for full prompt).  
   - Attach your Perplexity API credentials.  
   - Connect Schedule Trigger output to this node.

3. **Create Code Node (Response Perplexity)**  
   - Type: `code` (JavaScript)  
   - Use JavaScript code to clean the raw response string and parse it into an array of news items.  
   - Connect Perplexity node output to this node.

4. **Create SplitInBatches Node (Loop Over News)**  
   - Type: `splitInBatches`  
   - Default batch size (1) is sufficient to process each news item separately.  
   - Connect Response Perplexity node output to this node.

5. **Create MongoDB Node (DB News)**  
   - Type: `mongoDb`  
   - Operation: `insert`  
   - Collection: `news`  
   - Fields to insert: `headline`, `timestamp`, `source`, `summary`, `url`, `category`, `language`, `metadata`  
   - Attach MongoDB credentials.  
   - Connect Loop Over News output to this node.

6. **Create Code Node (News Aggregation)**  
   - Type: `code` (JavaScript)  
   - Code to collect all news items back into a single JSON array under key `noticias`.  
   - Connect Loop Over News output (second output) to this node.

7. **Create Gmail Node (Send Message)**  
   - Type: `gmail`  
   - Set recipient email address (replace `<YOUR EMAIL>` with actual email).  
   - Subject: `[ News ] Record of all news items`  
   - Message: Use the provided HTML template confirming successful news data registration.  
   - Disable attribution.  
   - Attach Gmail OAuth2 credentials.  
   - Connect News node output to this node.

8. **Optional: Add Sticky Notes**  
   - Add sticky notes above each logical block for documentation and clarity as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow automatically fetches, formats, stores, and notifies about global news once daily at 8:00 AM UTC.                    | General workflow purpose                                        |
| Perplexity API used with a detailed prompt to ensure structured, quality, and neutral news data in JSON format.                | Perplexity API prompt details                                   |
| MongoDB stores news articles with fields suitable for typical news metadata including nested metadata for extensibility.      | MongoDB database schema                                          |
| Gmail node sends a professional styled HTML email confirming successful processing.                                            | Gmail OAuth2 credentials required                               |
| Replace placeholder `<YOUR EMAIL>` with the actual recipient email to receive notifications.                                  | User configuration required                                     |
| Documentation sticky notes in the workflow provide detailed explanations for each block and the overall workflow goal.        | Workflow documentation embedded in sticky notes                 |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---