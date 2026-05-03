Launch job vacancies from ATS to Google Calendar, ClickUp and LinkedIn with GPT-4o

https://n8nworkflows.xyz/workflows/launch-job-vacancies-from-ats-to-google-calendar--clickup-and-linkedin-with-gpt-4o-12040


# Launch job vacancies from ATS to Google Calendar, ClickUp and LinkedIn with GPT-4o

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Vacancy Launch: Calendar, ClickUp & AI LinkedIn Post*  
**Stated title (user):** *Launch job vacancies from ATS to Google Calendar, ClickUp and LinkedIn with GPT-4o* (the actual workflow uses an OpenAI model “gpt-4.1-mini”, not GPT‑4o)

**Purpose:**  
This workflow is triggered by an ATS webhook (example: Recrutei). It normalizes vacancy data, creates two Google Calendar events (SLA and expiration), creates a ClickUp task for operational tracking, generates a LinkedIn marketing post using an OpenAI chat model, publishes it to LinkedIn, and then logs the publication as a comment back on the ClickUp task.

### 1.1 Input Reception & Normalization
Receives the vacancy payload, extracts a small set of core fields, converts date strings, and builds an internal ATS URL.

### 1.2 Calendar Deadline Tracking (Google Calendar)
Creates two all-day-like events (implemented as 24h start/end ISO timestamps): SLA date and expiration date.

### 1.3 Operational Tracking (ClickUp)
Merges the two calendar branches and creates a single ClickUp task to track the vacancy.

### 1.4 Marketing Content Preparation (Data + Prompt)
Re-loads the original webhook payload, normalizes boolean flags, and composes a structured “job details” prompt text.

### 1.5 AI Generation + Publishing + Audit Log
Sends the composed prompt to an OpenAI chat model, posts the returned message to LinkedIn, and adds a comment to the ClickUp task indicating the post was published.

---

## 2. Block-by-Block Analysis

### Block 2.1 — ATS Webhook Intake & Field Extraction
**Overview:** Receives the ATS event and extracts a small subset of key fields used for calendar creation and URL building.

**Nodes involved:** Webhook → Arranging data

#### Node: **Webhook**
- **Type / Role:** `Webhook` (n8n-nodes-base.webhook) — workflow entry point.
- **Configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `calendar-trello-webhook`
  - Expects JSON payload; downstream nodes reference `$json.body.*`.
- **Key variables/expressions:** downstream uses `$json.body.id`, `$json.body.title`, etc.
- **Connections:**
  - Output → **Arranging data**
- **Edge cases / failures:**
  - Payload not JSON or missing `body` object → expressions like `$json.body.id` evaluate to `undefined`.
  - ATS sends dates in unexpected format (handled later in Code node but may become `null`).
  - Webhook path mismatch with ATS configuration.
- **Version specifics:** TypeVersion 2.1.

#### Node: **Arranging data**
- **Type / Role:** `Set` — maps/whitelists fields for the calendar branch.
- **Configuration (interpreted):**
  - Creates fields: `id` (number), `title` (string), `sla` (string), `expired` (string), `public_link` (string)
  - Values come from webhook body:
    - `id = {{ $json.body.id }}`
    - `title = {{ $json.body.title }}`
    - `sla = {{ $json.body.sla }}`
    - `expired = {{ $json.body.expired }}`
    - `public_link = {{ $json.body.public_link }}`
- **Connections:**
  - Input: **Webhook**
  - Output → **Generating vacancy ATS's URL**
- **Edge cases / failures:**
  - If webhook payload changes shape (e.g., `sla` nested elsewhere), fields become empty and later calendar events may fail.
- **Version specifics:** TypeVersion 3.4.

---

### Block 2.2 — Date Conversion + ATS URL Composition
**Overview:** Converts Brazilian date strings (DD/MM/YYYY) into ISO timestamps for Google Calendar, calculates end timestamps (+1 day) for 24h events, and generates the ATS internal pipeline URL.

**Nodes involved:** Generating vacancy ATS's URL

