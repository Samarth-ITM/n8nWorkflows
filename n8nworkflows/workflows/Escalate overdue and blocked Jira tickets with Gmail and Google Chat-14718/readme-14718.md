Escalate overdue and blocked Jira tickets with Gmail and Google Chat

https://n8nworkflows.xyz/workflows/escalate-overdue-and-blocked-jira-tickets-with-gmail-and-google-chat-14718


# Escalate overdue and blocked Jira tickets with Gmail and Google Chat

# 1. Workflow Overview

This workflow monitors Jira issues in open sprints and escalates notifications based on two risk dimensions:

- **Due date proximity / lateness**
- **Blocked status duration**

It can be started either manually for testing or by a daily schedule for production. It fetches Jira issues from a configured project, computes ticket urgency, then routes each issue through reminder, warning, escalation, or blocked-ticket handling branches. Notifications are sent via **Gmail** and **Google Chat webhook**.

## 1.1 Entry and Configuration

The workflow starts from either:

- **Manual Trigger** for testing
- **Schedule Trigger** for automated daily execution

Both feed a single configuration node where the Jira domain, project key, manager emails, and Google Chat webhook URL are defined.

## 1.2 Jira Retrieval and Issue Preparation

After configuration, the workflow retrieves all non-done Jira issues from open sprints in the configured project. It then reshapes the Jira payload into a simplified issue list and removes any issues without a due date.

## 1.3 Per-Issue Iteration and Deadline Insight Computation

Each issue is processed one by one via **Split In Batches**. A code node calculates:

- whether the issue is imminent
- whether it is overdue
- how many days remain or are overdue
- whether it is currently blocked based on labels

## 1.4 Main Routing Logic

A switch node evaluates each issue against several routes:

- imminent due date
- overdue up to 4 days
- overdue beyond 4 days
- blocked
- fallback continue-loop route

Because the switch enables **all matching outputs**, a single issue may trigger both a due-date branch and a blocked branch.

## 1.5 Due-Date Notification Escalation

The workflow applies progressive escalation for due-date-related cases:

- **0–2 days left**: email reminder to assignee
- **1–2 overdue days**: email warning to assignee
- **3–4 overdue days**: Google Chat notification to team channel
- **more than 4 overdue days**: email escalation to managers

Each branch includes a wait/rate-limit step before returning to the loop.

## 1.6 Blocked Ticket Escalation

If the issue has the `blocked` label, the workflow retrieves the Jira changelog, finds when the label was set, computes blocked duration, then routes as follows:

- **1 blocked day**: email warning to assignee
- **2 blocked days**: Google Chat notification
- **3 blocked days**: escalation email
- otherwise: continue loop

## 1.7 Loop Continuation and End

After each notification branch finishes, a code node returns a trivial success object back into the batch loop. When all issues have been processed, the loop completes at a NoOp end node.

---

# 2. Block-by-Block Analysis

## 2.1 Entry Triggers and Configuration

### Overview
This block defines how the workflow starts and centralizes the user-editable settings. It supports both manual execution and scheduled production runs.

### Nodes Involved
- When clicking ‘Execute workflow’
- Schedule Trigger
- ⚙️ CONFIG
- Note - What Is This Workflow
- Note - Manual Testing Trigger
- Note - Automatic Trigger
- Note - Config Instructions

### Node Details

#### When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for test runs.
- **Configuration choices:** No parameters; standard manual execution trigger.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **⚙️ CONFIG**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None functionally; only available when manually executing in editor.
- **Sub-workflow reference:** None.

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; production automation entry point.
- **Configuration choices:** Triggers at hour `7` on the defined interval schedule.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **⚙️ CONFIG**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** Timezone behavior depends on workflow/n8n instance timezone settings.
- **Sub-workflow reference:** None.

#### ⚙️ CONFIG
- **Type and technical role:** `n8n-nodes-base.set`; stores reusable configuration constants for downstream expressions.
- **Configuration choices:** Creates four string fields:
  - `JIRA_DOMAIN`
  - `JIRA_PROJECT_KEY`
  - `MANAGER_EMAILS`
  - `GOOGLE_CHAT_WEBHOOK_URL`
- **Key expressions or variables used:** Referenced downstream via expressions like:
  - `$('⚙️ CONFIG').first().json.JIRA_DOMAIN`
  - `$('⚙️ CONFIG').first().json.JIRA_PROJECT_KEY`
  - `$('⚙️ CONFIG').first().json.MANAGER_EMAILS`
  - `$('⚙️ CONFIG').first().json.GOOGLE_CHAT_WEBHOOK_URL`
- **Input and output connections:** Input from both triggers; output to **GET_JIRA_ISSUES**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Invalid Jira domain causes broken links and Jira API misconfiguration assumptions.
  - Invalid webhook URL causes HTTP 400/401/404 errors later.
  - `MANAGER_EMAILS` formatting must be compatible with Gmail send-to field.
- **Sub-workflow reference:** None.

#### Note - What Is This Workflow
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Describes purpose, setup, requirements, and external help link.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Note - Manual Testing Trigger
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Labels the manual trigger area.
- **Input and output connections:** None.

#### Note - Automatic Trigger
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Labels the schedule trigger area.
- **Input and output connections:** None.

#### Note - Config Instructions
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Indicates the config node is the only node meant to be edited.
- **Input and output connections:** None.

---

## 2.2 Jira Retrieval and Data Preparation

### Overview
This block queries Jira for all active sprint issues in the selected project, simplifies the payload, and filters out issues that have no due date.

### Nodes Involved
- GET_JIRA_ISSUES
- GET_ISSUES_LIST
- Note - Prepare Data Section

### Node Details

#### GET_JIRA_ISSUES
- **Type and technical role:** `n8n-nodes-base.jira`; fetches Jira issues from Jira Software Cloud.
- **Configuration choices:**
  - Operation: `getAll`
  - Return all: enabled
  - JQL:
    `project = {{ JIRA_PROJECT_KEY }} AND status != Done AND sprint in openSprints()`
- **Key expressions or variables used:**
  - `={{ $('⚙️ CONFIG').first().json.JIRA_PROJECT_KEY }}`
- **Input and output connections:** Input from **⚙️ CONFIG**; output to **GET_ISSUES_LIST**.
- **Version-specific requirements:** Type version `1`; requires Jira Software Cloud credentials.
- **Edge cases or potential failure types:**
  - Jira auth failure
  - Incorrect project key
  - JQL incompatibility if sprint fields are unavailable
  - Large result sets may increase runtime
- **Sub-workflow reference:** None.

