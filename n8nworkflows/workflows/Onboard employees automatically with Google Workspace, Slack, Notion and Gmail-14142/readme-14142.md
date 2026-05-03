Onboard employees automatically with Google Workspace, Slack, Notion and Gmail

https://n8nworkflows.xyz/workflows/onboard-employees-automatically-with-google-workspace--slack--notion-and-gmail-14142


# Onboard employees automatically with Google Workspace, Slack, Notion and Gmail

# 1. Workflow Overview

This workflow automates a multi-step employee onboarding process triggered by an HR system via webhook. It provisions a Google Workspace account, posts a Slack welcome message, creates a Notion onboarding page, sends a welcome email through Gmail, waits 7 days, checks onboarding progress in Notion, and then either alerts the manager or continues to Day 30 and posts a completion message.

Its main use case is standardized Day 0–30 onboarding for new employees using common SaaS systems. The design assumes HR sends a JSON payload with employee identity and organizational data, and that all integrations are pre-authorized.

## 1.1 Input Reception and Data Normalization
The workflow starts with an HTTP POST webhook and validates the incoming employee payload. It also derives normalized fields such as first name, last name, temporary password, and organizational unit path.

## 1.2 Account Provisioning and Welcome Actions
Once the payload is prepared, the workflow creates the employee’s Google Workspace account, posts a Slack welcome message, creates a Notion onboarding page with a checklist, and sends a welcome email.

## 1.3 Delayed Follow-Up
After initial onboarding actions, the workflow pauses for 7 days before checking whether onboarding tasks appear complete in Notion.

## 1.4 Progress Evaluation and Branching
The workflow counts returned Notion items and checks whether at least 3 tasks are complete. If yes, it waits until Day 30 and posts a completion message. If not, it sends a Slack alert for manager follow-up.

## 1.5 Documentation and Setup Guidance
Several sticky notes provide operating context, prerequisites, setup reminders, and a high-level execution summary. These notes are not executable but are important for maintaining and configuring the workflow.

---

# 2. Block-by-Block Analysis

## Block 1 — Workflow Documentation and Operating Context

**Overview:**  
This block contains non-executable sticky notes that document the workflow purpose, required credentials, setup steps, and overall logic. These notes are critical for implementation and maintenance but do not affect runtime execution.

**Nodes Involved:**  
- Overview
- Prerequisites
- Setup Required
- How It Works

### Node Details

#### Overview
- **Type and technical role:** Sticky Note; visual documentation node.
- **Configuration choices:** Describes the workflow as an “Employee Onboarding Orchestrator” for Day 0–30 onboarding.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Prerequisites
- **Type and technical role:** Sticky Note; setup documentation.
- **Configuration choices:** Lists required credentials and expected webhook payload structure.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** Misleading setup if scopes or payload format are not followed exactly.
- **Sub-workflow reference:** None.

#### Setup Required
- **Type and technical role:** Sticky Note; implementation checklist.
- **Configuration choices:** Identifies which nodes need credentials and which placeholder values must be replaced.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** If placeholders such as channel IDs or Notion DB ID are not replaced, runtime failures or misdirected messages will occur.
- **Sub-workflow reference:** None.

#### How It Works
- **Type and technical role:** Sticky Note; execution summary.
- **Configuration choices:** Explains the step-by-step business flow from webhook receipt through Day 30 completion.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

---

## Block 2 — Input Reception and Payload Construction

**Overview:**  
This block receives the onboarding request from an HR source and transforms it into a clean internal data structure. It also validates that all required fields are present before any downstream provisioning begins.

**Nodes Involved:**  
- New Employee Webhook
- Build Payload note
- Build Payload

### Node Details

#### New Employee Webhook
- **Type and technical role:** Webhook; workflow entry point.
- **Configuration choices:**  
  - HTTP method: `POST`  
  - Path: `onboarding`  
  - Response mode: `responseNode`
- **Key expressions or variables used:** Incoming request body is expected to contain:
  - `employee_name`
  - `employee_email`
  - `manager_email`
  - `department`
  - `start_date`
  - `employee_id`
- **Input and output connections:**  
  - Input: none, entry point  
  - Output: Build Payload
