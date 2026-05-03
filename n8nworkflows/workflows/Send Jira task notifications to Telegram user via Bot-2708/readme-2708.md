Send Jira task notifications to Telegram user via Bot

https://n8nworkflows.xyz/workflows/send-jira-task-notifications-to-telegram-user-via-bot-2708


# Send Jira task notifications to Telegram user via Bot

### 1. Workflow Overview

This workflow automates sending Jira issue notifications to Telegram users via a Telegram Bot. It targets teams using Jira for issue tracking and aims to promptly inform developers or stakeholders about new, updated, or reassigned Jira issues through Telegram messages. The workflow is triggered by Jira webhook events and processes them to identify the issue type and assignee, then sends a tailored notification to the corresponding Telegram user.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives Jira webhook POST requests with issue event data.
- **1.2 Validation and Filtering:** Checks the webhook payload for necessary fields and validates the assignee and event type.
- **1.3 Telegram Account Resolution:** Maps Jira assignee account IDs to Telegram chat IDs.
- **1.4 Event Type Routing:** Routes the flow based on the event type (issue created, updated, or assignee changed).
- **1.5 Notification Sending:** Sends formatted Telegram messages corresponding to the event type.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives incoming HTTP POST requests from Jira automation webhook triggers. It authenticates the request via header authentication and captures the issue event data.

- **Nodes Involved:**  
  - `jira-webhook`

- **Node Details:**  
  - **jira-webhook**  
    - Type: Webhook node  
    - Role: Entry point for Jira webhook events  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook path `1e4989bf-6a23-4415-bd17-72d08130c5c4`  
      - Authentication: Header authentication with configured credentials  
    - Input: External HTTP POST from Jira automation  
    - Output: JSON payload containing issue data and headers  
    - Edge Cases:  
      - Unauthorized requests if header auth fails  
      - Malformed or incomplete webhook payloads  
    - Notes: The webhook URL must be copied into Jira automation webhook action.

#### 2.2 Validation and Filtering

- **Overview:**  
  Validates the webhook payload to ensure the issue body exists, the assignee is defined, and the event type header is present. This prevents processing incomplete or irrelevant webhook calls.

- **Nodes Involved:**  
  - `check issue body, assignee and hook type`

- **Node Details:**  
  - **check issue body, assignee and hook type**  
    - Type: If node  
    - Role: Conditional gate to verify essential data presence  
    - Configuration:  
      - Checks that:  
        - `body` field in webhook JSON is not empty  
        - `headers.type` exists and is a string  
        - `body.fields.assignee` is not empty  
    - Input: Output from `jira-webhook`  
    - Output: Passes only valid events forward  
    - Edge Cases:  
      - Missing assignee (e.g., unassigned issues) will block flow  
      - Missing or malformed headers or body will block flow

#### 2.3 Telegram Account Resolution

- **Overview:**  
  Maps the Jira assignee's `accountId` to a Telegram `chatId` using a hardcoded dictionary. This enables sending messages to the correct Telegram user.

- **Nodes Involved:**  
  - `telegram account`  
  - `check tg account exists`

- **Node Details:**  
  - **telegram account**  
    - Type: Code node (JavaScript)  
    - Role: Extracts Jira assignee accountId and maps it to Telegram chatId  
    - Configuration:  
      - Reads `accountId` from webhook JSON path: `body.fields.assignee.accountId`  
      - Uses a hardcoded object mapping Jira account IDs to Telegram chat IDs  
      - Returns an object with `telegramChatId`  
    - Input: Validated webhook data  
    - Output: JSON with `telegramChatId`  
    - Edge Cases:  
      - Missing or unmapped accountId results in undefined `telegramChatId`  
      - Requires manual update of mapping for all team members  
  - **check tg account exists**  
    - Type: If node  
    - Role: Checks if `telegramChatId` exists before proceeding  
    - Configuration:  
      - Condition: `telegramChatId` exists and is a number  
    - Input: Output from `telegram account`  
    - Output: Only passes if Telegram chat ID is found  
    - Edge Cases:  
      - Missing chat ID blocks notification sending  
      - Prevents errors from sending messages to undefined recipients

#### 2.4 Event Type Routing

- **Overview:**  
  Routes the workflow based on the event type header (`type`) in the webhook request. Supports three event types: `created`, `updated`, and `change-assignee`.

- **Nodes Involved:**  
  - `check type`