#### GET_ISSUES_LIST
- **Type and technical role:** `n8n-nodes-base.code`; normalizes Jira issue payloads.
- **Configuration choices:** Extracts:
  - `key`
  - `due_date` from `fields.duedate`
  - `assignee` from `fields.assignee`
  - `labels` from `fields.labels`
  Filters out issues whose `due_date` is null.
- **Key expressions or variables used:** Uses `$input.all()`.
- **Input and output connections:** Input from **GET_JIRA_ISSUES**; output to **Loop Over Items**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - `fields.assignee` may be null for unassigned issues, which can later break expressions accessing `emailAddress` or `displayName`
  - `labels` may be missing or non-array in unusual Jira payloads
  - By filtering out null due dates, blocked issues without due dates are never processed
- **Sub-workflow reference:** None.

#### Note - Prepare Data Section
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Labels the data preparation area.
- **Input and output connections:** None.

---

## 2.3 Iteration and Per-Issue Date Analysis

### Overview
This block iterates through the prepared issues one at a time and computes the fields required for routing.

### Nodes Involved
- Loop Over Items
- ISSUE_DATE_INSIGHTS
- END PROCESS
- Note - Workflow End

### Node Details

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through issues sequentially.
- **Configuration choices:** Default options; batch size not explicitly set.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Input from **GET_ISSUES_LIST** and loop-back continuation nodes
  - Main output 0 to **END PROCESS**
  - Main output 1 to **ISSUE_DATE_INSIGHTS**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Loop behavior depends on proper reconnection from every branch
  - A broken branch can stall iteration
- **Sub-workflow reference:** None.

#### ISSUE_DATE_INSIGHTS
- **Type and technical role:** `n8n-nodes-base.code`; computes due-date and blockage indicators.
- **Configuration choices:** Produces:
  - `key`
  - `dueDate_string`
  - `today_string`
  - `isOverdue`
  - `isImminent`
  - `isBlocked`
  - `days_left`
  - `overdue_days`
  - `assignee`
- **Key expressions or variables used:**
  - `isBlocked: $input.last().json.labels.includes("blocked")`
  - `daysLeft = Math.ceil((due - now)/(1000*60*60*24))`
- **Input and output connections:** Input from **Loop Over Items**; output to **ROUTES**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - `labels.includes("blocked")` fails if `labels` is null/undefined
  - Date calculations use current server time, so timezone can shift ticket classification near midnight
  - `Math.ceil` with partial-day differences can produce unintuitive threshold behavior
- **Sub-workflow reference:** None.

#### END PROCESS
- **Type and technical role:** `n8n-nodes-base.noOp`; terminal node for completed iteration.
- **Configuration choices:** No action.
- **Input and output connections:** Input from **Loop Over Items** output 0; no output.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Note - Workflow End
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Labels workflow end area.
- **Input and output connections:** None.

---

## 2.4 Primary Routing Logic

### Overview
This block decides which escalation path(s) each issue should follow. It uses a multi-output switch with all matching outputs enabled, allowing simultaneous due-date and blocked routing.

### Nodes Involved
- ROUTES
- CONTINUE_LOOP
- Note - Level 1 Reminder
- Note - Level 2 Warning
- Note - Level 3 Escalation
- Note - Blocked Tickets Logic
- Note - Production Quotas Warning

### Node Details

#### ROUTES
- **Type and technical role:** `n8n-nodes-base.switch`; multi-branch router.
- **Configuration choices:** Five rule outputs:
  1. `isImminent == true`
  2. `isOverdue && overdue_days <= 4`
  3. `isOverdue && overdue_days > 4`
  4. `isBlocked == true`
  5. unconditional `true` fallback for loop continuation
- **Key expressions or variables used:**
  - `={{ $input.last().json.isImminent }}`
  - `={{ $input.last().json.isOverdue && $input.last().json.overdue_days <= 4}}`
  - `={{ $input.last().json.isOverdue && $input.last().json.overdue_days > 4 }}`
  - `={{ $input.last().json.isBlocked }}`
  - `={{ true // Continue Loop }}`
- **Input and output connections:**
  - Input from **ISSUE_DATE_INSIGHTS**
  - Output 0 to **SEND_DUEDATE_REMINDER**
  - Output 1 to **IS_OVERDUE_BETWEEN**
  - Output 2 to **MANAGER_ESCALATION**
  - Output 3 to **GET_ISSUE_CHANGELOG** and **DATA_FLOW**
  - Output 4 to **CONTINUE_LOOP**
- **Version-specific requirements:** Type version `3.4`, rules version `3`, `allMatchingOutputs: true`.
- **Edge cases or potential failure types:**
  - Because fallback is always true, every item also enters the continue path
  - With `allMatchingOutputs`, an item may trigger multiple notifications and multiple loop continuations
  - Parallel continuations can make execution logic harder to reason about
- **Sub-workflow reference:** None.

#### CONTINUE_LOOP
- **Type and technical role:** `n8n-nodes-base.code`; returns a success placeholder to re-enter the batch loop.
- **Configuration choices:** Returns `{ success: true }`.
- **Input and output connections:** Input from **ROUTES** fallback branch; output to **Loop Over Items**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** None significant.
- **Sub-workflow reference:** None.

#### Note - Level 1 Reminder
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Documents reminder rule for imminent due dates.

#### Note - Level 2 Warning
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Documents warning rules for overdue tickets.

#### Note - Level 3 Escalation
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Documents escalation rule beyond 4 overdue days.

#### Note - Blocked Tickets Logic
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Documents blocked escalation progression.

#### Note - Production Quotas Warning
- **Type and technical role:** Sticky note documentation.
- **Configuration choices:** Documents Gmail and Google Chat quotas plus retry strategy.
- **Input and output connections:** None.

---

## 2.5 Due-Date Reminder and Warning Paths

### Overview
This block handles imminent and moderately overdue due-date notifications. It sends either an assignee email or a Google Chat notification, then pauses for rate limiting before continuing the loop.

### Nodes Involved
- SEND_DUEDATE_REMINDER
- Wait - Rate Limit (Reminder)
- CONTINUE_LOOP_OVERDUE_WARNING
- IS_OVERDUE_BETWEEN
- SEND_DUEDATE_WARNING
- NOTIFY_TEAM
- Wait - Rate Limit (Warning)
- CONTINUE_LOOP_ESCALATION

### Node Details

#### SEND_DUEDATE_REMINDER
- **Type and technical role:** `n8n-nodes-base.gmail`; sends reminder email to ticket assignee.
- **Configuration choices:**
  - To: assignee email
  - Subject: `Jira DueDate Ticket Reminder 🧠`
  - Text body with Jira browse URL
  - `onError: continueRegularOutput`
  - `retryOnFail: true`, 5-second delay