- **Version-specific requirements:** Webhook node version 2.
- **Edge cases or potential failure types:**  
  - Payload missing required JSON fields
  - Incorrect content type or malformed JSON
  - Because response mode is `responseNode`, there is no explicit Response node in this workflow; depending on n8n version/runtime behavior, this may lead to response handling issues or hanging requests if not adjusted
- **Sub-workflow reference:** None.

#### Build Payload note
- **Type and technical role:** Sticky Note; local block documentation.
- **Configuration choices:** Explains that required fields are validated and a structured payload is built.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Build Payload
- **Type and technical role:** Code node; validation and transformation logic.
- **Configuration choices:**  
  - Reads from `$json.body` if present, otherwise `$json`
  - Verifies six required fields
  - Splits `employee_name` into first and last name
  - Normalizes emails to lowercase
  - Trims whitespace
  - Builds:
    - `temp_password` as `Onboard@<employee_id>!`
    - `org_unit` as `/Departments/<department>`
- **Key expressions or variables used:**  
  - `const body = $json.body || $json;`
  - Required fields array:
    `['employee_name','employee_email','manager_email','department','start_date','employee_id']`
- **Input and output connections:**  
  - Input: New Employee Webhook  
  - Output: Provision Google Account
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**  
  - Throws an error if any required field is missing
  - Single-word names produce same value for first and last name
  - Department names with special characters may create invalid Google org unit paths
  - Temporary password may fail policy checks if Google password policy differs
- **Sub-workflow reference:** None.

---

## Block 3 — Initial Provisioning and Communication Chain

**Overview:**  
This block performs the Day 0 actions in sequence: create the employee account, announce the employee in Slack, create a Notion onboarding page, and send a welcome email. The workflow is linear, and each operational node is configured to continue even if it errors.

**Nodes Involved:**  
- Provision Google note
- Provision Google Account
- Welcome Slack note
- Post Welcome Slack
- Create Notion note
- Create Notion Onboarding Page
- Send Welcome Email

### Node Details

#### Provision Google note
- **Type and technical role:** Sticky Note; local documentation.
- **Configuration choices:** Indicates the purpose is Google Workspace account creation via Admin Directory API.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Provision Google Account
- **Type and technical role:** HTTP Request; direct call to Google Admin SDK Directory API.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://admin.googleapis.com/admin/directory/v1/users`
  - Authentication: predefined credential type `googleOAuth2Api`
  - Retries enabled: 3 attempts, 2-second wait
  - `onError`: continue to error output
  - Response configured with `neverError: true`
- **Key expressions or variables used:**  
  The node clearly intends to send a Google user creation payload, but the exported configuration shows `bodyParameters.parameters` containing an empty object only. In practice, this node is incomplete unless the request body is manually defined.
- **Input and output connections:**  
  - Input: Build Payload  
  - Output: Post Welcome Slack
- **Version-specific requirements:** HTTP Request node version 4.2.
- **Edge cases or potential failure types:**  
  - Most important: missing request body will prevent valid Google user creation
  - OAuth scope may be insufficient for user creation
  - Admin SDK may require domain-wide delegation or super admin authorization depending on setup
  - Duplicate user email causes API conflict
  - Invalid org unit path
  - Password policy rejection
  - Since `neverError` is enabled and the node continues on error, downstream nodes may execute even if account creation failed
- **Sub-workflow reference:** None.

#### Welcome Slack note
- **Type and technical role:** Sticky Note; local documentation.
- **Configuration choices:** Notes that a welcome message is posted to `#general`.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Post Welcome Slack
- **Type and technical role:** Slack node; sends a Slack message.
- **Configuration choices:**  
  - Message text uses Build Payload data:
    - employee name
    - department
    - start date
  - Markdown enabled via `mrkdwn: true`
  - Retry enabled: 3 attempts
  - `onError`: continue
- **Key expressions or variables used:**  
  - `$('Build Payload').item.json.employee_name`
  - `$('Build Payload').item.json.department`
  - `$('Build Payload').item.json.start_date`
- **Input and output connections:**  
  - Input: Provision Google Account  
  - Output: Create Notion Onboarding Page
