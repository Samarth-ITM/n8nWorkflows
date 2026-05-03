Post New Articles from Feeds to Slack Channel

https://n8nworkflows.xyz/workflows/post-new-articles-from-feeds-to-slack-channel-5333


# Post New Articles from Feeds to Slack Channel

### 1. Workflow Overview

This workflow automates the discovery and sharing of new articles from a curated list of RSS feeds by posting them to a Slack channel. It is designed primarily for marketing teams, founders, or professionals who want to monitor industry news and share updates internally without duplication.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Feed Acquisition:** Scheduled trigger to start the workflow daily, fetching the list of RSS feeds from a Google Sheet.
- **1.2 Article Retrieval:** Reading the latest articles from each RSS feed.
- **1.3 Duplicate Filtering:** Comparing retrieved articles against previously posted articles to filter out duplicates.
- **1.4 Logging & Posting:** Appending new articles to the Google Sheet log and posting them to the designated Slack channel.
- **1.5 Documentation:** Embedded sticky note providing detailed explanation, prerequisites, and usage context.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Feed Acquisition

**Overview:**  
This block initiates the workflow on a scheduled basis and fetches the list of RSS feed URLs from a Google Sheet.

**Nodes Involved:**  
- Trigger Workflow (Cron)  
- Get Article Feeds (Google Sheets)

**Node Details:**

- **Trigger Workflow**  
  - Type: Cron Trigger  
  - Role: Starts workflow execution daily at 7:00 AM.  
  - Configuration: Trigger time set to hour 7 (7:00 AM), runs once per day.  
  - Inputs: None  
  - Outputs: Connects to "Get Article Feeds" node.  
  - Edge Cases: Cron misconfiguration; workflow not triggering if n8n instance is down at scheduled time.

- **Get Article Feeds**  
  - Type: Google Sheets Read  
  - Role: Reads the list of RSS feeds from a Google Sheet tab named "Feeds".  
  - Configuration: Reads the sheet with ID specified by the n8n variable `{{$vars.Daily_Industry_News_Automation_Google_Sheet}}`; targeting sheet tab with GID 1768028583 ("Feeds").  
  - Inputs: Trigger Workflow node output (empty data, trigger).  
  - Outputs: Provides feed URLs for the next step.  
  - Edge Cases: OAuth2 authentication failure with Google Sheets; sheet or tab missing; empty feed list.

---

#### 2.2 Article Retrieval

**Overview:**  
Reads the latest articles from each RSS feed URL obtained from the previous block.

**Nodes Involved:**  
- Read Latest Articles from Feeds (RSS Feed Read)  
- Get Historically Posted Articles from Google Sheet (Google Sheets Read)

**Node Details:**

- **Read Latest Articles from Feeds**  
  - Type: RSS Feed Read  
  - Role: Fetches articles from each feed URL.  
  - Configuration: Dynamically uses the `link` field from each feed entry as the RSS URL to read.  
  - Inputs: Output from "Get Article Feeds" (list of feed URLs).  
  - Outputs: Article entries from each feed.  
  - Edge Cases: Invalid or unreachable RSS URLs; feed format errors; network timeouts.

- **Get Historically Posted Articles from Google Sheet**  
  - Type: Google Sheets Read  
  - Role: Retrieves the list of articles already posted, used for duplicate filtering.  
  - Configuration: Reads from the Google Sheet tab with GID 0 ("Posted Articles") in the same spreadsheet as feeds.  
  - Inputs: Output from "Read Latest Articles from Feeds" (feeds data).  
  - Outputs: Previously posted article metadata (title, link, pubDate).  
  - Additional: `executeOnce` is true to avoid redundant calls during batch processing.  
  - Edge Cases: OAuth2 authentication failure; sheet missing or empty; data format inconsistencies.

---

#### 2.3 Duplicate Filtering

**Overview:**  
Filters out articles that have already been posted to avoid duplicates in Slack posts.

**Nodes Involved:**  
- Filter Unpublished Articles (Code)

**Node Details:**

- **Filter Unpublished Articles**  
  - Type: Code  
  - Role: Filters new articles by removing any whose links already exist in the posted articles list.  
  - Configuration: Uses JavaScript to extract the `link` field from previously posted articles and filters incoming articles accordingly.  
  - Key Expression:  
    ```js
    const sheetLinks = $('Get Historically Posted Articles from Google Sheet').all().map(row => row.json.link);
    const articles = $('Read Latest Articles from Feeds').all();
    return articles.filter(item => !sheetLinks.includes(item.json.link));
    ```  
  - Inputs: Data from "Read Latest Articles from Feeds" and "Get Historically Posted Articles from Google Sheet".  
  - Outputs: List of articles not yet posted.  
  - Edge Cases: Missing fields (`link`) in articles, resulting in false negatives; large volume causing performance issues; expression syntax errors.