- **Key expressions or variables used:**
  - `={{ $input.last().json.assignee.emailAddress }}`
  - `https://{{ $('⚙️ CONFIG').first().json.JIRA_DOMAIN }}/browse/{{ $input.last().json.key }}`
- **Input and output connections:** Input from **ROUTES**; output to **Wait - Rate Limit (Reminder)**.
- **Version-specific requirements:** Type version `2.2`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Assignee missing email
  - Gmail quota/rate limit exceeded
  - OAuth2 credential expiration
- **Sub-workflow reference:** None.

#### Wait - Rate Limit (Reminder)
- **Type and technical role:** `n8n-nodes-base.wait`; throttles reminder flow.
- **Configuration choices:** Wait amount `1` (unit not explicitly visible in JSON export, but intended as a short pause).
- **Input and output connections:** Input from **SEND_DUEDATE_REMINDER**; output to **CONTINUE_LOOP_OVERDUE_WARNING**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Wait duration interpretation depends on node UI defaults.
- **Sub-workflow reference:** None.

#### CONTINUE_LOOP_OVERDUE_WARNING
- **Type and technical role:** `n8n-nodes-base.code`; loop continuation marker.
- **Configuration choices:** Returns `{ success_leve_1: true }`.
- **Input and output connections:** Input from **Wait - Rate Limit (Reminder)**; output to **Loop Over Items**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### IS_OVERDUE_BETWEEN
- **Type and technical role:** `n8n-nodes-base.if`; sub-router for overdue days 1–4.
- **Configuration choices:** True if `overdue_days >= 1 && overdue_days <= 2`.
  - True branch: assignee warning email
  - False branch: team chat warning
- **Key expressions or variables used:**
  - `={{ $input.last().json.overdue_days >= 1 &&  $input.last().json.overdue_days <= 2 }}`
- **Input and output connections:** Input from **ROUTES** overdue branch; output 0 to **SEND_DUEDATE_WARNING**, output 1 to **NOTIFY_TEAM**.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:** Depends on correct numeric `overdue_days`.
- **Sub-workflow reference:** None.

#### SEND_DUEDATE_WARNING
- **Type and technical role:** `n8n-nodes-base.gmail`; sends overdue warning to assignee.
- **Configuration choices:**
  - To: assignee email
  - Subject: `Jira DueDate Ticket  Warning ⚠️`
  - Plain text message with issue URL
  - Continue on error and retry enabled
- **Key expressions or variables used:**
  - `={{ $input.last().json.assignee.emailAddress }}`
- **Input and output connections:** Input from **IS_OVERDUE_BETWEEN** true branch; output to **Wait - Rate Limit (Warning)**.
- **Version-specific requirements:** Type version `2.2`; Gmail OAuth2 required.
- **Edge cases or potential failure types:** Same Gmail/assignee issues as reminder node.
- **Sub-workflow reference:** None.

#### NOTIFY_TEAM
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends Google Chat card message for overdue issues.
- **Configuration choices:**
  - POST to configured webhook URL
  - JSON body with card sections
  - Header `Content-Type: application/json`
  - Batching: 1 item every 1100 ms
  - Retry enabled
- **Key expressions or variables used:**
  - `={{ $('⚙️ CONFIG').first().json.GOOGLE_CHAT_WEBHOOK_URL }}`
  - issue key, assignee name, due date, browse URL
- **Input and output connections:** Input from **IS_OVERDUE_BETWEEN** false branch; output to **Wait - Rate Limit (Warning)**.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Invalid webhook
  - Google Chat quota/rate-limit responses
  - Message schema rejection
  - Card text contains French phrases despite English workflow documentation
- **Sub-workflow reference:** None.

#### Wait - Rate Limit (Warning)
- **Type and technical role:** `n8n-nodes-base.wait`; throttles warning path.
- **Configuration choices:** Wait amount `1`.
- **Input and output connections:** Inputs from **SEND_DUEDATE_WARNING** and **NOTIFY_TEAM**; output to **CONTINUE_LOOP_ESCALATION**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Same as other wait nodes.
- **Sub-workflow reference:** None.

#### CONTINUE_LOOP_ESCALATION
- **Type and technical role:** `n8n-nodes-base.code`; loop continuation marker.
- **Configuration choices:** Returns `{ success_leve_2: true }`.
- **Input and output connections:** Input from **Wait - Rate Limit (Warning)**; output to **Loop Over Items**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.6 Due-Date Manager Escalation

### Overview
This block handles severely overdue tickets by emailing manager recipients, then rate-limiting before returning to the loop.

### Nodes Involved
- MANAGER_ESCALATION
- Wait - Rate Limit (Escalation)
- CONTINUE_LOOP_BLOCKED_WARNING

### Node Details

#### MANAGER_ESCALATION
- **Type and technical role:** `n8n-nodes-base.gmail`; manager escalation email for tickets overdue more than 4 days.
- **Configuration choices:**
  - To: `MANAGER_EMAILS` from config
  - Subject: `Jira DueDate Ticket Alert 🚨`
  - Plain text body with issue link
  - Continue on error and retry enabled
- **Key expressions or variables used:**
  - `={{ $('⚙️ CONFIG').first().json.MANAGER_EMAILS }}`
- **Input and output connections:** Input from **ROUTES** overdue>4 branch; output to **Wait - Rate Limit (Escalation)**.
- **Version-specific requirements:** Type version `2.2`; Gmail OAuth2 required.
- **Edge cases or potential failure types:**
  - Manager email list formatting may not be accepted as-is
  - Body text says “Hello {assignee}” even though email goes to managers
- **Sub-workflow reference:** None.

#### Wait - Rate Limit (Escalation)
- **Type and technical role:** `n8n-nodes-base.wait`; throttles escalation emails.
- **Configuration choices:** Wait amount `1`.
- **Input and output connections:** Input from **MANAGER_ESCALATION**; output to **CONTINUE_LOOP_BLOCKED_WARNING**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** None significant.
- **Sub-workflow reference:** None.

#### CONTINUE_LOOP_BLOCKED_WARNING
- **Type and technical role:** `n8n-nodes-base.code`; loop continuation marker.
- **Configuration choices:** Returns `{ success_leve_3: true }`.
- **Input and output connections:** Input from **Wait - Rate Limit (Escalation)**; output to **Loop Over Items**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.7 Blocked Ticket Changelog Analysis

### Overview
This block enriches blocked issues with changelog data so the workflow can estimate how long the issue has been blocked.