- **Version-specific requirements:** Slack node version 2.3.
- **Edge cases or potential failure types:**  
  - Channel destination is not visible in the exported parameters; likely must be configured manually
  - Missing or invalid Slack credential
  - Bot lacks `chat:write` or channel access
  - If Google provisioning failed, the workflow still posts welcome text unless explicit branching is added
- **Sub-workflow reference:** None.

#### Create Notion note
- **Type and technical role:** Sticky Note; local documentation.
- **Configuration choices:** Documents creation of a Notion onboarding page with a 5-task checklist.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Create Notion Onboarding Page
- **Type and technical role:** Notion node; creates a page and checklist content.
- **Configuration choices:**  
  - Title: `Onboarding: <employee_name>`
  - `pageId` is configured in resource-link mode with an empty value
  - Adds five `to_do` blocks:
    1. Complete IT setup and laptop configuration
    2. Read and sign the employee handbook
    3. Complete mandatory compliance training
    4. Schedule 1:1 with manager
    5. Set up all required software accounts
  - Retry enabled: 3 attempts
  - `onError`: continue
- **Key expressions or variables used:**  
  - `$('Build Payload').item.json.employee_name`
- **Input and output connections:**  
  - Input: Post Welcome Slack  
  - Output: Send Welcome Email
- **Version-specific requirements:** Notion node version 2.2.
- **Edge cases or potential failure types:**  
  - Exported config appears incomplete: `pageId` is empty and no database target is visible
  - If the intent is to create a database entry, this must be explicitly configured in n8n
  - Notion integration may lack access to the target page/database
  - Notion API structure may not match later “Check Completed Tasks” logic
- **Sub-workflow reference:** None.

#### Send Welcome Email
- **Type and technical role:** Gmail node; sends the employee’s welcome email.
- **Configuration choices:**  
  - Recipient: employee email
  - Subject personalized with first name
  - HTML body includes:
    - greeting
    - department
    - login details
    - temporary password
    - 5-item checklist summary
    - note about Notion onboarding page
  - Retry enabled: 3 attempts
  - `onError`: continue
- **Key expressions or variables used:**  
  - `$('Build Payload').item.json.employee_email`
  - `$('Build Payload').item.json.first_name`
  - `$('Build Payload').item.json.department`
  - `$('Build Payload').item.json.temp_password`
- **Input and output connections:**  
  - Input: Create Notion Onboarding Page  
  - Output: Wait 7 Days
- **Version-specific requirements:** Gmail node version 2.1.
- **Edge cases or potential failure types:**  
  - Gmail OAuth credential may not authorize sending from desired mailbox
  - Sending plain temporary passwords by email may violate internal security policy
  - If prior steps fail, email still sends because upstream nodes continue on error
- **Sub-workflow reference:** None.

---

## Block 4 — Seven-Day Delay and Task Check

**Overview:**  
This block pauses execution for 7 days and then queries Notion to determine whether enough onboarding tasks have been completed. The logic assumes the Notion query returns one item per completed task or equivalent result items that can be counted.

**Nodes Involved:**  
- Wait 7 Days note
- Wait 7 Days
- Check Tasks note
- Check Completed Tasks
- 3+ Tasks Complete?

### Node Details

#### Wait 7 Days note
- **Type and technical role:** Sticky Note; local documentation.
- **Configuration choices:** States that execution pauses for exactly 7 days.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Wait 7 Days
- **Type and technical role:** Wait node; delayed resumption.
- **Configuration choices:**  
  - Unit: days
  - Amount: 7
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: Send Welcome Email  
  - Output: Check Completed Tasks
- **Version-specific requirements:** Wait node version 1.1.
- **Edge cases or potential failure types:**  
  - Requires n8n execution persistence to survive restarts
  - Long waits depend on proper queue/database configuration in production
- **Sub-workflow reference:** None.

#### Check Tasks note
- **Type and technical role:** Sticky Note; local documentation.
- **Configuration choices:** States that Notion is queried for tasks with `Status = Done` for the employee.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Check Completed Tasks
- **Type and technical role:** Notion node; retrieves entries from a Notion database.
- **Configuration choices:**  
  - Resource: `databasePage`
  - Operation: `getAll`
  - Database ID: `YOUR_ONBOARDING_DB_ID`
  - Limit: 10
  - Retry enabled: 3 attempts
  - `onError`: continue
