Create a Daily Digest from Gmail, RSS, and Todoist

https://n8nworkflows.xyz/workflows/create-a-daily-digest-from-gmail--rss--and-todoist-3134


# Create a Daily Digest from Gmail, RSS, and Todoist

### 1. Workflow Overview

This workflow automates the creation and delivery of a **Daily Digest** email by aggregating data from three key sources: Gmail, RSS feeds, and Todoist tasks. It is designed to provide users with a concise, well-formatted summary of their latest emails, top news headlines, and pending tasks every morning.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow automatically on a daily basis.
- **1.2 Data Retrieval:** Fetches data concurrently from Gmail (emails), RSS feed (news), and Todoist (tasks).
- **1.3 Data Aggregation & Formatting:** Merges the collected data and formats it into a styled HTML email.
- **1.4 Email Delivery:** Sends the formatted daily digest email via Gmail.

This structure ensures modularity, parallel data fetching, and a clean final output for enhanced productivity.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically on a daily schedule, ensuring the digest is generated and sent consistently each day.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - **Type & Role:** Schedule Trigger node; initiates workflow execution based on time.  
    - **Configuration:** Set to trigger daily (default interval with empty object, meaning every day).  
    - **Expressions/Variables:** None.  
    - **Input/Output:** No input; outputs trigger signal to three parallel nodes (RSS, Gmail, Todoist).  
    - **Version Requirements:** Version 1.2 used; no special requirements.  
    - **Potential Failures:** Misconfiguration of schedule could cause no trigger; time zone considerations may affect timing.  
    - **Sub-workflow:** None.

#### 1.2 Data Retrieval

- **Overview:**  
  This block fetches the latest data from three external sources in parallel: RSS feed for news, Gmail for emails, and Todoist for tasks.

- **Nodes Involved:**  
  - RSS Feed: Times of India  
  - Gmail: Fetch Emails  
  - TodoList: Fetch Tasks

- **Node Details:**

  - **RSS Feed: Times of India**  
    - **Type & Role:** RSS Feed Read node; reads latest news items from a specified RSS URL.  
    - **Configuration:** URL set to "https://timesofindia.indiatimes.com/rssfeeds/-2128936835.cms". No additional options enabled.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Triggered by Schedule Trigger; outputs RSS items to Merge node.  
    - **Version Requirements:** Version 1.1 used; no special requirements.  
    - **Potential Failures:** RSS feed URL downtime or format changes; network issues.  
    - **Sub-workflow:** None.

  - **Gmail: Fetch Emails**  
    - **Type & Role:** Gmail node; fetches all emails from the connected Gmail account.  
    - **Configuration:** Operation set to "getAll" with no filters, retrieving all emails.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Triggered by Schedule Trigger; outputs emails to Merge node.  
    - **Credentials:** Uses OAuth2 credentials named "Gmail account".  
    - **Version Requirements:** Version 2.1 used; requires OAuth2 credentials properly configured.  
    - **Potential Failures:** Authentication errors, API rate limits, network issues.  
    - **Sub-workflow:** None.

  - **TodoList: Fetch Tasks**  
    - **Type & Role:** Todoist node; fetches pending tasks from Todoist.  
    - **Configuration:** Operation "getAll" with limit 5 tasks; no filters applied.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Triggered by Schedule Trigger; outputs tasks to Merge node.  
    - **Credentials:** Uses API credentials named "Todoist account".  
    - **Version Requirements:** Version 2.1 used; requires valid Todoist API token.  
    - **Potential Failures:** Authentication errors, API downtime, empty task list.  
    - **Sub-workflow:** None.

#### 1.3 Data Aggregation & Formatting

- **Overview:**  
  This block merges the three data streams and formats the combined data into a polished HTML email with inline CSS styling.

- **Nodes Involved:**  
  - Merge  
  - Format Digest: Merge & Style Data (Code node)

- **Node Details:**

  - **Merge**  
    - **Type & Role:** Merge node; combines the three inputs (RSS, Gmail, Todoist) into a single data stream.  
    - **Configuration:** Number of inputs set to 3, matching the three data sources.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Inputs from RSS Feed, Gmail Fetch, and Todoist Fetch nodes; outputs merged data to Code node.  
    - **Version Requirements:** Version 3 used; no special requirements.  
    - **Potential Failures:** Mismatch in input counts; if any input fails, merge may not complete properly.  
    - **Sub-workflow:** None.

  - **Format Digest: Merge & Style Data**  
    - **Type & Role:** Code node; processes merged data to extract top 5 items from each source and generates an HTML email body with inline CSS.  
    - **Configuration:** Custom JavaScript code that:  
      - Extracts top 5 news items, emails, and tasks.  
      - Formats tasks with emoji, content, due date, and link.  
      - Formats news headlines as clickable links.  
      - Formats emails with subject and snippet.  
      - Constructs an email subject line with emojis and counts.  
      - Creates a styled HTML email body with sections for tasks, news, and emails.  
      - Adds a footer with generation timestamp.  
    - **Expressions/Variables:** Uses `$input.all()`, `$node["Gmail: Fetch Emails"].all()`, `$node["TodoList: Fetch Tasks"].all()` to access node data.  
    - **Input/Output:** Input from Merge node; outputs JSON with `email.subject` and `email.body` for sending.  
    - **Version Requirements:** Version 2 used; requires JavaScript support.  
    - **Potential Failures:** JavaScript errors if input data structure changes; empty inputs causing empty sections; date formatting issues.  
    - **Sub-workflow:** None.

#### 1.4 Email Delivery