### Nodes Involved
- GET_ISSUE_CHANGELOG
- AGGREGATE_ITEMS
- DATA_FLOW
- MERGE_DATA
- GET_BLOCKED_ITEM
- COMPUTE_BLOCKED_DAYS

### Node Details

#### GET_ISSUE_CHANGELOG
- **Type and technical role:** `n8n-nodes-base.jira`; retrieves issue changelog entries.
- **Configuration choices:**
  - Operation: `changelog`
  - Issue key taken from current item
- **Key expressions or variables used:**
  - `={{ $input.last().json.key }}`
- **Input and output connections:** Input from **ROUTES** blocked branch; output to **AGGREGATE_ITEMS**.
- **Version-specific requirements:** Type version `1`; Jira credentials required.
- **Edge cases or potential failure types:**
  - Jira auth/permission issues
  - Changelog unavailable or incomplete
- **Sub-workflow reference:** None.

#### AGGREGATE_ITEMS
- **Type and technical role:** `n8n-nodes-base.aggregate`; collects all changelog entries into one field.
- **Configuration choices:**
  - Aggregate all item data
  - Include specified fields: `created, items`
  - Output field name: `jira_issue_changelog`
- **Input and output connections:** Input from **GET_ISSUE_CHANGELOG**; output to **MERGE_DATA** input 1.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Empty changelog set could produce missing or empty aggregated array
- **Sub-workflow reference:** None.

#### DATA_FLOW
- **Type and technical role:** `n8n-nodes-base.code`; preserves original current issue payload for later merge.
- **Configuration choices:** Returns current item unchanged.
- **Key expressions or variables used:** `...$input.last().json`
- **Input and output connections:** Input from **ROUTES** blocked branch; output to **MERGE_DATA** input 0.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** None significant.
- **Sub-workflow reference:** None.

#### MERGE_DATA
- **Type and technical role:** `n8n-nodes-base.merge`; combines original issue data with aggregated changelog by position.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: position
  - `executeOnce: true`
- **Input and output connections:** Inputs from **DATA_FLOW** and **AGGREGATE_ITEMS**; output to **GET_BLOCKED_ITEM**.
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:**
  - Combining by position assumes both inputs align exactly
  - `executeOnce: true` may create unexpected behavior if multiple blocked items are processed concurrently
- **Sub-workflow reference:** None.

#### GET_BLOCKED_ITEM
- **Type and technical role:** `n8n-nodes-base.code`; extracts changelog event where the labels field was set to `blocked`.
- **Configuration choices:** Filters `jira_issue_changelog` where:
  - `item.items[0].field == "labels"`
  - `item.items[0].toString === "blocked"`
- **Key expressions or variables used:** Accesses `jira_issue_changelog`.
- **Input and output connections:** Input from **MERGE_DATA**; output to **COMPUTE_BLOCKED_DAYS**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Assumes `items[0]` exists
  - Assumes first changed field is labels
  - Fails if `blocked` label was added alongside other labels or later removed/re-added
  - Multiple matches are not disambiguated
- **Sub-workflow reference:** None.

#### COMPUTE_BLOCKED_DAYS
- **Type and technical role:** `n8n-nodes-base.code`; computes days since blocked label was applied.
- **Configuration choices:** Uses `blocked_item[0].created`, compares with current date, floors day difference.
- **Key expressions or variables used:**
  - `const blocked_date = new Date($input.last().json.blocked_item[0].created)`
  - `blocked_days = Math.floor(diff / (1000 * 60 * 60 * 24))`
- **Input and output connections:** Input from **GET_BLOCKED_ITEM**; output to **Switch**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Crashes if `blocked_item` is empty
  - Timezone and partial-day effects can shift exact blocked day count
- **Sub-workflow reference:** None.

---

## 2.8 Blocked Ticket Routing and Notifications

### Overview
This block routes blocked tickets according to how many days they have been blocked and sends the corresponding notification.

### Nodes Involved
- Switch
- SEND_BLOCKED_WARNING
- NOTIFY_TEAM_BLOCKED
- MANAGER_ESCALATION_BLOCKED
- Wait - Rate Limit (Blocked Escalation)
- CONTINUE_LOOP_BLOCKED_ESCALATION

### Node Details

#### Switch
- **Type and technical role:** `n8n-nodes-base.switch`; blocked-day router.
- **Configuration choices:** Four outputs:
  1. `blocked_days == 1`
  2. `blocked_days == 2`
  3. `blocked_days == 3`
  4. unconditional continue path
- **Key expressions or variables used:**
  - `={{ $input.last().json.blocked_days }} == 1`
  - `== 2`
  - string equals `"3"` for third branch
  - fallback `={{ true // continue loop }}`
- **Input and output connections:**
  - Input from **COMPUTE_BLOCKED_DAYS**
  - Outputs to **SEND_BLOCKED_WARNING**, **NOTIFY_TEAM_BLOCKED**, **MANAGER_ESCALATION_BLOCKED**, and **Wait - Rate Limit (Blocked Escalation)** respectively
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Third condition compares as string while previous ones compare numbers
  - Greater than 3 blocked days do not escalate further; they simply continue
- **Sub-workflow reference:** None.

#### SEND_BLOCKED_WARNING
- **Type and technical role:** `n8n-nodes-base.gmail`; warning email for 1-day blocked issue.
- **Configuration choices:**
  - Sends to current assignee email
  - Subject: `Jira Blocked Ticket Warning ⚠️`
  - Plain text body
  - Retry enabled
- **Key expressions or variables used:**
  - `={{ $input.last().json.data.assignee.emailAddress }}`
- **Input and output connections:** Input from **Switch** output 0; output to **Wait - Rate Limit (Blocked Escalation)**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Missing `data.assignee`
  - Gmail credential or quota issues
- **Sub-workflow reference:** None.

#### NOTIFY_TEAM_BLOCKED
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends blocked warning to Google Chat.
- **Configuration choices:**
  - POST to configured webhook URL
  - JSON card body
  - Header `Content-Type: application/json`
  - Batching 1 item every 1100 ms
- **Key expressions or variables used:**
  - Uses `GOOGLE_CHAT_WEBHOOK_URL`
  - References top-level fields like `key`, `assignee`, `dueDate_string`
- **Input and output connections:** Input from **Switch** output 1; output to **Wait - Rate Limit (Blocked Escalation)**.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - This node appears inconsistent with upstream payload shape, which at this stage is `{ blocked_days, data }`
  - Expressions such as `$input.last().json.key` may be undefined here unless n8n auto-resolves differently than expected
  - Very likely payload mapping bug
- **Sub-workflow reference:** None.

