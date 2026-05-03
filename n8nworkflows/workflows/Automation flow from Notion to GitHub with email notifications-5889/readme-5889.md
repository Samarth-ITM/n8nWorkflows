Automation flow from Notion to GitHub with email notifications

https://n8nworkflows.xyz/workflows/automation-flow-from-notion-to-github-with-email-notifications-5889


# Automation flow from Notion to GitHub with email notifications

### 1. Workflow Overview

This workflow automates the synchronization of task or feature request data from a Notion database into GitHub issues, coupled with email notifications to team members. Its primary use case is to streamline project management by automatically creating GitHub issues for tasks marked "To develop" in Notion and updating Notion with issue progress. For tasks not in development, it notifies the team via email about status changes.

The workflow is logically divided into three main blocks:

- **1.1 Data Acquisition and Sorting:** Triggered every 12 hours, it fetches all pages from a specified Notion database, restructures the data fields for consistency, and routes items based on their status.
- **1.2 GitHub Issue Management:** For tasks marked "To develop," it creates corresponding GitHub issues and updates the Notion page with the issue URL and status.
- **1.3 Team Notification:** For tasks not in "To develop" status, it gathers Notion users, filters out bots, aggregates their emails, and sends a notification email about the issue status.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition and Sorting

**Overview:**  
This block triggers periodically to pull all entries from a Notion database representing tasks/features, restructures key fields, and routes data based on task status.

**Nodes Involved:**  
- Schedule Trigger  
- Get many database pages (Notion)  
- Sort Issues Fields (Set)  
- Switch (Status-based routing)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Configuration:* Runs every 12 hours (hourly interval = 12)  
  - *Input/Output:* No input; triggers "Get many database pages" node.  
  - *Edge Cases:* Missed executions if n8n instance is down; configurable to webhook trigger for real-time with Notion paid plan.  
  - *Notes:* Starting point of the workflow.

- **Get many database pages (Notion)**  
  - *Type:* Notion node, resource: databasePage, operation: getAll  
  - *Configuration:* Retrieves all pages from a specific Notion database (ID: 224315ef-5fbb-804c-b0ac-daa3fd2204d1). Uses "Notion account" credentials. Returns all pages without pagination limits.  
  - *Input:* Triggered by Schedule Trigger  
  - *Output:* Supplies raw Notion page data downstream.  
  - *Edge Cases:* API rate limits, authentication errors, empty database.  
  - *Version:* v2.2

- **Sort Issues Fields (Set)**  
  - *Type:* Set node  
  - *Configuration:* Maps raw Notion page properties into standardized fields:  
    - Title ← page `name`  
    - Description ← `property_description`  
    - Labels ← first label from `property_labels` array  
    - Repository ← first repository from `property_repository` array  
    - Status ← `property_status`  
    - DatabasePageId ← page ID  
  - *Input:* Output from Notion pages node  
  - *Output:* Cleaned and structured JSON for routing  
  - *Edge Cases:* Missing or malformed properties, empty arrays causing undefined fields.  
  - *Version:* v3.4

- **Switch (Status-based routing)**  
  - *Type:* Switch node  
  - *Configuration:* Routes data based on the `Status` field:  
    - If Status equals "To develop" → path 1 (GitHub Issue Creation)  
    - Else if Status equals "Done" → path 2 (Team Notification)  
  - *Input:* Structured data from Set node  
  - *Output:* Two branches leading to different processing paths  
  - *Edge Cases:* Status field missing or having unexpected values; unhandled statuses ignored.  
  - *Version:* v3.2

---

#### 2.2 GitHub Issue Management (For "To develop" Tasks)

**Overview:**  
This block creates GitHub issues from Notion tasks marked "To develop" and updates the Notion page with the issue URL and new status "In progress."

**Nodes Involved:**  
- Create an issue (GitHub)  
- Set Status and Issue URL (Notion Update)

**Node Details:**

- **Create an issue (GitHub)**  
  - *Type:* GitHub node, operation: createIssue  
  - *Configuration:*  
    - Repository owner: Fixed to "From6Agency"  
    - Repository name: Dynamically taken from Notion field `Repository`  
    - Issue title: From Notion `Title` field  
    - Issue body: From Notion `Description` field  
    - Labels and assignees: Empty arrays (no labels or assignees assigned)  
    - Authentication: OAuth2 credentials ("GitHub account")  
  - *Input:* Items filtered by Switch with Status "To develop"  
  - *Output:* Returns created issue metadata, including `html_url`  
  - *Edge Cases:* OAuth token expiration, repository access rights, rate limits, invalid repository name or missing title/description.  
  - *Version:* v1.1

