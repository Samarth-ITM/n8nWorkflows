Re-engage HubSpot leads using Azure OpenAI GPT-4o, Gmail and Slack

https://n8nworkflows.xyz/workflows/re-engage-hubspot-leads-using-azure-openai-gpt-4o--gmail-and-slack-13518


# Re-engage HubSpot leads using Azure OpenAI GPT-4o, Gmail and Slack

## 1. Workflow Overview

**Purpose:** Re-engage HubSpot leads that were previously marked **BAD_TIMING** by generating a short AI-written follow-up email, creating a **Gmail draft** for human review, and notifying the sales rep in **Slack**. Runs automatically **every 24 hours**.

**Target use cases:**
- Reactivation of “cold” HubSpot leads without auto-sending emails
- SDR/AE assistance with consistent tone and reduced manual writing
- Operational safety via Slack alerts when AI generation fails

### 1.1 Daily scheduling & HubSpot lead retrieval
Fetches up to 20 contacts/leads from HubSpot, including activity/status properties.

### 1.2 Filtering to only “Bad Timing” leads
Ensures only leads with `hs_lead_status` containing `BAD_TIMING` are processed.

### 1.3 Controlled looping (batch processing)
Processes leads one-by-one to reduce rate-limit risk and isolate failures.

### 1.4 AI email generation (Azure OpenAI GPT-4o via LangChain Agent)
Uses lead fields to generate a concise follow-up body with specific tone rules.

### 1.5 Draft creation in Gmail + Slack notification
Creates a Gmail draft and notifies a Slack user (sales owner/rep) that a draft is ready.

### 1.6 Failure handling path (AI errors)
If AI fails, builds an error payload and alerts the rep on Slack, then continues the loop.

---

## 2. Block-by-Block Analysis

### Block 1 — Daily scheduling & HubSpot lead retrieval
**Overview:** Triggers once per day and retrieves recent leads/contacts from HubSpot with selected activity fields for downstream filtering and personalization.

**Nodes involved:**
- **Daily Trigger (Every 24h)**
- **HubSpot: Fetch Recent Leads + Activity Fields**

#### Node: Daily Trigger (Every 24h)
- **Type / Role:** `Schedule Trigger` — workflow entry point
- **Configuration (interpreted):** Runs every **24 hours**.
- **Inputs/Outputs:** No inputs; outputs to HubSpot fetch node.
- **Edge cases / failures:** None typical; only n8n scheduler availability/timezone considerations.

#### Node: HubSpot: Fetch Recent Leads + Activity Fields
- **Type / Role:** `HubSpot` — pulls records to process
- **Configuration (interpreted):**
  - Operation: **Get All**
  - Limit: **20**
  - Auth: **App Token**
  - Requests additional contact properties:
    - `notes_last_contacted`, `hs_last_sales_activity_timestamp`, `hs_notes_last_activity`,
      `notes_last_updated`, `createdate`, `first_deal_created_date`, `hs_lead_status`
- **Key variables/fields used later:**
  - `properties.hs_lead_status.value`
  - activity timestamps and created date
  - also relies later on identity/email fields (`identity-profiles`) even though the code/prompt uses both styles
- **Connections:**
  - In: Schedule Trigger
  - Out: IF filter node
- **Edge cases / failures:**
  - **Auth/token** invalid or missing scopes
  - HubSpot API **rate limiting**
  - Returned records may not contain expected nested fields (`identity-profiles`, `properties.email`, etc.), causing expression errors downstream

---

### Block 2 — Filtering to “Bad Timing” leads
**Overview:** Keeps only leads whose `hs_lead_status` contains `BAD_TIMING` to avoid contacting active or irrelevant leads.

**Nodes involved:**
- **Filter: Lead Status = BAD_TIMING**

#### Node: Filter: Lead Status = BAD_TIMING
- **Type / Role:** `IF` — conditional routing
- **Configuration (interpreted):**
  - Condition: string **contains** comparison
  - Left value: `{{ $json.properties.hs_lead_status?.value || '' }}`
  - Right value: `BAD_TIMING`
- **Connections:**
  - In: HubSpot fetch
  - True out: Split in Batches
  - False out: (not connected; leads are effectively dropped)
- **Edge cases / failures:**
  - `hs_lead_status` missing → safely handled by `?.value || ''`
  - If HubSpot uses different status naming/casing, filter may miss intended leads