#### MANAGER_ESCALATION_BLOCKED
- **Type and technical role:** `n8n-nodes-base.gmail`; escalation email for blocked ticket at 3 days.
- **Configuration choices:**
  - Sends to `data.assignee.emailAddress`
  - Subject: `Jira Blocked Ticket Alert 🚨`
  - Plain text body
- **Key expressions or variables used:**
  - `={{ $input.last().json.data.assignee.emailAddress }}`
- **Input and output connections:** Input from **Switch** output 2; output to **Wait - Rate Limit (Blocked Escalation)**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Despite its name and sticky note, it sends to assignee, not managers
  - Likely configuration bug
- **Sub-workflow reference:** None.

#### Wait - Rate Limit (Blocked Escalation)
- **Type and technical role:** `n8n-nodes-base.wait`; throttles blocked notification paths.
- **Configuration choices:** Wait amount `1`.
- **Input and output connections:** Inputs from **SEND_BLOCKED_WARNING**, **NOTIFY_TEAM_BLOCKED**, **MANAGER_ESCALATION_BLOCKED**, and Switch fallback output; output to **CONTINUE_LOOP_BLOCKED_ESCALATION**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** None significant.
- **Sub-workflow reference:** None.

#### CONTINUE_LOOP_BLOCKED_ESCALATION
- **Type and technical role:** `n8n-nodes-base.code`; returns control to main loop.
- **Configuration choices:** Returns `{ success_leve_4: true }`.
- **Input and output connections:** Input from **Wait - Rate Limit (Blocked Escalation)**; output to **Loop Over Items**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual test entry point |  | ⚙️ CONFIG | ## MANUAL TRIGGER FOR TESTING |
| Schedule Trigger | Schedule Trigger | Automated daily entry point |  | ⚙️ CONFIG | ## AUTOMATIC TRIGGER FOR PRODUCTION |
| ⚙️ CONFIG | Set | Central configuration values | When clicking ‘Execute workflow’; Schedule Trigger | GET_JIRA_ISSUES | ## ⚙️ CONFIGURATION NODE\nSet all your values here — **this is the only node you need to edit** |
| GET_JIRA_ISSUES | Jira | Fetch open sprint issues from Jira | ⚙️ CONFIG | GET_ISSUES_LIST | ## PREPARE DATA 📊 |
| GET_ISSUES_LIST | Code | Simplify Jira issue payload and filter due dates | GET_JIRA_ISSUES | Loop Over Items | ## PREPARE DATA 📊 |
| Loop Over Items | Split In Batches | Iterate issues sequentially | GET_ISSUES_LIST; CONTINUE_LOOP; CONTINUE_LOOP_OVERDUE_WARNING; CONTINUE_LOOP_ESCALATION; CONTINUE_LOOP_BLOCKED_WARNING; CONTINUE_LOOP_BLOCKED_ESCALATION | END PROCESS; ISSUE_DATE_INSIGHTS |  |
| ISSUE_DATE_INSIGHTS | Code | Compute due date and blocked indicators | Loop Over Items | ROUTES |  |
| ROUTES | Switch | Main routing for due-date and blocked logic | ISSUE_DATE_INSIGHTS | SEND_DUEDATE_REMINDER; IS_OVERDUE_BETWEEN; MANAGER_ESCALATION; GET_ISSUE_CHANGELOG; DATA_FLOW; CONTINUE_LOOP | ## LEVEL 1 -> REMINDER ASSIGNEE 🧠\n- [0 - 2] Imminent DueDate → Send email to ticket assignee\n## LEVEL 2 -> (WARNING ASSIGNEE OR NOTIFICATION CHANNEL) -> WARNING ⚠️\n\n* **if [1 - 2] OverDue Days** → Send Email to ticket assignee\n\n* **if ]2 - 4] OverDue Days** → Send notification to google chat dedicated channel\n## LEVEL 3 -> ESCALATION MANAGER -> ALERT 🚨\n* Greater than 4 OverDue Days → Send email to manager(s)\n## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| CONTINUE_LOOP | Code | Fallback loop continuation | ROUTES | Loop Over Items |  |
| SEND_DUEDATE_REMINDER | Gmail | Send imminent due-date reminder to assignee | ROUTES | Wait - Rate Limit (Reminder) | ## LEVEL 1 -> REMINDER ASSIGNEE 🧠\n- [0 - 2] Imminent DueDate → Send email to ticket assignee |
| Wait - Rate Limit (Reminder) | Wait | Throttle reminder emails | SEND_DUEDATE_REMINDER | CONTINUE_LOOP_OVERDUE_WARNING | ## LEVEL 1 -> REMINDER ASSIGNEE 🧠\n- [0 - 2] Imminent DueDate → Send email to ticket assignee |
| CONTINUE_LOOP_OVERDUE_WARNING | Code | Return from reminder branch to loop | Wait - Rate Limit (Reminder) | Loop Over Items | ## LEVEL 1 -> REMINDER ASSIGNEE 🧠\n- [0 - 2] Imminent DueDate → Send email to ticket assignee |
| IS_OVERDUE_BETWEEN | If | Split overdue 1–2 days vs 3–4 days | ROUTES | SEND_DUEDATE_WARNING; NOTIFY_TEAM | ## LEVEL 2 -> (WARNING ASSIGNEE OR NOTIFICATION CHANNEL) -> WARNING ⚠️\n\n* **if [1 - 2] OverDue Days** → Send Email to ticket assignee\n\n* **if ]2 - 4] OverDue Days** → Send notification to google chat dedicated channel |
| SEND_DUEDATE_WARNING | Gmail | Send overdue warning to assignee | IS_OVERDUE_BETWEEN | Wait - Rate Limit (Warning) | ## LEVEL 2 -> (WARNING ASSIGNEE OR NOTIFICATION CHANNEL) -> WARNING ⚠️\n\n* **if [1 - 2] OverDue Days** → Send Email to ticket assignee\n\n* **if ]2 - 4] OverDue Days** → Send notification to google chat dedicated channel |
| NOTIFY_TEAM | HTTP Request | Send overdue team alert to Google Chat | IS_OVERDUE_BETWEEN | Wait - Rate Limit (Warning) | ## LEVEL 2 -> (WARNING ASSIGNEE OR NOTIFICATION CHANNEL) -> WARNING ⚠️\n\n* **if [1 - 2] OverDue Days** → Send Email to ticket assignee\n\n* **if ]2 - 4] OverDue Days** → Send notification to google chat dedicated channel |
| Wait - Rate Limit (Warning) | Wait | Throttle warning notifications | SEND_DUEDATE_WARNING; NOTIFY_TEAM | CONTINUE_LOOP_ESCALATION | ## LEVEL 2 -> (WARNING ASSIGNEE OR NOTIFICATION CHANNEL) -> WARNING ⚠️\n\n* **if [1 - 2] OverDue Days** → Send Email to ticket assignee\n\n* **if ]2 - 4] OverDue Days** → Send notification to google chat dedicated channel |
| CONTINUE_LOOP_ESCALATION | Code | Return from warning branch to loop | Wait - Rate Limit (Warning) | Loop Over Items | ## LEVEL 2 -> (WARNING ASSIGNEE OR NOTIFICATION CHANNEL) -> WARNING ⚠️\n\n* **if [1 - 2] OverDue Days** → Send Email to ticket assignee\n\n* **if ]2 - 4] OverDue Days** → Send notification to google chat dedicated channel |
| MANAGER_ESCALATION | Gmail | Send severe overdue escalation email | ROUTES | Wait - Rate Limit (Escalation) | ## LEVEL 3 -> ESCALATION MANAGER -> ALERT 🚨\n* Greater than 4 OverDue Days → Send email to manager(s) |
| Wait - Rate Limit (Escalation) | Wait | Throttle manager escalation emails | MANAGER_ESCALATION | CONTINUE_LOOP_BLOCKED_WARNING | ## LEVEL 3 -> ESCALATION MANAGER -> ALERT 🚨\n* Greater than 4 OverDue Days → Send email to manager(s) |
| CONTINUE_LOOP_BLOCKED_WARNING | Code | Return from escalation branch to loop | Wait - Rate Limit (Escalation) | Loop Over Items | ## LEVEL 3 -> ESCALATION MANAGER -> ALERT 🚨\n* Greater than 4 OverDue Days → Send email to manager(s) |
| GET_ISSUE_CHANGELOG | Jira | Fetch blocked issue changelog | ROUTES | AGGREGATE_ITEMS | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| DATA_FLOW | Code | Preserve current blocked item data for merge | ROUTES | MERGE_DATA | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| AGGREGATE_ITEMS | Aggregate | Collect changelog entries into one array | GET_ISSUE_CHANGELOG | MERGE_DATA | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| MERGE_DATA | Merge | Combine original issue data with changelog | DATA_FLOW; AGGREGATE_ITEMS | GET_BLOCKED_ITEM | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| GET_BLOCKED_ITEM | Code | Find changelog event where blocked label was added | MERGE_DATA | COMPUTE_BLOCKED_DAYS | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| COMPUTE_BLOCKED_DAYS | Code | Compute days since blocked label event | GET_BLOCKED_ITEM | Switch | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| Switch | Switch | Route blocked issues by blocked-day count | COMPUTE_BLOCKED_DAYS | SEND_BLOCKED_WARNING; NOTIFY_TEAM_BLOCKED; MANAGER_ESCALATION_BLOCKED; Wait - Rate Limit (Blocked Escalation) | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| SEND_BLOCKED_WARNING | Gmail | Email assignee about blocked ticket | Switch | Wait - Rate Limit (Blocked Escalation) | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| NOTIFY_TEAM_BLOCKED | HTTP Request | Send blocked-ticket alert to Google Chat | Switch | Wait - Rate Limit (Blocked Escalation) | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| MANAGER_ESCALATION_BLOCKED | Gmail | Send blocked-ticket escalation email | Switch | Wait - Rate Limit (Blocked Escalation) | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| Wait - Rate Limit (Blocked Escalation) | Wait | Throttle blocked notifications | Switch; SEND_BLOCKED_WARNING; NOTIFY_TEAM_BLOCKED; MANAGER_ESCALATION_BLOCKED | CONTINUE_LOOP_BLOCKED_ESCALATION | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| CONTINUE_LOOP_BLOCKED_ESCALATION | Code | Return from blocked branch to loop | Wait - Rate Limit (Blocked Escalation) | Loop Over Items | ## Blocked Tickets (WARNING ASSIGNEE  ⚠️  OR ESCALATION EMAIL 🚨)\n* Blocked + 1 Days → Send Email to ticket assignee\n* Blocked + 2 Days -> Notify google chat team\n* Blocked + 3 Days → send email to manager.  |
| END PROCESS | NoOp | Terminal end node after loop completion | Loop Over Items |  | ## WORKFLOW END |
| Note - What Is This Workflow | Sticky Note | Documentation note |  |  |  |
| Note - Manual Testing Trigger | Sticky Note | Documentation note |  |  |  |
| Note - Automatic Trigger | Sticky Note | Documentation note |  |  |  |
| Note - Config Instructions | Sticky Note | Documentation note |  |  |  |
| Note - Prepare Data Section | Sticky Note | Documentation note |  |  |  |
| Note - Level 1 Reminder | Sticky Note | Documentation note |  |  |  |
| Note - Level 2 Warning | Sticky Note | Documentation note |  |  |  |
| Note - Level 3 Escalation | Sticky Note | Documentation note |  |  |  |
| Note - Blocked Tickets Logic | Sticky Note | Documentation note |  |  |  |
| Note - Production Quotas Warning | Sticky Note | Documentation note |  |  |  |
| Note - Workflow End | Sticky Note | Documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:
   `Jira Ticket Deadline & Blockage Notifier`.

