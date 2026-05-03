Automated MIT AI News Delivery to Discord with Deduplication

https://n8nworkflows.xyz/workflows/automated-mit-ai-news-delivery-to-discord-with-deduplication-11210


# Automated MIT AI News Delivery to Discord with Deduplication

### 1. Workflow Overview

This workflow automates the daily delivery of the latest MIT Artificial Intelligence news headlines to a Discord channel, ensuring no duplicate articles are posted. It is designed for teams or communities interested in staying updated with fresh AI news from MIT without repetitive notifications. The workflow is structured into three main logical blocks:

- **1.1 Daily Trigger & Fetch**: Triggers the workflow daily at a set time and fetches MIT AI news articles from their RSS feed.
- **1.2 Filtering & Deduplication**: Filters articles published within the last 24 hours and checks against a local n8n Data Table to avoid duplicates.
- **1.3 Notification Delivery**: Sends new, unique articles as formatted messages to a Discord channel via webhook.

Additional setup instructions and notes are provided via sticky notes for user guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger & Fetch

- **Overview:**  
  This block initiates the workflow on a daily schedule and retrieves the latest AI news articles from MIT’s RSS feed.

- **Nodes Involved:**  
  - Scheduled Daily Trigger  
  - mit-artificial-intelligence (articles)  
  - Filter 24h (articles)  
  - Cleaning Up Data

- **Node Details:**  

  **Scheduled Daily Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow execution daily at 9:00 AM by default.  
  - Configuration: Set to trigger once daily at hour 9, minute 0 (modifiable).  
  - Inputs: None (trigger node).  
  - Outputs: Connects to RSS Feed Reader node.  
  - Edge Cases: Misconfigured time zones or disabled scheduling could prevent execution.

  **mit-artificial-intelligence (articles)**  
  - Type: RSS Feed Read  
  - Role: Fetches articles from MIT AI news RSS feed.  
  - Configuration: URL set to https://news.mit.edu/topic/mitartificial-intelligence2-rss.xml.  
  - Inputs: Trigger node output.  
  - Outputs: Emits list of articles with metadata like title, creator, pubDate, and link.  
  - Edge Cases: RSS feed downtime, malformed XML, or connection failures.

  **Filter 24h (articles)**  
  - Type: Filter  
  - Role: Filters articles published in the last 24 hours.  
  - Configuration: DateTime condition comparing article `pubDate` to current time minus 24 hours.  
  - Inputs: RSS feed articles.  
  - Outputs: Passes only recent articles downstream.  
  - Edge Cases: Articles with missing or invalid publication dates may be excluded unintentionally.

  **Cleaning Up Data**  
  - Type: Set  
  - Role: Normalizes and extracts relevant fields (`creator`, `title`, `pubDate`, `link`) as strings.  
  - Configuration: Assigns each field explicitly from incoming JSON data.  
  - Inputs: Filtered articles.  
  - Outputs: Cleaned article data for deduplication.  
  - Edge Cases: Missing fields could cause empty string assignments.

---

#### 1.2 Filtering & Deduplication

- **Overview:**  
  This block avoids re-sending articles previously notified by checking if the article exists in an n8n Data Table, adding new entries if absent.

- **Nodes Involved:**  
  - Avoid Duplicated Articles  
  - Register New Article  
  - Wait 1sec

- **Node Details:**  

  **Avoid Duplicated Articles**  
  - Type: Data Table (Row Not Exists)  
  - Role: Checks if an article with the same `link` and `title` does not exist in the Data Table `mit_ai_news_sent`.  
  - Configuration: Filters on both `link` and `title` columns, operation set to "rowNotExists" to pass only new articles.  
  - Inputs: Cleaned article data.  
  - Outputs: Only unique articles proceed; duplicates stop here.  
  - Edge Cases: If the Data Table is missing or misconfigured, this node could fail or allow duplicates.

  **Register New Article**  
  - Type: Data Table (Insert Row)  
  - Role: Inserts the new article's metadata (`creator`, `title`, `link`, `pubDate`) into the Data Table `mit_ai_news_sent`.  
  - Configuration: Uses dynamic expressions to map article data to table columns.  
  - Inputs: Unique articles from previous node.  
  - Outputs: Passes the article forward after registration.  
  - Edge Cases: Table write permission issues or connectivity problems with Data Table service.

  **Wait 1sec**  
  - Type: Wait  
  - Role: Inserts a 1-second delay before sending notification, possibly to avoid rate limits or to sequence operations.  
  - Configuration: Fixed 1 second wait time.  
  - Inputs: Registered new articles.  
  - Outputs: Delayed output to notification block.  
  - Edge Cases: Delay errors are rare; workflow timing sensitivity should be considered if modified.

