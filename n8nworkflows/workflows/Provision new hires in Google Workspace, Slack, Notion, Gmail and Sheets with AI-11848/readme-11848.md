Provision new hires in Google Workspace, Slack, Notion, Gmail and Sheets with AI

https://n8nworkflows.xyz/workflows/provision-new-hires-in-google-workspace--slack--notion--gmail-and-sheets-with-ai-11848


# Provision new hires in Google Workspace, Slack, Notion, Gmail and Sheets with AI

## 1. Workflow Overview

**Purpose:** Automate new employee onboarding by receiving new hire details via a webhook, provisioning accounts in **Google Workspace**, **Slack**, and **Notion** in parallel, generating an **AI-based welcome package**, then **emailing the employee**, **notifying HR**, and **logging** the onboarding in **Google Sheets**, finally returning a JSON response to the webhook caller.

**Primary use cases**
- HR/IT receives new hire form submissions and wants consistent, fast provisioning across tools.
- Generate standardized but personalized onboarding communications at scale.
- Maintain an onboarding audit trail in a spreadsheet and notify internal channels.

### 1.1 Input Reception & Normalization
Receives employee data (POST) and normalizes/derives fields like `workEmail` and `onboardingId`.

### 1.2 Parallel Account Provisioning (Google, Slack, Notion)
Creates/invites user accounts across systems in parallel, then merges results.

### 1.3 Provisioning Result Compilation
Consolidates provisioning outcomes and employee details into a single structured object.

### 1.4 AI Welcome Package Generation
Calls an OpenAI chat model through an n8n LangChain Agent and parses its JSON-like output into email-ready content.

### 1.5 Notifications & Logging + Webhook Response
Conditionally emails the employee (if a personal email exists), always notifies HR Slack channel and appends a row to Google Sheets, merges branches, and responds to the webhook.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization
**Overview:** Accepts a POST webhook payload, then creates a clean internal schema with defaults and computed identifiers used downstream.  
**Nodes involved:** `New Employee Form`, `Prepare Employee Data`

#### Node: New Employee Form
- **Type / role:** Webhook Trigger (`n8n-nodes-base.webhook`) — entry point for new hire submissions.
- **Configuration choices:**
  - **Method:** POST
  - **Path:** `new-employee`
  - **Response mode:** “responseNode” (execution must reach **Respond to Webhook**)
- **Key variables/fields expected:** `body.firstName`, `body.lastName`, `body.personalEmail`, `body.department`, `body.jobTitle`, `body.startDate`, `body.managerId`
- **Connections:**
  - **Out:** to `Prepare Employee Data`
- **Edge cases / failure modes:**
  - Missing `body` keys (handled later via defaults except `personalEmail`)
  - If downstream fails and response node is not reached, webhook caller may hang/receive error depending on n8n settings/timeouts.
- **Version notes:** Node v2 supports response node mode; ensure n8n webhook configuration is consistent with your instance base URL/proxy.

#### Node: Prepare Employee Data
- **Type / role:** Set (`n8n-nodes-base.set`) — normalizes input, sets defaults, derives computed fields.
- **Configuration choices (interpreted):**
  - Produces canonical fields:
    - `firstName`: from `body.firstName`, default `"New"`
    - `lastName`: from `body.lastName`, default `"Employee"`
    - `personalEmail`: from `body.personalEmail` (no default)
    - `department`: default `"General"`
    - `jobTitle`: default `"Team Member"`
    - `startDate`: default to current date formatted `yyyy-MM-dd`
    - `managerId`: default `''`
    - `workEmail`: computed `firstname.lastname@company.com` lowercased
    - `onboardingId`: `ONB-YYYYMMDD-<random 6 chars>`
- **Key expressions / variables:**
  - Date: `$now.format('yyyy-MM-dd')`, `$now.format('yyyyMMdd')`
  - Random: `Math.random().toString(36).substring(2, 8).toUpperCase()`
  - Email: `($json.body.firstName || 'new').toLowerCase()`
- **Connections:**
  - **Out (fan-out parallel):** `Create Google Account`, `Invite to Slack`, `Create Notion Onboarding`