2. **Add a Manual Trigger node**
   - Type: **Manual Trigger**
   - Keep default settings.
   - This is for testing.

3. **Add a Schedule Trigger node**
   - Type: **Schedule Trigger**
   - Configure it to run daily at hour **7**.
   - Keep it as the production entry point.

4. **Add a Set node named `⚙️ CONFIG`**
   - Type: **Set**
   - Add four string fields:
     1. `JIRA_DOMAIN`
     2. `JIRA_PROJECT_KEY`
     3. `MANAGER_EMAILS`
     4. `GOOGLE_CHAT_WEBHOOK_URL`
   - Example values:
     - `JIRA_DOMAIN = my-company.atlassian.net`
     - `JIRA_PROJECT_KEY = PROJ`
     - `MANAGER_EMAILS = manager1@example.com,manager2@example.com`
     - `GOOGLE_CHAT_WEBHOOK_URL = https://chat.googleapis.com/...`

5. **Connect both triggers to `⚙️ CONFIG`**
   - `Manual Trigger -> ⚙️ CONFIG`
   - `Schedule Trigger -> ⚙️ CONFIG`

6. **Add a Jira node named `GET_JIRA_ISSUES`**
   - Type: **Jira Software Cloud**
   - Operation: **Get All**
   - Return All: **true**
   - JQL:
     `project = {{ $('⚙️ CONFIG').first().json.JIRA_PROJECT_KEY }} AND status != Done AND sprint in openSprints()`
   - Configure Jira credentials:
     - Use **Jira Software Cloud API** credentials
     - Ensure the account can read issues and changelog data