#### Node: **Generating vacancy ATS's URL**
- **Type / Role:** `Code` — transforms date strings, adds calculated fields, and composes a link.
- **Configuration (interpreted):**
  - Parses `sla` and `expired` strings expected in `DD/MM/YYYY`.
  - Produces:
    - `sla` as ISO at 00:00:00.000 (local JS Date converted to ISO)
    - `expired` as ISO at 00:00:00.000
    - `sla_end` = `sla` + 1 day (00:00 next day)
    - `expired_end` = `expired` + 1 day
    - `link` = `https://app.recrutei.com.br/vacancy/${id}/pipe`
  - Keeps original JSON fields via `...item.json`.
- **Key logic notes:**
  - Uses `new Date(year, monthIndex, day)` then `toISOString()`; the resulting ISO is UTC and can shift day depending on timezone.
- **Connections:**
  - Input: **Arranging data**
  - Outputs (two parallel branches):
    - → **Create event for the SLA**
    - → **Create event for the expired date**
- **Edge cases / failures:**
  - If `sla` / `expired` is missing or not `DD/MM/YYYY`, `processDate()` returns `null`. Google Calendar node will likely error on invalid start/end.
  - Timezone offset issues: events can appear on the previous/next day in Google Calendar if the account timezone differs. (Because ISO is UTC; “all-day” events are better modeled using `date` fields rather than `dateTime` when possible.)
- **Version specifics:** TypeVersion 2.

---

### Block 2.3 — Calendar Event Creation (SLA + Expiration)
**Overview:** Creates two separate Google Calendar events to track the SLA date and the vacancy expiration/application deadline.

**Nodes involved:** Create event for the SLA, Create event for the expired date

#### Node: **Create event for the SLA**
- **Type / Role:** `Google Calendar` — create event entry for SLA date.
- **Configuration (interpreted):**
  - **Start:** `{{ $json.sla }}`
  - **End:** `{{ $json.sla_end }}`
  - **Calendar:** selected from list (value is empty in exported JSON; must be chosen in UI)
  - **Summary:** `SLA date for {{ $json.title }}`
  - **Description:** includes vacancy title, internal link (`$json.link`), and public link (`$json.public_link`)
- **Connections:**
  - Input: **Generating vacancy ATS's URL**
  - Output → **Merge** (input index 0)
- **Edge cases / failures:**
  - Missing/invalid calendar selection → node fails.
  - OAuth/credentials expired → auth error.
  - Invalid ISO timestamps or end <= start → API rejection.
- **Version specifics:** TypeVersion 1.3.

#### Node: **Create event for the expired date**
- **Type / Role:** `Google Calendar` — create event entry for application deadline.
- **Configuration (interpreted):**
  - **Start:** `{{ $json.expired }}`
  - **End:** `{{ $json.expired_end }}`
  - **Summary:** `Expire date of the vacancy "{{ $json.title }}"`
  - **Description:** similar to SLA event, describing application deadline and links.
  - **Calendar:** must be selected in UI (export shows empty).
- **Connections:**
  - Input: **Generating vacancy ATS's URL**
  - Output → **Merge** (input index 1)
- **Edge cases / failures:** same as SLA node.
- **Version specifics:** TypeVersion 1.3.

---

### Block 2.4 — Branch Consolidation & ClickUp Task Creation
**Overview:** Merges both calendar event branches into a single continuation point, then creates a ClickUp task to track the vacancy.

**Nodes involved:** Merge → Create a task for the vacancy

#### Node: **Merge**
- **Type / Role:** `Merge` — joins the two calendar paths.
- **Configuration (interpreted):**
  - Uses default merge behavior (not explicitly set in parameters).
  - Practically, it acts as a synchronization point so the workflow proceeds after both branches produce output.
- **Connections:**
  - Inputs:
    - From **Create event for the SLA** (index 0)
    - From **Create event for the expired date** (index 1)
  - Output → **Create a task for the vacancy**
- **Edge cases / failures:**
  - If one calendar node errors and stops execution, the merge may never receive both inputs, preventing task creation.
  - Default merge mode may not preserve the exact item you expect if multiple items are received (this workflow appears to handle a single vacancy at a time).
- **Version specifics:** TypeVersion 3.2.

#### Node: **Create a task for the vacancy**
- **Type / Role:** `ClickUp` — creates a task for operational tracking.
- **Configuration (interpreted):**
  - **Auth:** OAuth2
  - **Task name:** `{{ $('Generating vacancy ATS\'s URL').item.json.title }}`
    - Note: references the **Generating vacancy ATS's URL** node directly, not the merge output.
  - **Additional Fields:** `startDateTime = true` (enables datetime interpretation for start date fields; however no explicit start date is set here).
  - **Execute Once:** enabled (`executeOnce: true`) — prevents multiple task creations if multiple input items arrive.
