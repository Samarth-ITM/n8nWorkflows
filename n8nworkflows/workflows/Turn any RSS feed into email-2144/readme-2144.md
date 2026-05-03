Turn any RSS feed into email

https://n8nworkflows.xyz/workflows/turn-any-rss-feed-into-email-2144


# Turn any RSS feed into email

### 1. Workflow Overview

This workflow automates the transformation of any RSS feed into an email newsletter by sending an email notification every time a new post is published in the last hour from the specified RSS feeds. It targets users who want to keep up-to-date with blogs or websites that do not have a native newsletter feature, delivering fresh content directly to their inbox every hour.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Periodically initiates the process every hour.
- **1.2 RSS Feeds Management**: Defines and splits multiple RSS feed URLs to process them individually.
- **1.3 RSS Feed Reading and Filtering**: Reads each RSS feed, filters posts published within the last hour.
- **1.4 Email Dispatch**: Sends an email notification for each new post detected.
- **1.5 Auxiliary and Control**: Handles batch processing and no-op fallbacks.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution every hour at the 30th minute, ensuring regular polling of RSS feeds for new content.

- **Nodes Involved:**  
  - Every 1 hour

- **Node Details:**  
  - **Node Name:** Every 1 hour  
  - **Type:** Schedule Trigger  
  - **Configuration:**  
    - Trigger set to every hour at minute 30 (i.e., 00:30, 01:30, 02:30, etc.)  
  - **Input/Output:**  
    - No input, initiates the workflow  
    - Output connects to "List of RSS feeds" node  
  - **Edge Cases:**  
    - If n8n server is offline during trigger time, the execution will be missed until next trigger.  
  - **Version:** 1.1

---

#### 1.2 RSS Feeds Management

- **Overview:**  
  This block manages the list of RSS feeds to be monitored, splitting them for individual processing.

- **Nodes Involved:**  
  - List of RSS feeds  
  - Split Out

- **Node Details:**  

  - **Node Name:** List of RSS feeds  
    - **Type:** Set  
    - **Configuration:**  
      - Defines an array field named `urls` containing RSS feed URLs.  
      - Default feeds included:  
        - https://www.anildash.com/feed.xml  
        - https://sive.rs/en.atom  
    - **Input/Output:**  
      - Input from "Every 1 hour" trigger  
      - Output to "Split Out" node  
    - **Edge Cases:**  
      - Ensure URLs are valid and accessible; invalid URLs will cause downstream errors in RSS reading.  
    - **Version:** 3.3

  - **Node Name:** Split Out  
    - **Type:** Split Out  
    - **Configuration:**  
      - Splits the array field `urls` into individual items, each containing a single RSS feed URL.  
    - **Input/Output:**  
      - Input from "List of RSS feeds"  
      - Output to "Loop Over Items" node  
    - **Edge Cases:**  
      - Empty or malformed array may lead to no iterations or errors.  
    - **Version:** 1

---

#### 1.3 RSS Feed Reading and Filtering

- **Overview:**  
  This block processes each RSS feed URL, reads its feed, and filters posts published within the last hour.

- **Nodes Involved:**  
  - Loop Over Items  
  - RSS Read  
  - If published in the last hour  
  - No Operation, do nothing