---

#### 2.4 Logging & Posting

**Overview:**  
Logs the new articles into the Google Sheet to prevent reposting and simultaneously posts them to the Slack channel.

**Nodes Involved:**  
- Append New Articles to Google Sheet (Google Sheets Append)  
- Post New Articles to Slack Channel (Slack)

**Node Details:**

- **Append New Articles to Google Sheet**  
  - Type: Google Sheets Append  
  - Role: Adds newly identified articles to the "Posted Articles" Google Sheet tab to maintain posting history.  
  - Configuration: Appends `link`, `title`, and `pubDate` fields from each article to the sheet with GID 0.  
  - Inputs: Output from "Filter Unpublished Articles".  
  - Outputs: Data forwarded to Slack posting node.  
  - Edge Cases: OAuth2 failure; schema mismatch; concurrency issues if multiple runs overlap.

- **Post New Articles to Slack Channel**  
  - Type: Slack Message Post  
  - Role: Posts a formatted message with article title, link, and publication date to the specified Slack channel.  
  - Configuration:  
    - Authentication: OAuth2.  
    - Channel ID: `C09281Y80AH` (hardcoded to "daily-industry-news-automation").  
    - Message Text: `"=*{{$json["title"]}}* | {{$json["link"]}} _Published: {{$json["pubDate"]}}_"` — bold title, link, and italicized publish date.  
  - Inputs: Output from "Append New Articles to Google Sheet".  
  - Outputs: None (end of flow).  
  - Edge Cases: Slack API rate limits; authentication token expiration; invalid channel ID.

---

#### 2.5 Documentation

**Overview:**  
Provides a comprehensive sticky note within the workflow describing its purpose, prerequisites, setup, and intended use cases.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Inline documentation for users and maintainers.  
  - Content Highlights:  
    - Workflow purpose and stepwise process.  
    - OAuth2 credential prerequisites for Google Sheets and Slack.  
    - Google Sheet tab and column naming conventions.  
    - Environment variables used.  
    - Trigger schedule details.  
    - Intended use cases such as marketing digests and internal communication.  
  - Position: Top-left corner for visibility.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                             |
|-----------------------------------|---------------------|-----------------------------------------------|--------------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------|
| Trigger Workflow                  | Cron Trigger        | Starts the workflow daily at 7:00 AM          | None                                 | Get Article Feeds                      | cron-based Workflow trigger. Recommend once per day.                                                  |
| Get Article Feeds                 | Google Sheets       | Reads RSS feed URLs from "Feeds" tab          | Trigger Workflow                     | Read Latest Articles from Feeds        | Get a list of RSS feeds to poll from the Feeds Google Sheet (title, link).                            |
| Read Latest Articles from Feeds  | RSS Feed Read       | Fetches latest articles from each feed URL    | Get Article Feeds                   | Get Historically Posted Articles from Google Sheet | Get articles from each feed in Google Sheet.                                                   |
| Get Historically Posted Articles from Google Sheet | Google Sheets | Retrieves posted articles to check duplicates | Read Latest Articles from Feeds      | Filter Unpublished Articles             | Get a list of articles previously posted from Posted Articles Google Sheet (title, link, pubDate).    |
| Filter Unpublished Articles       | Code                | Filters out articles already posted            | Read Latest Articles from Feeds, Get Historically Posted Articles from Google Sheet | Append New Articles to Google Sheet    | Filter previously published articles from those received from the feeds.                             |
| Append New Articles to Google Sheet | Google Sheets       | Logs new articles to "Posted Articles" tab    | Filter Unpublished Articles          | Post New Articles to Slack Channel     | Append the un-published articles to the Published Articles Google Sheet.                             |
| Post New Articles to Slack Channel | Slack               | Posts new articles to Slack channel            | Append New Articles to Google Sheet  | None                                  | Post new articles to Slack Channel.                                                                  |
| Sticky Note                      | Sticky Note         | Documentation and workflow explanation         | None                                 | None                                  | # 📄 Post New Articles from Feeds to Slack Channel ... (full content as described in section 2.5)     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node:**  
   - Type: Cron  
   - Set to trigger daily at 7:00 AM (Hour: 7).  
   - Name it "Trigger Workflow".