- **Connections:**
  - Input: **Merge**
  - Output → **Recover vacancy data**
- **Edge cases / failures:**
  - ClickUp workspace/list not specified in the exported snippet; in n8n UI you must choose Space/List (otherwise it will fail).
  - OAuth token revoked/expired → 401.
  - `executeOnce` can hide issues if you test with multiple items; only one will create a task.
- **Version specifics:** TypeVersion 1.
- **Sub-workflow reference:** none.

---

### Block 2.5 — Recover Full Vacancy Payload & Normalize Booleans
**Overview:** Re-reads the original webhook payload (not the reduced “Arranging data” fields) and converts numeric flags (0/1) into “yes/no” strings for cleaner prompt generation.

**Nodes involved:** Recover vacancy data

#### Node: **Recover vacancy data**
- **Type / Role:** `Code` — fetches items from a previous node and transforms them.
- **Configuration (interpreted):**
  - Uses `$items("Webhook")` to retrieve the original webhook items at runtime.
  - Converts:
    - `fixed_remuneration`, `remote`, `pcd`, `is_inclusive` from `1/0` to `'yes'/'no'`.
  - Outputs `json: body` (the full vacancy object).
- **Connections:**
  - Input: **Create a task for the vacancy** (used only as an execution chain step; data actually comes from Webhook node via `$items("Webhook")`)
  - Output → **Generates a prompt with the vacancy data**
- **Edge cases / failures:**
  - If the Webhook node name changes, `$items("Webhook")` breaks.
  - If boolean fields arrive as true/false instead of 0/1, conversion will output `'no'` for anything other than `1`.
- **Version specifics:** TypeVersion 2.

---

### Block 2.6 — Prompt Construction (Structured Job Details)
**Overview:** Converts the vacancy JSON into a structured markdown-like text, cleaning HTML from the description and formatting compensation.

**Nodes involved:** Generates a prompt with the vacancy data

#### Node: **Generates a prompt with the vacancy data**
- **Type / Role:** `Code` — builds `detailedJob` text.
- **Configuration (interpreted):**
  - Reads `vagaData = $input.item.json`.
  - Uses a mapping (`fieldMap`) to:
    - Provide English labels for fields.
    - Skip internal/ID fields.
    - Treat `description` as long text: strips common HTML tags and formats lists.
    - Formats `remuneration` as USD currency if it’s a number (hard-coded `en-US` + `USD`).
  - Output: `{ detailedJob: outputText }`
- **Connections:**
  - Input: **Recover vacancy data**
  - Output → **Message a model**
- **Edge cases / failures:**
  - If `description` contains complex HTML, the regex cleanup may produce messy text.
  - Currency assumption (USD) may be wrong for non-US jobs.
  - If `remuneration` comes as a string, it won’t be formatted as currency.
- **Version specifics:** TypeVersion 2.

---

### Block 2.7 — AI Post Generation (OpenAI Chat Model)
**Overview:** Sends the structured job details to an OpenAI chat model with a system instruction to produce a LinkedIn marketing post.

**Nodes involved:** Message a model

#### Node: **Message a model**
- **Type / Role:** `OpenAI (LangChain)` — chat completion.
- **Configuration (interpreted):**
  - **Model:** `gpt-4.1-mini` (selected via list)
  - **Messages:**
    - A message with content `{{ $json.detailedJob }}` (role not explicitly set in JSON; in this node it is typically treated as a “user” message)
    - A **system** message: “transform this information into a marketing post… via LinkedIn.”
- **Connections:**
  - Input: **Generates a prompt with the vacancy data**
  - Output → **Create a post**
- **Edge cases / failures:**
  - Missing/invalid OpenAI credentials → auth error.
  - Model name availability depends on your OpenAI account/region and n8n node version.
  - Output shape: this workflow expects `$json.message.content` downstream; if node output schema changes, LinkedIn post will fail.
- **Version specifics:** TypeVersion 1.8.

---

### Block 2.8 — Publish to LinkedIn + Log to ClickUp
**Overview:** Publishes the generated text to LinkedIn, then comments on the ClickUp task to record that the vacancy was posted.