- **Node Details:**  

  - **Node Name:** Loop Over Items  
    - **Type:** Split In Batches  
    - **Configuration:**  
      - Processes incoming items one by one (default batch size) to handle each RSS feed URL separately.  
    - **Input/Output:**  
      - Input from "Split Out"  
      - Output to two nodes: "No Operation, do nothing" (for empty batches or fallback) and "RSS Read"  
    - **Edge Cases:**  
      - Batch size or concurrency misconfiguration can lead to delays or overload.  
    - **Version:** 3

  - **Node Name:** RSS Read  
    - **Type:** RSS Feed Read  
    - **Configuration:**  
      - Reads RSS feed from URL provided dynamically via expression: `={{ $json.urls }}`  
      - Retries on failure with 5-second intervals.  
      - Continues workflow even if feed retrieval fails (`onError` set to `continueRegularOutput`).  
    - **Input/Output:**  
      - Input from "Loop Over Items"  
      - Output to "If published in the last hour"  
    - **Edge Cases:**  
      - Network issues, invalid feed formats, or unavailable feeds can cause read failures.  
      - Retry attempts might delay workflow.  
    - **Version:** 1

  - **Node Name:** If published in the last hour  
    - **Type:** If (Conditional)  
    - **Configuration:**  
      - Filters posts where the publication date (`isoDate`) is:  
        - After (more recent than) one hour ago (`DateTime.now().minus({hour: 1})`)  
        - Before or equal to current time (`DateTime.now()`)  
      - Uses ISO date parsing for accurate datetime comparison.  
    - **Input/Output:**  
      - Input from "RSS Read"  
      - Output (true branch) to "Send email with each post"  
      - No false branch connection (posts older than 1 hour are ignored)  
    - **Edge Cases:**  
      - Posts without `isoDate` field will fail the expression and be excluded.  
      - Timezone inconsistencies in RSS feed may affect filtering accuracy.  
    - **Version:** 2

  - **Node Name:** No Operation, do nothing  
    - **Type:** No Operation  
    - **Configuration:**  
      - Placeholder node, does nothing but allows workflow control and visualization.  
    - **Input/Output:**  
      - Input from "Loop Over Items" (used as a fallback or control branch)  
      - No outputs  
    - **Edge Cases:**  
      - None, purely passive.  
    - **Version:** 1

---

#### 1.4 Email Dispatch

- **Overview:**  
  Sends an email for each new RSS post detected within the last hour.

- **Nodes Involved:**  
  - Send email with each post

- **Node Details:**  

  - **Node Name:** Send email with each post  
    - **Type:** Gmail (Sending Email)  
    - **Configuration:**  
      - Sends email to a configured recipient address (placeholder: "SET YOUR EMAIL HERE")  
      - Subject template: `"New post from {{ $json.link.extractDomain() }}: {{ $json.title }}"`  
      - Message body includes:  
        - Link domain and URL  
        - Content snippet from the RSS post  
      - Attribution appended automatically (default Gmail behavior)  
      - Requires Gmail OAuth2 credentials named "Personal Gmail account"  
    - **Input/Output:**  
      - Input from "If published in the last hour" (true branch)  
      - No output (end node)  
    - **Edge Cases:**  
      - Invalid or expired Gmail OAuth2 credentials will cause authentication failure.  
      - Sending limits or API quota restrictions may stop email dispatch.  
      - Missing or malformed post data (e.g., missing `link` or `title`) can lead to incomplete emails.  
    - **Version:** 2.1

---

#### 1.5 Auxiliary and Control

- **Overview:**  
  Contains sticky notes for user guidance and workflow clarity.

- **Nodes Involved:**  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**  

  - **Node Name:** Sticky Note2  
    - **Type:** Sticky Note  
    - **Content:** "👆 Add your RSS feeds urls here."  
    - **Position:** Near "List of RSS feeds" node  
    - **Purpose:** User instruction for configuring RSS feed URLs.  

  - **Node Name:** Sticky Note3  
    - **Type:** Sticky Note  
    - **Content:**  
      ```
      ### 👨‍🎤 Setup
      1. Add your email and email creds
      2. Add the RSS feed URLs you want to follow
      ```  
    - **Position:** Left sidebar area for general setup instructions.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                  | Input Node(s)         | Output Node(s)                  | Sticky Note                            |
|-------------------------|------------------------|--------------------------------|-----------------------|--------------------------------|--------------------------------------|
| Every 1 hour            | Schedule Trigger       | Starts workflow every hour      | -                     | List of RSS feeds              |                                      |
| List of RSS feeds       | Set                    | Defines RSS feed URLs           | Every 1 hour          | Split Out                     | 👆 Add your RSS feeds urls here.      |
| Split Out               | Split Out              | Splits feed URLs array          | List of RSS feeds     | Loop Over Items               | 👆 Add your RSS feeds urls here.      |
| Loop Over Items         | Split In Batches       | Processes each feed URL individually | Split Out           | No Operation, do nothing; RSS Read |                                      |
| No Operation, do nothing | No Operation           | Placeholder for control         | Loop Over Items       | -                            |                                      |
| RSS Read                | RSS Feed Read          | Reads RSS feed from URL         | Loop Over Items       | If published in the last hour |                                      |
| If published in the last hour | If (Conditional)        | Filters posts by published date | RSS Read              | Send email with each post     |                                      |
| Send email with each post | Gmail                  | Sends notification email        | If published in the last hour | -                            |                                      |
| Sticky Note2            | Sticky Note            | RSS feed URLs setup hint        | -                     | -                            | 👆 Add your RSS feeds urls here.      |
| Sticky Note3            | Sticky Note            | General setup instructions      | -                     | -                            | ### 👨‍🎤 Setup 1. Add your email and email creds 2. Add the RSS feed URLs you want to follow |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Name: `Every 1 hour`
   - Type: Schedule Trigger
   - Parameters:  
     - Set interval to trigger every hour at minute 30 (e.g., set hours interval with triggerAtMinute = 30)

