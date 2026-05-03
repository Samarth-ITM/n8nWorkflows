Turn Gmail meeting summaries into HubSpot CRM records with OpenAI

https://n8nworkflows.xyz/workflows/turn-gmail-meeting-summaries-into-hubspot-crm-records-with-openai-12476


# Turn Gmail meeting summaries into HubSpot CRM records with OpenAI

## 1. Workflow Overview

**Purpose:** Convert incoming Gmail “meeting summary” emails into structured sales data using OpenAI, then **create/update HubSpot CRM objects** (Contact, Deal, and Meeting Engagement) automatically.

**Primary use cases:**
- Sales/discovery call summaries (from tools like tl;dv, Zoom summaries, internal recap emails)
- Removing manual CRM entry by extracting: problem, budget, decision maker, timing, competitors, next steps, and contact/company identity.

### 1.1 Input Reception (Gmail)
Listens for new emails from a specific sender in Gmail Inbox.

### 1.2 Summary Preparation
Normalizes email content into a single `summary` field to feed the AI model.

### 1.3 AI Processing (OpenAI)
Sends the summary to an OpenAI chat model with a strict JSON-only extraction instruction.

### 1.4 Parsing & Validation
Parses the model’s output into JSON so downstream HubSpot nodes can reference fields.

### 1.5 HubSpot CRM Updates
Creates/updates a Contact, updates a Deal, then logs a Meeting engagement associated to the Deal + Contact.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Gmail)

**Overview:** Triggers the workflow when a new email arrives that matches sender + Inbox filter.  
**Nodes involved:** `Gmail Trigger`

#### Node: Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — polling trigger for new Gmail messages.
- **Configuration (interpreted):**
  - Polling schedule: **every minute**
  - Filters:
    - Sender must equal: `user@example.com`
    - Label: `INBOX`
    - Excludes Spam/Trash
  - `simple: false` (returns fuller email payload than simplified mode)
- **Key variables/fields used downstream:**
  - Downstream expects the email text at **`$json.text`** (used in the Set node).
- **Connections:**
  - Output → `Prepare Meeting Summary`
- **Version specifics:** Node typeVersion `1.3`
- **Edge cases / failures:**
  - Gmail OAuth2 expired/invalid → auth failure.
  - Trigger may not return `text` depending on email format; HTML-only emails may behave differently.
  - Sender filtering is strict; summaries from another address won’t trigger.

---

### Block 2 — Prepare & Normalize Summary

**Overview:** Creates a dedicated `summary` field from the email content for consistent AI prompting.  
**Nodes involved:** `Prepare Meeting Summary`

#### Node: Prepare Meeting Summary
- **Type / role:** `n8n-nodes-base.set` — sets/renames fields for downstream use.
- **Configuration (interpreted):**
  - Creates field `summary` as: `{{ $json.text }}`
- **Key expressions/variables:**
  - `={{ $json.text }}`
- **Connections:**
  - Input ← `Gmail Trigger`
  - Output → `AI Extraction`
- **Version specifics:** typeVersion `3.4`
- **Edge cases / failures:**
  - If the trigger payload does not include `text`, `summary` becomes empty/undefined → AI extraction quality drops or prompt becomes blank.

---

### Block 3 — AI Extraction (Structured Sales Data)

**Overview:** Uses OpenAI to extract CRM-ready fields from the meeting summary and returns **JSON-only** content.  
**Nodes involved:** `AI Extraction`

#### Node: AI Extraction
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat completion via LangChain-based node.
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
  - Messages:
    - **System**: “sales operations assistant” + strict rules:
      - If email missing, set `email = null`
      - If no contact email, extract company name instead
      - Return ONLY valid JSON (no markdown/code fences)
      - Required keys: `company, email, problem, budget, decision_maker, timing, competitors, next_actions`
    - **User**: includes `Meeting summary: {{ $json.summary }}`
  - Built-in tools: none enabled
- **Key expressions/variables:**
  - `{{ $json.summary }}`
- **Connections:**
  - Input ← `Prepare Meeting Summary`
  - Output → `Parse AI JSON Output`