- **Overview:**  
  Sends the formatted daily digest email to the specified recipient using Gmail.

- **Nodes Involved:**  
  - Gmail: Send Digest

- **Node Details:**

  - **Gmail: Send Digest**  
    - **Type & Role:** Gmail node; sends an email with the digest content.  
    - **Configuration:**  
      - `sendTo` set to "youremail@gmail.com" (user must replace with their own email).  
      - `subject` and `message` dynamically set from the Code node output (`$json.email.subject` and `$json.email.body`).  
      - No additional options enabled.  
    - **Expressions/Variables:** Uses expressions to dynamically set subject and message.  
    - **Input/Output:** Input from Code node; no output.  
    - **Credentials:** Uses OAuth2 credentials named "Gmail account".  
    - **Version Requirements:** Version 2.1 used; requires valid Gmail OAuth2 credentials.  
    - **Potential Failures:** Authentication errors, invalid recipient email, Gmail API limits, HTML rendering issues in email clients.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role               | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                      |
|----------------------------|-------------------------|------------------------------|----------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger        | Initiates daily workflow run | None                             | RSS Feed: Times of India, Gmail: Fetch Emails, TodoList: Fetch Tasks |                                                                                                 |
| RSS Feed: Times of India   | RSS Feed Read           | Fetches latest news headlines| Schedule Trigger                 | Merge                           |                                                                                                 |
| Gmail: Fetch Emails        | Gmail                   | Retrieves latest emails      | Schedule Trigger                 | Merge                           |                                                                                                 |
| TodoList: Fetch Tasks      | Todoist                 | Retrieves pending tasks      | Schedule Trigger                 | Merge                           |                                                                                                 |
| Merge                     | Merge                   | Combines data from all sources| RSS Feed, Gmail Fetch, Todoist Fetch | Format Digest: Merge & Style Data |                                                                                                 |
| Format Digest: Merge & Style Data | Code                   | Formats merged data into HTML email | Merge                           | Gmail: Send Digest              |                                                                                                 |
| Gmail: Send Digest         | Gmail                   | Sends the daily digest email | Format Digest: Merge & Style Data | None                           | **Important:** Replace "youremail@gmail.com" with your own email address in the "To" field.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set the trigger to run daily (default interval).  
   - Position it as the starting node.

3. **Add an RSS Feed Read node:**  
   - Name it "RSS Feed: Times of India".  
   - Set the URL to `https://timesofindia.indiatimes.com/rssfeeds/-2128936835.cms`.  
   - Connect the output of Schedule Trigger to this node.

4. **Add a Gmail node to fetch emails:**  
   - Name it "Gmail: Fetch Emails".  
   - Set operation to "getAll" with no filters.  
   - Configure Gmail OAuth2 credentials (create or select existing).  
   - Connect the output of Schedule Trigger to this node.

5. **Add a Todoist node to fetch tasks:**  
   - Name it "TodoList: Fetch Tasks".  
   - Set operation to "getAll".  
   - Set limit to 5 tasks.  
   - Configure Todoist API credentials.  
   - Connect the output of Schedule Trigger to this node.

6. **Add a Merge node:**  
   - Set "Number of Inputs" to 3.  
   - Connect outputs of RSS Feed, Gmail Fetch, and Todoist Fetch nodes to inputs 1, 2, and 3 respectively.

7. **Add a Code node:**  
   - Name it "Format Digest: Merge & Style Data".  
   - Paste the provided JavaScript code that:  
     - Extracts top 5 items from each input.  
     - Formats tasks with emoji, content, due date, and link.  
     - Formats news headlines as clickable links.  
     - Formats emails with subject and snippet.  
     - Creates an HTML email body with inline CSS styling.  
     - Sets a dynamic subject line with emojis and counts.  
   - Connect the output of Merge node to this Code node.

8. **Add a Gmail node to send the digest:**  
   - Name it "Gmail: Send Digest".  
   - Set operation to "send".  
   - Set "To" field to your email address (replace "youremail@gmail.com" with your own).  
   - Set "Subject" to expression: `{{$json.email.subject}}`.  
   - Set "Message" to expression: `{{$json.email.body}}`.  
   - Use the same Gmail OAuth2 credentials as the fetch node.  
   - Connect the output of the Code node to this node.

9. **Activate the workflow.**

10. **Test the workflow manually or wait for the scheduled trigger to run.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Replace the placeholder email address in the final Gmail node with your actual email address.   | Critical to ensure the digest is sent to the correct recipient.                                    |
| The workflow uses inline CSS for email formatting to ensure compatibility across email clients. | Styling is embedded directly in the HTML for consistent rendering.                                |
| This workflow is ideal for busy professionals, entrepreneurs, team leaders, and productivity enthusiasts. | Use case scenario described in the overview section.                                              |
| Keywords for search and categorization: daily digest automation, email summary, RSS integration, task management, Gmail automation, Todoist workflow, Cron trigger | Useful for tagging and documentation purposes.                                                    |
| For Gmail OAuth2 setup, ensure you have created credentials in Google Cloud Console and authorized n8n. | Refer to official n8n documentation for OAuth2 Gmail setup.                                       |
| Todoist API token must be generated from Todoist developer settings and configured in n8n credentials. | See Todoist API documentation for token generation.                                               |
| RSS feed URL can be replaced with any other valid RSS feed URL to customize news source.         | Adaptable to user preferences.                                                                     |

---

This document provides a complete, detailed reference for understanding, reproducing, and maintaining the "Create a Daily Digest from Gmail, RSS, and Todoist" workflow in n8n.