- **Edge cases / failure modes:**
  - `personalEmail` can be undefined; later handled by IF node.
  - `workEmail` uses `@company.com` hardcoded; must match your domain.
  - Names with spaces/accents may produce invalid emails (no sanitization beyond lowercase).
- **Version notes:** Set node v3.4 supports the “assignments” structure used here.

---

### Block 2 — Parallel Account Provisioning (Google, Slack, Notion)
**Overview:** Runs three provisioning actions concurrently, then merges their outputs into a single stream to continue processing.  
**Nodes involved:** `Create Google Account`, `Invite to Slack`, `Create Notion Onboarding`, `Merge Google & Slack`, `Merge All Accounts`

#### Node: Create Google Account
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Google Admin SDK Directory API to create a user.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://admin.googleapis.com/admin/directory/v1/users`
  - **Body:** JSON with `primaryEmail`, `name`, `password`, `changePasswordAtNextLogin`, `orgUnitPath`
  - **Auth:** predefined credential type `googleApi`
- **Key expressions:**
  - `primaryEmail`: `{{ $json.workEmail }}`
  - `orgUnitPath`: `/{{ $json.department }}`
  - Password fixed: `TempPass123!`
- **Connections:**
  - **In:** from `Prepare Employee Data`
  - **Out:** to `Merge Google & Slack` (input index 0)
- **Edge cases / failure modes:**
  - **Auth/permissions:** Google credential must be authorized for Admin SDK; service account + domain-wide delegation typically required.
  - **Org unit path:** `/DepartmentName` may not exist → API error.
  - **User exists:** conflict error if email already provisioned.
  - **Hardcoded temp password:** security risk; may violate org policy.
- **Version notes:** Node v4.2 supports credential-based auth with JSON body templating.

#### Node: Invite to Slack
- **Type / role:** Slack (`n8n-nodes-base.slack`) — invites user to workspace.
- **Configuration choices:**
  - Resource: `user`
  - Operation: `invite`
  - (Email is not explicitly mapped in shown parameters; typically Slack invite requires email—verify the node’s mapped fields in your n8n UI.)
- **Connections:**
  - **In:** from `Prepare Employee Data`
  - **Out:** to `Merge Google & Slack` (input index 1)
- **Edge cases / failure modes:**
  - Missing invite email mapping (could result in runtime parameter error).
  - Slack token missing scopes (admin/user management) → `not_allowed_token_type` / permission errors.
  - User already exists / already invited.
- **Version notes:** Slack node v2.2 behavior depends on Slack API changes; ensure your workspace supports the invite method used.

#### Node: Create Notion Onboarding
- **Type / role:** Notion (`n8n-nodes-base.notion`) — creates an onboarding page.
- **Configuration choices:**
  - Creates a page titled: `Onboarding: First Last`
  - `pageId` configured in “url” mode but **empty** in JSON → must be set to a parent page/database.
  - Adds a minimal block definition (heading_2 placeholder)
- **Connections:**
  - **In:** from `Prepare Employee Data`
  - **Out:** to `Merge All Accounts` (input index 1)
- **Edge cases / failure modes:**
  - Missing `pageId` (parent) will fail.
  - Notion integration lacks access to the parent page/database.
- **Version notes:** Notion node v2.2 requires an internal integration token; also Notion API imposes rate limits.

#### Node: Merge Google & Slack
- **Type / role:** Merge (`n8n-nodes-base.merge`) — combines Google + Slack results by item position.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `position`
- **Connections:**
  - **In 0:** `Create Google Account`
  - **In 1:** `Invite to Slack`
  - **Out:** `Merge All Accounts` (input index 0)
- **Edge cases / failure modes:**
  - If one branch errors or returns 0 items, merge may produce unexpected empty output.
  - If either branch returns multiple items, “by position” must align counts.

#### Node: Merge All Accounts
- **Type / role:** Merge — combines (Google+Slack merged) with Notion result.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `position`
- **Connections:**
  - **In 0:** `Merge Google & Slack`
  - **In 1:** `Create Notion Onboarding`
  - **Out:** `Compile Provisioning Results`