- **Version specifics:** typeVersion `2.1`
- **Edge cases / failures:**
  - OpenAI credential/auth failure or quota issues.
  - Model may still return non-JSON (despite instruction), which will break the next node’s `JSON.parse`.
  - Extracted `budget` may be non-numeric (e.g., “$10k–$20k”) which can be problematic for HubSpot “amount” depending on expectations.
  - `next_actions` might not be an array → later `.join('\n')` will fail.

---

### Block 4 — Parse AI JSON Output

**Overview:** Extracts the AI response text from the OpenAI node output structure and parses it into JSON fields.  
**Nodes involved:** `Parse AI JSON Output`

#### Node: Parse AI JSON Output
- **Type / role:** `n8n-nodes-base.function` — custom JavaScript to parse AI output.
- **Configuration (interpreted):**
  - Reads: `const text = $json.output[0].content[0].text;`
  - Returns: `JSON.parse(text)` as the new item JSON
- **Key variables/structure assumptions:**
  - Assumes the OpenAI node outputs:
    - `$json.output[0].content[0].text` containing raw JSON string
- **Connections:**
  - Input ← `AI Extraction`
  - Output → `Create or Update Contact`
- **Version specifics:** typeVersion `1`
- **Edge cases / failures:**
  - If the OpenAI output schema differs (e.g., different property names), `text` access throws.
  - If the AI returns invalid JSON, `JSON.parse` throws and execution fails.
  - If AI returns JSON with trailing commas, comments, or code fences → parse fails.

---

### Block 5 — HubSpot CRM Updates

**Overview:** Uses the parsed AI fields to create/update a HubSpot Contact, update a Deal, and create a Meeting engagement associated to both.  
**Nodes involved:** `Create or Update Contact`, `Update Deal`, `Create Meeting Engagement`

#### Node: Create or Update Contact
- **Type / role:** `n8n-nodes-base.hubspot` — HubSpot CRM Contact upsert-like action (as configured).
- **Configuration (interpreted):**
  - Resource: `contact`
  - Auth: `appToken`
  - **Email field is set to `"="` (literal)**, which is likely misconfigured.
  - No additional fields mapped.
- **Key expressions/variables:**
  - Intended likely: `={{ $('Parse AI JSON Output').item.json.email }}` (but currently not set)
- **Connections:**
  - Input ← `Parse AI JSON Output`
  - Output → `Update Deal`
- **Version specifics:** typeVersion `1`
- **Edge cases / failures:**
  - As-is, HubSpot will likely reject the request (invalid email) or create a bad record.
  - If AI output has `email = null`, HubSpot contact creation by email may fail unless you handle nulls or use another identifier.
  - Auth/token permissions may block contact write.

#### Node: Update Deal
- **Type / role:** `n8n-nodes-base.hubspot` — updates/creates a Deal (depending on node operation mode; named “Update Deal”).
- **Configuration (interpreted):**
  - Sets Deal stage: `closedwon`
  - Pipeline: `default`
  - Amount: `{{ budget }}`
  - Deal name: `{{ company }}`
  - Description: `{{ problem }}`
- **Key expressions/variables:**
  - `={{ $('Parse AI JSON Output').item.json.budget }}`
  - `={{ $('Parse AI JSON Output').item.json.company }}`
  - `={{ $('Parse AI JSON Output').item.json.problem }}`
- **Connections:**
  - Input ← `Create or Update Contact`
  - Output → `Create Meeting Engagement`
- **Version specifics:** typeVersion `1`
- **Edge cases / failures:**
  - Updating a deal typically requires a Deal ID; if the node is truly “update” without an ID, it may fail or be misconfigured.
  - Setting stage to `closedwon` automatically may be undesired or may violate pipeline rules.
  - Amount format issues (string ranges, currency symbols).
  - Missing company/problem → empty fields.

#### Node: Create Meeting Engagement
- **Type / role:** `n8n-nodes-base.hubspot` — creates a HubSpot **meeting engagement** and associates it.
- **Configuration (interpreted):**
  - Resource: `engagement`
  - Type: `meeting`
  - Body (meeting notes) built from parsed AI fields:
    - `problem`
    - `next_actions` joined by newline
  - Associations:
    - Deal IDs: `={{ $json.dealId }}`
    - Contact IDs: `={{ $node["Create or Update Contact"].json.properties.hs_object_id.value }}`
