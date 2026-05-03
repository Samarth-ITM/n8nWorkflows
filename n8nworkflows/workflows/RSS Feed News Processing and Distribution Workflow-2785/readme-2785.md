RSS Feed News Processing and Distribution Workflow

https://n8nworkflows.xyz/workflows/rss-feed-news-processing-and-distribution-workflow-2785


# RSS Feed News Processing and Distribution Workflow

### 1. Workflow Overview

This workflow automates the process of monitoring multiple RSS feeds, filtering recent news articles, formatting them, and distributing updates as comments on a Trello card while notifying a moderator via email. It is designed for professionals and teams such as content managers, marketers, and team leads who need to stay updated on relevant news without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Scheduled trigger initiates the workflow and reads multiple RSS feeds simultaneously.
- **1.2 Data Aggregation and Normalization:** Merges RSS feed data and transforms raw feed items into a consistent format with key fields.
- **1.3 Filtering and Sorting:** Filters articles by publication date (default last 7 days), sorts them descending by date, and limits the number of articles.
- **1.4 Formatting:** Converts the filtered news items into Markdown format for readability.
- **1.5 Publishing and Notification:** Posts the formatted news as a comment on a specified Trello card and sends an email notification to a moderator.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow on a weekly schedule and reads data from three distinct RSS feeds concurrently.

- **Nodes Involved:**  
  - Schedule Trigger  
  - RSS Read Testing Catalog  
  - RSS Read marktechpost  
  - RSS Read  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution weekly on Monday at 7 AM.  
    - Configuration: Interval set to weekly, triggering at day 1 (Monday), hour 7.  
    - Inputs: None (start node)  
    - Outputs: Connects to all three RSS Read nodes simultaneously.  
    - Edge Cases: Misconfigured schedule could cause missed or multiple triggers.

  - **RSS Read Testing Catalog**  
    - Type: RSS Feed Read  
    - Role: Reads RSS feed from https://www.testingcatalog.com/rss/  
    - Configuration: URL set; SSL errors ignored (ignoreSSL=true).  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Feeds data into the Merge node (index 2).  
    - Edge Cases: Network errors, invalid RSS format, SSL issues.

  - **RSS Read marktechpost**  
    - Type: RSS Feed Read  
    - Role: Reads RSS feed from https://www.marktechpost.com/feed/  
    - Configuration: URL set; default SSL behavior.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Feeds data into the Merge node (index 1).  
    - Edge Cases: Network errors, invalid RSS format.

  - **RSS Read**  
    - Type: RSS Feed Read  
    - Role: Reads RSS feed from https://www.artificial-intelligence.blog/ai-news?format=rss  
    - Configuration: URL set; default SSL behavior.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Feeds data into the Merge node (index 0).  
    - Edge Cases: Network errors, invalid RSS format.

#### 2.2 Data Aggregation and Normalization

- **Overview:**  
  This block merges the three RSS feed outputs into a single data stream and normalizes each news item by extracting and standardizing key fields.

- **Nodes Involved:**  
  - Merge  
  - Transform date  

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines three input streams from RSS Read nodes into one unified stream.  
    - Configuration: Number of inputs set to 3; merges all inputs.  
    - Inputs: From the three RSS Read nodes.  
    - Outputs: To Transform date node.  
    - Edge Cases: Missing input from any RSS feed may reduce data volume; node expects all three inputs.

  - **Transform date**  
    - Type: Set  
    - Role: Extracts and standardizes fields from each RSS item for downstream processing.  
    - Configuration: Assigns fields:  
      - title (string) from item title  
      - content (string) from contentSnippet  
      - link (string) from link  
      - isoDate (number) as timestamp from isoDate string  
      - categories (array) from categories  
    - Inputs: From Merge node.  
    - Outputs: To Filter by date node.  
    - Edge Cases: Missing or malformed isoDate may cause timestamp conversion errors.

#### 2.3 Filtering and Sorting

- **Overview:**  
  Filters news items to include only those published within the last 7 days, sorts them by date descending, and limits the total number of articles to 10.

- **Nodes Involved:**  
  - Filter by date (more than 7 days)  
  - Sort by date  
  - Limit news to x  

