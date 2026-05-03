Monitor Stuck Tasks in Monday.com with Automated Slack Alerts

https://n8nworkflows.xyz/workflows/monitor-stuck-tasks-in-monday-com-with-automated-slack-alerts-7737


# Monitor Stuck Tasks in Monday.com with Automated Slack Alerts

---

### 1. Workflow Overview

This workflow monitors a specific group of tasks in a Monday.com board and automatically sends Slack alerts for any tasks flagged as "Stuck". Its main goal is to enable proactive communication by notifying relevant team members about blocked tasks, helping to reduce delays and improve project management efficiency.

**Logical blocks:**

- **1.1 Manual Trigger:** Initiates the workflow manually.
- **1.2 Retrieve Monday.com Items:** Fetches all items from a specified Monday.com board group.
- **1.3 Data Preparation:** Extracts and formats relevant task details (name, status, due date).
- **1.4 Filtering Stuck Tasks:** Filters tasks where the status is exactly "Stuck".
- **1.5 Slack Alert:** Sends a Slack message alerting the assigned user or a channel about the stuck tasks.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block serves as the entry point, allowing a user to manually execute the workflow on demand.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - **Type & Role:** Manual trigger node; starts the workflow when manually executed.  
    - **Configuration:** No parameters set; standard manual trigger.  
    - **Key Variables:** None.  
    - **Input/Output:** No input connections; outputs to "Get many items1".  
    - **Version:** 1  
    - **Edge Cases:** No external dependencies; no expected failures.  
    - **Sub-workflow:** None.

#### 1.2 Retrieve Monday.com Items

- **Overview:**  
  Retrieves all items from a specified Monday.com board and group using the Monday.com API.

- **Nodes Involved:**  
  - Get many items1

- **Node Details:**

  - **Get many items1**  
    - **Type & Role:** Monday.com node; fetches data from a Monday.com board.  
    - **Configuration:**  
      - Board ID: "9865886546" (configured to target a specific board).  
      - Group ID: "new_group29179" (targets a specific group within the board).  
      - Resource: boardItem  
      - Operation: getAll (fetching all items in the group).  
    - **Credentials:** Uses Monday.com API personal token credential.  
    - **Key Expressions:** None; static IDs configured.  
    - **Input/Output:** Input from manual trigger; outputs full items JSON to "Set Columns".  
    - **Version:** 1  
    - **Edge Cases:**  
      - Authentication failure if token invalid or expired.  
      - API rate limits or downtime.  
      - Empty group resulting in zero items.  
    - **Sub-workflow:** None.

#### 1.3 Data Preparation

- **Overview:**  
  Extracts and formats relevant columns from the raw Monday.com items to simplify downstream filtering and alerting.

- **Nodes Involved:**  
  - Set Columns

- **Node Details:**

  - **Set Columns**  
    - **Type & Role:** Set node; maps and renames fields from the Monday.com response.  
    - **Configuration:**  
      - Sets three fields:  
        - `name`: takes `$json.name` (item name)  
        - `Status`: extracts the text from the second column in `column_values` (index 1)  
        - `Due Date`: extracts the value from the third column in `column_values` (index 2)  
      - Uses expressions to dynamically map values for each item.  
    - **Input/Output:** Input from "Get many items1"; output to "Filter for Stuck Items".  
    - **Version:** 3.4  
    - **Edge Cases:**  
      - If column indexes change or columns missing, may produce undefined or error in expressions.  
      - Null or empty values in columns should be handled gracefully.  
    - **Sub-workflow:** None.

#### 1.4 Filtering Stuck Tasks

- **Overview:**  
  Filters items to only those whose Status is exactly "Stuck".

- **Nodes Involved:**  
  - Filter for Stuck Items

- **Node Details:**

  - **Filter for Stuck Items**  
    - **Type & Role:** Filter node; conditionally passes only tasks with Status = "Stuck".  
    - **Configuration:**  
      - Condition: `$json.Status` equals "Stuck" (case sensitive, strict type validation).  
      - Combinator: AND (only one condition here).  
    - **Input/Output:** Input from "Set Columns"; outputs filtered items to "Alert Team".  
    - **Version:** 2.2  
    - **Edge Cases:**  
      - Case sensitivity means "stuck" or "STUCK" will not match.  
      - Tasks with missing or null Status will be excluded.  
      - If Status column mapping is wrong, filtering will fail.  
    - **Sub-workflow:** None.

#### 1.5 Slack Alert

- **Overview:**  
  Sends a Slack message alert to a specified user or channel for each stuck task detected.

- **Nodes Involved:**  
  - Alert Team