- **Key expressions/variables:**
  - Body:
    - `{{ $('Parse AI JSON Output').item.json.problem + '\n\nNext Steps:\n' + $('Parse AI JSON Output').item.json.next_actions.join('\n') }}`
  - Deal association:
    - `={{ $json.dealId }}`
  - Contact association:
    - `={{ $node["Create or Update Contact"].json.properties.hs_object_id.value }}`
- **Connections:**
  - Input ← `Update Deal`
  - Output → (end)
- **Version specifics:** typeVersion `1`
- **Edge cases / failures:**
  - `$json.dealId` is expected from the previous node’s output; if `Update Deal` doesn’t output `dealId`, association will fail.
  - If `next_actions` is not an array, `.join()` throws.
  - HubSpot engagement APIs are sensitive to association formats; missing/incorrect IDs will error.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for meeting summary emails | — | Prepare Meeting Summary | ## Step:1 Gmail Trigger — Meeting Summary Intake\n\nTriggers the workflow when a new meeting summary email is received in Gmail. |
| Prepare Meeting Summary | n8n-nodes-base.set | Normalize email content into `summary` for AI | Gmail Trigger | AI Extraction | ## Step:2 Prepare & Normalize Summary\n\nExtracts and prepares the meeting summary text from the email for AI processing. |
| AI Extraction | @n8n/n8n-nodes-langchain.openAi | Extract structured CRM fields from summary | Prepare Meeting Summary | Parse AI JSON Output | ## Step:3 AI Extraction — Structured Sales Data\n\nUses OpenAI to extract structured sales insights from the meeting summary. |
| Parse AI JSON Output | n8n-nodes-base.function | Parse AI response text into JSON | AI Extraction | Create or Update Contact | ## Step:4 Parse AI JSON Output\n\nCleans and parses the AI response into valid JSON fields for CRM usage. |
| Create or Update Contact | n8n-nodes-base.hubspot | Create/update HubSpot contact | Parse AI JSON Output | Update Deal | ## Step:5 HubSpot CRM Updates\n\nCreates or updates HubSpot contact, deal, and meeting engagement automatically. |
| Update Deal | n8n-nodes-base.hubspot | Update HubSpot deal fields/stage | Create or Update Contact | Create Meeting Engagement | ## Step:5 HubSpot CRM Updates\n\nCreates or updates HubSpot contact, deal, and meeting engagement automatically. |
| Create Meeting Engagement | n8n-nodes-base.hubspot | Create meeting engagement and associate to deal/contact | Update Deal | — | ## Step:5 HubSpot CRM Updates\n\nCreates or updates HubSpot contact, deal, and meeting engagement automatically. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / description | — | — | ## Meeting Debrief → HubSpot CRM Automation\n\nThis workflow automatically converts meeting summary emails into structured HubSpot CRM updates using AI. It is designed for teams that use meeting recording or note-taking tools (such as tl;dv, Zoom summaries, or internal recap emails) and want to eliminate manual CRM data entry after sales or discovery calls.Instead of sales reps writing notes and updating HubSpot manually, this automation reads the meeting summary email, extracts key sales information using OpenAI, and creates CRM-ready records automatically.\n### How it works\n\n\t•\tThe workflow starts when a new meeting summary email arrives in Gmail.\n\t•\tThe email body is cleaned and normalized into a single summary field.\n\t•\tThe summary is sent to OpenAI, which extracts structured sales data.\n\t•\tThe AI response is parsed into valid JSON.\n\t•\tHubSpot records are created or updated automatically.\n### Setup steps:\n\n\t1.\tConnect your Gmail account (used to receive meeting summaries).\n\t2.\tConnect your OpenAI API credentials.\n\t3.\tConnect your HubSpot account.\n\t4.\tUpdate the Gmail sender filter. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section header/comment | — | — | ## Step:1 Gmail Trigger — Meeting Summary Intake\n\nTriggers the workflow when a new meeting summary email is received in Gmail. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section header/comment | — | — | ## Step:2 Prepare & Normalize Summary\n\nExtracts and prepares the meeting summary text from the email for AI processing. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section header/comment | — | — | ## Step:3 AI Extraction — Structured Sales Data\n\nUses OpenAI to extract structured sales insights from the meeting summary. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section header/comment | — | — | ## Step:4 Parse AI JSON Output\n\nCleans and parses the AI response into valid JSON fields for CRM usage. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Section header/comment | — | — | ## Step:5 HubSpot CRM Updates\n\nCreates or updates HubSpot contact, deal, and meeting engagement automatically. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Meeting Debrief → HubSpot CRM”**
   - (Optional) Add sticky notes for each step to mirror the structure.