- **Edge cases / failure modes:** Same alignment concerns as above; partial failures can prevent downstream steps.

---

### Block 3 — Provisioning Result Compilation
**Overview:** Collects employee data and provisioning responses into a single structured JSON object used by AI generation and notifications.  
**Nodes involved:** `Compile Provisioning Results`

#### Node: Compile Provisioning Results
- **Type / role:** Code (`n8n-nodes-base.code`) — builds normalized summary object.
- **Configuration choices (interpreted):**
  - Pulls data using n8n’s node data accessors:
    - `$('Prepare Employee Data').first().json`
    - `$('Create Google Account').first().json`
    - `$('Invite to Slack').first().json`
    - `$('Create Notion Onboarding').first().json`
  - Produces:
    - `employee` object
    - `accounts.google/slack/notion` status fields derived from response shapes
    - timestamps: `provisionedAt`
- **Connections:**
  - **In:** from `Merge All Accounts`
  - **Out:** to `AI Welcome Generator`
- **Edge cases / failure modes:**
  - If any upstream node failed and produced no items, `.first()` may throw or produce undefined fields.
  - Assumes Google response has `id`, Slack response has `ok`, Notion response has `id`/`url`—may differ depending on node output structure or errors.
- **Version notes:** Code node v2 uses modern n8n runtime; `$()` node accessor works in current n8n versions.

---

### Block 4 — AI Welcome Package Generation
**Overview:** Uses an OpenAI chat model via LangChain Agent to generate onboarding text, then parses it into structured content and constructs email subject/body.  
**Nodes involved:** `OpenAI Chat Model`, `AI Welcome Generator`, `Prepare Welcome Package`

#### Node: OpenAI Chat Model
- **Type / role:** LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM to the agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.7`
- **Connections:**
  - **Out (ai_languageModel):** to `AI Welcome Generator` (as the agent’s language model)
- **Edge cases / failure modes:**
  - Missing OpenAI credentials / invalid key.
  - Rate limits or model availability.
- **Version notes:** Node v1.2; requires n8n’s LangChain nodes package support.

#### Node: AI Welcome Generator
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — prompts model to output a JSON welcome package.
- **Configuration choices:**
  - Prompt includes employee name, department, title, start date.
  - Requests JSON keys: `welcomeMessage`, `firstWeekSchedule` (array), `successTips` (array)
  - System message: friendly HR assistant tone
- **Connections:**
  - **In:** from `Compile Provisioning Results`
  - **Model input:** from `OpenAI Chat Model`
  - **Out:** to `Prepare Welcome Package`
- **Edge cases / failure modes:**
  - Model may return non-JSON or extra text; downstream parsing attempts to recover.
  - Hallucinated formatting may break JSON parsing.

#### Node: Prepare Welcome Package
- **Type / role:** Code — parses agent output, sets fallbacks, and composes final email content.
- **Configuration choices (interpreted):**
  - Reads provisioning data from `Compile Provisioning Results`
  - Reads AI output from `$input.first().json` and expects text at `aiResponse.output`
  - Extracts first `{ ... }` block via regex and `JSON.parse`
  - On parse failure, injects a default welcome package
  - Builds:
    - `emailSubject`: `Welcome to <department>! - <name>`
    - `emailBody`: plain text with schedule and tips formatted
- **Connections:**
  - **Out:** to `Can Send Welcome Email?`
- **Edge cases / failure modes:**
  - If the agent output is already structured (not in `.output`) the parser may miss it.
  - Regex `{[\s\S]*}` can capture too much if multiple JSON objects appear.
  - Missing arrays handled with `(aiContent.firstWeekSchedule || [])`.
- **Version notes:** Code node v2.

---

### Block 5 — Notifications, Logging & Webhook Response
**Overview:** If personal email exists, send the welcome email; regardless, notify HR in Slack and append to Google Sheets; then respond to the original webhook.  
**Nodes involved:** `Can Send Welcome Email?`, `Send Welcome Email`, `Skip Email`, `Notify HR Channel`, `Log to Onboarding Sheet`, `Merge Notification Results`, `Respond to Webhook`

#### Node: Can Send Welcome Email?
- **Type / role:** IF (`n8n-nodes-base.if`) — checks if `employee.personalEmail` exists.
- **Configuration choices:**
  - Condition: string “exists” on `{{ $json.employee.personalEmail }}`
- **Connections:**
  - **True branch:** `Send Welcome Email`, `Notify HR Channel`, `Log to Onboarding Sheet`
  - **False branch:** `Skip Email`, `Notify HR Channel`, `Log to Onboarding Sheet`
- **Edge cases / failure modes:**
  - “exists” may pass on whitespace; consider trimming validation if needed.
  - If Gmail sending should use `workEmail` instead, logic must change.

#### Node: Send Welcome Email
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — sends email to employee’s personal email.
- **Configuration choices:**
  - To: `{{ $json.employee.personalEmail }}`
  - Subject: `{{ $json.emailSubject }}`
  - Message: `{{ $json.emailBody }}`
- **Connections:**
  - **Out:** to `Merge Notification Results` (branch index 0)
- **Edge cases / failure modes:**
  - Gmail OAuth not configured or lacks send scope.
  - Sending limits, blocked content, invalid recipient.
- **Version notes:** Gmail node v2.1.

#### Node: Skip Email
- **Type / role:** NoOp (`n8n-nodes-base.noOp`) — placeholder for the “no email” branch.
- **Connections:**
  - **Out:** to `Merge Notification Results` (branch index 1)

#### Node: Notify HR Channel
- **Type / role:** Slack — posts onboarding notification to an HR channel.
- **Configuration choices:**
  - Channel: `#hr-notifications` (selected by name)
  - Message includes employee details and onboarding ID