---

### Block 3 — Controlled looping (batch processing)
**Overview:** Iterates through eligible leads sequentially, enabling stable processing and easy continuation after success/failure paths.

**Nodes involved:**
- **Batch Leads (Split In Batches)**

#### Node: Batch Leads (Split In Batches)
- **Type / Role:** `SplitInBatches` — loop controller
- **Configuration (interpreted):**
  - Default batch size (not explicitly set in parameters)
  - `reset: false` (keeps state within execution for iteration)
- **Connections:**
  - In: IF true path
  - Output 1 (done): not used
  - Output 2 (next item): to AI agent
  - Receives loop-backs from:
    - Slack notify success → back into this node to continue next lead
    - Slack failure alert → back into this node to continue next lead
- **Edge cases / failures:**
  - If no items, downstream never runs (expected)
  - If large lead volume, consider explicit batch size and HubSpot pagination strategy

---

### Block 4 — AI generation with Azure OpenAI (LangChain agent)
**Overview:** Generates a short re-engagement email body using GPT-4o with strict tone rules and personalization. If generation errors, the workflow continues with an alert path.

**Nodes involved:**
- **Execute Re-Engagement Follow-Up Email Body with Azure OpenAI**
- **AI: Generate Re-Engagement Follow-Up Email Body**
- **Parse AI JSON** (note: mismatched expectations; see below)

#### Node: Execute Re-Engagement Follow-Up Email Body with Azure OpenAI
- **Type / Role:** `Azure OpenAI Chat Model` — provides LLM to agent
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Used as the **language model** input to the agent node via `ai_languageModel` connection
- **Connections:**
  - Out (ai_languageModel): to AI agent node
- **Version-specific requirements:**
  - Requires the `@n8n/n8n-nodes-langchain` package support and correct Azure OpenAI resource/deployment mapping in credentials.
- **Edge cases / failures:**
  - Azure OpenAI credential misconfig (endpoint, API key, deployment/model mapping)
  - Content filtering/blocked responses (provider policy)
  - Timeouts/rate limits

#### Node: AI: Generate Re-Engagement Follow-Up Email Body
- **Type / Role:** `LangChain Agent` — prompt orchestration + output creation
- **Configuration (interpreted):**
  - **Prompt text** includes lead data:
    - Email: `{{ $json["identity-profiles"][0].identities[0].value }}`
    - Lead Status: `{{ $json.properties.hs_lead_status.value }}`
    - Timestamps: `notes_last_contacted`, `createdate`
  - **System message** sets SDR persona and rules:
    - concise (3–5 lines), polite, not salesy
    - do not mention “BAD_TIMING”
    - soft CTA
    - output only email body (no subject)
  - `onError: continueErrorOutput` enabled, so errors produce an item on the error output.
  - `hasOutputParser: true` (agent is expected to structure output, but actual downstream nodes assume JSON—see next node)
- **Connections:**
  - Main output → Parse AI JSON
  - Error output → Build AI Error Payload
  - Receives LLM model via `ai_languageModel` connection from the Azure OpenAI node
- **Edge cases / failures:**
  - **Data path mismatch:** `identity-profiles` may not exist on HubSpot output depending on endpoint/object; many HubSpot nodes return `properties.email` instead.
  - If any of these are missing, expressions may evaluate to errors or empty strings.
  - LLM may output **plain text**, but downstream “Parse AI JSON” attempts JSON parsing.
  - Token/length issues unlikely due to short prompt, but possible.

#### Node: Parse AI JSON
- **Type / Role:** `Code` — normalizes AI output into `{subject, body, toEmail, ...}`
- **Configuration (interpreted):**
  - Reads: `const rawOutput = $input.first().json.output || ''`
  - Defaults:
    - `subject = 'Following up'`
    - `body = 'Hi, just checking in...'`
  - Attempts: `JSON.parse(rawOutput)` and expects keys `subject` and `body`
  - Pulls lead data from: `$('Batch Leads (Split In Batches)').first().json`
  - Returns:
    - `subject`, `body`, `toEmail`, `firstName`, `lastName`, `ownerId`
- **Connections:**
  - In: AI agent main output
  - Out: Gmail draft node