**Nodes involved:** Create a post → Create a comment in vacancy task

#### Node: **Create a post**
- **Type / Role:** `LinkedIn` — publishes a post.
- **Configuration (interpreted):**
  - **Text:** `{{ $json.message.content }}`
  - Additional fields: none set.
- **Connections:**
  - Input: **Message a model**
  - Output → **Create a comment in vacancy task**
- **Edge cases / failures:**
  - LinkedIn API permissions vary (member vs organization posting). Missing scopes can block posting.
  - Rate limits and moderation constraints can reject content.
  - If `$json.message.content` is undefined, the post may be empty → API rejection.
- **Version specifics:** TypeVersion 1.

#### Node: **Create a comment in vacancy task**
- **Type / Role:** `ClickUp` — adds a comment on the created task.
- **Configuration (interpreted):**
  - **Resource:** comment
  - **Comment on:** task
  - **Task ID:** `{{ $('Create a task for the vacancy').item.json.id }}`
  - **Comment text:** `Vacancy posted in LinkedIn`
  - **Auth:** OAuth2
- **Connections:**
  - Input: **Create a post**
  - Output: end of workflow
- **Edge cases / failures:**
  - If task creation failed earlier, task ID expression resolves to undefined and commenting fails.
  - OAuth expired → 401.
- **Version specifics:** TypeVersion 1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Entry point receiving ATS payload | — | Arranging data | ## Data Ingestion  \nReceives vacancy details via Webhook. The Code node converts Brazilian date strings (DD/MM/YYYY) into ISO format and calculates the 24h duration required for Google Calendar events. |
| Arranging data | Set | Extract core fields for calendar + URL generation | Webhook | Generating vacancy ATS's URL | ## Data Ingestion  \nReceives vacancy details via Webhook. The Code node converts Brazilian date strings (DD/MM/YYYY) into ISO format and calculates the 24h duration required for Google Calendar events. |
| Generating vacancy ATS's URL | Code | Convert dates to ISO (+ end date) and generate ATS URL | Arranging data | Create event for the SLA; Create event for the expired date | ## Data Ingestion  \nReceives vacancy details via Webhook. The Code node converts Brazilian date strings (DD/MM/YYYY) into ISO format and calculates the 24h duration required for Google Calendar events. |
| Create event for the SLA | Google Calendar | Create SLA deadline event | Generating vacancy ATS's URL | Merge | ## Event Creation  \nGenerates two distinct entries on the Recruiter's calendar. It maps the vacancy URL and public link into the event description for quick access. |
| Create event for the expired date | Google Calendar | Create application deadline/expiration event | Generating vacancy ATS's URL | Merge | ## Event Creation  \nGenerates two distinct entries on the Recruiter's calendar. It maps the vacancy URL and public link into the event description for quick access. |
| Merge | Merge | Synchronize both calendar branches before task creation | Create event for the SLA; Create event for the expired date | Create a task for the vacancy | ## Task Registration  \nConsolidates the flow to ensure that, regardless of calendar successes, a task is created in ClickUp to track the vacancy's operational progress. |
| Create a task for the vacancy | ClickUp | Create tracking task for vacancy | Merge | Recover vacancy data | ## Task Registration  \nConsolidates the flow to ensure that, regardless of calendar successes, a task is created in ClickUp to track the vacancy's operational progress. |
| Recover vacancy data | Code | Re-load webhook payload and normalize boolean flags | Create a task for the vacancy | Generates a prompt with the vacancy data | ## Preparing vacancy to be publish  \nIt recovers the data from the webhook and then inputs into a prompt, ready to |
| Generates a prompt with the vacancy data | Code | Build structured prompt text (clean HTML, format fields) | Recover vacancy data | Message a model | ## Preparing vacancy to be publish  \nIt recovers the data from the webhook and then inputs into a prompt, ready to |
| Message a model | OpenAI (LangChain) | Generate LinkedIn marketing post from prompt | Generates a prompt with the vacancy data | Create a post | ## Generating post and registering in vacancy task  \nThe agent receives all the data in a single prompt and generates a marketing post to be send to LinkedIn, then, it register this action in the vacancy task in clickup |
| Create a post | LinkedIn | Publish the generated post | Message a model | Create a comment in vacancy task | ## Generating post and registering in vacancy task  \nThe agent receives all the data in a single prompt and generates a marketing post to be send to LinkedIn, then, it register this action in the vacancy task in clickup |
| Create a comment in vacancy task | ClickUp | Log publication back to ClickUp task | Create a post | — | ## Generating post and registering in vacancy task  \nThe agent receives all the data in a single prompt and generates a marketing post to be send to LinkedIn, then, it register this action in the vacancy task in clickup |
| Sticky Note | Sticky Note | Documentation / overview | — | — | ## Overview: Automated Vacancy Launch & AI Marketing  \n\nThis workflow streamlines the entire job opening process by connecting your ATS to your operational and marketing tools. It not only manages deadlines but also automates the promotion of the vacancy.\n\n1. **Schedule:** Creates SLA and Expiration events in Google Calendar based on ATS dates.\n2. **Track:** Creates a central task in ClickUp to manage the selection process.\n3. **Content Generation:** Uses GPT-4o to analyze the job description and write a compelling marketing post.\n4. **Publish:** Automatically posts the job to LinkedIn and logs the action back in the ClickUp task.\n\n## Setup steps\n1. **Webhook:** Configure your Recrutei ATS (or similar) to trigger this workflow.\n2. **Google Calendar:** Select the calendar for deadline tracking.\n3. **ClickUp:** Map the Team, Space, and List where vacancy tasks should be created.\n4. **OpenAI:** Ensure you have a valid API Key.\n5. **LinkedIn:** Connect your profile or company page. |
| Sticky Note1 | Sticky Note | Documentation / block header | — | — | |
| Sticky Note2 | Sticky Note | Documentation / block header | — | — | |
| Sticky Note3 | Sticky Note | Documentation / block header | — | — | |
| Sticky Note4 | Sticky Note | Documentation / block header | — | — | |
| Sticky Note5 | Sticky Note | Documentation / block header | — | — | |

