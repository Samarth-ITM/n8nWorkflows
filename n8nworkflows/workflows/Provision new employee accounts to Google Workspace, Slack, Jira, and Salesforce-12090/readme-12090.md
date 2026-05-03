Provision new employee accounts to Google Workspace, Slack, Jira, and Salesforce

https://n8nworkflows.xyz/workflows/provision-new-employee-accounts-to-google-workspace--slack--jira--and-salesforce-12090


# Provision new employee accounts to Google Workspace, Slack, Jira, and Salesforce

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automate employee onboarding by provisioning accounts across **Google Workspace**, **Slack**, **Jira**, and **Salesforce**, then sending a **welcome email** once provisioning steps complete.

**Target use cases:**
- HR/IT onboarding automation triggered by a form submission
- Department-based account provisioning (e.g., Engineering → Jira, Sales → Salesforce)
- Parallelizing common tasks to reduce onboarding time

### 1.1 Input Reception (Webhook)
Receives new hire data via HTTP POST.

### 1.2 Global Configuration (Shared Variables)
Sets shared constants like Slack channel ID, default password, and email subject.

### 1.3 Parallel Common Provisioning (Google Workspace + Slack)
Creates a Google Workspace user and invites them to a Slack channel simultaneously.

### 1.4 Department-Based Provisioning (Routing)
Routes by department to create either a Jira user (Engineering) or a Salesforce user (Sales).

### 1.5 Synchronization & Notification
Waits for all provisioning branches to finish, then emails the new hire.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception (Webhook)

**Overview:** Accepts a new hire submission via webhook. This is the workflow entry point and supplies the payload used by all later nodes.

**Nodes involved:**
- **New Hire Form** (Webhook)

**Node details:**

#### New Hire Form
- **Type / role:** `Webhook` trigger (`n8n-nodes-base.webhook`) — entry point receiving POST data.
- **Configuration choices:**
  - **HTTP Method:** POST
  - **Path:** `new-hire` (endpoint becomes `/webhook/new-hire` or `/webhook-test/new-hire` depending on mode)
  - **Response Mode:** `lastNode` (HTTP response is whatever the last executed node returns)
- **Key variables / expected input fields:**
  - Expects at least: `name`, `email`, `department`, `start_date`
- **Connections:**
  - Output → **⚙️ CONFIGURATION**
- **Edge cases / failures:**
  - Missing fields (e.g., no `email`) will break downstream expressions (`split('@')`) or API calls.
  - `responseMode: lastNode` can cause long request times/timeouts if external APIs are slow; consider responding early if used in real-time form submission.
- **Version notes:** TypeVersion **2.1**.

---

### Block 1.2 — Global Configuration

**Overview:** Enriches incoming payload with global settings used by multiple downstream nodes (Slack channel, default password, welcome email subject).

**Nodes involved:**
- **⚙️ CONFIGURATION** (Set)

**Node details:**

#### ⚙️ CONFIGURATION
- **Type / role:** `Set` (`n8n-nodes-base.set`) — adds constants and keeps incoming fields.
- **Configuration choices:**
  - Adds fields:
    - `Slack_Channel_ID` = `C12345678`
    - `Default_Password` = `Welcome2025!`
    - `Welcome_Email_Subject` = `Welcome to the team!`
  - **Include other fields:** enabled (so original webhook payload remains available)
- **Connections:**
  - Input ← **New Hire Form**
  - Outputs (fan-out, parallel):
    - → **Create G-Suite Account**
    - → **Invite to General Channel**
    - → **Check Department**
- **Edge cases / failures:**
  - Password policy mismatch: Google Workspace may reject `Default_Password` if complexity requirements differ.
  - Hard-coded channel ID requires correct workspace/channel; inviting will fail if invalid.
- **Version notes:** TypeVersion **3.4**.

---

### Block 1.3 — Parallel Common Provisioning (Google Workspace + Slack)

**Overview:** Executes common onboarding steps in parallel: creates the Google Workspace account and invites the user to a Slack channel.

**Nodes involved:**
- **Create G-Suite Account** (Google Admin / GSuite Admin)
- **Invite to General Channel** (Slack)
- **Sync All Tasks** (Merge) — receives these results