- **Key expressions or variables used:**  
  None visible in the exported config for filtering by employee or status.
- **Input and output connections:**  
  - Input: Wait 7 Days  
  - Output: 3+ Tasks Complete?
- **Version-specific requirements:** Notion node version 2.2.
- **Edge cases or potential failure types:**  
  - Major design gap: no filter is configured, so this fetches up to 10 database pages rather than “completed tasks for this employee”
  - If the database contains multiple employees, the count will be inaccurate
  - If the onboarding page uses to-do blocks rather than database pages, this node does not actually query those checklist items
  - Invalid or unreplaced `YOUR_ONBOARDING_DB_ID` causes failure
- **Sub-workflow reference:** None.

#### 3+ Tasks Complete?
- **Type and technical role:** IF node; evaluates whether enough tasks are complete.
- **Configuration choices:**  
  - Uses loose type validation
  - Condition: `$items().length >= 3`
- **Key expressions or variables used:**  
  - `={{ $items().length }}`
- **Input and output connections:**  
  - Input: Check Completed Tasks  
  - True output: Wait Until Day 30  
  - False output: Alert Manager — Incomplete Tasks
- **Version-specific requirements:** IF node version 2.2.
- **Edge cases or potential failure types:**  
  - Accuracy fully depends on previous Notion node returning only relevant completed tasks
  - If Notion returns zero items because of auth or query failure and errors are continued, the false branch may trigger a misleading manager alert
- **Sub-workflow reference:** None.

---

## Block 5 — Completion Path and Escalation Path

**Overview:**  
This block handles the post-check branching. Employees with sufficient task completion continue to Day 30 and trigger a completion message; otherwise, a manager alert is sent immediately after the 7-day check.

**Nodes Involved:**  
- Wait Until Day 30
- Day 30 Completion Message
- Alert Manager — Incomplete Tasks

### Node Details

#### Wait Until Day 30
- **Type and technical role:** Wait node; delayed continuation to Day 30.
- **Configuration choices:**  
  - Unit: days
  - Amount: 23
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: 3+ Tasks Complete? (true branch)  
  - Output: Day 30 Completion Message
- **Version-specific requirements:** Wait node version 1.1.
- **Edge cases or potential failure types:**  
  - Same persistence considerations as other wait nodes
  - Assumes a fixed Day 7 + 23-day sequence equals Day 30, regardless of actual hire date/timezone nuances
- **Sub-workflow reference:** None.

#### Day 30 Completion Message
- **Type and technical role:** Slack node; posts a completion notification.
- **Configuration choices:**  
  - Text announces 30-day onboarding completion
  - Uses employee name and department
  - Markdown enabled
  - Retry enabled: 3 attempts
  - `onError`: continue
- **Key expressions or variables used:**  
  - `$('Build Payload').item.json.employee_name`
  - `$('Build Payload').item.json.department`
- **Input and output connections:**  
  - Input: Wait Until Day 30  
  - Output: none
- **Version-specific requirements:** Slack node version 2.3.
- **Edge cases or potential failure types:**  
  - Channel destination is not visible in export and likely requires manual configuration
  - If the task check logic is inaccurate, this message may be sent incorrectly
- **Sub-workflow reference:** None.

#### Alert Manager — Incomplete Tasks
- **Type and technical role:** Slack node; sends escalation message when task completion is insufficient.
- **Configuration choices:**  
  - Text includes:
    - employee name
    - employee email
    - department
    - start date
    - request for follow-up
  - Markdown enabled
  - Retry enabled: 3 attempts
  - `onError`: continue
- **Key expressions or variables used:**  
  - `$('Build Payload').item.json.employee_name`
  - `$('Build Payload').item.json.employee_email`
  - `$('Build Payload').item.json.department`
  - `$('Build Payload').item.json.start_date`
- **Input and output connections:**  
  - Input: 3+ Tasks Complete? (false branch)  
  - Output: none
