Sync your GitHub issues to your Notion database

https://n8nworkflows.xyz/workflows/sync-your-github-issues-to-your-notion-database-1804


# Sync your GitHub issues to your Notion database

### 1. Workflow Overview

This workflow automates synchronization between GitHub issues and a Notion database. It listens for issue events (such as creation, edits, closure, reopening, and deletion) in a specified GitHub repository and reflects these changes in a Notion database. If the Notion database does not exist, it creates a new page for the issue automatically. The workflow is structured into logical blocks that handle event triggering, conditional routing based on issue actions, data creation or update in Notion, and filtered searches within Notion.

Logical blocks include:

- **1.1 GitHub Trigger:** Listens for issue events from GitHub.
- **1.2 Conditional Routing (IF & Switch):** Determines if the issue is new or an update and routes accordingly.
- **1.3 New Issue Handling:** Creates a new page in Notion for new issues.
- **1.4 Existing Issue Handling:** Builds filters and searches the Notion database for the corresponding page.
- **1.5 Issue Status Update:** Updates, deletes, closes, or reopens issue pages in Notion based on the GitHub event.

---

### 2. Block-by-Block Analysis

#### 1.1 GitHub Trigger

- **Overview:**  
  Initiates the workflow when an issue event occurs in the specified GitHub repository.

- **Nodes Involved:**  
  - `Trigger on issues`

- **Node Details:**

  - **Trigger on issues**  
    - *Type & Role:* GitHub Trigger node; listens for GitHub issue events.  
    - *Configuration:*  
      - Owner: `John-n8n`  
      - Repository: `DemoRepo`  
      - Events: Issues (triggers on any issue-related event)  
      - WebhookId set for receiving events automatically.  
    - *Key Expressions:* Uses event payload, especially `body.action` and `body.issue` data.  
    - *Connections:* Outputs to the IF node.  
    - *Version Requirements:* n8n version supporting GitHub trigger nodes with webhooks.  
    - *Potential Failures:* Authentication errors if GitHub credentials are invalid; webhook delivery issues; rate limiting by GitHub.  
    - *Credentials:* Requires GitHub OAuth credentials configured.  

#### 1.2 Conditional Routing (IF & Switch)

- **Overview:**  
  Determines if the GitHub event is for a new issue or an update, and further routes updated issues based on the specific action (edited, deleted, closed, reopened).

- **Nodes Involved:**  
  - `IF`  
  - `Create custom Notion filters`  
  - `Find database page`  
  - `Switch`  
  - `Note` (sticky note for explanation)

- **Node Details:**

  - **IF**  
    - *Type & Role:* Conditional node to check if the issue event is for a newly opened issue.  
    - *Configuration:* Checks if `body.action == "opened"`.  
    - *Connections:*  
      - True output: Routes to `Create database page` (new issue handling).  
      - False output: Routes to `Create custom Notion filters` (existing issue handling).  
    - *Potential Failures:* Expression syntax errors if payload structure changes.  

  - **Create custom Notion filters**  
    - *Type & Role:* Function node; builds a JSON filter to find an existing Notion page by the GitHub issue ID.  
    - *Configuration:*  
      - Skips processing if action is "opened" (new issue).  
      - Constructs a Notion filter with an OR clause to find pages where property ‘Issue ID’ equals the GitHub issue ID.  
    - *Expressions:* Uses `$items("Trigger on issues")`, reads issue ID from event payload.  
    - *Connections:* Outputs to `Find database page`.  
    - *Potential Failures:* Logic errors if GitHub payload structure changes; JSON filter syntax issues.  

  - **Find database page**  
    - *Type & Role:* Notion node; searches the Notion database for pages matching the filter created above.  
    - *Configuration:*  
      - Resource: databasePage  
      - Operation: getAll with `returnAll=true`  
      - Database ID: `5026700b-6693-473a-8100-8cc6ddef62a6`  
      - Filter: Uses JSON filter from `Create custom Notion filters`.  
    - *Connections:* Outputs to `Switch`.  
    - *Potential Failures:* Notion API errors (rate limits, invalid filters); missing or changed database ID.  
    - *Credentials:* Requires Notion API credentials.  

  - **Switch**  
    - *Type & Role:* Routes the data based on the action performed on the GitHub issue for existing issues.  
    - *Configuration:*  
      - Evaluates `body.action` against values: `edited`, `deleted`, `closed`, `reopened`.  
      - Routes to corresponding Notion update nodes.  
    - *Connections:*  
      - Edited → `Edit issue`  
      - Deleted → `Delete issue`  
      - Closed → `Close issue`  
      - Reopened → `Reopen issue`  
    - *Potential Failures:* Expression errors; unhandled actions.  

  - **Note**  
    - *Type & Role:* Sticky Note node; documents that the IF and Switch nodes control workflow routing based on GitHub issue actions.  
    - *No connections.*  

#### 1.3 New Issue Handling