> Note: Sticky Note nodes are included because they exist as nodes in the workflow JSON, but they do not execute or connect to other nodes.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Vacancy Launch: Calendar, ClickUp & AI LinkedIn Post**
   - Keep it **inactive** until credentials and external systems are set.

2. **Add the entry node: Webhook**
   - Node: **Webhook**
   - Method: **POST**
   - Path: `calendar-trello-webhook`
   - Save the workflow to generate the Production URL.
   - In your ATS (e.g., Recrutei), configure an outbound webhook to this URL.

3. **Add “Arranging data” (Set node)**
   - Node type: **Set**
   - Add fields (assignments) reading from the webhook body:
     - `id` (Number): `{{ $json.body.id }}`
     - `title` (String): `{{ $json.body.title }}`
     - `sla` (String): `{{ $json.body.sla }}`
     - `expired` (String): `{{ $json.body.expired }}`
     - `public_link` (String): `{{ $json.body.public_link }}`
   - Connect: **Webhook → Arranging data**

4. **Add “Generating vacancy ATS's URL” (Code node)**
   - Node type: **Code**
   - Paste logic that:
     - parses `DD/MM/YYYY` → ISO
     - creates `sla_end`, `expired_end` (+1 day)
     - creates `link = https://app.recrutei.com.br/vacancy/${id}/pipe`
   - Connect: **Arranging data → Generating vacancy ATS's URL**

5. **Add Google Calendar node for SLA**
   - Node: **Google Calendar** (Create Event operation)
   - Credentials: connect Google OAuth2 for the recruiter/calendar owner.
   - Calendar: select your target calendar (deadline tracking).
   - Start: `{{ $json.sla }}`
   - End: `{{ $json.sla_end }}`
   - Summary: `SLA date for {{ $json.title }}`
   - Description: include `{{ $json.link }}` and `{{ $json.public_link }}`
   - Connect: **Generating vacancy ATS's URL → Create event for the SLA**

6. **Add Google Calendar node for Expiration**
   - Node: **Google Calendar** (Create Event)
   - Use same credentials and calendar selection.
   - Start: `{{ $json.expired }}`
   - End: `{{ $json.expired_end }}`
   - Summary: `Expire date of the vacancy "{{ $json.title }}"`
   - Description: include `{{ $json.link }}` and `{{ $json.public_link }}`
   - Connect: **Generating vacancy ATS's URL → Create event for the expired date**

