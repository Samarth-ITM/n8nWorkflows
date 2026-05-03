Update people through Zulip about open tickets in Zammad

https://n8nworkflows.xyz/workflows/update-people-through-zulip-about-open-tickets-in-zammad-1575


# Update people through Zulip about open tickets in Zammad

### 1. Workflow Overview

This workflow automates daily reporting of open tickets from Zammad to Zulip, specifically targeting the customer support team’s daily standup updates. It runs on weekdays at 08:30 and summarizes open tickets by their status categories before sending the summary message to a Zulip stream.

The workflow is logically divided into these blocks:

- **1.1 Triggering Input:** Initiates the workflow either manually or by a scheduled cron job.
- **1.2 Ticket Retrieval:** Fetches all tickets from Zammad.
- **1.3 Ticket Filtering and Aggregation:** Counts tickets per status category.
- **1.4 Notification Dispatch:** Sends the ticket summary to a Zulip stream.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering Input

- **Overview:** This block starts the workflow. It supports both manual execution and scheduled automated runs on weekdays at 08:30.
- **Nodes Involved:** 
  - `On clicking 'execute'`
  - `Standup Cron`

##### Node: On clicking 'execute'

- **Type & Role:** Manual Trigger node; allows manual initiation of the workflow.
- **Configuration:** No parameters configured; triggers immediately on manual execution.
- **Expressions:** None.
- **Inputs:** None (start node).
- **Outputs:** Connects to `List Tickets`.
- **Version Requirements:** Compatible with all n8n versions supporting manual triggers.
- **Edge Cases:** Manual execution bypasses scheduling; no data input validation needed.
- **Sub-workflow:** None.

##### Node: Standup Cron

- **Type & Role:** Cron Trigger node; schedules automatic workflow execution.
- **Configuration:** Set to trigger at 08:30 AM, Monday through Friday (`cronExpression: 0 30 8 * * 1-5`).
- **Expressions:** None.
- **Inputs:** None (start node).
- **Outputs:** Connects to `List Tickets`.
- **Version Requirements:** Requires n8n version supporting Cron node with custom cron expressions.
- **Edge Cases:** Cron misconfiguration or server downtime may prevent triggering.
- **Sub-workflow:** None.

---

#### 2.2 Ticket Retrieval

- **Overview:** Retrieves all tickets from Zammad via API.
- **Nodes Involved:** `List Tickets`

##### Node: List Tickets

- **Type & Role:** Zammad node configured to fetch ticket data.
- **Configuration:** 
  - Resource: `ticket`
  - Operation: `getAll`
  - Return all tickets (`returnAll: true`)
- **Expressions:** None.
- **Inputs:** Triggered from `On clicking 'execute'` and `Standup Cron`.
- **Outputs:** Connects to `Ticket Filtering`.
- **Credentials:** Uses `Zammad Token Auth account` (token-based authentication).
- **Version Requirements:** Requires the Zammad node version compatible with token-based authentication.
- **Edge Cases:**
  - API authentication failure (invalid or expired token).
  - API rate limits or timeouts.
  - Large ticket volume may cause performance issues.
- **Sub-workflow:** None.

---

#### 2.3 Ticket Filtering and Aggregation

- **Overview:** Processes the full list of tickets, counting how many fall into key status categories for reporting.
- **Nodes Involved:** `Ticket Filtering`

##### Node: Ticket Filtering

- **Type & Role:** Function node to aggregate ticket counts by status.
- **Configuration:**
  - Custom JavaScript code iterates over all tickets.
  - Counts tickets with `state_id` matching specific statuses:
    - `1`: new tickets
    - `2`: open tickets
    - `3`: pending reminder
    - `7`: pending close
  - Returns a single JSON object with aggregated counts.
- **Key Expressions/Variables:**
  - Accesses `items` array input.
  - Uses properties `json.state_id`.
- **Inputs:** Receives all tickets from `List Tickets`.
- **Outputs:** Passes aggregated counts to `Notify for Standup`.
- **Version Requirements:** Compatible with n8n versions supporting Function node.
- **Edge Cases:**
  - Unexpected or missing `state_id` fields.
  - Empty ticket lists (will return zero counts).
  - Potential JavaScript runtime errors if input format changes.
- **Sub-workflow:** None.

---

#### 2.4 Notification Dispatch

- **Overview:** Sends a formatted summary message about ticket counts to a Zulip stream for daily standups.
- **Nodes Involved:** `Notify for Standup`

##### Node: Notify for Standup

- **Type & Role:** Zulip node to send a message to a stream.
- **Configuration:**
  - Operation: `sendStream`
  - Stream: `customer support`
  - Topic: `tickets`
  - Content: Multiline message summarizing ticket counts using expressions to inject data from `Ticket Filtering` node:
    ```
    :ticket: Support Tickets Summary:
    * Open: {{$node["Ticket Filtering"].json["open"]}}
    * New: {{$node["Ticket Filtering"].json["new"]}}
    * Pending Close {{$node["Ticket Filtering"].json["pendingClose"]}}
    * Pending Reminder {{$node["Ticket Filtering"].json["pendingReminder"]}}
    ```