**Node details:**

#### Create G-Suite Account
- **Type / role:** `gSuiteAdmin` (`n8n-nodes-base.gSuiteAdmin`) — creates a Google Workspace user.
- **Configuration choices (interpreted):**
  - `firstName` = first token of `name`: `{{ $json.name.split(' ')[0] }}`
  - `lastName` = second token of `name` or empty: `{{ $json.name.split(' ')[1] || '' }}`
  - `username` = local-part of email: `{{ $json.email.split('@')[0] }}`
  - `password` = `{{ $json.Default_Password }}`
  - No extra optional fields set
- **Connections:**
  - Input ← **⚙️ CONFIGURATION**
  - Output → **Sync All Tasks** (input index 0)
- **Edge cases / failures:**
  - Name parsing: multi-part surnames (e.g., “Van der Meer”) will truncate; `lastName` becomes only the second word.
  - If `email` is missing/invalid, `split('@')` fails or yields unexpected username.
  - Google Admin auth/scopes issues (service account domain-wide delegation typically required).
  - Username conflicts if the email local part already exists.
- **Version notes:** TypeVersion **1**.
- **Credentials:** Not shown in node; in practice requires Google Workspace Admin credentials/scopes.

#### Invite to General Channel
- **Type / role:** `Slack` (`n8n-nodes-base.slack`) — invites the user to a channel.
- **Configuration choices (interpreted):**
  - Resource: **channel**
  - Operation: **invite**
  - `channelId` from config: `{{ $json.Slack_Channel_ID }}`
  - `userIds`: set to `[ {{ $json.email }} ]`
- **Connections:**
  - Input ← **⚙️ CONFIGURATION**
  - Output → **Sync All Tasks** (input index 1)
- **Important integration note:**
  - Many Slack APIs require a **Slack User ID** (e.g., `U123...`) rather than an email. If this node expects user IDs, inviting by email will fail. You may need a “lookup user by email” step first (Slack: `users.lookupByEmail`) and pass the returned ID.
- **Edge cases / failures:**
  - User not yet exists in Slack → invite fails unless your process also creates Slack users (this workflow does not).
  - Missing scopes: `channels:read`, `channels:manage` / `conversations:write` (scope names vary) and user invitation permissions.
  - Private channel vs public channel permissions.
- **Version notes:** TypeVersion **2.4**.
- **Credentials:** Slack API credential set (`slackApi Credential`).

#### Sync All Tasks (partial role in this block)
- Receives outputs from Google and Slack branches and holds them for synchronization (details in Block 1.5).

---

### Block 1.4 — Department-Based Provisioning (Routing + Jira/Salesforce)

**Overview:** Routes new hire provisioning based on `department`. Engineering creates a Jira user; Sales creates a Salesforce user.

**Nodes involved:**
- **Check Department** (Switch)
- **Create Jira User** (Jira)
- **Create Salesforce User** (Salesforce)

**Node details:**

#### Check Department
- **Type / role:** `Switch` (`n8n-nodes-base.switch`) — conditional branching.
- **Configuration choices:**
  - Rule 1 output: **Engineering** if `{{ $json.department }}` equals `"Engineering"`
  - Rule 2 output: **Sales** if `{{ $json.department }}` equals `"Sales"`
  - Outputs are renamed to match output keys (for readability)
- **Connections:**
  - Input ← **⚙️ CONFIGURATION**
  - Output 0 (Engineering) → **Create Jira User**
  - Output 1 (Sales) → **Create Salesforce User**
- **Edge cases / failures:**
  - No default/fallback route: any other department (or casing differences like “engineering”) results in **no branch executed**, leaving Merge waiting for an input that never arrives (see Merge risks).
- **Version notes:** TypeVersion **3.4**.

#### Create Jira User
- **Type / role:** `Jira` (`n8n-nodes-base.jira`) — creates a Jira user.
- **Configuration choices:**
  - Resource: `user`
  - Sets:
    - `displayName` = `{{ $json.name }}`
    - `emailAddress` = `{{ $json.email }}`
    - `username` is literally set to `"<UNKNOWN>"` (as configured)