- **Connections:**
  - **Out:** to `Merge Notification Results` (branch index 0)
- **Edge cases / failure modes:**
  - Channel name mismatch, bot not in channel, token scope issues.

#### Node: Log to Onboarding Sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — appends a row to a tracking sheet.
- **Configuration choices:**
  - Operation: append
  - `documentId` and `sheetName` are present but **empty** in JSON → must be configured.
- **Connections:** (no outgoing connection shown in the workflow)
- **Edge cases / failure modes:**
  - Missing spreadsheet configuration will fail.
  - Headers/columns mismatch can misplace data unless field mapping is configured in UI.
- **Version notes:** Google Sheets node v4.5.

#### Node: Merge Notification Results
- **Type / role:** Merge — used to continue after either email-sent or email-skipped path.
- **Configuration choices:**
  - Mode: `chooseBranch` (passes through whichever branch produced data)
- **Connections:**
  - **In 0:** from `Send Welcome Email` and `Notify HR Channel`
  - **In 1:** from `Skip Email`
  - **Out:** to `Respond to Webhook`
- **Edge cases / failure modes:**
  - Because multiple nodes feed input 0 (email + Slack), which one arrives/continues can be non-obvious; “chooseBranch” may not synchronize multiple notifications.
  - If you require “wait for Slack + Sheets + (optional Gmail)” you’d typically use additional merges or “Merge by waiting” patterns.

#### Node: Respond to Webhook
- **Type / role:** Respond to Webhook (`n8n-nodes-base.respondToWebhook`) — returns JSON to the original webhook request.
- **Configuration choices:**
  - Respond with JSON:
    - `success: true`
    - `onboardingId` and `employee` pulled from `Prepare Welcome Package`
    - static message: “Employee onboarding initiated successfully”
- **Key expressions:**
  - `{{ $('Prepare Welcome Package').first().json.onboardingId }}`
  - `{{ $('Prepare Welcome Package').first().json.employee.name }}`
- **Connections:** terminal
- **Edge cases / failure modes:**
  - If execution doesn’t reach here (error upstream), webhook will not get the intended response.
  - If `Prepare Welcome Package` produced no items, expressions may fail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Visual documentation | — | — | ## Employee Onboarding Automation; Automatically provisions new employees across multiple systems: Google Workspace account creation; Slack workspace invitation; Notion workspace setup; AI-generated welcome materials; **Trigger**: Form submission or manual webhook; **Output**: Complete onboarding with notifications |