- **Node Details:**

  - **Alert Team**  
    - **Type & Role:** Slack node; sends chat messages via Slack API.  
    - **Configuration:**  
      - Text: dynamic message `"{{ $json.name }} task is stuck"` (inserts task name).  
      - Recipient: user mode by ID, with a placeholder `"yourid"` (should be replaced with actual Slack user ID).  
      - Authentication: OAuth2 using Slack Bot User OAuth token.  
    - **Credentials:** Slack OAuth2 API credential pre-configured with required scopes.  
    - **Input/Output:** Input from "Filter for Stuck Items"; no further output.  
    - **Version:** 2.3  
    - **Edge Cases:**  
      - Requires valid Slack OAuth2 token with appropriate scopes (`chat:write`, `channels:read`, `groups:read`, `users:read`).  
      - Invalid user ID or channel will cause message sending to fail.  
      - Rate limits or API downtime possible.  
      - Empty input (no stuck tasks) results in no messages sent.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                 | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                   |
|-----------------------------|--------------------|--------------------------------|----------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Start workflow manually         | —                          | Get many items1           |                                                                                                               |
| Get many items1             | Monday.com         | Retrieve all items from board/group | When clicking ‘Execute workflow’ | Set Columns              | See sticky note for Monday.com API token setup and board/group config                                         |
| Set Columns                 | Set                | Map and format task columns      | Get many items1             | Filter for Stuck Items    |                                                                                                               |
| Filter for Stuck Items      | Filter             | Filter tasks where Status = "Stuck" | Set Columns                 | Alert Team                |                                                                                                               |
| Alert Team                 | Slack              | Send Slack alert for stuck tasks | Filter for Stuck Items      | —                        | See sticky note for Slack OAuth2 app scopes setup and selecting user/channel                                  |
| Sticky Note64               | Sticky Note        | Workflow purpose and overview    | —                          | —                        | # 🚨 Monday.com “Stuck” Items → Slack Alerts (n8n) workflow explanation                                       |
| Sticky Note22               | Sticky Note        | Setup instructions for Monday.com and Slack credentials | —                          | —                        | Detailed setup instructions for Monday.com API token and Slack OAuth2 authentication                          |
| Sticky Note68               | Sticky Note        | Monday.com node connection guide | —                          | —                        | Setup instructions for Monday.com API token and node configuration                                           |
| Sticky Note69               | Sticky Note        | Slack API connection guide       | —                          | —                        | Slack app creation, scopes, installation, and credential setup instructions                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually to check for stuck tasks.

2. **Add a Monday.com Node**  
   - Type: Monday.com  
   - Operation: getAll  
   - Resource: boardItem  
   - Parameters:  
     - Board ID: `9865886546` (adjust to your board)  
     - Group ID: `new_group29179` (adjust to your group)  
   - Credentials: Create a new Monday.com API credential using your personal API token (see Monday.com Admin → API).  
   - Connect output of Manual Trigger to this node.

3. **Add a Set Node**  
   - Type: Set  
   - Purpose: Extract relevant fields from Monday.com items.  
   - Assignments:  
     - `name` = `{{$json["name"]}}`  
     - `Status` = `{{$json["column_values"][1]["text"]}}` (make sure this index matches your status column)  
     - `Due Date` = `{{$json["column_values"][2]["value"]}}`  
   - Connect output of Monday.com node to this node.

4. **Add a Filter Node**  
   - Type: Filter  
   - Condition: `$json.Status` equals `Stuck` (case sensitive)  
   - Connect output of Set node to Filter node.

5. **Add a Slack Node**  
   - Type: Slack  
   - Operation: Send message  
   - Text: `{{$json.name}} task is stuck`  
   - Recipient: Select mode "User" and specify the Slack user ID to alert (replace `"yourid"` with actual Slack user ID).  
   - Authentication: OAuth2 using Slack Bot User OAuth token.  
   - Credentials: Create Slack OAuth2 API credential with scopes:  
     - `chat:write`  
     - `channels:read`  
     - `groups:read`  
     - `users:read`  
   - Connect output of Filter node to Slack node.

6. **Save and Execute**  
   - Test the workflow by clicking “Execute workflow” manually.  
   - Confirm Slack messages are sent for any tasks with status "Stuck".

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow checks Monday.com board/group for items with Status = "Stuck" and sends Slack alerts to users/channels.          | Overview in Sticky Note64                                                                                 |
| Monday.com API token must be generated via Admin → API, then saved in n8n credentials as Monday.com API.                  | [Generate Monday API Token](https://developer.monday.com/api-reference/docs/authentication)               |
| Slack app must be created with OAuth scopes: `chat:write`, `channels:read`, `groups:read`, `users:read`, then installed.  | Slack API: <https://api.slack.com/apps>                                                                   |
| Contact for help mapping custom columns or routing alerts: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/, Site: https://ynteractive.com | Contact info in Sticky Note22                                                                              |

---

**Disclaimer:** The text and analysis provided are based exclusively on an automated workflow created with n8n, respecting all applicable content policies. No illegal or offensive content is included. All data processed is legal and publicly available.