- **Critical logic mismatch (important):**
  - The agent’s system rules say “Output only the email body text. No subject line. No explanations.”
  - This means the agent will likely output **plain text**, not JSON, so JSON.parse will fail and the node will use default subject/body.
  - Additionally, Gmail draft uses the agent output directly for the message (see Gmail node), not the `body` created here—so this node’s `body` may be unused.
- **Edge cases / failures:**
  - If `leadData.properties.email` / firstname / lastname don’t exist, output fields become `undefined`
  - Expression `$('Batch Leads...')` relies on node execution context; if the node name changes, it breaks

---

### Block 5 — Gmail draft creation + Slack notification (success path)
**Overview:** Creates a draft email in Gmail (human review) and notifies a Slack user that the lead needs follow-up, then continues to the next batch item.

**Nodes involved:**
- **Gmail: Create Follow-Up Draft Email**
- **Slack: Notify Owner to Re-Engage Lead**

#### Node: Gmail: Create Follow-Up Draft Email
- **Type / Role:** `Gmail` — draft creation
- **Configuration (interpreted):**
  - Creates a draft with:
    - Subject: `{{ $json.subject }}` (from Parse AI JSON)
    - Message/body: `{{ $('AI: Generate Re-Engagement Follow-Up Email Body').item.json.output }}`
  - **No “to” field is configured** in parameters shown (draft may be created without recipient unless Gmail node defaults or hidden fields exist).
- **Connections:**
  - In: Parse AI JSON
  - Out: Slack notify node
- **Edge cases / failures:**
  - OAuth2 token expiration or missing Gmail scopes (draft creation)
  - If AI output is empty, draft may be blank
  - If recipient is required by node operation and not provided, the node may error
  - Inconsistent usage: subject from Parse node, body from AI node directly (not from Parse node’s `body`)

#### Node: Slack: Notify Owner to Re-Engage Lead
- **Type / Role:** `Slack` — sends notification
- **Configuration (interpreted):**
  - Sends message to a **specific Slack user** (selected by ID: `U09HMPVD466`)
  - Message includes:
    - lead email via `identity-profiles` path
    - lead status
  - Text indicates lead was bad timing and suggests retry outreach
- **Connections:**
  - In: Gmail draft node
  - Out: loops back to SplitInBatches to process next lead
- **Edge cases / failures:**
  - Slack auth/token issues
  - If `identity-profiles` is missing, message text expression can fail (or show blank)
  - Hardcoded user ID means it does not actually notify the HubSpot owner dynamically

---

### Block 6 — Failure handling (AI errors)
**Overview:** If the AI agent errors, the workflow constructs a minimal context payload and alerts the sales rep in Slack, then continues the loop.

**Nodes involved:**
- **Build AI Error Payload**
- **Slack: Alert AI Draft Failure**

#### Node: Build AI Error Payload
- **Type / Role:** `Code` — transforms error into Slack-friendly fields
- **Configuration (interpreted):**
  - Pulls lead from batch node: `$('Batch Leads (Split In Batches)').first().json`
  - Produces:
    - `firstName`, `lastName`, `toEmail`
    - `errorMessage` from `$input.first().json.error?.message || 'AI generation failed'`
- **Connections:**
  - In: AI agent error output
  - Out: Slack failure alert
- **Edge cases / failures:**
  - If batch lead properties missing, fields become empty/Unknown (handled partially)
  - If error object shape differs, may not capture message

#### Node: Slack: Alert AI Draft Failure
- **Type / Role:** `Slack` — notifies about AI generation failure
- **Configuration (interpreted):**
  - Sends warning message to the same Slack user ID (`U09HMPVD466`)
  - Includes lead name/email and error message
- **Connections:**
  - In: Build AI Error Payload
  - Out: loops back to SplitInBatches to continue processing next lead
