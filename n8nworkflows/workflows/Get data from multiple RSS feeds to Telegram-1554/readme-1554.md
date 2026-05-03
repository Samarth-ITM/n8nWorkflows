Get data from multiple RSS feeds to Telegram

https://n8nworkflows.xyz/workflows/get-data-from-multiple-rss-feeds-to-telegram-1554


# Get data from multiple RSS feeds to Telegram

### 1. Workflow Overview

This workflow automates the retrieval and distribution of the latest news items from multiple RSS feeds, categorizing them by topic and sending them to specific Telegram channels. It periodically polls configured RSS sources, filters out previously processed items to only handle new content, and routes messages conditionally based on keywords and source URLs.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & RSS Source Initialization:** Periodic activation and definition of RSS feed URLs.
- **1.2 Batch Processing & RSS Reading:** Sequentially process each RSS feed URL to fetch recent items.
- **1.3 New Content Filtering:** Detect and forward only new RSS items that haven’t been processed before.
- **1.4 Conditional Categorization:** Use conditional checks to classify RSS items by topic or source.
- **1.5 Telegram Notification:** Send filtered news items to appropriate Telegram channels.
- **1.6 Data Reset Utility:** A function node to clear stored processed RSS IDs for maintenance.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & RSS Source Initialization

- **Overview:** This block triggers the workflow every 10 minutes and defines the list of RSS feed URLs to be processed.
- **Nodes Involved:** `Cron`, `RSS Source`

- **Node Details:**

  - **Cron**
    - Type: Trigger (Cron)
    - Role: Periodically triggers the workflow every 10 minutes.
    - Configuration: Set to run every 10 minutes.
    - Inputs: None (trigger node).
    - Outputs: Connects to `RSS Source`.
    - Edge Cases: Cron node failures are rare but can occur if n8n server is down.
  
  - **RSS Source**
    - Type: Function
    - Role: Provides the list of RSS feed URLs as JSON objects.
    - Configuration: Hardcoded list of 5 RSS feed URLs related to tech and security news.
    - Key Expressions: None; static JSON array returned.
    - Inputs: From `Cron`.
    - Outputs: To `SplitInBatches`.
    - Edge Cases: If URLs are malformed, subsequent RSS fetches will fail.

#### 2.2 Batch Processing & RSS Reading

- **Overview:** Processes each RSS feed URL one at a time to avoid API overload, and fetches the latest feed items.
- **Nodes Involved:** `SplitInBatches`, `RSS Feed Read`

- **Node Details:**

  - **SplitInBatches**
    - Type: Splitter
    - Role: Processes the RSS URLs one-by-one (batch size = 1).
    - Configuration: Batch size set to 1.
    - Inputs: From `RSS Source`.
    - Outputs: To `RSS Feed Read`.
    - Edge Cases: If batch processing stalls, workflow might hang or slow down.
  
  - **RSS Feed Read**
    - Type: RSS Feed Reader
    - Role: Reads RSS items from the current URL batch.
    - Configuration: URL parameter dynamically assigned from the current batch item (`{{$node["SplitInBatches"].json["url"]}}`).
    - Inputs: From `SplitInBatches`.
    - Outputs: To `only get new RSS` and back to `SplitInBatches` (loop).
    - Edge Cases: Network errors, invalid RSS format, empty feeds.

#### 2.3 New Content Filtering

- **Overview:** Filters out any RSS items that have already been processed and stored in workflow static data to avoid duplicate notifications.
- **Nodes Involved:** `only get new RSS`

- **Node Details:**

  - **only get new RSS**
    - Type: Function
    - Role: Compares current batch RSS items against stored IDs (using `isoDate`) and returns only new, unprocessed items.
    - Configuration:
      - Uses workflow static data (global) to maintain list of previously seen `isoDate` values.
      - Updates static data with new IDs after filtering.
    - Inputs: From `RSS Feed Read`.
    - Outputs: To `IF-1`.
    - Edge Cases:
      - On first run, static data is empty, so all items are considered new.
      - If `isoDate` is missing or duplicated, filtering logic may fail.
      - Static data size growth over time could impact performance.

#### 2.4 Conditional Categorization

- **Overview:** Categorizes new RSS items based on source URL and keywords in the title, routing them to different Telegram channels.
- **Nodes Involved:** `IF-1`, `IF-2`