- **Set Status and Issue URL (Notion Update)**  
  - *Type:* Notion node, operation: update databasePage  
  - *Configuration:*  
    - Updates the Notion page with ID from the current item's `DatabasePageId`  
    - Sets `Status` property to "In progress"  
    - Sets `Issue URL` property to the GitHub issue URL (`html_url`)  
  - *Input:* Output from GitHub create issue node  
  - *Output:* Confirmation of update operation  
  - *Edge Cases:* Notion permission errors, invalid page ID, API limits, missing `html_url`.  
  - *Version:* v2.2

---

#### 2.3 Team Notification (For Non-"To develop" Tasks)

**Overview:**  
For tasks not marked "To develop," this block retrieves all Notion users, filters out bot accounts, aggregates their emails, and sends a notification email about the task status.

**Nodes Involved:**  
- Get many users (Notion)  
- Map Notion Users (Set)  
- Exclude Bot (Switch)  
- Group Recipients (Aggregate)  
- Send a message (Gmail)

**Node Details:**

- **Get many users (Notion)**  
  - *Type:* Notion node, resource: user, operation: getAll  
  - *Configuration:* Retrieves all users associated with the Notion workspace  
  - *Input:* From Switch node output for non-"To develop" tasks  
  - *Output:* List of user objects including names, emails, types  
  - *Edge Cases:* Large user lists, API limits, permission errors.  
  - *Version:* v2.2

- **Map Notion Users (Set)**  
  - *Type:* Set node  
  - *Configuration:* Maps user properties:  
    - Name ← user `name`  
    - Email ← user `person.email`  
    - type ← user `type` (e.g., "person", "bot")  
  - *Input:* Output from Get many users  
  - *Output:* Simplified user list JSON  
  - *Edge Cases:* Missing emails, non-person types.  
  - *Version:* v3.4

- **Exclude Bot (Switch)**  
  - *Type:* Switch node  
  - *Configuration:* Passes through only users where `type` equals "person" to exclude bots (e.g., notifications@noreply)  
  - *Input:* Output from Map Notion Users  
  - *Output:* Filtered user list  
  - *Edge Cases:* Unexpected user types, empty user list after filtering.  
  - *Version:* v3.2

- **Group Recipients (Aggregate)**  
  - *Type:* Aggregate node  
  - *Configuration:* Aggregates the filtered users' `Email` fields into a single array  
  - *Input:* Output from Exclude Bot  
  - *Output:* Single aggregated item with array of emails  
  - *Edge Cases:* Empty email list, aggregation failures.  
  - *Version:* v1

- **Send a message (Gmail)**  
  - *Type:* Gmail node, operation: sendEmail  
  - *Configuration:*  
    - Recipient emails: joined string of aggregated emails  
    - Subject: "A Github Issue has been closed !"  
    - Message body: Text message indicating the issue title and repository where the issue was closed  
    - Email type: Plain text  
    - Authentication: Gmail OAuth2 credentials ("Gmail account")  
  - *Input:* From Group Recipients node  
  - *Output:* Email sent confirmation  
  - *Edge Cases:* OAuth token issues, invalid email addresses, sending limits, message content errors.  
  - *Version:* v2.1

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                              | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                   |
|-------------------------|-----------------------------|----------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger             | Triggers workflow every 12 hours             | -                           | Get many database pages      | ## 🔹 SECTION 1: Detect and Sort Issues from Notion ... Start trigger                                        |
| Get many database pages  | Notion                      | Fetches all pages from Notion database       | Schedule Trigger            | Sort Issues Fields           | ## 🔹 SECTION 1: Detect and Sort Issues from Notion ... Fetches all rows                                    |
| Sort Issues Fields       | Set                         | Restructures Notion data fields               | Get many database pages     | Switch                      | ## 🔹 SECTION 1: Detect and Sort Issues from Notion ... Restructures fields                                  |
| Switch                  | Switch                      | Routes data by Status field                    | Sort Issues Fields          | Create an issue, Get many users | ## 🔹 SECTION 1: Detect and Sort Issues from Notion ... Routes based on status                               |
| Create an issue          | GitHub                      | Creates GitHub issue for "To develop" tasks  | Switch (To develop path)    | Set Status and Issue URL     | ## 🔹 SECTION 2: GitHub Issue Creation (IF "To develop") ... Creates GitHub issue                            |
| Set Status and Issue URL | Notion                      | Updates Notion page with status and issue URL| Create an issue             | -                           | ## 🔹 SECTION 2: GitHub Issue Creation (IF "To develop") ... Updates Notion status and URL                   |
| Get many users           | Notion                      | Retrieves team members for notification       | Switch (non "To develop")   | Map Notion Users            | ## 🔹 SECTION 3: Notify Team on Already In-Progress Items ... Retrieves list of users                        |
| Map Notion Users         | Set                         | Formats user data for email notifications     | Get many users              | Exclude Bot                 | ## 🔹 SECTION 3: Notify Team on Already In-Progress Items ... Maps and formats user data                     |
| Exclude Bot              | Switch                      | Filters out bot users from notification list  | Map Notion Users            | Group Recipients            | ## 🔹 SECTION 3: Notify Team on Already In-Progress Items ... Excludes bots                                 |
| Group Recipients         | Aggregate                   | Aggregates user emails into a single array    | Exclude Bot                 | Send a message              | ## 🔹 SECTION 3: Notify Team on Already In-Progress Items ... Aggregates emails                             |
| Send a message           | Gmail                       | Sends notification email to team               | Group Recipients            | -                           | ## 🔹 SECTION 3: Notify Team on Already In-Progress Items ... Sends email notification                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger every 12 hours (field: hours interval = 12).  
   - This node initiates the workflow.