7. **Add Merge node**
   - Node: **Merge**
   - Keep default configuration.
   - Connect:
     - **Create event for the SLA → Merge (Input 1 / index 0)**
     - **Create event for the expired date → Merge (Input 2 / index 1)**

8. **Add ClickUp node to create the vacancy task**
   - Node: **ClickUp** (Create Task)
   - Authentication: **OAuth2** (connect ClickUp OAuth in n8n credentials)
   - Select the appropriate **Workspace/Team**, **Space**, **List** in the node UI (required even though not visible in the export).
   - Task Name expression:
     - `{{ $('Generating vacancy ATS\'s URL').item.json.title }}`
   - Additional option: enable **Start Date Time** (`startDateTime = true`) if you plan to set datetime fields.
   - Enable **Execute Once** if you want to guard against multiple items.
   - Connect: **Merge → Create a task for the vacancy**

9. **Add “Recover vacancy data” (Code node)**
   - Node type: **Code**
   - Logic:
     - `const webhookItems = $items("Webhook");`
     - Convert `fixed_remuneration`, `remote`, `pcd`, `is_inclusive` from `0/1` to `'no'/'yes'`
     - Return `body` as output JSON
   - Connect: **Create a task for the vacancy → Recover vacancy data**

10. **Add “Generates a prompt with the vacancy data” (Code node)**
   - Node type: **Code**
   - Build `detailedJob` text:
     - map fields to labeled output
     - strip simple HTML from `description`
     - format `remuneration` as currency (adjust currency/locale as needed)
   - Output JSON must include: `{ detailedJob: "...text..." }`
   - Connect: **Recover vacancy data → Generates a prompt with the vacancy data**

11. **Add OpenAI “Message a model” node**
   - Node type: **OpenAI (LangChain)**
   - Credentials: OpenAI API key credential in n8n.
   - Model: select **gpt-4.1-mini** (or replace with GPT‑4o if you prefer and it’s available).
   - Messages:
     - User message content: `{{ $json.detailedJob }}`
     - System message: instruction to turn job info into a LinkedIn marketing post.
   - Connect: **Generates a prompt with the vacancy data → Message a model**

12. **Add LinkedIn “Create a post” node**
   - Node type: **LinkedIn**
   - Credentials: LinkedIn OAuth (member or organization access depending on posting target).
   - Post text: `{{ $json.message.content }}`
   - Connect: **Message a model → Create a post**

13. **Add ClickUp “Create a comment in vacancy task” node**
   - Node type: **ClickUp**
   - Resource: **Comment**
   - Comment on: **Task**
   - Task ID: `{{ $('Create a task for the vacancy').item.json.id }}`
   - Comment text: `Vacancy posted in LinkedIn`
   - Connect: **Create a post → Create a comment in vacancy task**

14. **Add sticky notes (optional)**
   - Add Sticky Notes to document blocks (as in the original workflow). They do not affect execution.

15. **Test end-to-end**
   - Use Webhook test URL first; send a sample payload (matching ATS structure with `body.sla` and `body.expired` as `DD/MM/YYYY`).
   - Verify:
     - Calendar events appear on correct dates/timezone
     - ClickUp task created in correct list
     - OpenAI returns output in expected field (`message.content`)
     - LinkedIn post is published
     - ClickUp comment is added to the same task

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The sticky note overview mentions GPT‑4o, but the configured model in the workflow is **gpt-4.1-mini**. | Align documentation/expectations or change the model in the OpenAI node. |
| Date conversion uses `toISOString()` which is UTC-based; Google Calendar may display dates shifted depending on timezone. | Consider using all-day events (`date` fields) or explicitly managing timezones if accuracy is critical. |
| “Recover vacancy data” depends on node name `Webhook` via `$items("Webhook")`. | Renaming the Webhook node will break the code unless updated accordingly. |
| ClickUp and Google Calendar nodes require proper OAuth2 credentials and (for ClickUp) correct Space/List selection. | Configure credentials in n8n Credentials and select resources in node UI. |
| Workflow overview sticky note content: “Overview: Automated Vacancy Launch & AI Marketing” + setup steps for Webhook, Google Calendar, ClickUp, OpenAI, LinkedIn. | Internal project documentation contained in the workflow canvas. |