Get GitHub Issue Updates and Send Notifications to Telegram

https://n8nworkflows.xyz/workflows/get-github-issue-updates-and-send-notifications-to-telegram-3021


# Get GitHub Issue Updates and Send Notifications to Telegram

### 1. Workflow Overview

This workflow automates the monitoring of GitHub issues and sends notifications to a Telegram user or group. It is designed for developers, managers, and DevOps teams who want timely updates on new or updated GitHub issues without manually checking the repository.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Periodically initiates the workflow every 10 minutes.
- **1.2 Fetch GitHub Issues**: Retrieves open issues from a specified GitHub repository with optional filters.
- **1.3 Data Extraction and Mapping**: Extracts and formats key issue details such as title, URL, creation date, and comment count.
- **1.4 Filtering Issues**: Filters issues based on the number of comments to reduce noise.
- **1.5 Telegram Notification**: Sends formatted issue notifications to a Telegram user or group.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution every 10 minutes to check for new or updated GitHub issues automatically.

- **Nodes Involved:**  
  - Run every 10 minutes

- **Node Details:**  
  - **Run every 10 minutes**  
    - Type: Schedule Trigger  
    - Configuration: Interval set to trigger every 10 minutes.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to the "Get Github Issues" node to start the data retrieval process.  
    - Edge Cases: If the node fails, the workflow will not run on schedule; ensure n8n instance uptime.  
    - Version: 1.2  

---

#### 1.2 Fetch GitHub Issues

- **Overview:**  
  Retrieves open issues from a specified GitHub repository using GitHub API with filters such as issue state, labels, and a time window.

- **Nodes Involved:**  
  - Get Github Issues  
  - Sticky Note (Get Github Issues HTTP Request)

- **Node Details:**  
  - **Get Github Issues**  
    - Type: GitHub node (API integration)  
    - Configuration:  
      - Owner and repository name must be set (editable fields).  
      - Filters:  
        - `state` set to "open" to fetch only open issues.  
        - `labels` set to "Bug" (can be customized).  
        - `since` set dynamically to 30 minutes ago (`new Date(Date.now() - 30 * 60 * 1000).toISOString()`) to fetch recent issues only.  
    - Credentials: Requires GitHub Personal Access Token with `repo` or `public_repo` permissions.  
    - Inputs: Triggered by the schedule node.  
    - Outputs: Passes raw GitHub issues data to the next node.  
    - Edge Cases:  
      - Authentication errors if token is invalid or expired.  
      - API rate limits may cause failures or incomplete data.  
      - Repository or owner misconfiguration leads to empty or error responses.  
    - Version: 1  

  - **Sticky Note (Get Github Issues HTTP Request)**  
    - Type: Sticky Note (documentation aid)  
    - Content: Instructions to edit OWNER and REPO NAME and explanation of query parameters (`state`, `since`, `labels`).  
    - Position: Visual aid only, no functional impact.

---

#### 1.3 Data Extraction and Mapping

- **Overview:**  
  Extracts relevant fields from the raw GitHub API response to simplify downstream processing and messaging.

- **Nodes Involved:**  
  - Map title, url, created, comments  
  - Sticky Note1 (Extract Fields)

- **Node Details:**  
  - **Map title, url, created, comments**  
    - Type: Set node (data transformation)  
    - Configuration: Maps and assigns the following fields from the GitHub issue JSON:  
      - `title` (string)  
      - `html_url` (string) — direct link to the issue  
      - `created_at` (string) — issue creation timestamp  
      - `comments` (number) — number of comments on the issue  
    - Inputs: Receives raw issue data from "Get Github Issues".  
    - Outputs: Passes simplified issue data to the filter node.  
    - Edge Cases: Missing fields in GitHub response could cause empty values; expressions assume fields exist.  
    - Version: 3.4  

  - **Sticky Note1 (Extract Fields)**  
    - Type: Sticky Note  
    - Content: Notes about extracting fields like title, comments, created_at from the GitHub API response.

---

#### 1.4 Filtering Issues

- **Overview:**  
  Filters issues to only process those with fewer than 5 comments, reducing notification noise for heavily discussed issues.

- **Nodes Involved:**  
  - Check for comments  
  - Sticky Note2 (Filter on Fields)

- **Node Details:**  
  - **Check for comments**  
    - Type: Filter node  
    - Configuration:  
      - Condition: Pass only issues where `comments` < 5.  
      - Uses strict number comparison on the `comments` field.  
    - Inputs: Receives mapped issue data.  
    - Outputs:  
      - Issues passing the filter proceed to Telegram notification.  
      - Issues failing the filter are discarded (no output).  
    - Edge Cases: Issues without a `comments` field or with non-numeric values may cause filter errors.  
    - Version: 2.2  

  - **Sticky Note2 (Filter on Fields)**  
    - Type: Sticky Note  
    - Content: Explains filtering issues based on the number of comments.

---

#### 1.5 Telegram Notification

- **Overview:**  
  Sends a formatted message containing the issue title and URL to a Telegram user or group via a bot.

- **Nodes Involved:**  
  - Send Message to @user  
  - Sticky Note3 (Send message to Telegram User)