- **Node Details:**

  - **Filter by date (more than 7 days)**  
    - Type: Filter  
    - Role: Filters out news items older than 7 days.  
    - Configuration: Condition checks if isoDate timestamp is greater than (current time - 7 days in milliseconds).  
    - Inputs: From Transform date node.  
    - Outputs: To Sort by date node.  
    - Edge Cases: Incorrect date calculation or missing isoDate may filter out valid items.

  - **Sort by date**  
    - Type: Sort  
    - Role: Sorts filtered news items by isoDate descending (newest first).  
    - Configuration: Sort field isoDate, order descending.  
    - Inputs: From Filter by date node.  
    - Outputs: To Limit news to x node.  
    - Edge Cases: Items with missing isoDate may be sorted incorrectly.

  - **Limit news to x**  
    - Type: Limit  
    - Role: Limits output to maximum 10 news items.  
    - Configuration: maxItems set to 10.  
    - Inputs: From Sort by date node.  
    - Outputs: To Transform new to MD node.  
    - Edge Cases: If fewer than 10 items, all pass through.

#### 2.4 Formatting

- **Overview:**  
  Converts the limited news items into a single Markdown-formatted string for posting.

- **Nodes Involved:**  
  - Transform new to MD  

- **Node Details:**

  - **Transform new to MD**  
    - Type: Code (JavaScript)  
    - Role: Iterates over all news items and concatenates them into a Markdown list with links and content snippets.  
    - Configuration: Custom JS code loops through all input items, building a Markdown string with format:  
      `- [title](link "‌"): \ncontent\n\n`  
    - Inputs: From Limit news to x node.  
    - Outputs: To Publish comment node.  
    - Edge Cases: Empty input results in empty string; malformed fields may affect formatting.

#### 2.5 Publishing and Notification

- **Overview:**  
  Posts the formatted Markdown news update as a comment on a specified Trello card and sends an email notification to a moderator.

- **Nodes Involved:**  
  - Publish comment  
  - Send revision email  

- **Node Details:**

  - **Publish comment**  
    - Type: Trello  
    - Role: Posts the Markdown news update as a comment on a Trello card.  
    - Configuration:  
      - Text set to the Markdown string from previous node.  
      - cardId set to "dFtYLRXv" (specific Trello card).  
      - Resource: cardComment.  
    - Credentials: Uses Trello API credentials named "Trello account".  
    - Inputs: From Transform new to MD node.  
    - Outputs: To Send revision email node.  
    - Edge Cases: Trello API errors, invalid card ID, auth failures.

  - **Send revision email**  
    - Type: Gmail  
    - Role: Sends a notification email to moderator to review the Trello comment.  
    - Configuration:  
      - Recipient: thomas@pollup.net  
      - Subject: "Update for Trello done"  
      - Message: Notification text referencing Trello card URL.  
      - Email type: text/plain.  
    - Credentials: Uses Gmail OAuth2 credentials named "Gmail account".  
    - Inputs: From Publish comment node.  
    - Outputs: None (end node).  
    - Edge Cases: Email sending failures, auth errors.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                         | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                                            |
