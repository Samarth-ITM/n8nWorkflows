Create LinkedIn posts from web pages using Airtable, Dumpling AI, Claude and PostPulse

https://n8nworkflows.xyz/workflows/create-linkedin-posts-from-web-pages-using-airtable--dumpling-ai--claude-and-postpulse-14841


# Create LinkedIn posts from web pages using Airtable, Dumpling AI, Claude and PostPulse

The user wants a comprehensive analysis and documentation of this n8n workflow. Let me parse through the workflow JSON carefully.

Workflow name: "Website Articles to LinkedIn Post copy"

Nodes:
1. Airtable Trigger (n8n-nodes-base.airtableTrigger) - id: 8578bf54...
2. Extract URL Details (n8n-nodes-base.httpRequest) - id: 9d5102e9...
3. Message a model (@n8n/n8n-nodes-langchain.anthropic) - id: ce5ba654...
4. Schedule a light post (@postpulse/n8n-nodes-postpulse.postPulse) - id: 9cd94275...
5. Update record (n8n-nodes-base.airtable) - id: d0c768f7...
6. Sticky Note2 - id: 8907147d...
7. Sticky Note1 - id: 9d231e1b...
8. Sticky Note3 - id: 196535a9...

Connections:
- Airtable Trigger → Extract URL Details
- Extract URL Details → Message a model
- Message a model → Schedule a light post
- Schedule a light post → Update record

Now let me analyze each node in detail.