- **Version-specific requirements:** Slack node version 2.3.
- **Edge cases or potential failure types:**  
  - Channel destination is not visible in export and likely requires manual configuration
  - This sends to Slack, not directly to `manager_email`; operationally this is a team alert, not an email to the manager
  - False alerts are possible if Notion query logic is incomplete
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documents workflow purpose and scope |  |  | ## Employee Onboarding Orchestrator<br>Version 1.0.0 — HR<br><br>Automates the complete Day 0-30 onboarding sequence. Provisions a Google Workspace account, sends a Slack welcome, creates a Notion onboarding page with a 5-task checklist, and sends a welcome email. Checks task completion at Day 7 — alerts the manager if tasks are incomplete — then sends a Day 30 completion message.<br><br>Only credentials need to be configured. All parameters, prompts, and logic are ready to use. |
| Prerequisites | Sticky Note | Documents required credentials and payload format |  |  | ## Prerequisites<br>- Google Workspace Admin SDK<br>  (Google OAuth2 with admin.directory.user.create scope)<br>- Slack Bot token with chat:write scope<br>- Notion integration token with database access<br>- Gmail OAuth2 credential<br><br>Webhook payload (JSON POST):<br>{<br>  "employee_name": "Jane Smith",<br>  "employee_email": "jane@company.com",<br>  "manager_email": "mgr@company.com",<br>  "department": "Engineering",<br>  "start_date": "2025-04-01",<br>  "employee_id": "EMP-0042"<br>} |
| Setup Required | Sticky Note | Documents mandatory credential and placeholder setup |  |  | ## Setup Required<br>1. Provision Google Account<br>   Set Google OAuth2 credential<br>2. Post Welcome Slack + Alert Manager<br>   Set Slack credential<br>   Replace C00GENERAL000 with real channel ID<br>   Replace C00MANAGERS000 with real channel ID<br>3. Create Notion Onboarding Page<br>   Set Notion credential<br>   Replace YOUR_ONBOARDING_DB_ID with real DB ID<br>4. Check Completed Tasks<br>   Same Notion credential and DB ID<br>5. Send Welcome Email<br>   Set Gmail credential |
| How It Works | Sticky Note | Summarizes execution logic |  |  | ## How It Works<br>1. HR system sends a POST to the webhook<br>   with the new employee details<br>2. Google Workspace account is provisioned<br>   with a temporary password<br>3. Welcome message posted to #general<br>4. Notion page created with 5-task checklist<br>5. Welcome email sent with login details<br>   and checklist summary<br>6. Execution pauses 7 days<br>7. Notion queried for tasks with Status = Done<br>8. Fewer than 3 done: manager alert posted<br>   3 or more done: execution pauses 23 more<br>   days then sends Day 30 completion message |
| New Employee Webhook | Webhook | Receives HR onboarding POST request |  | Build Payload |  |
| Build Payload note | Sticky Note | Documents payload validation and structuring |  |  | Validates required fields and builds a clean structured payload |
| Build Payload | Code | Validates input and generates normalized onboarding fields | New Employee Webhook | Provision Google Account | Validates required fields and builds a clean structured payload |
| Provision Google note | Sticky Note | Documents Google account creation step |  |  | Creates Google Workspace account via Admin Directory API |
| Provision Google Account | HTTP Request | Creates Google Workspace user through Admin SDK | Build Payload | Post Welcome Slack | Creates Google Workspace account via Admin Directory API |
| Welcome Slack note | Sticky Note | Documents welcome Slack post |  |  | Posts welcome message to #general channel |
| Post Welcome Slack | Slack | Posts onboarding welcome message | Provision Google Account | Create Notion Onboarding Page | Posts welcome message to #general channel |
| Create Notion note | Sticky Note | Documents Notion onboarding page creation |  |  | Creates onboarding page with 5-task checklist in Notion database |
| Create Notion Onboarding Page | Notion | Creates onboarding page/checklist in Notion | Post Welcome Slack | Send Welcome Email | Creates onboarding page with 5-task checklist in Notion database |
| Send Welcome Email | Gmail | Sends welcome email with login details and checklist summary | Create Notion Onboarding Page | Wait 7 Days |  |
| Wait 7 Days note | Sticky Note | Documents 7-day pause |  |  | Pauses workflow execution for exactly 7 days |
| Wait 7 Days | Wait | Delays execution for 7 days | Send Welcome Email | Check Completed Tasks | Pauses workflow execution for exactly 7 days |
| Check Tasks note | Sticky Note | Documents Notion follow-up query |  |  | Queries Notion for tasks with Status = Done for this employee |
| Check Completed Tasks | Notion | Retrieves Notion records intended to represent completed onboarding tasks | Wait 7 Days | 3+ Tasks Complete? | Queries Notion for tasks with Status = Done for this employee |
| 3+ Tasks Complete? | IF | Branches based on completed task count | Check Completed Tasks | Wait Until Day 30; Alert Manager — Incomplete Tasks |  |
| Wait Until Day 30 | Wait | Delays completion message until Day 30 | 3+ Tasks Complete? | Day 30 Completion Message |  |
| Day 30 Completion Message | Slack | Posts final completion notification | Wait Until Day 30 |  |  |
| Alert Manager — Incomplete Tasks | Slack | Posts escalation message for insufficient task completion | 3+ Tasks Complete? |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a complete rebuild sequence in n8n.