2. **Add Notion node "Get many database pages"**  
   - Resource: databasePage  
   - Operation: getAll  
   - Database ID: `224315ef-5fbb-804c-b0ac-daa3fd2204d1` (replace with your database ID)  
   - Credentials: Link your Notion API credentials.  
   - Connect Schedule Trigger output to this node.

3. **Add Set node "Sort Issues Fields"**  
   - Map the following fields from Notion raw data:  
     - Title ← `name`  
     - Description ← `property_description`  
     - Labels ← first element of `property_labels` array  
     - Repository ← first element of `property_repository` array  
     - Status ← `property_status`  
     - DatabasePageId ← Notion page ID (`id`)  
   - Connect "Get many database pages" output to this node.

4. **Add Switch node "Switch"**  
   - Add two rules based on the `Status` field:  
     - Rule 1: Status equals "To develop"  
     - Rule 2: Status equals "Done"  
   - Connect "Sort Issues Fields" output to this node.

5. **Create GitHub node "Create an issue"**  
   - Operation: createIssue  
   - Owner: Fixed to "From6Agency" (replace if needed)  
   - Repository: Set dynamically from `Repository` field  
   - Title: Set dynamically from `Title` field  
   - Body: Set dynamically from `Description` field  
   - Labels & Assignees: Leave empty or configure as needed  
   - Authentication: Connect GitHub OAuth2 credentials  
   - Connect Switch output path "To develop" to this node.

6. **Create Notion node "Set Status and Issue URL"**  
   - Operation: update databasePage  
   - Page ID: Set dynamically from `DatabasePageId` of the current item  
   - Properties to update:  
     - Status: set to "In progress"  
     - Issue URL: set to GitHub issue `html_url`  
   - Credentials: Notion API credentials  
   - Connect output of "Create an issue" node to this node.

7. **Create Notion node "Get many users"**  
   - Resource: user  
   - Operation: getAll  
   - Credentials: Notion API credentials  
   - Connect Switch output path "Done" to this node.

8. **Add Set node "Map Notion Users"**  
   - Map user properties:  
     - Name ← `name`  
     - Email ← `person.email`  
     - type ← `type`  
   - Connect "Get many users" output to this node.

9. **Add Switch node "Exclude Bot"**  
   - Rule: Pass only users where `type` equals "person"  
   - Connect "Map Notion Users" output to this node.

10. **Add Aggregate node "Group Recipients"**  
    - Aggregate field: `Email` into an array  
    - Connect "Exclude Bot" output to this node.

11. **Add Gmail node "Send a message"**  
    - Operation: sendEmail  
    - Send To: Use aggregated emails joined by commas  
    - Subject: "A Github Issue has been closed !"  
    - Message: Text indicating the issue title and repository from the Switch node's item (use expressions to reference)  
    - Authentication: Connect Gmail OAuth2 credentials  
    - Connect "Group Recipients" output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| The workflow uses a 12-hour polling interval due to Notion API limitations; upgrading to a webhook trigger requires Notion paid plan. | Sticky Note 1, Section 1                                                                                      |
| GitHub OAuth2 credentials must have repo creation permissions; ensure the OAuth token scope includes `repo`.                  | Sticky Note 2, Section 2                                                                                      |
| Gmail OAuth2 requires user consent for sending emails on their behalf; configure scopes accordingly.                         | Sticky Note 3, Section 3                                                                                      |
| Workflow handles only two statuses explicitly ("To develop" and "Done"); other statuses are ignored and do not trigger actions.| Switch node configuration                                                                                     |
| For email notifications, bots and automation accounts are excluded to avoid spamming non-human recipients.                    | Exclude Bot node functionality                                                                                 |
| Replace all fixed IDs and credentials with your own environment specifics when deploying the workflow in your context.        | General best practice                                                                                          |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.