**Airtable Trigger:**
- Type: n8n-nodes-base.airtableTrigger, version 1
- Base ID: appV8W8qiCqqy4cIz (via URL: https://airtable.com/appV8W8qiCqqy4cIz/tblQKHqO4ltrAN0jS)
- Table ID: tblQKHqO4ltrAN0jS
- Polling: everyMinute
- Trigger field: "URL link"
- Authentication: airtableTokenApi (Personal Access Token)
- Additional fields: none

**Extract URL Details:**
- Type: n8n-nodes-base.httpRequest, version 4.4
- URL: https://app.dumplingai.com/api/v1/extract
- Method: POST
- Headers: Content-Type: application/json, Authorization: Bearer YOUR_TOKEN_HERE
- JSON Body: uses url from $json.fields['URL link'] and schema with title and description fields
- Output: results with description (and title)

**Message a model:**
- Type: @n8n/n8n-nodes-langchain.anthropic, version 1
- Model: claude-haiku-4-5-20251001
- Message: Creates a long form LinkedIn post using the article summary from $json.results.description
- Credentials: anthropicApi

**Schedule a light post:**
- Type: @postpulse/n8n-nodes-postpulse.postPulse, version 1
- Operation: scheduleLight
- Content: $json.content[0].text
- Scheduled Time: 2026-04-01T00:00:00
- Social Media Account: LINKEDIN|1204
- Credentials: postPulseOAuth2Api

**Update record:**
- Type: n8n-nodes-base.airtable, version 2.2
- Operation: update
- Base: appV8W8qiCqqy4cIz (LinkedIn Post from URL)
- Table: tblQKHqO4ltrAN0jS (Table 1)
- Columns:
  - id: from $('Airtable Trigger').item.json.id
  - Post Status: "Scheduled"
  - Generated LinkedIn Post: $json.publications[0].posts[0].content
- Matching column: id
- Credentials: airtableTokenApi

**Sticky Notes:**
- Sticky Note 2: Covers workflow purpose, steps, and notes about refining Claude prompt
- Sticky note 1: "Kick-Off workflow" - covers Airtable Trigger, Extract URL Details, Message a model area
- Sticky note 3: "Schedule Post and Updates Airtable" - covers Schedule a light post and Update record area

Now let me structure the document according to the required format.

Let me think about the logical blocks:
1. Input Reception: Airtable Trigger
2. URL Data Extraction: Extract URL Details (HTTP request to Dumpling AI)
3. AI Content Generation: Message a model (Claude)
4. Post Scheduling: Schedule a light post (PostPulse)
5. Status Update: Update record (Airtable)

Now I need to create a comprehensive document.

Let me also note edge cases:
- Airtable trigger: might not fire if no new records, authentication issues
- Extract URL Details: invalid URL, Dumpling AI API errors, timeout, auth token needed
- Message a model: Claude API errors, rate limits, content policy
- Schedule a light post: PostPulse OAuth2 token expiration, invalid social media account
- Update record: Airtable rate limits, record not found

Let me also note the sticky notes carefully:

Sticky Note2 content:
**Workflow Purpose**
A LinkedIn post generation workflow that extracts data from websites, creates content with Claude, and schedule post in PostPulse using data from Airtable.

## Workflow Steps
**1. Airtable Trigger** 
* Triggers the worflow whenever a new table is created in Airtable. 

**2. Extract URL Details**
* Extracts website summary using Dumpling AI's extract endpoint and the url field from Airtable.

**3. Create LinkedIn Post**
* Creates LinkedIn post using Claude and the extracted website summary.

**4. Schedule a Light Post**
* Schedule post to go out daily. 

**5. Update Record**
* Update status of Airtable record to "scheduled" and save generated LinkedIn post.

## Notes
Prompt used in Claude can be refined to create an output that sounds more like the users "LinkedIn voice".

Sticky Note1 content:
## Kick-Off workflow
Searches Airtable and uses AI to extract website data and create a social media post with Dumpling AI and Claude respectively.

Sticky Note3 content:
## Schedule Post and Updates Airtable
Schedules generated social media post in PostPulse and updates a record in Airtable.

Now mapping sticky notes to nodes based on positions:
- Sticky Note2 position: [-736,-576] - this is a general overview note, not directly covering specific nodes
- Sticky Note1 position: [-32,-176] - covers area near Airtable Trigger [0,0] and Extract URL Details [208,0] and possibly Message a model [416,0]
- Sticky Note3 position: [688,-176] - covers area near Schedule a light post [736,0] and Update record [944,0]

So for the summary table:
- Airtable Trigger → Sticky Note1
- Extract URL Details → Sticky Note1
- Message a model → Sticky Note1
- Schedule a light post → Sticky Note3
- Update record → Sticky Note3

And Sticky Note2 is a general purpose note that could apply to all nodes. I'll include it for all nodes as well or just mention it as a global note. Actually the instructions say: "If a sticky note visually covers multiple nodes, duplicate its content in each affected row." Sticky Note2 is positioned far away at [-736,-576] so it doesn't visually cover any specific node. It's more of a documentation note. I'll include it as a general note but not duplicate it into each node row.

Actually, let me reconsider. The instruction says "If a sticky note visually covers multiple nodes, duplicate its content in each affected row." Sticky Note2 is at position [-736,-576] which is far from all nodes, so it doesn't visually cover any node. I'll treat it as a general documentation note.

For Sticky Note1 and Sticky Note3, I need to determine which nodes they cover. Based on positions:
- Sticky Note1: position [-32,-176], size 688x288. This covers roughly from x=-32 to x=656, y=-176 to y=112. This would cover Airtable Trigger (0,0), Extract URL Details (208,0), and Message a model (416,0).
- Sticky Note3: position [688,-176], size 448x304. This covers roughly from x=688 to x=1136, y=-176 to y=128. This would cover Schedule a light post (736,0) and Update record (944,0).

Now let me write the complete document.# Create LinkedIn Posts from Web Pages Using Airtable, Dumpling AI, Claude and PostPulse

---

## 1. Workflow Overview

This workflow automates the creation and scheduling of LinkedIn posts sourced from web page URLs stored in an Airtable base. When a new record containing a **URL link** field value is created in Airtable, the workflow is triggered. It then:

1. **Retrieves the URL** from the Airtable record.
2. **Extracts article data** (title and summary) from the web page via Dumpling AI's extraction API.
3. **Generates a long-form LinkedIn post** using Anthropic's Claude model, based on the extracted summary.
4. **Schedules the generated post** in PostPulse for publication on LinkedIn.
5. **Updates the original Airtable record** with the generated post text and marks the status as *Scheduled*.

The workflow operates as a single linear pipeline with no branching or sub-workflow calls, and all processing is stateless aside from the Airtable record updates.

---

## 2. Block-by-Block Analysis

### Block 1 – Input Reception

**Overview:** This block listens for new records in a specific Airtable table. Whenever a record is created containing a value in the *URL link* field, the workflow is triggered and the record data is passed downstream.

**Nodes Involved:** Airtable Trigger

| Attribute | Detail |
|---|---|
| **Node Name** | Airtable Trigger |
| **Type** | `n8n-nodes-base.airtableTrigger` (v1) |
| **Configuration** | Base: *LinkedIn Post from URL* (`appV8W8qiCqqy4cIz`), Table: *Table 1* (`tblQKHqO4ltrAN0jS`), Trigger field: `URL link`, Polling interval: every 1 minute, Authentication: Airtable Personal Access Token |
| **Key Expressions / Variables** | `$json.fields['URL link']` – the URL stored in the Airtable record that downstream nodes will consume |
| **Input Connections** | None (entry point) |
| **Output Connections** | → Extract URL Details |
| **Credentials Required** | Airtable Personal Access Token (with read access to the specified base and table) |
| **Edge Cases / Failure Types** | • Token lacks permission → 403 error. • Polling frequency too high → Airtable API rate limits. • If a record has no value in the *URL link* field, the trigger still fires but downstream extraction may fail. |

---

### Block 2 – URL Data Extraction

**Overview:** This block calls Dumpling AI's `/extract` endpoint, sending the URL obtained from Airtable. Dumpling AI returns a structured response containing a title and a description/summary of the web page content.

**Nodes Involved:** Extract URL Details

| Attribute | Detail |
|---|---|
| **Node Name** | Extract URL Details |
| **Type** | `n8n-nodes-base.httpRequest` (v4.4) |
| **Configuration** | Method: POST, URL: `https://app.dumplingai.com/api/v1/extract`, Body: JSON with `url` sourced from `$json.fields['URL link']` and a `schema` object defining `title` (string) and `description` (string) fields. Headers: `Content-Type: application/json`, `Authorization: Bearer YOUR_TOKEN_HERE` |
| **Key Expressions / Variables** | `{{ $json.fields['URL link'] }}` – the URL sent to Dumpling AI. Output path `$json.results.description` – the extracted summary consumed by the next node. |
| **Input Connections** | ← Airtable Trigger |
| **Output Connections** | → Message a model |
| **Credentials Required** | Dumpling AI Bearer token (manually set in the Authorization header; not stored as an n8n credential) |
| **Edge Cases / Failure Types** | • Invalid or unreachable URL → Dumpling AI returns an error. • Expired or missing Bearer token → 401 Unauthorized. • Network timeout → httpRequest node times out. • Schema mismatch – if Dumpling AI cannot parse the page, `results.description` may be empty or null, causing Claude to receive no context. |

---

### Block 3 – AI Content Generation

**Overview:** This block uses Anthropic's Claude (Haiku 4.5) to compose a long-form, attention-grabbing LinkedIn post based on the web page summary returned by Dumpling AI.

**Nodes Involved:** Message a model

| Attribute | Detail |
|---|---|
| **Node Name** | Message a model |
| **Type** | `@n8n/n8n-nodes-langchain.anthropic` (v1) |
| **Configuration** | Model: `claude-haiku-4-5-20251001`, Prompt: instructs Claude to create a long-form LinkedIn post that is attention-catching, scroll-stopping, insightful, and detailed, using the provided article summary. The summary is injected via `{{ $json.results.description }}`. The prompt explicitly requests only the post text as output (no preamble). |
| **Key Expressions / Variables** | `$json.results.description` – the extracted summary from Dumpling AI. Output: `$json.content[0].text` – the generated LinkedIn post text. |
| **Input Connections** | ← Extract URL Details |
| **Output Connections** | → Schedule a light post |
| **Credentials Required** | Anthropic API key (stored as n8n credential *Anthropic account*) |
| **Edge Cases / Failure Types** | • Anthropic API rate limit or key issue → 429/401 errors. • Content policy violation → Anthropic may refuse to generate. • Empty or very short summary → Claude produces a generic or irrelevant post. • If `results.description` is null/undefined, the prompt may be incomplete. |

---

### Block 4 – Post Scheduling

**Overview:** This block takes the generated LinkedIn post text and schedules it for publication via PostPulse, targeting a specific LinkedIn social media account.

**Nodes Involved:** Schedule a light post

| Attribute | Detail |
|---|---|
| **Node Name** | Schedule a light post |
| **Type** | `@postpulse/n8n-nodes-postpulse.postPulse` (v1) |
| **Configuration** | Operation: `scheduleLight`, Content: `{{ $json.content[0].text }}` (the Claude output), Scheduled time: `2026-04-01T00:00:00` (hard-coded), Social media account: `LINKEDIN|1204` |
| **Key Expressions / Variables** | `$json.content[0].text` – the LinkedIn post text from Claude. Output includes `$json.publications[0].posts[0].content` which is consumed by the next node. |
| **Input Connections** | ← Message a model |
| **Output Connections** | → Update record |
| **Credentials Required** | PostPulse OAuth2 credential (stored as *PostPulse account 2*) |
| **Edge Cases / Failure Types** | • OAuth2 token expired → PostPulse returns 401. • Invalid or disconnected social media account ID → 400 error. • Hard-coded date in the past → PostPulse may reject or immediately publish. • Rate limits on PostPulse scheduling API. |

---

### Block 5 – Status Update in Airtable

**Overview:** This block writes the generated LinkedIn post text back to the originating Airtable record and updates the *Post Status* field to *Scheduled*, closing the data loop.

**Nodes Involved:** Update record

| Attribute | Detail |
|---|---|
| **Node Name** | Update record |
| **Type** | `n8n-nodes-base.airtable` (v2.2) |
| **Configuration** | Operation: update, Base: *LinkedIn Post from URL* (`appV8W8qiCqqy4cIz`), Table: *Table 1* (`tblQKHqO4ltrAN0jS`), Matching column: `id` (value from `$('Airtable Trigger').item.json.id`), Columns mapped: **Generated LinkedIn Post** = `{{ $json.publications[0].posts[0].content }}`, **Post Status** = `Scheduled` |
| **Key Expressions / Variables** | `$('Airtable Trigger').item.json.id` – the Airtable record ID from the trigger. `$json.publications[0].posts[0].content` – the post content returned by PostPulse. |
| **Input Connections** | ← Schedule a light post |
| **Output Connections** | None (end of workflow) |
| **Credentials Required** | Airtable Personal Access Token (same as trigger) |
| **Edge Cases / Failure Types** | • Record ID not found (e.g., record deleted mid-execution) → 404. • Write permission missing → 403. • Airtable API rate limits → 429. • If PostPulse response structure changes and `publications[0].posts[0].content` is undefined, the update will write a null/empty value. |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Airtable Trigger | n8n-nodes-base.airtableTrigger (v1) | Entry point – triggers workflow when a new record with a URL link is created | — | Extract URL Details | Kick-Off workflow — Searches Airtable and uses AI to extract website data and create a social media post with Dumpling AI and Claude respectively. |
| Extract URL Details | n8n-nodes-base.httpRequest (v4.4) | Calls Dumpling AI to extract title and summary from the web URL | Airtable Trigger | Message a model | Kick-Off workflow — Searches Airtable and uses AI to extract website data and create a social media post with Dumpling AI and Claude respectively. |
| Message a model | @n8n/n8n-nodes-langchain.anthropic (v1) | Generates a long-form LinkedIn post using Claude based on the extracted summary | Extract URL Details | Schedule a light post | Kick-Off workflow — Searches Airtable and uses AI to extract website data and create a social media post with Dumpling AI and Claude respectively. |
| Schedule a light post | @postpulse/n8n-nodes-postpulse.postPulse (v1) | Schedules the generated LinkedIn post for publishing via PostPulse | Message a model | Update record | Schedule Post and Updates Airtable — Schedules generated social media post in PostPulse and updates a record in Airtable. |
| Update record | n8n-nodes-base.airtable (v2.2) | Writes the generated post text and status back to the Airtable record | Schedule a light post | — | Schedule Post and Updates Airtable — Schedules generated social media post in PostPulse and updates a record in Airtable. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Website Articles to LinkedIn Post copy".

2. **Add the Airtable Trigger node:**
   - Type: `Airtable Trigger` (`n8n-nodes-base.airtableTrigger`, version 1).
   - Base: Select or enter the Airtable base ID `appV8W8qiCqqy4cIz` (name: *LinkedIn Post from URL*).
   - Table: Select or enter `tblQKHqO4ltrAN0jS` (name: *Table 1*).
   - Trigger Field: `URL link`.
   - Poll Times: Set mode to "everyMinute" (poll every 1 minute).
   - Authentication: Select or create an **Airtable Personal Access Token** credential with read and write access to the base.

3. **Add the Extract URL Details node:**
   - Type: `HTTP Request` (`n8n-nodes-base.httpRequest`, version 4.4).
   - Method: POST.
   - URL: `https://app.dumplingai.com/api/v1/extract`.
   - Enable "Send Body" and "Send Headers".
   - Specify Body as JSON:
     ```
     {
       "url": "{{ $json.fields['URL link'] }}",
       "schema": {
         "title": "Title of the article",
         "description": "website summary with key points"
       }
     }
     ```
   - Headers:
     - `Content-Type`: `application/json`
     - `Authorization`: `Bearer YOUR_TOKEN_HERE` (replace with your Dumpling AI API token).
   - Connect: Airtable Trigger → Extract URL Details.

4. **Add the Message a model (Claude) node:**
   - Type: `Anthropic` (`@n8n/n8n-nodes-langchain.anthropic`, version 1).
   - Model: Select `claude-haiku-4-5-20251001`.
   - Message content (prompt):
     ```
     Create a long form linkedIn post using the provided article summary below. The post should be attention catching, scroll stopping, insightful and detailed.

     Here's the website summary:
     {{ $json.results.description }}

     Note: Only create the response and the output.
     ```
   - Credential: Create or select an **Anthropic API** credential with a valid API key.
   - Connect: Extract URL Details → Message a model.

5. **Add the Schedule a light post (PostPulse) node:**
   - Type: `PostPulse` (`@postpulse/n8n-nodes-postpulse.postPulse`, version 1).
   - Operation: `scheduleLight`.
   - Content: `{{ $json.content[0].text }}`.
   - Scheduled Time: Set to the desired future date/time, e.g. `2026-04-01T00:00:00` (adjust as needed).
   - Social Media Account: `LINKEDIN|1204` (your specific LinkedIn account ID in PostPulse).
   - Credential: Create or select a **PostPulse OAuth2** credential (ensure the OAuth2 flow is completed and the LinkedIn account is connected).
   - Connect: Message a model → Schedule a light post.

6. **Add the Update record (Airtable) node:**
   - Type: `Airtable` (`n8n-nodes-base.airtable`, version 2.2).
   - Operation: `update`.
   - Base: Select *LinkedIn Post from URL* (`appV8W8qiCqqy4cIz`).
   - Table: Select *Table 1* (`tblQKHqO4ltrAN0jS`).
   - Mapping Mode: "Define Below".
   - Matching Column: `id` (use the record ID from the trigger).
   - Columns to update:
     - `id`: `{{ $('Airtable Trigger').item.json.id }}` (used for matching; read-only).
     - `Generated LinkedIn Post`: `{{ $json.publications[0].posts[0].content }}`.
     - `Post Status`: `Scheduled` (select from options: *Scheduled* / *Not Scheduled*).
   - Credential: Use the same **Airtable Personal Access Token** credential as the trigger.
   - Connect: Schedule a light post → Update record.

7. **Add sticky notes (optional, for documentation):**
   - Create a sticky note titled **"Workflow Purpose"** summarising the five steps and noting that the Claude prompt can be refined for a personal "LinkedIn voice".
   - Create a sticky note titled **"Kick-Off workflow"** positioned near nodes 1–3 describing the extraction and generation steps.
   - Create a sticky note titled **"Schedule Post and Updates Airtable"** positioned near nodes 4–5 describing the scheduling and update steps.

8. **Activate the workflow** once all credentials are configured and nodes are connected.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The Claude prompt can be refined to better match a user's personal "LinkedIn voice" — adjust the prompt in the *Message a model* node. | General best practice |
| The scheduled time in the PostPulse node is hard-coded to `2026-04-01T00:00:00`. Update this value or use an expression to set a dynamic date before deployment. | PostPulse node configuration |
| Dumpling AI Bearer token is set directly in the HTTP Request header (`Authorization: Bearer YOUR_TOKEN_HERE`). Consider migrating this to an n8n credential or environment variable for better security. | Extract URL Details node |
| Airtable trigger fires on record creation (when the *URL link* field is populated). Ensure the field is filled at creation time; updates to an existing record may not re-trigger the workflow unless the trigger field is changed. | Airtable Trigger node |
| PostPulse OAuth2 credential must be re-authorised if the connected LinkedIn account is disconnected or the token expires. | PostPulse credential |
| The workflow uses Claude Haiku 4.5 (`claude-haiku-4-5-20251001`). If this model version is deprecated in the future, update the model selection in the Anthropic node. | Anthropic node |