- **Edge cases / failures:**
  - Slack auth issues
  - If name/email missing, message becomes less actionable

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Trigger (Every 24h) | Schedule Trigger | Entry point; runs every 24 hours | — | HubSpot: Fetch Recent Leads + Activity Fields | ## ⏰ Daily Schedule & Lead Fetch (HubSpot)<br>Runs every 24 hours to fetch leads from HubSpot with activity timestamps.<br>• Scheduled trigger<br>• Pulls recent lead activity<br>• Feeds data into filtering logic |
| HubSpot: Fetch Recent Leads + Activity Fields | HubSpot | Fetch HubSpot leads/contacts with selected properties | Daily Trigger (Every 24h) | Filter: Lead Status = BAD_TIMING | ## ⏰ Daily Schedule & Lead Fetch (HubSpot)<br>Runs every 24 hours to fetch leads from HubSpot with activity timestamps.<br>• Scheduled trigger<br>• Pulls recent lead activity<br>• Feeds data into filtering logic |
| Filter: Lead Status = BAD_TIMING | IF | Keep only BAD_TIMING leads | HubSpot: Fetch Recent Leads + Activity Fields | Batch Leads (Split In Batches) | ## 🚦 Filter Leads Marked as “Bad Timing”<br>Only processes leads that were previously marked as bad timing in HubSpot.<br>• Prevents spamming active leads<br>• Targets cold or paused opportunities<br>• Keeps follow-ups intentional |
| Batch Leads (Split In Batches) | Split In Batches | Iterate through leads one-by-one | Filter: Lead Status = BAD_TIMING; Slack: Notify Owner to Re-Engage Lead; Slack: Alert AI Draft Failure | AI: Generate Re-Engagement Follow-Up Email Body | ## 🔄 Loop Over Eligible Leads<br>Processes leads one-by-one to avoid rate limits and ensure clean AI output.<br>• Handles each lead independently<br>• Supports retries on failure<br>• Keeps processing stable at scale |
| Execute Re-Engagement Follow-Up Email Body with Azure OpenAI | Azure OpenAI Chat Model (LangChain) | Provides GPT-4o model to agent | — | AI: Generate Re-Engagement Follow-Up Email Body (ai_languageModel) | ## ✍️ AI Follow-Up Email Generator<br>Generates short, polite re-engagement emails for cold leads.<br>• Personalized using lead data<br>• Non-pushy, friendly tone<br>• Soft CTA to re-open conversation |
| AI: Generate Re-Engagement Follow-Up Email Body | LangChain Agent | Generates follow-up email content (and emits errors) | Batch Leads (Split In Batches); Azure OpenAI model via ai_languageModel | Parse AI JSON (success); Build AI Error Payload (error) | ## ✍️ AI Follow-Up Email Generator<br>Generates short, polite re-engagement emails for cold leads.<br>• Personalized using lead data<br>• Non-pushy, friendly tone<br>• Soft CTA to re-open conversation |
| Parse AI JSON | Code | Attempts to parse AI output into subject/body + lead fields | AI: Generate Re-Engagement Follow-Up Email Body | Gmail: Create Follow-Up Draft Email | ## 📨 Gmail Draft Creation<br>Creates a draft follow-up email for sales to review and send.<br>• Human-in-the-loop safety<br>• No auto-send to customers<br>• Keeps outreach controlled |
| Gmail: Create Follow-Up Draft Email | Gmail | Creates Gmail draft | Parse AI JSON | Slack: Notify Owner to Re-Engage Lead | ## 📨 Gmail Draft Creation<br>Creates a draft follow-up email for sales to review and send.<br>• Human-in-the-loop safety<br>• No auto-send to customers<br>• Keeps outreach controlled |
| Slack: Notify Owner to Re-Engage Lead | Slack | Notifies rep that draft exists, then continue loop | Gmail: Create Follow-Up Draft Email | Batch Leads (Split In Batches) | ## 🔔 Sales Rep Notification (Slack)<br>Notifies the assigned owner that a follow-up draft was created.<br>• Includes lead email + status<br>• Prompts timely outreach<br>• Keeps sales informed |
| Build AI Error Payload | Code | Builds payload for Slack failure message | AI: Generate Re-Engagement Follow-Up Email Body (error output) | Slack: Alert AI Draft Failure | ## ⚠️ AI Failure Handling & Fallback<br>Handles cases where AI fails to generate a follow-up email.<br>• Notifies sales on Slack<br>• Prevents silent lead drops<br>• Enables manual follow-up |
| Slack: Alert AI Draft Failure | Slack | Alerts rep that AI failed, then continue loop | Build AI Error Payload | Batch Leads (Split In Batches) | ## ⚠️ AI Failure Handling & Fallback<br>Handles cases where AI fails to generate a follow-up email.<br>• Notifies sales on Slack<br>• Prevents silent lead drops<br>• Enables manual follow-up |
| Sticky Note | Sticky Note | Documentation only | — | — | ## 🔁 AI-Powered HubSpot Lead Re-Engagement Workflow<br>### What this workflow does<br>This workflow automatically re-engages HubSpot leads that were previously marked as “Bad Timing” and have gone cold. Every 24 hours, it fetches stale leads from HubSpot, filters only those with the BAD_TIMING status, and loops through them one by one.<br><br>For each eligible lead, an AI agent generates a short, polite follow-up email that re-opens the conversation without sounding pushy or salesy. The email draft is created in Gmail so the sales rep can review before sending. At the same time, the owner is notified on Slack so they are aware a follow-up was prepared.<br><br>If AI generation fails for any lead, the workflow safely falls back and alerts the sales rep to follow up manually. This ensures no leads are silently skipped and sales never miss reactivation opportunities.<br><br>### Setup checklist<br>• Connect HubSpot App Token<br>• Connect Azure OpenAI credentials<br>• Connect Gmail OAuth2<br>• Connect Slack API<br>• Review email tone inside AI prompt<br><br>### Customization ideas<br>• Change retry cadence (daily → weekly)<br>• Add CRM status updates after follow-up<br>• Push follow-ups into a CRM task queue |
| Sticky Note1 | Sticky Note | Documentation only | — | — | ## ⏰ Daily Schedule & Lead Fetch (HubSpot)<br>Runs every 24 hours to fetch leads from HubSpot with activity timestamps.<br><br>• Scheduled trigger<br>• Pulls recent lead activity<br>• Feeds data into filtering logic |
| Sticky Note2 | Sticky Note | Documentation only | — | — | ## 🚦 Filter Leads Marked as “Bad Timing”<br>Only processes leads that were previously marked as bad timing in HubSpot.<br><br>• Prevents spamming active leads<br>• Targets cold or paused opportunities<br>• Keeps follow-ups intentional |
| Sticky Note3 | Sticky Note | Documentation only | — | — | ## 🔄 Loop Over Eligible Leads<br>Processes leads one-by-one to avoid rate limits and ensure clean AI output.<br><br>• Handles each lead independently<br>• Supports retries on failure<br>• Keeps processing stable at scale |
| Sticky Note4 | Sticky Note | Documentation only | — | — | ## ✍️ AI Follow-Up Email Generator<br>Generates short, polite re-engagement emails for cold leads.<br><br>• Personalized using lead data<br>• Non-pushy, friendly tone<br>• Soft CTA to re-open conversation |
| Sticky Note5 | Sticky Note | Documentation only | — | — | ## 📨 Gmail Draft Creation<br>Creates a draft follow-up email for sales to review and send.<br><br>• Human-in-the-loop safety<br>• No auto-send to customers<br>• Keeps outreach controlled |
| Sticky Note6 | Sticky Note | Documentation only | — | — | ## 🔔 Sales Rep Notification (Slack)<br>Notifies the assigned owner that a follow-up draft was created.<br><br>• Includes lead email + status<br>• Prompts timely outreach<br>• Keeps sales informed |
| Sticky Note7 | Sticky Note | Documentation only | — | — | ## ⚠️ AI Failure Handling & Fallback<br>Handles cases where AI fails to generate a follow-up email.<br><br>• Notifies sales on Slack<br>• Prevents silent lead drops<br>• Enables manual follow-up |
| Sticky Note8 | Sticky Note | Documentation only | — | — | ## 🔐 Required Credentials & Access<br>• HubSpot App Token<br>• Azure OpenAI (LLM generation)<br>• Gmail OAuth2 (draft emails)<br>• Slack API (sales alerts)<br><br>Never auto-send emails without review.<br>Keep humans in the loop for outreach. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: **AI-Powered HubSpot Lead Re-Engagement Workflow**
- (Optional) Set execution order setting to **v1** if you want to match the original behavior.