|---------------------------|---------------------|---------------------------------------|----------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger    | Initiates workflow weekly             | None                             | RSS Read Testing Catalog, RSS Read marktechpost, RSS Read |                                                                                                                        |
| RSS Read Testing Catalog   | RSS Feed Read       | Reads Testing Catalog RSS feed        | Schedule Trigger                 | Merge                         | ## RSS sources \nHere you can add up to nine sources of RSS. Modify merge node and duplicate RSS nodes accordingly.    |
| RSS Read marktechpost      | RSS Feed Read       | Reads Marktechpost RSS feed            | Schedule Trigger                 | Merge                         | ## RSS sources \nHere you can add up to nine sources of RSS. Modify merge node and duplicate RSS nodes accordingly.    |
| RSS Read                  | RSS Feed Read       | Reads Artificial Intelligence Blog RSS| Schedule Trigger                 | Merge                         | ## RSS sources \nHere you can add up to nine sources of RSS. Modify merge node and duplicate RSS nodes accordingly.    |
| Merge                     | Merge               | Combines all RSS feed outputs          | RSS Read Testing Catalog, RSS Read marktechpost, RSS Read | Transform date                | ## RSS sources \nHere you can add up to nine sources of RSS. Modify merge node and duplicate RSS nodes accordingly.    |
| Transform date            | Set                 | Normalizes RSS item fields             | Merge                           | Filter by date (more than 7 days) | ## Age and number of the news \nChange 7 days in filter node; adjust limit news node for number of articles.            |
| Filter by date (more than 7 days) | Filter         | Filters news older than 7 days         | Transform date                  | Sort by date                  | ## Age and number of the news \nChange 7 days in filter node; adjust limit news node for number of articles.            |
| Sort by date              | Sort                | Sorts news descending by date          | Filter by date (more than 7 days) | Limit news to x               | ## Age and number of the news \nChange 7 days in filter node; adjust limit news node for number of articles.            |
| Limit news to x           | Limit               | Limits news to 10 items                 | Sort by date                   | Transform new to MD           | ## Age and number of the news \nChange 7 days in filter node; adjust limit news node for number of articles.            |
| Transform new to MD       | Code                | Formats news items into Markdown       | Limit news to x                | Publish comment              |                                                                                                                        |
| Publish comment           | Trello              | Posts Markdown news as Trello comment  | Transform new to MD             | Send revision email          |                                                                                                                        |
| Send revision email       | Gmail               | Sends notification email to moderator  | Publish comment                | None                         |                                                                                                                        |
| Sticky Note               | Sticky Note         | Notes on RSS sources and merge node    | None                          | None                         | ## RSS sources \nHere you can add up to nine sources of RSS. Modify merge node and duplicate RSS nodes accordingly.    |
| Sticky Note1              | Sticky Note         | Notes on date filtering and news limit | None                          | None                         | ## Age and number of the news \nChange 7 days in filter node; adjust limit news node for number of articles.            |
| Sticky Note2              | Sticky Note         | Workflow description and usage details | None                          | None                         | ## RSS Feed News Processing and Distribution Workflow\n\nFull workflow purpose, setup, and customization instructions.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure: Set interval to weekly, trigger on Monday at 7:00 AM.

2. **Create Three RSS Feed Read Nodes**  
   - Node 1:  
     - Name: RSS Read Testing Catalog  
     - Type: RSS Feed Read  
     - URL: https://www.testingcatalog.com/rss/  
     - Options: Ignore SSL errors enabled.  
     - Connect input from Schedule Trigger.  
   - Node 2:  
     - Name: RSS Read marktechpost  
     - Type: RSS Feed Read  
     - URL: https://www.marktechpost.com/feed/  
     - Default SSL options.  
     - Connect input from Schedule Trigger.  
   - Node 3:  
     - Name: RSS Read  
     - Type: RSS Feed Read  
     - URL: https://www.artificial-intelligence.blog/ai-news?format=rss  
     - Default SSL options.  
     - Connect input from Schedule Trigger.

3. **Create Merge Node**  
   - Type: Merge  
   - Configure: Number of inputs = 3  
   - Connect inputs from the three RSS Read nodes (assign each to one input index).  
   - Connect output to Transform date node.

4. **Create Transform date Node**  
   - Type: Set  
   - Configure assignments:  
     - title: `{{$json["title"]}}`  
     - content: `{{$json["contentSnippet"]}}`  
     - link: `{{$json["link"]}}`  
     - isoDate: `{{ new Date($json["isoDate"]).getTime() }}` (number)  
     - categories: `{{$json["categories"]}}` (array)  
   - Connect input from Merge node.  
   - Connect output to Filter by date node.

5. **Create Filter by date Node**  
   - Type: Filter  
   - Configure condition:  
     - isoDate > `Date.now() - 7 * 24 * 60 * 60 * 1000` (filters last 7 days)  
   - Connect input from Transform date node.  
   - Connect output to Sort by date node.

6. **Create Sort by date Node**  
   - Type: Sort  
   - Configure: Sort by field isoDate, order descending.  
   - Connect input from Filter by date node.  
   - Connect output to Limit news to x node.

7. **Create Limit news to x Node**  
   - Type: Limit  
   - Configure: maxItems = 10  
   - Connect input from Sort by date node.  
   - Connect output to Transform new to MD node.