- **Node Details:**  
  - **Send Message to @user**  
    - Type: Telegram node  
    - Configuration:  
      - Message text template: `"New Issue:  {{ $json.title }} [Link]({{ $json.html_url }})"` — uses Markdown link formatting.  
      - Chat ID: Configured in Telegram credentials (can be user ID or username).  
      - Bot Token: Stored securely in Telegram API credentials.  
    - Inputs: Receives filtered issue data from the filter node.  
    - Outputs: None (terminal node).  
    - Edge Cases:  
      - Invalid bot token or chat ID causes message send failure.  
      - Telegram API rate limits or downtime may delay or block messages.  
      - Message formatting errors if fields are missing or malformed.  
    - Version: 1.2  

  - **Sticky Note3 (Send message to Telegram User)**  
    - Type: Sticky Note  
    - Content:  
      - Instructions to create a Telegram bot and obtain the bot token.  
      - Notes on configuring the chat ID (user ID or username).  
      - Link to Telegram bot token creation tutorial: https://core.telegram.org/bots/tutorial#obtain-your-bot-token

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                 | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                  |
|---------------------------|---------------------|--------------------------------|-------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------|
| Run every 10 minutes       | Schedule Trigger    | Periodic workflow trigger       | None                    | Get Github Issues         |                                                                                                              |
| Get Github Issues          | GitHub              | Fetch open GitHub issues        | Run every 10 minutes    | Map title, url, created, comments | Get Github Issues HTTP Request - Edit OWNER and REPO NAME; filters: state, since, labels                      |
| Map title, url, created, comments | Set                 | Extract and map issue fields    | Get Github Issues       | Check for comments        | Extract Fields - Extract title, comments, created_at, etc.                                                   |
| Check for comments         | Filter              | Filter issues by comment count  | Map title, url, created, comments | Send Message to @user    | Filter on Fields - Filter issues based on number of comments                                                  |
| Send Message to @user      | Telegram            | Send issue notification message | Check for comments      | None                      | Send message to Telegram User - Configure bot token and chat ID; see https://core.telegram.org/bots/tutorial#obtain-your-bot-token |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 10 minutes.  
   - Name it "Run every 10 minutes".

2. **Add a GitHub node**  
   - Type: GitHub  
   - Name it "Get Github Issues".  
   - Configure credentials with a valid GitHub Personal Access Token (with `repo` or `public_repo` scope).  
   - Set resource to "repository".  
   - Set operation to "getRepositoryIssues".  
   - Set repository owner and repository name (e.g., your GitHub username and repo).  
   - Under filters, set:  
     - `state` = "open"  
     - `labels` = "Bug" (or any label you want to filter by)  
     - `since` = `={{ new Date(Date.now() - 30 * 60 * 1000).toISOString() }}` (to fetch issues updated in the last 30 minutes)  
   - Connect the output of "Run every 10 minutes" to this node.

3. **Add a Set node**  
   - Type: Set  
   - Name it "Map title, url, created, comments".  
   - Add assignments:  
     - `title` = `={{ $json.title }}` (string)  
     - `html_url` = `={{ $json.html_url }}` (string)  
     - `created_at` = `={{ $json.created_at }}` (string)  
     - `comments` = `={{ $json.comments }}` (number)  
   - Connect output of "Get Github Issues" to this node.

4. **Add a Filter node**  
   - Type: Filter  
   - Name it "Check for comments".  
   - Set condition:  
     - `comments` < 5 (number comparison)  
   - Connect output of "Map title, url, created, comments" to this node.

5. **Add a Telegram node**  
   - Type: Telegram  
   - Name it "Send Message to @user".  
   - Configure Telegram API credentials with your bot token.  
   - Set the message text to:  
     `New Issue:  {{ $json.title }} [Link]({{ $json.html_url }})`  
   - Set the chat ID to your Telegram user ID or group chat ID.  
   - Connect the "true" output of "Check for comments" filter node to this node.

6. **Optional: Add Sticky Notes**  
   - Add sticky notes near relevant nodes to document configuration tips and instructions, e.g.:  
     - Near GitHub node: instructions to edit owner/repo and filter parameters.  
     - Near Set node: note about extracted fields.  
     - Near Filter node: explanation of filtering logic.  
     - Near Telegram node: instructions for bot creation and chat ID retrieval with link to https://core.telegram.org/bots/tutorial#obtain-your-bot-token.

7. **Activate the workflow**  
   - Save and activate the workflow to run every 10 minutes and send notifications for new GitHub issues matching the criteria.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Create a Telegram bot and obtain the bot token via BotFather on Telegram.                                     | https://core.telegram.org/bots/tutorial#obtain-your-bot-token                                   |
| Find your Telegram chat ID using bots like @getmyid_bot to configure message recipient.                       | https://t.me/getmyid_bot                                                                         |
| GitHub Personal Access Token must have `repo` or `public_repo` permissions to access repository issues.      | GitHub Developer Settings > Personal Access Tokens                                              |
| Customize issue filters by modifying GitHub node parameters such as `labels` or `state`.                      | Workflow setup guide section                                                                    |
| To trigger notifications on GitHub events instead of polling, consider using GitHub webhooks if repository allows. | GitHub Webhooks documentation                                                                    |

---

This documentation provides a complete understanding of the workflow’s structure, node configurations, and operational logic, enabling users and AI agents to reproduce, modify, or troubleshoot the workflow effectively.