7. **Connect `⚙️ CONFIG -> GET_JIRA_ISSUES`**

8. **Add a Code node named `GET_ISSUES_LIST`**
   - Paste logic that:
     - maps Jira issues to `key`, `due_date`, `assignee`, and `labels`
     - filters out issues with null due dates
   - Equivalent logic:
     - iterate over all input items
     - extract:
       - `key = issue.key`
       - `due_date = issue.fields.duedate`
       - `assignee = issue.fields.assignee`
       - `labels = issue.fields.labels`
     - keep only items where `due_date` is not null

9. **Connect `GET_JIRA_ISSUES -> GET_ISSUES_LIST`**

10. **Add a Split In Batches node named `Loop Over Items`**
    - Type: **Split In Batches**
    - Use default settings, or explicitly set batch size to **1** for predictable sequential handling.

11. **Connect `GET_ISSUES_LIST -> Loop Over Items`**

12. **Add a NoOp node named `END PROCESS`**
    - This acts as the terminal node after batch completion.

13. **Connect `Loop Over Items` main output 0 -> `END PROCESS`**

14. **Add a Code node named `ISSUE_DATE_INSIGHTS`**
    - Compute:
      - `isOverdue`
      - `isImminent`
      - `isBlocked`
      - `days_left`
      - `overdue_days`
      - plus `key`, `dueDate_string`, `today_string`, `assignee`
    - Use logic equivalent to:
      - parse due date
      - compare with current date
      - `isImminent = daysLeft >= 0 && daysLeft <= 2`
      - `isOverdue = daysLeft < 0`
      - `overdue_days = abs(daysLeft)` if overdue else 0
      - `isBlocked = labels includes "blocked"`

15. **Connect `Loop Over Items` main output 1 -> `ISSUE_DATE_INSIGHTS`**

16. **Add a Switch node named `ROUTES`**
    - Enable **All Matching Outputs**
    - Create 5 outputs:
      1. `isImminent` is true
      2. `isOverdue && overdue_days <= 4` is true
      3. `isOverdue && overdue_days > 4` is true
      4. `isBlocked` is true
      5. unconditional true fallback

17. **Connect `ISSUE_DATE_INSIGHTS -> ROUTES`**

## Due-date reminder branch

18. **Add a Gmail node named `SEND_DUEDATE_REMINDER`**
    - Operation: send email
    - To:
      `{{ $input.last().json.assignee.emailAddress }}`
    - Subject:
      `Jira DueDate Ticket Reminder 🧠`
    - Message:
      include assignee display name and Jira URL:
      `https://{{ $('⚙️ CONFIG').first().json.JIRA_DOMAIN }}/browse/{{ $input.last().json.key }}`
    - Email type: **text**
    - Error handling: **Continue**
    - Retry on fail: enabled
    - Wait between retries: `5000 ms`
    - Credentials: **Gmail OAuth2**

19. **Add a Wait node named `Wait - Rate Limit (Reminder)`**
    - Set a short wait, intended as about **1 second**

20. **Add a Code node named `CONTINUE_LOOP_OVERDUE_WARNING`**
    - Return a simple object such as `{ success_leve_1: true }`

21. **Connect**
    - `ROUTES output 0 -> SEND_DUEDATE_REMINDER`
    - `SEND_DUEDATE_REMINDER -> Wait - Rate Limit (Reminder)`
    - `Wait - Rate Limit (Reminder) -> CONTINUE_LOOP_OVERDUE_WARNING`
    - `CONTINUE_LOOP_OVERDUE_WARNING -> Loop Over Items`

## Due-date warning branch

22. **Add an If node named `IS_OVERDUE_BETWEEN`**
    - Condition:
      `overdue_days >= 1 && overdue_days <= 2`

23. **Connect `ROUTES output 1 -> IS_OVERDUE_BETWEEN`**

24. **Add a Gmail node named `SEND_DUEDATE_WARNING`**
    - To assignee email
    - Subject:
      `Jira DueDate Ticket  Warning ⚠️`
    - Body with issue URL
    - Email type text
    - Continue on error
    - Retry on fail enabled

25. **Add an HTTP Request node named `NOTIFY_TEAM`**
    - Method: **POST**
    - URL:
      `{{ $('⚙️ CONFIG').first().json.GOOGLE_CHAT_WEBHOOK_URL }}`
    - Send headers: yes
    - Header:
      `Content-Type: application/json`
    - Body type: JSON
    - Send body: yes
    - Configure Google Chat card payload containing:
      - issue key
      - assignee display name
      - due date
      - text note about overdue/imminent state
      - button linking to Jira issue
    - In options, set batching:
      - batch size `1`
      - interval around `1100 ms`

26. **Add a Wait node named `Wait - Rate Limit (Warning)`**
    - Short wait, intended as about **1 second**

27. **Add a Code node named `CONTINUE_LOOP_ESCALATION`**
    - Return `{ success_leve_2: true }`

28. **Connect**
    - `IS_OVERDUE_BETWEEN true -> SEND_DUEDATE_WARNING`
    - `IS_OVERDUE_BETWEEN false -> NOTIFY_TEAM`
    - `SEND_DUEDATE_WARNING -> Wait - Rate Limit (Warning)`
    - `NOTIFY_TEAM -> Wait - Rate Limit (Warning)`
    - `Wait - Rate Limit (Warning) -> CONTINUE_LOOP_ESCALATION`
    - `CONTINUE_LOOP_ESCALATION -> Loop Over Items`

## Due-date manager escalation branch

29. **Add a Gmail node named `MANAGER_ESCALATION`**
    - To:
      `{{ $('⚙️ CONFIG').first().json.MANAGER_EMAILS }}`
    - Subject:
      `Jira DueDate Ticket Alert 🚨`
    - Body with issue URL
    - Continue on error
    - Retry on fail enabled

30. **Add a Wait node named `Wait - Rate Limit (Escalation)`**
    - Short wait, intended as about **1 second**

31. **Add a Code node named `CONTINUE_LOOP_BLOCKED_WARNING`**
    - Return `{ success_leve_3: true }`

32. **Connect**
    - `ROUTES output 2 -> MANAGER_ESCALATION`
    - `MANAGER_ESCALATION -> Wait - Rate Limit (Escalation)`
    - `Wait - Rate Limit (Escalation) -> CONTINUE_LOOP_BLOCKED_WARNING`
    - `CONTINUE_LOOP_BLOCKED_WARNING -> Loop Over Items`