---

#### 1.3 Notification Delivery

- **Overview:**  
  Sends a formatted notification message for each new article to a predefined Discord channel via webhook.

- **Nodes Involved:**  
  - MIT AI Articles

- **Node Details:**  

  **MIT AI Articles**  
  - Type: Discord Webhook  
  - Role: Sends article information as a Discord message.  
  - Configuration:  
    - Content template includes title (bold), author (italic), publication date (inline code), and link.  
    - Uses webhook authentication with a configured Discord webhook credential named "MIT AI News".  
  - Inputs: Articles after delay.  
  - Outputs: None (terminal node).  
  - Edge Cases: Discord webhook misconfiguration, rate limits, or network issues could cause message failures.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role              | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                           |
|---------------------------|-------------------------|-----------------------------|----------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------|
| Scheduled Daily Trigger    | Schedule Trigger        | Workflow initiator          | None                             | mit-artificial-intelligence (articles) | ## 0. Daily Trigger - Workflow executes daily at 9am (modifiable)                                   |
| mit-artificial-intelligence (articles) | RSS Feed Read          | Fetch MIT AI News           | Scheduled Daily Trigger          | Filter 24h (articles)                |                                                                                                     |
| Filter 24h (articles)      | Filter                  | Filter articles from last 24h | mit-artificial-intelligence (articles) | Cleaning Up Data                    | ## 1. Fetch & Filter - Get MIT AI News and filter for the last 24h                                  |
| Cleaning Up Data           | Set                     | Normalize article fields    | Filter 24h (articles)            | Avoid Duplicated Articles            |                                                                                                     |
| Avoid Duplicated Articles  | Data Table (Row Not Exists) | Deduplicate articles       | Cleaning Up Data                 | Register New Article                 | ## 2. Deduplication Logic - Check if the link exists in the Data Table, add if new                  |
| Register New Article       | Data Table (Insert Row) | Save new article to Data Table | Avoid Duplicated Articles      | Wait 1sec                          |                                                                                                     |
| Wait 1sec                 | Wait                    | Delay before sending        | Register New Article             | MIT AI Articles                    |                                                                                                     |
| MIT AI Articles            | Discord Webhook          | Send notification to Discord | Wait 1sec                      | None                              | ## 3. Notify - Send to Discord                                                                       |
| Readme Note                | Sticky Note             | Setup instructions          | None                             | None                              | ## ⚠️ SETUP REQUIRED (Read First) - Instructions on setting up Data Table and connecting nodes       |
| Chapter 1                  | Sticky Note             | Describes Fetch & Filter    | None                             | None                              | ## 1. Fetch & Filter - Get MIT AI News and filter for the last 24h                                  |
| Chapter 2                  | Sticky Note             | Describes Deduplication    | None                             | None                              | ## 2. Deduplication Logic - Check if the link exists in the Data Table, add if new                  |
| Chapter 3                  | Sticky Note             | Describes Notification block | None                            | None                              | ## 3. Notify - Send to Discord                                                                       |
| Sticky Note                | Sticky Note             | Describes daily trigger     | None                             | None                              | ## 0. Daily Trigger - Workflow executes daily at 9am                                                |
| Sticky Note13              | Sticky Note             | Overview and testing tips   | None                             | None                              | ## 🧪 Try It Out! - Summary, testing, customization, and requirements                                |
| Sticky Note2               | Sticky Note             | Author information          | None                             | None                              | ## Hey, I'm Vasyl Pavlyuchok - Author bio and contact info                                         |
| Sticky Note6               | Sticky Note             | Author image                | None                             | None                              | ![Vasyl Pavlyuchok](https://assets.zyrosite.com/mp86MKQeprTxrnEW/profile-pic-2025_10-Yan0j4jpxeUzzGlm.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Data Table:**  
   - Go to n8n dashboard > Data Tables.  
   - Create a new table named `mit_ai_news_sent` with columns:  
     - `creator` (String)  
     - `title` (String)  
     - `link` (String)  
     - `pubDate` (Date)  

2. **Add Scheduled Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9:00 AM (adjust as needed).  
   - No credentials required.

3. **Add RSS Feed Read Node:**  
   - Type: RSS Feed Read  
   - URL: `https://news.mit.edu/topic/mitartificial-intelligence2-rss.xml`  
   - Connect Scheduled Trigger output to this node.

4. **Add Filter Node:**  
   - Type: Filter  
   - Condition:  
     - Field: `pubDate` (date-time)  
     - Operation: After or equals  
     - Value: Current date/time minus 24 hours (set expression accordingly, e.g., `={{ (new Date(Date.now() - 24*60*60*1000)).toISOString() }}`)  
   - Connect RSS Feed Read output to this node.

5. **Add Set Node (Cleaning Up Data):**  
   - Type: Set  
   - Fields to assign:  
     - `creator` = `{{$json.creator}}`  
     - `title` = `{{$json.title}}`  
     - `pubDate` = `{{$json.pubDate}}`  
     - `link` = `{{$json.link}}`  
   - Connect Filter output to this node.

6. **Add Data Table Node (Avoid Duplicated Articles):**  
   - Type: Data Table  
   - Operation: Row Not Exists  
   - Data Table: Select `mit_ai_news_sent`  
   - Filters (all conditions):  
     - `link` equals `{{$json.link}}`  
     - `title` equals `{{$json.title}}`  
   - Connect Set node output to this node.

7. **Add Data Table Node (Register New Article):**  
   - Type: Data Table  
   - Operation: Insert Row  
   - Data Table: Select `mit_ai_news_sent`  
   - Map fields:  
     - `creator` = `{{$json.creator}}`  
     - `title` = `{{$json.title}}`  
     - `link` = `{{$json.link}}`  
     - `pubDate` = `{{$json.pubDate}}`  
   - Connect previous Data Table output to this node.

8. **Add Wait Node:**  
   - Type: Wait  
   - Duration: 1 second  
   - Connect Register New Article output to this node.

9. **Add Discord Node:**  
   - Type: Discord  
   - Authentication: Webhook (create/use a Discord webhook credential named "MIT AI News")  
   - Content template:  
     ```
     Title: **{{ $json.title }}**
     Author: *{{ $json.creator }}*
     Date: `{{ $json.pubDate }}`
     Link: {{ $json.link }}
     ```  
   - Connect Wait node output to this node.

10. **Test the Workflow:**  
    - Ensure Data Table and Discord credentials are properly configured.  
    - Run the workflow manually or wait for scheduled trigger.  
    - Verify that new articles published in the last 24 hours are posted to Discord without duplicates.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| ## ⚠️ SETUP REQUIRED (Read First) - Create a Data Table `mit_ai_news_sent` with required columns before running.                 | Setup instructions sticky note in workflow.                                                             |
| ## 🧪 Try It Out! - Explains workflow purpose, testing instructions, and customization tips including schedule and RSS source.    | Overview and test guidance sticky note.                                                                  |
| ## Hey, I'm Vasyl Pavlyuchok - Designer of AI-powered automation systems with contact info and social links.                    | Author bio sticky note.                                                                                   |
| Video tutorials and automation tips can be found on YouTube: [https://www.youtube.com/@vasylpavlyuchok](https://www.youtube.com/@vasylpavlyuchok) | Author's YouTube channel linked in sticky notes.                                                        |
| LinkedIn profile for professional connection: [https://www.linkedin.com/in/vasyl-pavlyuchok/](https://www.linkedin.com/in/vasyl-pavlyuchok/) | Author's LinkedIn linked in sticky notes.                                                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.