2) **Add Schedule Trigger**
- Node: **Schedule Trigger**
- Set interval: **Every 24 hours**

3) **Add HubSpot node to fetch leads**
- Node: **HubSpot**
- Authentication: **App Token**
- Operation: **Get All**
- Limit: **20**
- Additional Fields → Properties:
  - `notes_last_contacted`
  - `hs_last_sales_activity_timestamp`
  - `hs_notes_last_activity`
  - `notes_last_updated`
  - `createdate`
  - `first_deal_created_date`
  - `hs_lead_status`
- **Credentials:** create/select a **HubSpot App Token** credential.
- Connect: **Schedule Trigger → HubSpot**

4) **Add IF filter for BAD_TIMING**
- Node: **IF**
- Condition (String → contains):
  - Left: `{{ $json.properties.hs_lead_status?.value || '' }}`
  - Right: `BAD_TIMING`
- Connect: **HubSpot → IF**
- Use the **true** output for processing.

5) **Add Split In Batches**
- Node: **Split In Batches**
- Options:
  - Reset: **false**
- Connect: **IF (true) → Split In Batches**
- Use the “next batch/item” output (in this workflow it is the second output) to continue.

6) **Add Azure OpenAI Chat Model**
- Node: **Azure OpenAI Chat Model** (LangChain) / `lmChatAzureOpenAi`
- Model: `gpt-4o`
- **Credentials:** configure **Azure OpenAI** credential (endpoint/resource, API key, and deployment/model mapping as required by your n8n version).
- This node does not connect via “main”; it connects via the agent’s **ai_languageModel** input.