- **Node Details:**

  - **IF-1**
    - Type: Conditional (If)
    - Role: Checks if the RSS item's `link` contains the string `techcommunity.microsoft.com`.
    - Configuration: String contains operation.
    - Inputs: From `only get new RSS`.
    - Outputs:
      - True branch: to `Telegram_M365`.
      - False branch: to `IF-2`.
    - Edge Cases: If `link` is missing or malformed, condition may fail.
  
  - **IF-2**
    - Type: Conditional (If)
    - Role: Checks if the RSS item `title` matches regex keywords related to security topics (both English and Chinese terms).
    - Configuration:
      - Regex string match with keywords such as "Security", "安全", "駭", "漏洞", etc.
      - Combined with "any" operation on multiple keywords.
    - Inputs: From `IF-1` (false branch).
    - Outputs:
      - True branch: to `Telegram_Security`.
      - False branch: to `Telegram_IT`.
    - Edge Cases: Regex failures if title is missing or non-string.
    - Version-specific: Regex behavior depends on n8n version but generally stable.

#### 2.5 Telegram Notification

- **Overview:** Sends the filtered and categorized RSS items to the correct Telegram chat channels with title and link.
- **Nodes Involved:** `Telegram_M365`, `Telegram_Security`, `Telegram_IT`

- **Node Details:**

  - **Telegram_M365**
    - Type: Telegram node
    - Role: Sends Microsoft 365 related news to a designated Telegram chat.
    - Configuration:
      - Message text: RSS `title` and `link`.
      - Chat ID: placeholder `"TelegramID"` (to be replaced).
      - Credentials: Uses `M365_RSS` Telegram API credentials.
    - Inputs: From `IF-1` (true).
    - Outputs: None.
    - Edge Cases: Telegram API errors, invalid chat ID, rate limits.
  
  - **Telegram_Security**
    - Type: Telegram node
    - Role: Sends security-related news items to a designated Telegram chat.
    - Configuration: Same as above but with `Security_RSS` credentials.
    - Inputs: From `IF-2` (true).
    - Outputs: None.
    - Edge Cases: Same as above.
  
  - **Telegram_IT**
    - Type: Telegram node
    - Role: Sends other IT-related news items to a designated Telegram chat.
    - Configuration: Same as above but with `IT_RSS` credentials.
    - Inputs: From `IF-2` (false).
    - Outputs: None.
    - Edge Cases: Same as above.

#### 2.6 Data Reset Utility

- **Overview:** Clears the stored list of processed RSS IDs from the global static data. Useful for maintenance or after major feed URL changes.
- **Nodes Involved:** `Clear Function`

- **Node Details:**

  - **Clear Function**
    - Type: Function
    - Role: Deletes the `oldRSSIds` key from global static data.
    - Configuration:
      - Accesses `getWorkflowStaticData('global')`.
      - Deletes `oldRSSIds`.
    - Inputs: None connected (manual or external trigger assumed).
    - Outputs: Empty JSON to continue flow if needed.
    - Edge Cases: If static data is already empty, no effect.
    - Usage: Not connected to main flow; likely to be triggered manually.

---

### 3. Summary Table

| Node Name        | Node Type             | Functional Role                     | Input Node(s)       | Output Node(s)          | Sticky Note                                      |
|------------------|-----------------------|-----------------------------------|---------------------|-------------------------|-------------------------------------------------|
| Cron             | Trigger (Cron)        | Periodic workflow trigger every 10 minutes | None                | RSS Source              |                                                 |
| RSS Source       | Function              | Provides list of RSS feed URLs    | Cron                | SplitInBatches          |                                                 |
| SplitInBatches   | Split In Batches      | Processes RSS URLs one at a time  | RSS Source          | RSS Feed Read           |                                                 |
| RSS Feed Read    | RSS Feed Read         | Fetches RSS items from URL        | SplitInBatches      | only get new RSS, SplitInBatches (loop) |                                                 |
| only get new RSS | Function              | Filters new RSS items not seen before | RSS Feed Read       | IF-1                    |                                                 |
| IF-1             | If                    | Checks if link contains Microsoft 365 domain | only get new RSS     | Telegram_M365 (true), IF-2 (false) |                                                 |
| IF-2             | If                    | Matches security keywords in title | IF-1 (false)        | Telegram_Security (true), Telegram_IT (false) |                                                 |
| Telegram_M365    | Telegram              | Sends Microsoft 365 news to Telegram | IF-1 (true)         | None                    |                                                 |
| Telegram_Security| Telegram              | Sends security news to Telegram   | IF-2 (true)         | None                    |                                                 |
| Telegram_IT      | Telegram              | Sends other IT news to Telegram   | IF-2 (false)        | None                    |                                                 |
| Clear Function   | Function              | Clears stored processed RSS IDs   | None (manual)       | None                    | Used for workflow maintenance to reset stored RSS IDs |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a `Cron` node:**
   - Set to trigger every 10 minutes.
   - This node initiates the workflow.