| Sticky Note1 | Sticky Note | Visual documentation | — | — | ### Step 1: Receive Employee Data; Form trigger receives new hire information including name, email, department, and start date. |
| Sticky Note2 | Sticky Note | Visual documentation | — | — | ### Step 2: Parallel Account Provisioning; Simultaneously creates accounts in Google Workspace, Slack, and Notion for faster onboarding. |
| Sticky Note3 | Sticky Note | Visual documentation | — | — | ### Step 3: AI Welcome Package; Generates personalized welcome message, training schedule, and team introduction using AI. |
| Sticky Note4 | Sticky Note | Visual documentation | — | — | ### Step 4: Notification & Logging; Sends welcome email to employee, notifies HR and manager, logs to tracking sheet. |
| New Employee Form | Webhook | Receives new hire payload | — | Prepare Employee Data | ### Step 1: Receive Employee Data; Form trigger receives new hire information including name, email, department, and start date. |
| Prepare Employee Data | Set | Normalize fields; compute email/id | New Employee Form | Create Google Account; Invite to Slack; Create Notion Onboarding | ### Step 1: Receive Employee Data; Form trigger receives new hire information including name, email, department, and start date. |
| Create Google Account | HTTP Request | Create Google Workspace user via Admin SDK | Prepare Employee Data | Merge Google & Slack | ### Step 2: Parallel Account Provisioning; Simultaneously creates accounts in Google Workspace, Slack, and Notion for faster onboarding. |
| Invite to Slack | Slack | Invite user to Slack workspace | Prepare Employee Data | Merge Google & Slack | ### Step 2: Parallel Account Provisioning; Simultaneously creates accounts in Google Workspace, Slack, and Notion for faster onboarding. |
| Create Notion Onboarding | Notion | Create onboarding page in Notion | Prepare Employee Data | Merge All Accounts | ### Step 2: Parallel Account Provisioning; Simultaneously creates accounts in Google Workspace, Slack, and Notion for faster onboarding. |
| Merge Google & Slack | Merge | Combine Google+Slack responses | Create Google Account; Invite to Slack | Merge All Accounts | ### Step 2: Parallel Account Provisioning; Simultaneously creates accounts in Google Workspace, Slack, and Notion for faster onboarding. |
| Merge All Accounts | Merge | Combine (Google+Slack) with Notion response | Merge Google & Slack; Create Notion Onboarding | Compile Provisioning Results | ### Step 2: Parallel Account Provisioning; Simultaneously creates accounts in Google Workspace, Slack, and Notion for faster onboarding. |
| Compile Provisioning Results | Code | Build consolidated provisioning summary | Merge All Accounts | AI Welcome Generator | ### Step 2: Parallel Account Provisioning; Simultaneously creates accounts in Google Workspace, Slack, and Notion for faster onboarding. |
| AI Welcome Generator | LangChain Agent | Generate welcome package with AI | Compile Provisioning Results | Prepare Welcome Package | ### Step 3: AI Welcome Package; Generates personalized welcome message, training schedule, and team introduction using AI. |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for agent | — | AI Welcome Generator | ### Step 3: AI Welcome Package; Generates personalized welcome message, training schedule, and team introduction using AI. |
| Prepare Welcome Package | Code | Parse AI output; build email subject/body | AI Welcome Generator | Can Send Welcome Email? | ### Step 3: AI Welcome Package; Generates personalized welcome message, training schedule, and team introduction using AI. |
| Can Send Welcome Email? | IF | Conditional based on personalEmail existence | Prepare Welcome Package | Send Welcome Email; Skip Email; Notify HR Channel; Log to Onboarding Sheet | ### Step 4: Notification & Logging; Sends welcome email to employee, notifies HR and manager, logs to tracking sheet. |
| Send Welcome Email | Gmail | Email welcome package | Can Send Welcome Email? (true) | Merge Notification Results | ### Step 4: Notification & Logging; Sends welcome email to employee, notifies HR and manager, logs to tracking sheet. |
| Skip Email | NoOp | Placeholder for no-email path | Can Send Welcome Email? (false) | Merge Notification Results | ### Step 4: Notification & Logging; Sends welcome email to employee, notifies HR and manager, logs to tracking sheet. |
| Notify HR Channel | Slack | Post HR notification | Can Send Welcome Email? (true/false) | Merge Notification Results | ### Step 4: Notification & Logging; Sends welcome email to employee, notifies HR and manager, logs to tracking sheet. |
| Log to Onboarding Sheet | Google Sheets | Append onboarding record | Can Send Welcome Email? (true/false) | — | ### Step 4: Notification & Logging; Sends welcome email to employee, notifies HR and manager, logs to tracking sheet. |
| Merge Notification Results | Merge | Continue after email or skip | Send Welcome Email; Skip Email; Notify HR Channel | Respond to Webhook | ### Step 4: Notification & Logging; Sends welcome email to employee, notifies HR and manager, logs to tracking sheet. |
| Respond to Webhook | Respond to Webhook | Return success payload to caller | Merge Notification Results | — | ### Step 4: Notification & Logging; Sends welcome email to employee, notifies HR and manager, logs to tracking sheet. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Employee Onboarding Automation with Multi-System Provisioning*.