7) **Add LangChain Agent for email generation**
- Node: **AI Agent** / `@n8n/n8n-nodes-langchain.agent`
- Set **On Error**: `continueErrorOutput`
- Prompt (Text) (use the same structure):
  - Include lead email, lead status, timestamps, and the “bad timing” context
- System message:
  - SDR assistant persona
  - 3–5 lines, polite, non-salesy
  - don’t mention CRM label “BAD_TIMING”
  - output only email body
- Connect:
  - **Split In Batches → Agent (main)**
  - **Azure OpenAI Chat Model → Agent (ai_languageModel)**

8) **Add Code node: Parse AI JSON**
- Node: **Code**
- Paste logic equivalent to:
  - Read `$input.first().json.output`
  - Try JSON.parse, else default subject/body
  - Pull lead data using `$('Batch Leads (Split In Batches)').first().json`
  - Return `{subject, body, toEmail, firstName, lastName, ownerId}`
- Connect: **Agent (main output) → Parse AI JSON**

9) **Add Gmail draft node**
- Node: **Gmail**
- Operation: **Create Draft** (draft creation)
- Subject: `{{ $json.subject }}`
- Message: `{{ $('AI: Generate Re-Engagement Follow-Up Email Body').item.json.output }}`
- **Credentials:** Gmail **OAuth2** with scopes allowing draft creation.
- Connect: **Parse AI JSON → Gmail**
- Important: if your Gmail node requires recipients, add **To** using:
  - `{{ $json.toEmail }}` (from Parse AI JSON) or your preferred field mapping.

10) **Add Slack node (success notify)**
- Node: **Slack**
- Operation: **Send message** (to user)
- Select: **user** and choose the target user (original uses `U09HMPVD466`)
- Message text: include lead email and lead status (as in workflow)
- **Credentials:** Slack API token/credential.
- Connect: **Gmail → Slack (success notify)**

11) **Loop continuation**
- Connect: **Slack (success notify) → Split In Batches**
- This causes the next lead to be processed.

12) **Failure path: Build error payload**
- Node: **Code**
- Build JSON payload with lead name/email and error message from the agent error object.
- Connect: **Agent (error output) → Build AI Error Payload**

13) **Failure path: Slack alert**
- Node: **Slack**
- Send warning message to the same Slack user
- Connect: **Build AI Error Payload → Slack (failure alert)**

14) **Loop continuation after failure**
- Connect: **Slack (failure alert) → Split In Batches**

15) **Credentials checklist**
- HubSpot: App Token
- Azure OpenAI: endpoint + key + deployment/model mapping
- Gmail: OAuth2 with Gmail scopes for drafts
- Slack: API token/credential with permission to DM/post to the chosen user

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Never auto-send emails without review. Keep humans in the loop for outreach. | From sticky note “🔐 Required Credentials & Access” |
| Setup checklist: HubSpot App Token, Azure OpenAI credentials, Gmail OAuth2, Slack API, review email tone inside AI prompt | From main sticky note “Setup checklist” |
| Customization ideas: change cadence, update CRM status after follow-up, push follow-ups into a CRM task queue | From main sticky note “Customization ideas” |

---