2. **Add a `Function` node named `RSS Source`:**
   - Paste the following JavaScript code to return RSS URLs:
     ```javascript
     return [
       { json: { url: 'https://feeds.feedburner.com/UnikosHardware' } },
       { json: { url: 'http://www.ithome.com.tw/rss.php' } },
       { json: { url: 'http://feeds.feedburner.com/playpc' } },
       { json: { url: 'https://lab.ocf.tw/feed/' } },
       { json: { url: 'https://techcommunity.microsoft.com/plugins/custom/microsoft/o365/custom-blog-rss?tid=3754543230341459569&board=microsoft_365blog' } }
     ];
     ```
   - Connect `Cron` → `RSS Source`.

3. **Add a `SplitInBatches` node:**
   - Set Batch Size to 1.
   - Connect `RSS Source` → `SplitInBatches`.

4. **Add an `RSS Feed Read` node:**
   - Set URL parameter to expression:
     ```
     {{$node["SplitInBatches"].json["url"]}}
     ```
   - Connect `SplitInBatches` → `RSS Feed Read`.
   - Create a loop connection from `RSS Feed Read` back to `SplitInBatches` to continue batches.

5. **Add a `Function` node named `only get new RSS`:**
   - Paste this JavaScript code:
     ```javascript
     const staticData = getWorkflowStaticData('global');
     const newRSSIds = items.map(item => item.json["isoDate"]);
     const oldRSSIds = staticData.oldRSSIds;

     if (!oldRSSIds) {
       staticData.oldRSSIds = newRSSIds;
       return items;
     }

     const actualNewRSSIds = newRSSIds.filter(id => !oldRSSIds.includes(id));
     const actualNewRSS = items.filter(data => actualNewRSSIds.includes(data.json['isoDate']));
     staticData.oldRSSIds = [...actualNewRSSIds, ...oldRSSIds];

     return actualNewRSS;
     ```
   - Connect `RSS Feed Read` → `only get new RSS`.

6. **Add an `If` node named `IF-1`:**
   - Condition: `link` contains `techcommunity.microsoft.com`.
   - Connect `only get new RSS` → `IF-1`.

7. **Add an `If` node named `IF-2`:**
   - Condition: `title` matches regex:
     ```
     資安|資訊安全|安全|外洩|監控|威脅|漏洞|封鎖|修補|攻擊|入侵|個資|隱私|私密|騙|社交工程|釣魚|駭|Security|security|Secure|secure
     ```
   - Use "any" combine operation.
   - Connect `IF-1` false branch → `IF-2`.

8. **Add three `Telegram` nodes:**
   - **Telegram_M365:**
     - Message: `{{$json["title"]}}\n{{$json["link"]}}`
     - Chat ID: Replace `"TelegramID"` with your actual chat ID.
     - Credentials: Use Telegram API credentials named `M365_RSS`.
     - Connect `IF-1` true branch → `Telegram_M365`.
   - **Telegram_Security:**
     - Same message format.
     - Chat ID: Replace `"TelegramID"`.
     - Credentials: `Security_RSS`.
     - Connect `IF-2` true branch → `Telegram_Security`.
   - **Telegram_IT:**
     - Same message format.
     - Chat ID: Replace `"TelegramID"`.
     - Credentials: `IT_RSS`.
     - Connect `IF-2` false branch → `Telegram_IT`.

9. **Optional: Add a `Function` node named `Clear Function` for maintenance:**
   - Code:
     ```javascript
     const staticData = getWorkflowStaticData('global');
     delete staticData.oldRSSIds;
     return [{ json: {} }];
     ```
   - Not connected to main flow; trigger manually when needed.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                       |
|------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow references three example workflows on n8n.io for RSS and Telegram integration | https://n8n.io/workflows/1507, https://n8n.io/workflows/617, https://n8n.io/workflows/199             |
| Telegram chat IDs must be replaced with actual chat identifiers for your bots | Telegram API documentation & n8n Telegram node docs                                                  |
| The regex used in IF-2 supports both English and Chinese security-related terms | Useful for multilingual news categorization                                                         |
| Static data storage in n8n is persistent per workflow and used here to avoid duplicate notifications | See n8n docs on Workflow Static Data                                                                |
| To reset stored old RSS IDs, run the `Clear Function` node manually           | Prevents blocking new items if old IDs get stuck                                                    |

---

This structured reference provides a detailed understanding of the full workflow, allowing maintenance, modification, or reproduction by both technical users and AI automation agents.