## 1. Create the workflow canvas documentation
1. Add four **Sticky Note** nodes:
   - **Overview**
   - **Prerequisites**
   - **Setup Required**
   - **How It Works**
2. Paste the corresponding operational notes into each one.
3. These notes are optional for execution, but recommended for maintainability.

## 2. Create the webhook entry point
4. Add a **Webhook** node named **New Employee Webhook**.
5. Configure:
   - **HTTP Method:** `POST`
   - **Path:** `onboarding`
   - **Response Mode:** preferably `On Received` or add an explicit response node if you want `responseNode`
6. Expected JSON payload:
   ```json
   {
     "employee_name": "Jane Smith",
     "employee_email": "jane@company.com",
     "manager_email": "mgr@company.com",
     "department": "Engineering",
     "start_date": "2025-04-01",
     "employee_id": "EMP-0042"
   }
   ```

## 3. Build the payload normalization step
7. Add a **Sticky Note** named **Build Payload note** with the comment about validation and clean payload construction.
8. Add a **Code** node named **Build Payload**.
9. Paste this logic into the JavaScript field:
   ```javascript
   const body = $json.body || $json;
   const required = ['employee_name','employee_email','manager_email','department','start_date','employee_id'];
   for (const f of required) {
     if (!body[f]) throw new Error(`Missing required field: ${f}`);
   }
   const parts = body.employee_name.trim().split(' ');
   return [{
     json: {
       employee_id: body.employee_id,
       employee_name: body.employee_name.trim(),
       first_name: parts[0],
       last_name: parts.slice(1).join(' ') || parts[0],
       employee_email: body.employee_email.toLowerCase().trim(),
       manager_email: body.manager_email.toLowerCase().trim(),
       department: body.department.trim(),
       start_date: body.start_date,
       temp_password: `Onboard@${body.employee_id}!`,
       org_unit: `/Departments/${body.department.trim()}`
     }
   }];
   ```
10. Connect **New Employee Webhook → Build Payload**.

## 4. Create Google Workspace provisioning
11. Add a **Sticky Note** named **Provision Google note**.
12. Add an **HTTP Request** node named **Provision Google Account**.
13. Configure:
   - **Method:** `POST`
   - **URL:** `https://admin.googleapis.com/admin/directory/v1/users`
   - **Authentication:** `Predefined Credential Type`
   - **Credential Type:** `Google OAuth2 API`
   - **Retry on Fail:** enabled
   - **Max Tries:** `3`
   - **Wait Between Tries:** `2000 ms`
   - **On Error:** `Continue (using error output or continue workflow)`
   - **Response:** set to never fail on non-2xx if you want behavior matching the export
14. Create/select a **Google OAuth2** credential with Admin SDK scopes appropriate for user creation, including at minimum the equivalent of Admin Directory user creation access.
15. Define the JSON body manually, because the exported workflow does not contain a usable body. Use fields from **Build Payload**, for example:
   ```json
   {
     "primaryEmail": "={{ $json.employee_email }}",
     "name": {
       "givenName": "={{ $json.first_name }}",
       "familyName": "={{ $json.last_name }}"
     },
     "password": "={{ $json.temp_password }}",
     "orgUnitPath": "={{ $json.org_unit }}",
     "changePasswordAtNextLogin": true
   }
   ```