- **Node Details:**  
  - **check type**  
    - Type: Switch node  
    - Role: Directs flow to the appropriate notification node based on event type  
    - Configuration:  
      - Switch on `headers.type` from webhook JSON  
      - Cases:  
        - `"created"` → Send Create notification  
        - `"updated"` → Send Update notification  
        - `"change-assignee"` → Send Assign Alert notification  
    - Input: Output from `check tg account exists`  
    - Output: One of three branches to send messages  
    - Edge Cases:  
      - Unknown or missing `type` header results in no notification  
      - Case-sensitive matching enforced

#### 2.5 Notification Sending

- **Overview:**  
  Sends formatted Telegram messages to the resolved Telegram chat ID, customized per event type with issue details.

- **Nodes Involved:**  
  - `Send Create`  
  - `Send Update`  
  - `Send Assign Alert`

- **Node Details:**  
  - **Send Create**  
    - Type: Telegram node  
    - Role: Sends notification for newly created issues  
    - Configuration:  
      - Text template includes issue type, project, key, title, description, and creation time formatted as `yyyy-MM-dd HH:mm`  
      - Chat ID from `telegram account` node  
      - Attribution disabled  
      - Uses Telegram API credentials with bot token  
    - Input: From `check type` on `created` branch  
    - Output: Telegram message sent  
    - Edge Cases:  
      - Telegram API errors (invalid token, chat ID)  
      - Empty fields in issue data handled gracefully by template  
  - **Send Update**  
    - Type: Telegram node  
    - Role: Sends notification for updated issues  
    - Configuration: Similar to `Send Create` but message prefixed with "⚠️ Update"  
    - Input: From `check type` on `updated` branch  
    - Output: Telegram message sent  
  - **Send Assign Alert**  
    - Type: Telegram node  
    - Role: Sends notification when issue is assigned or reassigned  
    - Configuration: Similar template, prefixed with "👩‍💻👨‍💻 Assigned to you"  
    - Input: From `check type` on `change-assignee` branch  
    - Output: Telegram message sent  
    - Edge Cases for all Telegram nodes:  
      - Network issues or Telegram API downtime  
      - Invalid chat IDs or revoked bot permissions

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                          | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                                   |
|-------------------------------|--------------------|----------------------------------------|------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------|
| jira-webhook                  | Webhook            | Receives Jira webhook POST requests    | External HTTP POST            | check issue body, assignee and hook type | Jira webhook URL must be copied into Jira automation webhook action. Header auth used for security.           |
| check issue body, assignee and hook type | If                 | Validates webhook payload and assignee | jira-webhook                 | telegram account                 |                                                                                                               |
| telegram account             | Code               | Maps Jira assignee accountId to Telegram chatId | check issue body, assignee and hook type | check tg account exists          | Add your Jira accountId and corresponding Telegram chatId in the code node mapping object.                     |
| check tg account exists       | If                 | Checks if Telegram chatId exists       | telegram account             | check type                      |                                                                                                               |
| check type                   | Switch             | Routes flow based on webhook event type | check tg account exists      | Send Create, Send Update, Send Assign Alert |                                                                                                               |
| Send Create                  | Telegram            | Sends Telegram message for new issues  | check type (created branch)  | None                           |                                                                                                               |
| Send Update                  | Telegram            | Sends Telegram message for updated issues | check type (updated branch)  | None                           |                                                                                                               |
| Send Assign Alert            | Telegram            | Sends Telegram message for reassigned issues | check type (change-assignee branch) | None                           |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a Webhook node named `jira-webhook`.  
   - Set HTTP Method to POST.  
   - Set Path to a unique string (e.g., `1e4989bf-6a23-4415-bd17-72d08130c5c4`).  
   - Enable Header Authentication and configure credentials with a header key and value (e.g., API key).  
   - Save credentials as `Header Auth account`.  
   - Note: Copy the webhook URL generated here for Jira automation setup.

2. **Create If Node for Validation**  
   - Add an If node named `check issue body, assignee and hook type`.  
   - Configure conditions (all must be true):  
     - `body` field in webhook JSON is not empty (`{{$json["body"]}}` not empty).  
     - `headers.type` exists and is a string.  
     - `body.fields.assignee` is not empty.  
   - Connect `jira-webhook` output to this node input.

3. **Create Code Node for Telegram Account Mapping**  
   - Add a Code node named `telegram account`.  
   - Paste the following JavaScript code (adjust the mapping):  
     ```javascript
     const accountId = $('jira-webhook').first().json.body.fields.assignee?.accountId;

     const telegramAccounts = {
       "[jira account id]": 00000000, // Replace with actual Telegram chat ID
     };

     const telegramChatId = telegramAccounts[accountId];

     return [{ telegramChatId }];
     ```  
   - Connect output of `check issue body, assignee and hook type` to this node.