- **Overview:**  
  Creates a new page in the Notion database when a new GitHub issue is opened.

- **Nodes Involved:**  
  - `Create database page`

- **Node Details:**

  - **Create database page**  
    - *Type & Role:* Notion node; creates a new page in the Notion database representing the GitHub issue.  
    - *Configuration:*  
      - Resource: `databasePage`  
      - Database ID: `5026700b-6693-473a-8100-8cc6ddef62a6`  
      - Title: The GitHub issue title from `body.issue.title`  
      - Properties:  
        - `Issue ID|number`: GitHub issue numeric ID from `body.issue.id`  
        - `Link|url`: URL to the GitHub issue from `body.issue.html_url`  
    - *Connections:* None (end node for new issues).  
    - *Potential Failures:* Notion API errors; invalid database ID; missing required properties; credential issues.  
    - *Credentials:* Requires Notion API credentials.  

#### 1.4 Existing Issue Handling

- **Overview:**  
  Finds the existing Notion page that corresponds to the GitHub issue so it can be updated or deleted as needed.

- **Nodes Involved:**  
  - `Create custom Notion filters` (also part of conditional routing)  
  - `Find database page`

- **Details:** See 1.2 Conditional Routing for detailed node descriptions.

#### 1.5 Issue Status Update

- **Overview:**  
  Depending on the GitHub issue action, updates the corresponding Notion page by editing, deleting, closing, or reopening.

- **Nodes Involved:**  
  - `Edit issue`  
  - `Delete issue`  
  - `Close issue`  
  - `Reopen issue`

- **Node Details:**

  - **Edit issue**  
    - *Type & Role:* Notion node; updates the title of the existing Notion page to match the GitHub issue title.  
    - *Configuration:*  
      - Operation: update  
      - Page ID: from `Find database page` output (`id`)  
      - Properties: Updates `Issue|title` to new GitHub issue title (`body.issue.title`)  
    - *Potential Failures:* Page ID not found; Notion API update errors.  

  - **Delete issue**  
    - *Type & Role:* Notion node; archives (deletes) the Notion page corresponding to the GitHub issue.  
    - *Configuration:*  
      - Operation: archive  
      - Page ID: from `Find database page` output (`id`)  
    - *Potential Failures:* Page ID not found; Notion API permission errors.  

  - **Close issue**  
    - *Type & Role:* Notion node; updates the "Closed" checkbox property to true.  
    - *Configuration:*  
      - Operation: update  
      - Page ID: from `Find database page` output (`id`)  
      - Properties: Sets `Closed|checkbox` to true  
    - *Potential Failures:* Property missing in database schema; API errors.  

  - **Reopen issue**  
    - *Type & Role:* Notion node; clears the "Closed" checkbox (unchecked).  
    - *Configuration:*  
      - Operation: update  
      - Page ID: from `Find database page` output (`id`)  
      - Properties: Sets `Closed|checkbox` to false or clears it (empty value)  
    - *Potential Failures:* Same as Close issue node.  

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                               | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                      |
|---------------------------|-------------------------|-----------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Trigger on issues          | GitHub Trigger          | Starts workflow on GitHub issue events        |                            | IF                            |                                                                                                |
| IF                        | If                      | Checks if issue is newly opened or updated    | Trigger on issues           | Create database page (true), Create custom Notion filters (false) |                                                                                                |
| Create database page       | Notion                  | Creates new Notion page for new GitHub issue  | IF                         |                               |                                                                                                |
| Create custom Notion filters | Function                | Builds a filter to find existing Notion page by issue ID | IF (false output)           | Find database page             |                                                                                                |
| Find database page         | Notion                  | Finds existing Notion page matching issue ID  | Create custom Notion filters | Switch                        |                                                                                                |
| Switch                    | Switch                  | Routes event based on GitHub issue action     | Find database page          | Edit issue, Delete issue, Close issue, Reopen issue | ## IF & Switch Depends on what action was taken on an issue in GitHub.                         |
| Edit issue                | Notion                  | Updates Notion page title for edited issues   | Switch                     |                               |                                                                                                |
| Delete issue              | Notion                  | Archives (deletes) Notion page for deleted issues | Switch                     |                               |                                                                                                |
| Close issue               | Notion                  | Sets the "Closed" checkbox to true in Notion | Switch                     |                               |                                                                                                |
| Reopen issue              | Notion                  | Clears the "Closed" checkbox in Notion       | Switch                     |                               |                                                                                                |
| Note                      | Sticky Note             | Explains IF & Switch logic                     |                            |                               | ## IF & Switch Depends on what action was taken on an issue in GitHub.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger Node:**
   - Add node: `GitHub Trigger`
   - Set Owner to `John-n8n`
   - Set Repository to `DemoRepo`
   - Select event: `issues`
   - Set webhook ID (auto-generated)
   - Configure GitHub OAuth credentials (ensure proper permissions for repo issues)
   - Connect output to `IF` node