16. Connect **Build Payload → Provision Google Account**.

## 5. Add the Slack welcome step
17. Add a **Sticky Note** named **Welcome Slack note**.
18. Add a **Slack** node named **Post Welcome Slack**.
19. Configure:
   - **Credential:** Slack Bot token
   - **Operation:** send message
   - **Channel:** replace placeholder with your real general channel ID, e.g. `C00GENERAL000`
   - **Text:**
     ```text
     Welcome to the team, *{{ $('Build Payload').item.json.employee_name }}*! They are joining the *{{ $('Build Payload').item.json.department }}* team on {{ $('Build Payload').item.json.start_date }}.
     ```
   - **mrkdwn:** enabled
   - **Retry on Fail:** enabled, 3 tries, 2000 ms
   - **On Error:** continue
20. Ensure the Slack app has `chat:write` and access to the target channel.
21. Connect **Provision Google Account → Post Welcome Slack**.

## 6. Create the Notion onboarding page
22. Add a **Sticky Note** named **Create Notion note**.
23. Add a **Notion** node named **Create Notion Onboarding Page**.
24. Configure a **Notion credential** using your integration token.
25. Share the target parent page or database with the integration.
26. Because the export is incomplete, choose one of these implementation patterns:

### Preferred pattern: create a database page
27. Set the node to create a page in your onboarding database.
28. Use your real onboarding database ID instead of `YOUR_ONBOARDING_DB_ID`.
29. Set the page title to:
   ```text
   Onboarding: {{ $('Build Payload').item.json.employee_name }}
   ```
30. Add properties such as:
   - Employee Name
   - Employee Email
   - Manager Email
   - Department
   - Start Date
   - Employee ID
   - Completed Task Count or Status if needed later
31. Add five child blocks of type **to_do**:
   - Complete IT setup and laptop configuration
   - Read and sign the employee handbook
   - Complete mandatory compliance training
   - Schedule 1:1 with manager
   - Set up all required software accounts
32. Enable retry and continue-on-error as in the source workflow.
33. Connect **Post Welcome Slack → Create Notion Onboarding Page**.

## 7. Send the welcome email
34. Add a **Gmail** node named **Send Welcome Email**.
35. Configure a **Gmail OAuth2 credential** authorized to send email.
36. Configure:
   - **To:** `={{ $('Build Payload').item.json.employee_email }}`
   - **Subject:** `=Welcome to the team, {{ $('Build Payload').item.json.first_name }}!`
   - **Message:** HTML body containing:
     - personalized greeting
     - department name
     - email and temporary password
     - first-week checklist
     - note that the Notion page has been created
37. Use this message body:
   ```html
   <h2>Welcome, {{ $('Build Payload').item.json.first_name }}!</h2>
   <p>We are thrilled to have you join the <strong>{{ $('Build Payload').item.json.department }}</strong> team.</p>
   <h3>Your login details</h3>
   <p><strong>Email:</strong> {{ $('Build Payload').item.json.employee_email }}<br><strong>Temporary password:</strong> {{ $('Build Payload').item.json.temp_password }}<br>Please change your password on first login.</p>
   <h3>Your first-week checklist</h3>
   <ul>
     <li>Complete IT setup and laptop configuration</li>
     <li>Read and sign the employee handbook</li>
     <li>Complete mandatory compliance training</li>
     <li>Schedule a 1:1 with your manager</li>
     <li>Set up all required software accounts</li>
   </ul>
   <p>Your onboarding page in Notion has been created. Your manager will share access shortly.</p>
   ```
38. Enable retries and continue-on-error.
39. Connect **Create Notion Onboarding Page → Send Welcome Email**.

## 8. Add the 7-day delay
40. Add a **Sticky Note** named **Wait 7 Days note**.
41. Add a **Wait** node named **Wait 7 Days**.
42. Configure:
   - **Unit:** `Days`
   - **Amount:** `7`
43. Connect **Send Welcome Email → Wait 7 Days**.

## 9. Add the Notion progress check
44. Add a **Sticky Note** named **Check Tasks note**.
45. Add a **Notion** node named **Check Completed Tasks**.
46. Configure:
   - **Resource:** database page
   - **Operation:** get many / get all
   - **Database ID:** your real onboarding tracking database ID
   - **Limit:** `10`
   - Retry and continue-on-error enabled