- **Connections:**
  - Input ← **Check Department** (Engineering output)
  - Output → **Sync All Tasks** (input index 2)
- **Edge cases / failures:**
  - Jira Cloud user creation commonly requires `accountId`-based handling; “username” may be deprecated depending on Jira product/version.
  - Using `"<UNKNOWN>"` is likely invalid and may cause API failure; should be derived (e.g., from email).
  - Org policies may prevent user creation via API or require invitation flows.
  - Auth/scopes and admin permissions.
- **Version notes:** TypeVersion **1**.
- **Credentials:** Not shown in JSON, but required in n8n for Jira node.

#### Create Salesforce User
- **Type / role:** `Salesforce` (`n8n-nodes-base.salesforce`) — creates/handles Salesforce user resource.
- **Configuration choices:**
  - Resource: `user`
  - `userId` set to `{{ $json.email }}` (note: Salesforce “User” creation typically requires many fields: Username, Email, Alias, ProfileId, TimeZoneSidKey, LocaleSidKey, EmailEncodingKey, LanguageLocaleKey, etc.)
- **Connections:**
  - Input ← **Check Department** (Sales output)
  - Output → **Sync All Tasks** (input index 3)
- **Edge cases / failures:**
  - This configuration appears insufficient for creating a Salesforce User in most orgs; may error due to missing required fields.
  - `userId` parameter meaning depends on operation (get/update/delete vs create). If operation is “create”, you usually pass a body, not an ID.
  - Duplicate usernames/emails; Salesforce usernames must be globally unique.
- **Version notes:** TypeVersion **1**.
- **Credentials:** Not shown in JSON, but required in n8n for Salesforce node.

---

### Block 1.5 — Synchronization & Notification

**Overview:** Waits for all parallel tasks to complete, then sends a welcome email to the new hire through Gmail.

**Nodes involved:**
- **Sync All Tasks** (Merge)
- **Send Welcome Email** (Gmail)

**Node details:**

#### Sync All Tasks
- **Type / role:** `Merge` (`n8n-nodes-base.merge`) — synchronization barrier across branches.
- **Configuration choices:**
  - Mode is “multiple inputs” behavior with **numberInputs = 4**
  - Inputs expected (by wiring):
    1. From **Create G-Suite Account**
    2. From **Invite to General Channel**
    3. From **Create Jira User**
    4. From **Create Salesforce User**
- **Connections:**
  - Inputs ← the four branches above
  - Output → **Send Welcome Email**
- **Critical edge case:**
  - Because **only one** of Jira *or* Salesforce runs (depending on department), the other branch will never emit an item. With `numberInputs = 4`, this can cause the merge to **wait indefinitely** (or produce no output), preventing the welcome email from sending.
  - To fix, typical approaches include:
    - Add a default branch that emits a “no-op” item into the missing path
    - Use separate merges per department
    - Use Merge modes that don’t require all inputs, or restructure to combine after the switch
- **Version notes:** TypeVersion **3.2**.

#### Send Welcome Email
- **Type / role:** `Gmail` (`n8n-nodes-base.gmail`) — sends onboarding email.
- **Configuration choices:**
  - Authentication: **serviceAccount**
  - To: `{{ $json.email }}`
  - Subject: `{{ $json.Welcome_Email_Subject }}`
  - Message body uses templated fields:
    - `{{ $json.name }}`
    - `{{ $json.start_date }}`
    - `{{ $json.department }}`
- **Connections:**
  - Input ← **Sync All Tasks**
  - As last node, its output becomes the webhook response (because webhook is set to `lastNode`)
- **Edge cases / failures:**
  - Service account must be authorized to send as/from the configured mailbox (domain delegation or configured sender).
  - If merge never emits, this node never runs.
  - Missing `start_date` or other fields results in blank content but not necessarily failure.