2. **Create IF Node:**
   - Add node: `IF`
   - Condition: String equals
     - Value1: `{{$json["body"]["action"]}}`
     - Value2: `opened`
   - True output connects to `Create database page`
   - False output connects to `Create custom Notion filters`

3. **Create Notion Node for New Issues ("Create database page"):**
   - Add node: `Notion`
   - Resource: `databasePage`
   - Operation: `create`
   - Database ID: `5026700b-6693-473a-8100-8cc6ddef62a6` (replace with your DB ID)
   - Title: Expression `{{$json["body"]["issue"]["title"]}}`
   - Properties:  
     - `Issue ID|number`: Expression `{{$node["Trigger on issues"].json["body"]["issue"]["id"]}}`  
     - `Link|url`: Expression `{{$node["Trigger on issues"].json["body"]["issue"]["html_url"]}}`  
   - Use Notion API credentials
   - No outputs needed

4. **Create Function Node ("Create custom Notion filters"):**
   - Add node: `Function`
   - Paste the following code:

     ```javascript
     const new_items = [];
     for (item of $items("Trigger on issues")) {
       if (item.json["body"]["action"] == "opened") {
         continue;
       }
       var new_item = { json: { notionfilter: "" } };
       let notionfilter = { or: [] };
       const filter = {
         property: "Issue ID",
         number: { equals: parseInt(item.json["body"]["issue"]["id"]) },
       };
       notionfilter["or"].push(filter);
       new_item.json.notionfilter = JSON.stringify(notionfilter);
       new_items.push(new_item);
     }
     return new_items;
     ```
   - Connect output to `Find database page`

5. **Create Notion Node to Find Existing Page ("Find database page"):**
   - Add node: `Notion`
   - Resource: `databasePage`
   - Operation: `getAll`
   - Database ID: same as above
   - Return all: true
   - Filter Type: JSON
   - Filter JSON: Expression `{{$node["Create custom Notion filters"].json["notionfilter"]}}`
   - Use Notion API credentials
   - Connect output to `Switch`

6. **Create Switch Node:**
   - Add node: `Switch`
   - Value1: Expression `{{$json["body"]["action"]}}`
   - Rules (string equals):  
     - edited → output 0  
     - deleted → output 1  
     - closed → output 2  
     - reopened → output 3  
   - Connect outputs accordingly:
     - 0 → `Edit issue`
     - 1 → `Delete issue`
     - 2 → `Close issue`
     - 3 → `Reopen issue`

7. **Create Notion Nodes for Each Action:**

   - **Edit issue:**  
     - Resource: `databasePage`  
     - Operation: `update`  
     - Page ID: Expression `{{$node["Find database page"].json["id"]}}`  
     - Properties:  
       - `Issue|title`: Expression `{{$node["Trigger on issues"].json["body"]["issue"]["title"]}}`  
     - Use Notion API credentials  

   - **Delete issue:**  
     - Operation: `archive`  
     - Page ID: Expression `{{$node["Find database page"].json["id"]}}`  
     - Use Notion API credentials  

   - **Close issue:**  
     - Operation: `update`  
     - Page ID: Expression `{{$node["Find database page"].json["id"]}}`  
     - Properties:  
       - `Closed|checkbox`: `true`  
     - Use Notion API credentials  

   - **Reopen issue:**  
     - Operation: `update`  
     - Page ID: Expression `{{$node["Find database page"].json["id"]}}`  
     - Properties:  
       - `Closed|checkbox`: leave unchecked (empty or false)  
     - Use Notion API credentials  

8. **Add Sticky Note for Documentation (Optional):**
   - Content: `## IF & Switch\nDepends on what action was taken on an issue in GitHub.`

9. **Verify Credentials:**
   - Setup Notion API credentials with access to the target database.
   - Setup GitHub OAuth credentials with appropriate repository permissions.

10. **Test Workflow:**
    - Open, edit, close, reopen, and delete issues in the GitHub repository.
    - Confirm corresponding Notion database pages are created, updated, or archived accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The Notion database must have properties named exactly: "Issue ID" (number), "Link" (url), "Issue" (title), and "Closed" (checkbox). | Notion database schema requirements for seamless syncing.                                      |
| GitHub webhook events for issues require repository admin rights to configure and authenticate.                 | https://docs.n8n.io/integrations/builtin/credentials/github/                                   |
| Notion API credentials require integration with access to the target database.                                  | https://docs.n8n.io/integrations/builtin/credentials/notion/                                   |
| The workflow handles the main GitHub issue lifecycle events: opened, edited, closed, reopened, and deleted only.| Other actions (e.g., labeled) are not handled and will be ignored.                              |
| Make sure to update the placeholder credentials "[UPDATE ME]" with actual configured credentials before use.   | Credential placeholders are present in all Notion and GitHub nodes.                            |

---

This documentation should enable developers and automation agents to understand, reproduce, and maintain the GitHub-to-Notion issues synchronization workflow effectively.