2. **Add “Gmail Trigger”**
   - Node type: **Gmail Trigger**
   - Credentials: **Gmail OAuth2**
   - Polling: **Every minute**
   - Filters:
     - Sender: `user@example.com` (replace with your meeting-summary sender)
     - Labels: `INBOX`
     - Include Spam/Trash: disabled
   - Connect: `Gmail Trigger` → `Prepare Meeting Summary`

3. **Add “Prepare Meeting Summary” (Set node)**
   - Node type: **Set**
   - Add a field:
     - Name: `summary`
     - Value (expression): `{{ $json.text }}`
   - Connect: `Prepare Meeting Summary` → `AI Extraction`

4. **Add “AI Extraction” (OpenAI)**
   - Node type: **OpenAI (LangChain)**
   - Credentials: **OpenAI API**
   - Model: `gpt-4.1-mini` (or equivalent)
   - Messages:
     - System message: include rules requiring **JSON-only** output and the required keys:
       - `company, email, problem, budget, decision_maker, timing, competitors, next_actions`
     - User message:
       - `Meeting summary:\n{{ $json.summary }}`
   - Connect: `AI Extraction` → `Parse AI JSON Output`

5. **Add “Parse AI JSON Output” (Function node)**
   - Node type: **Function**
   - Code (logic):
     - Read the AI response text from the OpenAI node output structure
     - `JSON.parse()` it
   - Connect: `Parse AI JSON Output` → `Create or Update Contact`

6. **Add “Create or Update Contact” (HubSpot)**
   - Node type: **HubSpot**
   - Credentials: **HubSpot App Token**
   - Resource: **Contact**
   - Configure email mapping (important):
     - Set Email to expression referencing parsed JSON, typically:
       - `{{ $json.email }}`
     - If you must support `null` emails, add logic (e.g., IF node) to avoid calling HubSpot create with an empty email, or use another lookup key.
   - Connect: `Create or Update Contact` → `Update Deal`

7. **Add “Update Deal” (HubSpot)**
   - Node type: **HubSpot**
   - Credentials: **HubSpot App Token**
   - Configure deal properties:
     - Deal Name: `{{ $json.company }}`
     - Amount: `{{ $json.budget }}`
     - Description: `{{ $json.problem }}`
     - Pipeline: `default`
     - Stage: `closedwon` (adjust to your process)
   - Ensure the node is configured to **create** a deal or has a **deal ID** if updating (depending on your intended behavior).
   - Connect: `Update Deal` → `Create Meeting Engagement`

8. **Add “Create Meeting Engagement” (HubSpot)**
   - Node type: **HubSpot**
   - Resource: **Engagement**
   - Type: **Meeting**
   - Body field expression:
     - Combine `problem` + “Next Steps” + join `next_actions` by newline
   - Associations:
     - Contact ID: reference the created/updated contact’s object id from the Contact node output
     - Deal ID: reference the deal id from the Deal node output
   - End of workflow.

9. **Credentials checklist**
   - Gmail OAuth2: authorized account has access to the mailbox receiving summaries.
   - OpenAI API: valid key and allowed model.
   - HubSpot App Token: must have scopes for contacts, deals, and engagements.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically converts meeting summary emails into structured HubSpot CRM updates using AI… Setup steps: Connect Gmail, OpenAI, HubSpot, update sender filter.” | From the workflow’s main sticky note (high-level description and setup guidance). |