4. **Create If Node to Check Telegram Chat ID Exists**  
   - Add an If node named `check tg account exists`.  
   - Condition: Check if `telegramChatId` exists and is a number (`={{$('telegram account').item.json.telegramChatId}}` exists).  
   - Connect output of `telegram account` to this node.

5. **Create Switch Node for Event Type Routing**  
   - Add a Switch node named `check type`.  
   - Configure rules based on `headers.type` field from webhook JSON:  
     - Case 1: equals `"created"`  
     - Case 2: equals `"updated"`  
     - Case 3: equals `"change-assignee"`  
   - Connect output of `check tg account exists` to this node.

6. **Create Telegram Nodes for Notifications**  
   - Add three Telegram nodes named `Send Create`, `Send Update`, and `Send Assign Alert`.  
   - For each node:  
     - Select Telegram API credentials (create or select existing with your bot token).  
     - Set `chatId` to `={{$("telegram account").item.json.telegramChatId}}`.  
     - Disable attribution (optional).  
     - Configure message text templates as follows:

     **Send Create:**  
     ```
     🆕 New {{$json["body"]["fields"]["issuetype"]["name"]}}

     🔰 Project: `{{$json["body"]["fields"]["project"]["name"]}}`

     🆔 Key: `{{$json["body"]["key"]}}`

     🔰 Title: `{{$json["body"]["fields"]["summary"]}}`

     🔰 Description: `{{$json["body"]["fields"]["description"]}}`

     Create Time: `{{DateTime.fromMillis($json["body"]["fields"]["created"]).format("yyyy-MM-dd HH:mm")}}`
     ```

     **Send Update:**  
     ```
     ⚠️ Update {{$json["body"]["fields"]["issuetype"]["name"]}}

     🔰 Project: `{{$json["body"]["fields"]["project"]["name"]}}`

     🆔 Key: `{{$json["body"]["key"]}}`

     🔰 Title: `{{$json["body"]["fields"]["summary"]}}`

     🔰 Description: `{{$json["body"]["fields"]["description"]}}`

     Create Time: `{{DateTime.fromMillis($json["body"]["fields"]["created"]).format("yyyy-MM-dd HH:mm")}}`
     ```

     **Send Assign Alert:**  
     ```
     👩‍💻👨‍💻 Assigned to you {{$json["body"]["fields"]["issuetype"]["name"]}}

     🔰 Project: `{{$json["body"]["fields"]["project"]["name"]}}`

     🆔 Key: `{{$json["body"]["key"]}}`

     🔰 Title: `{{$json["body"]["fields"]["summary"]}}`

     🔰 Description: `{{$json["body"]["fields"]["description"]}}`

     Create Time: `{{DateTime.fromMillis($json["body"]["fields"]["created"]).format("yyyy-MM-dd HH:mm")}}`
     ```

7. **Connect Switch Outputs to Telegram Nodes**  
   - Connect `"created"` case output to `Send Create`.  
   - Connect `"updated"` case output to `Send Update`.  
   - Connect `"change-assignee"` case output to `Send Assign Alert`.

8. **Credential Setup**  
   - Create Telegram API credentials with your Telegram Bot token.  
   - Create HTTP Header Auth credentials for Jira webhook authentication.

9. **Jira Automation Setup**  
   - In Jira, create automation rules with trigger `Issue Created` (and others as needed).  
   - Add action `Send webhook request` with:  
     - URL: webhook URL from `jira-webhook` node  
     - Method: POST  
     - Body: Issue Data (Automation format)  
     - Header: `type` with value matching event type (`created`, `updated`, `change-assignee`).

10. **Testing**  
    - Start the Telegram bot.  
    - Create or update Jira issues assigned to users with mapped Telegram chat IDs.  
    - Verify Telegram notifications are received.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To enable Telegram notifications, register a Telegram Bot and obtain its token.                                | Telegram Bot API: https://core.telegram.org/bots/api                                           |
| Jira automation webhook setup requires copying the webhook URL from the `jira-webhook` node.                   | Jira Global Automation settings → Create Rule → Send Webhook Request                           |
| Mapping Jira `accountId` to Telegram `chatId` must be manually maintained in the `telegram account` code node. | Update the JavaScript object with your team’s Jira account IDs and corresponding Telegram chat IDs. |
| Telegram Bot must be started and accessible for messages to be delivered.                                      | Telegram client app or bot must be active                                                      |
| Use header authentication on the webhook node to secure the endpoint from unauthorized requests.               | Configure HTTP Header Auth credentials in n8n                                                  |

---

This documentation fully describes the workflow structure, logic, and setup instructions to enable developers or automation agents to understand, reproduce, and maintain the Jira-to-Telegram notification integration.