- **Version notes:** TypeVersion **2.2**.
- **Credentials:** Google API credential (`googleApi Credential`).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Hire Form | Webhook | Entry point receiving new-hire payload | — | ⚙️ CONFIGURATION | ## 🚀 Orchestrate Employee Onboarding; Streamline HR & IT operations by automatically provisioning accounts for new employees based on their department.; ## ⚙️ How it works; 1. **Receives Data**: Triggers when a new hire form is submitted via Webhook.; 2. **Configuration**: Sets global variables (default passwords, channel IDs).; 3. **Parallel Provisioning**:; - Creates **Google Workspace** account.; - Invites user to a general **Slack** channel.; 4. **Conditional Logic**: Checks the department.; - **Engineering** → Creates **Jira** user.; - **Sales** → Creates **Salesforce** user.; 5. **Sync & Notify**: Waits for all tasks to complete, then sends a welcome email via Gmail.; ## 🛠️ How to set up; 1. **Credentials**: Configure credentials for Google Workspace (Admin), Slack, Jira, Salesforce, and Gmail.; 2. **Config Node**: Double-click the `⚙️ CONFIGURATION` node to set your specific `Slack_Channel_ID` and default password policy.; 3. **Webhook**: Use the Production URL for live forms, or Test URL for development.; ## 📦 Requirements; - Admin access to G-Suite, Slack, Jira, and Salesforce. |
| ⚙️ CONFIGURATION | Set | Defines shared constants and keeps payload fields | New Hire Form | Create G-Suite Account; Invite to General Channel; Check Department | (same note as above) |
| Create G-Suite Account | gSuiteAdmin | Creates Google Workspace user | ⚙️ CONFIGURATION | Sync All Tasks | ###  Parallel Provisioning; Executes common tasks (G-Suite & Slack) simultaneously for efficiency. |
| Invite to General Channel | Slack | Invites new hire to a Slack channel | ⚙️ CONFIGURATION | Sync All Tasks | ###  Parallel Provisioning; Executes common tasks (G-Suite & Slack) simultaneously for efficiency. |
| Check Department | Switch | Routes provisioning based on department | ⚙️ CONFIGURATION | Create Jira User; Create Salesforce User | (same note as above) |
| Create Jira User | Jira | Creates Jira account for Engineering | Check Department | Sync All Tasks | (same note as above) |
| Create Salesforce User | Salesforce | Creates Salesforce user for Sales | Check Department | Sync All Tasks | (same note as above) |
| Sync All Tasks | Merge | Synchronizes parallel branches before notification | Create G-Suite Account; Invite to General Channel; Create Jira User; Create Salesforce User | Send Welcome Email | (same note as above) |
| Send Welcome Email | Gmail | Sends welcome email after provisioning completes | Sync All Tasks | — | (same note as above) |
| Main Description | Sticky Note | Documentation/annotation node | — | — | ## 🚀 Orchestrate Employee Onboarding; Streamline HR & IT operations by automatically provisioning accounts for new employees based on their department.; ## ⚙️ How it works; 1. **Receives Data**: Triggers when a new hire form is submitted via Webhook.; 2. **Configuration**: Sets global variables (default passwords, channel IDs).; 3. **Parallel Provisioning**:; - Creates **Google Workspace** account.; - Invites user to a general **Slack** channel.; 4. **Conditional Logic**: Checks the department.; - **Engineering** → Creates **Jira** user.; - **Sales** → Creates **Salesforce** user.; 5. **Sync & Notify**: Waits for all tasks to complete, then sends a welcome email via Gmail.; ## 🛠️ How to set up; 1. **Credentials**: Configure credentials for Google Workspace (Admin), Slack, Jira, Salesforce, and Gmail.; 2. **Config Node**: Double-click the `⚙️ CONFIGURATION` node to set your specific `Slack_Channel_ID` and default password policy.; 3. **Webhook**: Use the Production URL for live forms, or Test URL for development.; ## 📦 Requirements; - Admin access to G-Suite, Slack, Jira, and Salesforce. |
| Step 2 Note | Sticky Note | Documentation/annotation node | — | — | ###  Parallel Provisioning; Executes common tasks (G-Suite & Slack) simultaneously for efficiency. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a Webhook node**
   - Name: **New Hire Form**
   - Method: **POST**
   - Path: `new-hire`
   - Response: **Last node**
   - Deploy and note the Test/Production URLs.