## Fallback continue branch

33. **Add a Code node named `CONTINUE_LOOP`**
    - Return `{ success: true }`

34. **Connect**
    - `ROUTES output 4 -> CONTINUE_LOOP`
    - `CONTINUE_LOOP -> Loop Over Items`

## Blocked issue changelog branch

35. **Add a Jira node named `GET_ISSUE_CHANGELOG`**
    - Operation: **Changelog**
    - Issue Key:
      `{{ $input.last().json.key }}`

36. **Add a Code node named `DATA_FLOW`**
    - Return the current input item unchanged.

37. **Add an Aggregate node named `AGGREGATE_ITEMS`**
    - Aggregate mode: all item data
    - Include specified fields:
      - `created`
      - `items`
    - Destination field:
      `jira_issue_changelog`

38. **Add a Merge node named `MERGE_DATA`**
    - Mode: **Combine**
    - Combine by: **Position**

39. **Connect**
    - `ROUTES output 3 -> GET_ISSUE_CHANGELOG`
    - `ROUTES output 3 -> DATA_FLOW`
    - `GET_ISSUE_CHANGELOG -> AGGREGATE_ITEMS`
    - `DATA_FLOW -> MERGE_DATA input 0`
    - `AGGREGATE_ITEMS -> MERGE_DATA input 1`

40. **Add a Code node named `GET_BLOCKED_ITEM`**
    - Filter changelog entries where:
      - changed field is `labels`
      - new value equals `blocked`
    - Return:
      - `blocked_item`
      - `data` containing original issue info

41. **Add a Code node named `COMPUTE_BLOCKED_DAYS`**
    - Read `blocked_item[0].created`
    - Compute day difference from current date
    - Return:
      - `blocked_days`
      - `data`

42. **Connect**
    - `MERGE_DATA -> GET_BLOCKED_ITEM`
    - `GET_BLOCKED_ITEM -> COMPUTE_BLOCKED_DAYS`

## Blocked issue routing branch

43. **Add a Switch node named `Switch`**
    - Output 0: `blocked_days == 1`
    - Output 1: `blocked_days == 2`
    - Output 2: `blocked_days == 3`
    - Output 3: unconditional continue path

44. **Connect `COMPUTE_BLOCKED_DAYS -> Switch`**

45. **Add a Gmail node named `SEND_BLOCKED_WARNING`**
    - To:
      `{{ $input.last().json.data.assignee.emailAddress }}`
    - Subject:
      `Jira Blocked Ticket Warning ⚠️`
    - Body with issue URL from `data.key`

46. **Add an HTTP Request node named `NOTIFY_TEAM_BLOCKED`**
    - POST to Google Chat webhook
    - Use JSON card payload
    - Ideally, reference `data.key`, `data.assignee.displayName`, and issue URL based on `data.key`
    - Note: the exported workflow seems to reference top-level fields here; correct this during rebuild.

47. **Add a Gmail node named `MANAGER_ESCALATION_BLOCKED`**
    - Intended behavior: send to managers after 3 blocked days
    - Recommended configuration:
      `{{ $('⚙️ CONFIG').first().json.MANAGER_EMAILS }}`
    - Note: the exported workflow actually sends to assignee; this appears incorrect.

48. **Add a Wait node named `Wait - Rate Limit (Blocked Escalation)`**
    - Short wait, intended as about **1 second**

49. **Add a Code node named `CONTINUE_LOOP_BLOCKED_ESCALATION`**
    - Return `{ success_leve_4: true }`

50. **Connect**
    - `Switch output 0 -> SEND_BLOCKED_WARNING`
    - `Switch output 1 -> NOTIFY_TEAM_BLOCKED`
    - `Switch output 2 -> MANAGER_ESCALATION_BLOCKED`
    - `Switch output 3 -> Wait - Rate Limit (Blocked Escalation)`
    - `SEND_BLOCKED_WARNING -> Wait - Rate Limit (Blocked Escalation)`
    - `NOTIFY_TEAM_BLOCKED -> Wait - Rate Limit (Blocked Escalation)`
    - `MANAGER_ESCALATION_BLOCKED -> Wait - Rate Limit (Blocked Escalation)`
    - `Wait - Rate Limit (Blocked Escalation) -> CONTINUE_LOOP_BLOCKED_ESCALATION`
    - `CONTINUE_LOOP_BLOCKED_ESCALATION -> Loop Over Items`

## Credentials setup

51. **Configure Jira credentials**
    - Use Jira Software Cloud account credentials with access to:
      - issue search
      - issue changelog reading

52. **Configure Gmail credentials**
    - Use Gmail OAuth2
    - Ensure the authorized account can send messages to assignees and managers

53. **Configure Google Chat**
    - Create an incoming webhook for the target Google Chat space
    - Paste the webhook URL into `GOOGLE_CHAT_WEBHOOK_URL`

## Recommended corrections while rebuilding

54. **Fix blocked manager escalation recipient**
    - Use manager emails for `MANAGER_ESCALATION_BLOCKED`, not assignee email.

55. **Fix blocked Google Chat payload references**
    - In blocked branch, the payload after `COMPUTE_BLOCKED_DAYS` is nested under `data`
    - Use:
      - `{{ $input.last().json.data.key }}`
      - `{{ $input.last().json.data.assignee.displayName }}`
      - `https://{{ $('⚙️ CONFIG').first().json.JIRA_DOMAIN }}/browse/{{ $input.last().json.data.key }}`

56. **Harden null handling**
    - Add checks for:
      - missing assignee
      - missing assignee email
      - empty labels array
      - missing blocked changelog event

57. **Optionally simplify loop continuation**
    - Current design has several separate continuation code nodes
    - You may replace them with a single reusable continuation pattern if desired

58. **Activate the workflow**
    - Test manually first
    - Then enable the schedule trigger for daily runs

### Sub-workflow setup
This workflow does **not** invoke any sub-workflow and does not contain any Execute Workflow node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Visit aixautomation.tech for help | [https://aixautomation.tech/](https://aixautomation.tech/) |
| Requirements listed in the workflow notes: Jira Software Cloud, Gmail (OAuth2), Google Chat Webhook | Internal workflow documentation |
| Production warning: Google Chat per-space limit is 60 messages/minute; per-project limit is 3,000/minute; max message size about 32 KB | Internal workflow documentation |
| Production warning: Gmail daily limit is about 500 recipients/day for free Gmail and about 2,000/day for Google Workspace | Internal workflow documentation |
| Suggested mitigation in notes: Split In Batches size 1, Wait node around 1 second, retry on failure 3 times, 5-second delay, continue on error | Internal workflow documentation |