8. **Create Transform new to MD Node**  
   - Type: Code (JavaScript)  
   - Paste the following JS code:  
     ```javascript
     let ret = "";
     for (const item of $input.all()) {
       ret += `- [${item.json.title}](${item.json.link} "‌"): \n${item.json.content}\n\n`;
     }
     return { data: ret };
     ```  
   - Connect input from Limit news to x node.  
   - Connect output to Publish comment node.

9. **Create Publish comment Node**  
   - Type: Trello  
   - Configure:  
     - Resource: cardComment  
     - cardId: "dFtYLRXv" (replace with your Trello card ID)  
     - Text: `{{$json["data"]}}` (Markdown string from previous node)  
   - Set Trello credentials (OAuth or API key/token).  
   - Connect input from Transform new to MD node.  
   - Connect output to Send revision email node.

10. **Create Send revision email Node**  
    - Type: Gmail  
    - Configure:  
      - Send To: thomas@pollup.net (replace with your moderator email)  
      - Subject: "Update for Trello done"  
      - Message: "The Trello comment for https://trello.com/c/dFtYLRXv has been updated. Please check."  
      - Email Type: Text  
    - Set Gmail OAuth2 credentials.  
    - Connect input from Publish comment node.

11. **Optional: Add Sticky Notes**  
    - Add notes for RSS sources, date filtering, and workflow description for documentation and maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| ## RSS sources \nHere you can add up to nine sources of RSS. To do so, modify the merge node for the number of RSS feeds you want, duplicate the RSS node and wire it to the trigger and the merge node.                                                                                                                                                                                                                                                | Sticky Note attached to RSS Read nodes and Merge node.                                                |
| ## Age and number of the news \nHere you can set the number of days behind by changing the 7 by any number in the filter by date node: `Date.now() - 7 * 24 * 60 * 60 * 1000` \nYou can also modify the number of news in the "limit news to x" node.                                                                                                                                                                                                                 | Sticky Note attached to Transform date, Filter by date, Sort by date, and Limit news nodes.           |
| ## RSS Feed News Processing and Distribution Workflow\n\nThis workflow is designed for professionals and teams who need to monitor multiple RSS feeds, filter the latest content, and distribute actionable updates as a Trello comment. Ideal for content managers, marketers, and team leads managing news or content pipelines.\n\nManually monitoring RSS feeds and keeping track of the latest content can be time-consuming. This workflow automates the aggregation, filtering, and distribution of news, ensuring that only relevant and timely updates are shared with your team or audience.\n\nWhat this workflow does:\n1. Aggregates RSS Feeds: Pulls data from up to three RSS feeds simultaneously.\n2. Filters Content: Filters articles based on their publication date (default: last 7 days).\n3. Organizes and Sorts: Sorts filtered articles by date for clarity.\n4. Formats Updates: Transforms news items into Markdown format for better readability.\n5. Publishes and Notifies: Posts comments to Trello cards and sends an email to a moderator to check the comment.\n\nSetup:\n1. Connect your RSS feeds by configuring the RSS Read nodes.\n2. Link your Trello and Gmail accounts for seamless integration.\n3. Adjust the schedule trigger to set how often the workflow should run (e.g., daily, weekly).\n4. Test the workflow to ensure all connections and configurations are correct.\n\nHow to customize this workflow to your needs:\n- Change the Number of RSS Feeds: Add or remove RSS Read nodes and update the merge configuration accordingly.\n- Adjust the Date Filter: Modify the date logic in the “Filter by date” node to include more or fewer days.\n- Limit the Number of Articles: Adjust the limit in the “Limit news to x” node.\n- Custom Formatting: Update the Transform node to format the news items differently.\n- Alternative Notifications: Replace Trello and Gmail with other integrations, such as Slack or Microsoft Teams.\n\nThis workflow ensures your team stays informed with minimal effort and delivers content updates in an organized and professional manner. | Sticky Note attached to the workflow overview area (large note).                                      |

---

This documentation provides a complete, structured reference for understanding, reproducing, and modifying the "RSS Feed News Processing and Distribution Workflow" in n8n. It covers all nodes, their configurations, connections, and potential edge cases, enabling both advanced users and automation agents to work effectively with this workflow.