47. Important: the exported workflow does not include the required filters. To make the logic actually work, add filters such as:
   - employee identifier equals current employee ID or email
   - task status equals `Done`
48. If your design stores checklist items as page blocks rather than database rows, replace this node with a Notion API strategy that truly counts completed to-do items for the specific employee. The exported node does not do this correctly by itself.
49. Connect **Wait 7 Days → Check Completed Tasks**.

## 10. Add the completion threshold check
50. Add an **IF** node named **3+ Tasks Complete?**.
51. Configure the condition:
   - Left value: `={{ $items().length }}`
   - Operator: `greater than or equal`
   - Right value: `3`
52. Leave type handling loose if desired to match the export.
53. Connect **Check Completed Tasks → 3+ Tasks Complete?**

## 11. Add the success branch to Day 30
54. Add a **Wait** node named **Wait Until Day 30**.
55. Configure:
   - **Unit:** `Days`
   - **Amount:** `23`
56. Connect the **true** output of **3+ Tasks Complete? → Wait Until Day 30**.

57. Add a **Slack** node named **Day 30 Completion Message**.
58. Configure:
   - Slack credential
   - target channel, typically your HR or general operations channel
   - text:
     ```text
     *{{ $('Build Payload').item.json.employee_name }}* has completed their 30-day onboarding in the *{{ $('Build Payload').item.json.department }}* department. All tasks marked complete.
     ```
   - mrkdwn enabled
   - retries enabled
   - continue-on-error enabled
59. Connect **Wait Until Day 30 → Day 30 Completion Message**.

## 12. Add the incomplete-task alert branch
60. Add a **Slack** node named **Alert Manager — Incomplete Tasks**.
61. Configure:
   - Slack credential
   - manager or operations alert channel, replacing placeholder such as `C00MANAGERS000`
   - text:
     ```text
     *Action required:* {{ $('Build Payload').item.json.employee_name }} has fewer than 3 onboarding tasks completed after 7 days.

     Employee: {{ $('Build Payload').item.json.employee_email }}
     Department: {{ $('Build Payload').item.json.department }}
     Start date: {{ $('Build Payload').item.json.start_date }}

     Please follow up directly.
     ```
   - mrkdwn enabled
   - retries enabled
   - continue-on-error enabled
62. Connect the **false** output of **3+ Tasks Complete? → Alert Manager — Incomplete Tasks**.

## 13. Final credential checklist
63. Configure these credentials before activation:
   - **Google OAuth2 API** for Admin SDK user creation
   - **Slack Bot Token**
   - **Notion Integration Token**
   - **Gmail OAuth2**
64. Replace all placeholders:
   - Slack general channel ID
   - Slack manager/alert channel ID
   - Notion database ID
   - Any parent page IDs required by Notion
65. Test each node individually where possible.

## 14. Recommended corrections before production use
66. Add a proper **Respond to Webhook** node or change the webhook response mode to a simpler mode.
67. Make **Provision Google Account** send a complete JSON request body.
68. Rework the Notion model so the Day 7 check truly counts completed tasks for the current employee.
69. Consider explicit branching on provisioning success before sending credentials by email.
70. Consider replacing emailed temporary passwords with a secure invite or password reset flow.

**Sub-workflow setup:**  
This workflow does not call or expose any sub-workflow nodes. There are no child workflows to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is labeled “Employee Onboarding Orchestrator”, version 1.0.0, HR. | Internal workflow branding from the canvas notes |
| The workflow assumes only credentials need to be configured, but the exported operational configuration is incomplete in several places, especially Google request body, Slack channel targets, and Notion tracking logic. | Implementation caution |
| Webhook payload fields expected: employee_name, employee_email, manager_email, department, start_date, employee_id. | Runtime input contract |
| Required services: Google Workspace Admin SDK, Slack Bot, Notion integration, Gmail OAuth2. | Credential planning |
| Security caution: sending temporary passwords by email may not align with company security policies. | Operational/security note |
| Long waits of 7 and 23 days require persistent n8n execution storage and reliable worker infrastructure. | Deployment note |