2) **Add a Set node**
   - Name: **⚙️ CONFIGURATION**
   - Enable **Include Other Fields**
   - Add fields:
     - `Slack_Channel_ID` (string) e.g. `C12345678`
     - `Default_Password` (string) e.g. `Welcome2025!`
     - `Welcome_Email_Subject` (string) e.g. `Welcome to the team!`
   - Connect: **New Hire Form → ⚙️ CONFIGURATION**

3) **Add Google Workspace Admin node (GSuite Admin)**
   - Name: **Create G-Suite Account**
   - Operation: create user (as supported by the node)
   - Map fields with expressions:
     - First name: `{{ $json.name.split(' ')[0] }}`
     - Last name: `{{ $json.name.split(' ')[1] || '' }}`
     - Username: `{{ $json.email.split('@')[0] }}`
     - Password: `{{ $json.Default_Password }}`
   - Configure Google Workspace Admin credentials (typically service account + domain-wide delegation; ensure proper scopes).
   - Connect: **⚙️ CONFIGURATION → Create G-Suite Account**

4) **Add Slack node**
   - Name: **Invite to General Channel**
   - Resource: **Channel**
   - Operation: **Invite**
   - Channel ID: `{{ $json.Slack_Channel_ID }}`
   - User IDs: set to `{{ $json.email }}` (note: if Slack requires user ID, insert an additional Slack “lookup by email” node before this and pass the returned user ID)
   - Configure Slack API credentials (OAuth token with required scopes).
   - Connect: **⚙️ CONFIGURATION → Invite to General Channel**

5) **Add a Switch node**
   - Name: **Check Department**
   - Add rules:
     - If `{{ $json.department }}` equals `Engineering` → output “Engineering”
     - If `{{ $json.department }}` equals `Sales` → output “Sales”
   - Connect: **⚙️ CONFIGURATION → Check Department**

6) **Add Jira node**
   - Name: **Create Jira User**
   - Resource: **User**
   - Set:
     - Display Name: `{{ $json.name }}`
     - Email Address: `{{ $json.email }}`
     - Username: replace `"<UNKNOWN>"` with a real value (commonly derived from email local-part) depending on your Jira deployment requirements
   - Configure Jira credentials (URL, email/token for Jira Cloud, or appropriate auth).
   - Connect: **Check Department (Engineering) → Create Jira User**

7) **Add Salesforce node**
   - Name: **Create Salesforce User**
   - Resource: **User**
   - Configure the correct operation (create vs update/get) and supply required Salesforce fields for user creation in your org (Profile, Alias, Username, Locale settings, etc.).
   - Configure Salesforce OAuth credentials.
   - Connect: **Check Department (Sales) → Create Salesforce User**

8) **Add a Merge node for synchronization**
   - Name: **Sync All Tasks**
   - Set **Number of Inputs = 4**
   - Connect inputs:
     - **Create G-Suite Account → Sync All Tasks** (input 0)
     - **Invite to General Channel → Sync All Tasks** (input 1)
     - **Create Jira User → Sync All Tasks** (input 2)
     - **Create Salesforce User → Sync All Tasks** (input 3)
   - Important: adjust design to avoid deadlock (because only one of Jira/Salesforce runs). For example, add a “no-op” item to the missing branch or restructure merges after the Switch.

9) **Add Gmail node**
   - Name: **Send Welcome Email**
   - Auth: **Service Account**
   - To: `{{ $json.email }}`
   - Subject: `{{ $json.Welcome_Email_Subject }}`
   - Body (example):
     - `Hi {{ $json.name }}, ... Start Date: {{ $json.start_date }} ... Department: {{ $json.department }}`
   - Configure Google API credential for Gmail sending (service account with delegation or authorized sender).
   - Connect: **Sync All Tasks → Send Welcome Email**

10) **Add Sticky Notes (optional but recommended)**
   - Add **Main Description** note containing the workflow explanation and setup requirements.
   - Add **Step 2 Note** near the parallel provisioning nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## 🚀 Orchestrate Employee Onboarding — Streamline HR & IT operations by automatically provisioning accounts for new employees based on their department. (Includes: how it works, setup steps, requirements.) | Sticky note: **Main Description** |
| ### Parallel Provisioning — Executes common tasks (G-Suite & Slack) simultaneously for efficiency. | Sticky note: **Step 2 Note** |