2. **Add Webhook node** named **New Employee Form**  
   - Type: *Webhook*  
   - HTTP Method: **POST**  
   - Path: **new-employee**  
   - Response Mode: **Using “Respond to Webhook” node** (responseNode)

3. **Add Set node** named **Prepare Employee Data** and connect: `New Employee Form → Prepare Employee Data`  
   - Add fields (expressions):
     - `firstName`: `{{$json.body.firstName || 'New'}}`
     - `lastName`: `{{$json.body.lastName || 'Employee'}}`
     - `personalEmail`: `{{$json.body.personalEmail}}`
     - `department`: `{{$json.body.department || 'General'}}`
     - `jobTitle`: `{{$json.body.jobTitle || 'Team Member'}}`
     - `startDate`: `{{$json.body.startDate || $now.format('yyyy-MM-dd')}}`
     - `managerId`: `{{$json.body.managerId || ''}}`
     - `workEmail`: `{{($json.body.firstName || 'new').toLowerCase()}}.{{($json.body.lastName || 'employee').toLowerCase()}}@company.com`
     - `onboardingId`: `{{'ONB-' + $now.format('yyyyMMdd') + '-' + Math.random().toString(36).substring(2, 8).toUpperCase()}}`

4. **Add HTTP Request node** named **Create Google Account** and connect from `Prepare Employee Data`  
   - Method: **POST**  
   - URL: `https://admin.googleapis.com/admin/directory/v1/users`  
   - Authentication: **Google** (credential type `googleApi`)  
   - Body content type: **JSON**  
   - Body:
     - `primaryEmail`: `{{$json.workEmail}}`
     - `name.givenName`: `{{$json.firstName}}`
     - `name.familyName`: `{{$json.lastName}}`
     - `password`: `TempPass123!`
     - `changePasswordAtNextLogin`: `true`
     - `orgUnitPath`: `/{{$json.department}}`
   - **Credentials:** configure Google API credential with Admin SDK Directory API access (often service account + domain-wide delegation).

5. **Add Slack node** named **Invite to Slack** and connect from `Prepare Employee Data`  
   - Resource: **User**  
   - Operation: **Invite**  
   - Map the invite email to `{{$json.workEmail}}` or `{{$json.personalEmail}}` (depending on your policy; confirm required Slack field in UI).  
   - **Credentials:** Slack OAuth/token with permissions to invite users.

6. **Add Notion node** named **Create Notion Onboarding** and connect from `Prepare Employee Data`  
   - Operation: create a page (under a parent page or database)  
   - Title: `Onboarding: {{$json.firstName}} {{$json.lastName}}`  
   - **Set the Parent Page/Database** (pageId URL or database) — this is required.  
   - **Credentials:** Notion integration token with access to the chosen parent.

7. **Add Merge node** named **Merge Google & Slack**  
   - Mode: **Combine**  
   - Combine by: **Position**  
   - Connect:
     - `Create Google Account → Merge Google & Slack (Input 0)`
     - `Invite to Slack → Merge Google & Slack (Input 1)`