3. **Add a Set node:**
   - Name: `List of RSS feeds`
   - Connect input from `Every 1 hour`
   - Parameters:  
     - Add a field named `urls` of type Array  
     - Value example:  
       ```json
       [
         "https://www.anildash.com/feed.xml",
         "https://sive.rs/en.atom"
       ]
       ```

4. **Add a Split Out node:**
   - Name: `Split Out`
   - Connect input from `List of RSS feeds`
   - Parameters:  
     - Field to split out: `urls`

5. **Add a Split In Batches node:**
   - Name: `Loop Over Items`
   - Connect input from `Split Out`
   - Parameters:  
     - Leave default batch size (1) to process feeds sequentially

6. **Add a No Operation node:**
   - Name: `No Operation, do nothing`
   - Connect input from `Loop Over Items` (optional branch, for fallback or control)

7. **Add an RSS Feed Read node:**
   - Name: `RSS Read`
   - Connect input from `Loop Over Items`
   - Parameters:  
     - URL: Use an expression to dynamically read current batch feed URL: `={{ $json.urls }}`
     - Enable `Retry On Fail` with a 5-second wait between attempts
     - On error: Set to `continueRegularOutput` to avoid workflow stop on feed errors

8. **Add an If node:**
   - Name: `If published in the last hour`
   - Connect input from `RSS Read`
   - Parameters:  
     - Set condition to check if `isoDate` of each post is within the last hour:  
       - Condition 1: `DateTime.fromISO($json.isoDate)` is after `DateTime.now().minus({hour: 1})`  
       - Condition 2: `DateTime.fromISO($json.isoDate)` is before or equal to `DateTime.now()`

9. **Add a Gmail node:**
   - Name: `Send email with each post`
   - Connect input from `If published in the last hour` (true branch)
   - Parameters:  
     - Send To: Set your target email address  
     - Subject (expression): `New post from {{ $json.link.extractDomain() }}: {{ $json.title }}`  
     - Message (expression):  
       ```
       Check out this new post from {{ $json.link.extractDomain() }} at {{ $json.link }}

       ----

       {{ $json.contentSnippet }}
       ```  
     - Enable append attribution (optional)  
   - Credentials:  
     - Set Gmail OAuth2 credentials with a valid personal Gmail account

10. **Add Sticky Notes for clarity (optional):**
    - Add a sticky note near `List of RSS feeds` with content: "👆 Add your RSS feeds urls here."
    - Add a general sticky note with setup instructions:
      ```
      ### 👨‍🎤 Setup
      1. Add your email and email creds
      2. Add the RSS feed URLs you want to follow
      ```

11. **Connect nodes as per the logical flow:**
    - `Every 1 hour` → `List of RSS feeds` → `Split Out` → `Loop Over Items`  
    - `Loop Over Items` → `No Operation, do nothing` (optional)  
    - `Loop Over Items` → `RSS Read` → `If published in the last hour` → `Send email with each post`

12. **Activate the workflow and test.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                    |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow is useful for RSS feeds without native newsletters.                                         | Workflow purpose                                                  |
| You can adjust the workflow to send notifications via Telegram or Slack by replacing the email node.     | Suggested enhancements                                            |
| Refer to n8n community forums for OAuth2 Gmail credential setup instructions.                             | https://community.n8n.io                                         |
| The workflow retries RSS feed reading to improve reliability in case of temporary network issues.        | Node configuration notes                                         |
| Be mindful of Gmail sending limits and API quotas when scaling email dispatch.                            | Gmail API documentation and best practices                       |
| Screenshot attached in original workflow description shows node layout and connections for visual aid.  | FileId: 753 (not accessible here, but reference for user)       |