- **Expressions:** Uses mustache syntax to pull counts dynamically.
- **Inputs:** Gets aggregated ticket counts from `Ticket Filtering`.
- **Outputs:** None (end node).
- **Credentials:** Uses `Zulip n8n Bot` credentials.
- **Version Requirements:** Requires n8n version supporting Zulip node and API.
- **Edge Cases:**
  - Zulip API authentication failure.
  - Network issues preventing message delivery.
  - Stream or topic misconfiguration leading to message not appearing.
- **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name            | Node Type             | Functional Role                   | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                         |
|----------------------|-----------------------|---------------------------------|------------------------|------------------------|---------------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger        | Manual workflow start trigger   | —                      | List Tickets           |                                                                                                   |
| Standup Cron         | Cron Trigger           | Scheduled workflow start trigger | —                      | List Tickets           | Daily stand-up open days.                                                                          |
| List Tickets          | Zammad                 | Fetch all tickets from Zammad   | On clicking 'execute', Standup Cron | Ticket Filtering       | Get all tickets.                                                                                   |
| Ticket Filtering      | Function               | Aggregate ticket counts by status | List Tickets           | Notify for Standup      | Filter tickets by status.                                                                          |
| Notify for Standup    | Zulip                  | Send ticket summary to Zulip stream | Ticket Filtering       | —                      | Sends a summary to customer support stream.                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named `On clicking 'execute'` with no parameters.
   - Add a **Cron Trigger** node named `Standup Cron`.
     - Set Trigger Times to Custom Cron Expression: `0 30 8 * * 1-5` (8:30 AM Monday to Friday).
   - Connect both trigger nodes’ output to the next node `List Tickets`.

2. **Add Zammad Node to List Tickets:**
   - Add a **Zammad** node named `List Tickets`.
   - Set resource to `ticket`.
   - Set operation to `getAll`.
   - Enable `Return All` to fetch all tickets.
   - Configure credentials using a Zammad API Token (create or reuse a token with read access to tickets).
   - Connect inputs from both triggers.
   - Connect output to the `Ticket Filtering` node.

3. **Add Function Node for Filtering:**
   - Add a **Function** node named `Ticket Filtering`.
   - Paste the following JavaScript code into the function editor:
     ```javascript
     let newTickets = 0;
     let openTickets = 0;
     let pendingReminder = 0;
     let pendingClose = 0;

     for (let i = 0; i < items.length; i++) {
       const ticket = items[i];
       if (ticket.json.state_id === 1) {
         newTickets++;
       }
       if (ticket.json.state_id === 2) {
         openTickets++;
       }
       if (ticket.json.state_id === 3) {
         pendingReminder++;
       }
       if (ticket.json.state_id === 7) {
         pendingClose++;
       }
     }

     return [{
       json: {
         "new": newTickets,
         open: openTickets,
         pendingReminder: pendingReminder,
         pendingClose: pendingClose
       }
     }];
     ```
   - Connect input from `List Tickets`.
   - Connect output to `Notify for Standup`.

4. **Add Zulip Node for Notification:**
   - Add a **Zulip** node named `Notify for Standup`.
   - Set operation to `sendStream`.
   - Set Stream to `customer support`.
   - Set Topic to `tickets`.
   - Set Content to:
     ```
     :ticket: Support Tickets Summary:
     * Open: {{$node["Ticket Filtering"].json["open"]}}
     * New: {{$node["Ticket Filtering"].json["new"]}}
     * Pending Close {{$node["Ticket Filtering"].json["pendingClose"]}}
     * Pending Reminder {{$node["Ticket Filtering"].json["pendingReminder"]}}
     ```
   - Configure credentials with Zulip API credentials for a bot user authorized to post in the `customer support` stream.
   - Connect input from `Ticket Filtering`.

5. **Activate Workflow:**
   - Save and activate the workflow.
   - Confirm credentials for Zammad and Zulip are valid.
   - Test manual trigger via `On clicking 'execute'`.
   - Verify scheduled runs via `Standup Cron` on weekdays at 08:30.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                    |
|------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Cron expression used: `0 30 8 * * 1-5` triggers workflow weekdays at 08:30 AM.                  | Standard cron syntax for scheduling in n8n Cron node |
| Zammad API token authentication is used; ensure token has sufficient permissions to read tickets. | https://admin-docs.zammad.org/en/latest/api/tokens.html |
| Zulip bot credentials require access to the `customer support` stream to post messages.         | Zulip API docs: https://zulip.com/api/             |
| Summary message formatting uses Mustache templating to inject dynamic ticket counts.            | n8n Expressions documentation: https://docs.n8n.io/nodes/expressions/ |
| JavaScript function assumes ticket JSON contains `state_id` field; changes in Zammad API may require updates. | Zammad ticket states: https://admin-docs.zammad.org/en/latest/api/tickets.html |

---

This document fully covers the workflow structure, node configurations, and provides detailed guidance to reproduce or modify the process, including error considerations and integration details.