8. **Add Merge node** named **Merge All Accounts**  
   - Mode: **Combine**, by **Position**  
   - Connect:
     - `Merge Google & Slack → Merge All Accounts (Input 0)`
     - `Create Notion Onboarding → Merge All Accounts (Input 1)`

9. **Add Code node** named **Compile Provisioning Results** and connect: `Merge All Accounts → Compile Provisioning Results`  
   - Paste logic equivalent to:
     - Read `Prepare Employee Data`, Google/Slack/Notion outputs
     - Build `provisioningResults` object with `employee`, `accounts`, `provisionedAt`

10. **Add OpenAI Chat Model node** named **OpenAI Chat Model**  
   - Model: `gpt-4o-mini`  
   - Temperature: `0.7`  
   - **Credentials:** OpenAI API key configured in n8n.

11. **Add LangChain Agent node** named **AI Welcome Generator**  
   - Prompt type: “define” (custom prompt)  
   - Prompt asks for JSON with keys `welcomeMessage`, `firstWeekSchedule`, `successTips` using employee fields.  
   - System message: friendly HR assistant style.  
   - Connect:
     - `Compile Provisioning Results → AI Welcome Generator (main)`
     - `OpenAI Chat Model → AI Welcome Generator (ai_languageModel)`

12. **Add Code node** named **Prepare Welcome Package** and connect: `AI Welcome Generator → Prepare Welcome Package`  
   - Implement:
     - Parse AI output text into JSON (with fallback defaults)
     - Construct `emailSubject` and `emailBody`
     - Merge with provisioning summary

13. **Add IF node** named **Can Send Welcome Email?** and connect: `Prepare Welcome Package → Can Send Welcome Email?`  
   - Condition: “String exists” on `{{$json.employee.personalEmail}}`

14. **Add Gmail node** named **Send Welcome Email**  
   - To: `{{$json.employee.personalEmail}}`  
   - Subject: `{{$json.emailSubject}}`  
   - Message: `{{$json.emailBody}}`  
   - Connect from **true** output of IF.  
   - **Credentials:** Gmail OAuth2 with send permission.

15. **Add NoOp node** named **Skip Email**  
   - Connect from **false** output of IF.

16. **Add Slack node** named **Notify HR Channel**  
   - Post message to channel `#hr-notifications` (or your channel)  
   - Include employee details and onboardingId  
   - Connect from **both** IF outputs (true and false).

17. **Add Google Sheets node** named **Log to Onboarding Sheet**  
   - Operation: **Append**  
   - Select Spreadsheet (Document) and Sheet tab  
   - Map columns (at minimum: onboardingId, name, workEmail, department, jobTitle, startDate, provisionedAt, notion page URL if desired)  
   - Connect from **both** IF outputs (true and false).  
   - **Credentials:** Google Sheets OAuth/service account.

18. **Add Merge node** named **Merge Notification Results**  
   - Mode: **Choose Branch**  
   - Connect:
     - `Send Welcome Email → Merge Notification Results (Input 0)`
     - `Notify HR Channel → Merge Notification Results (Input 0)` (as in the provided workflow)
     - `Skip Email → Merge Notification Results (Input 1)`

19. **Add Respond to Webhook node** named **Respond to Webhook** and connect: `Merge Notification Results → Respond to Webhook`  
   - Respond with: JSON  
   - Response body includes:
     - `success: true`
     - `onboardingId` from `Prepare Welcome Package`
     - `employee` name from `Prepare Welcome Package`
     - message string

20. **Activate and test**
   - Send a POST request to `/webhook/new-employee` with JSON body containing at least `firstName`, `lastName`, and optionally `personalEmail`, etc.
   - Verify Google/Slack/Notion provisioning, email behavior, Slack post, and Sheets append.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer / compliance context |
| Hardcoded temporary password `TempPass123!` is present in the Google user creation request. | Security/compliance consideration; replace with secure generation + secret handling |
| Notion parent `pageId` and Google Sheets `documentId`/`sheetName` are empty in the workflow and must be configured before production use. | Required configuration to avoid runtime failures |
| Slack “invite user” typically requires an email field mapping; confirm and set it explicitly in the Slack node UI. | Prevent missing-parameter errors |