2. **Create a Google Sheets node to read feeds:**  
   - Type: Google Sheets (Read)  
   - Name: "Get Article Feeds"  
   - Credentials: Configure OAuth2 for Google Sheets.  
   - Document ID: Use a variable or hardcode your Google Sheet ID containing feeds.  
   - Sheet Name: Set to the GID or name of the "Feeds" tab (e.g., `1768028583`).  
   - Columns expected: `title`, `link`.  
   - Connect "Trigger Workflow" output to this node.

3. **Create an RSS Feed Read node:**  
   - Type: RSS Feed Read  
   - Name: "Read Latest Articles from Feeds"  
   - URL: Use expression to get the feed URL dynamically: `={{ $json.link }}`  
   - Connect "Get Article Feeds" output to this node.

4. **Create another Google Sheets node to read posted articles:**  
   - Type: Google Sheets (Read)  
   - Name: "Get Historically Posted Articles from Google Sheet"  
   - Credentials: Same Google Sheets OAuth2 credential.  
   - Document ID: Same as feeds sheet.  
   - Sheet Name: GID or name of "Posted Articles" tab (`gid=0`).  
   - Columns expected: `title`, `link`, `pubDate`.  
   - Set `Execute Once` to true.  
   - Connect "Read Latest Articles from Feeds" output to this node.

5. **Create a Code node to filter unpublished articles:**  
   - Type: Code  
   - Name: "Filter Unpublished Articles"  
   - JavaScript code:  
     ```js
     const sheetLinks = $('Get Historically Posted Articles from Google Sheet').all().map(row => row.json.link);
     const articles = $('Read Latest Articles from Feeds').all();
     return articles.filter(item => !sheetLinks.includes(item.json.link));
     ```  
   - Connect outputs of "Read Latest Articles from Feeds" and "Get Historically Posted Articles from Google Sheet" as inputs to this node.

6. **Create a Google Sheets Append node:**  
   - Type: Google Sheets (Append)  
   - Name: "Append New Articles to Google Sheet"  
   - Credentials: Google Sheets OAuth2.  
   - Document ID: Same spreadsheet.  
   - Sheet Name: "Posted Articles" tab (GID `0`).  
   - Mapping: Append fields `link`, `title`, and `pubDate` from incoming JSON data.  
   - Connect output from "Filter Unpublished Articles" node.

7. **Create a Slack node to post messages:**  
   - Type: Slack  
   - Name: "Post New Articles to Slack Channel"  
   - Credentials: Configure OAuth2 for Slack.  
   - Channel: Set to your target Slack channel ID (e.g., `C09281Y80AH`).  
   - Message text: Use expression:  
     ```  
     =*{{$json["title"]}}* | {{$json["link"]}} _Published: {{$json["pubDate"]}}_  
     ```  
   - Connect output from "Append New Articles to Google Sheet".

8. **Optionally, add a Sticky Note node for documentation:**  
   - Type: Sticky Note  
   - Name: "Sticky Note"  
   - Content: Include workflow purpose, prerequisites, environment variables, and scheduling instructions as per internal documentation.

9. **Verify the entire flow connections:**  
   - Trigger Workflow → Get Article Feeds → Read Latest Articles from Feeds → Get Historically Posted Articles from Google Sheet → Filter Unpublished Articles → Append New Articles to Google Sheet → Post New Articles to Slack Channel.

10. **Test the workflow:**  
    - Run manually or wait for scheduled execution to ensure articles are fetched, filtered, logged, and posted correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Google Sheets OAuth2 and Slack OAuth2 credentials must be configured in n8n before running this workflow.                  | Credential setup instructions within n8n documentation.                                        |
| The Google Sheet must have two tabs named `Feeds` (with columns `title`, `link`) and `Posted Articles` (with `title`, `link`, `pubDate`). | Spreadsheet setup requirement.                                                                 |
| Slack channel ID `C09281Y80AH` is hardcoded but should be replaced with your actual Slack channel ID for posting messages. | Slack channel configuration.                                                                   |
| Cron trigger is set by default to 7:00 AM daily but can be adjusted in the "Trigger Workflow" node to suit business hours. | Scheduling customization.                                                                       |
| Workflow is ideal for marketing teams and founders to automate curated news postings and avoid duplicate article shares.   | Use case summary from sticky note.                                                             |
| More details and instructions are embedded in the workflow’s sticky note for easy reference by maintainers and users.     | Sticky Note node content as comprehensive internal documentation.                               |

---

**Disclaimer:**  
The content provided is extracted exclusively from an automated n8n workflow with strict adherence to content policies. All data processed is legal and publicly accessible.