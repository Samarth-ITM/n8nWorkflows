Generate product feature announcements from Notion to Google Docs with GPT-5 Mini and Claude

https://n8nworkflows.xyz/workflows/generate-product-feature-announcements-from-notion-to-google-docs-with-gpt-5-mini-and-claude-12535


# Generate product feature announcements from Notion to Google Docs with GPT-5 Mini and Claude

## 1. Workflow Overview

**Purpose:** This workflow watches a Notion “Content Plan” database for entries marked ready, generates an SEO/AI-search-friendly outline with **GPT‑5 Mini**, writes a full product announcement article with **Claude Sonnet 4.5**, creates a **Google Doc** containing the generated Markdown (with YAML front matter), then writes the Google Docs link back to Notion and marks the item as completed.

**Primary use case:** Automating product feature announcement drafts (blog-ready) from internal Notion planning into Google Docs, with status gating to avoid accidental generation.

### 1.1 Input Reception & Gating (Notion → Filters)
Poll Notion hourly, only proceed if the page has `Status = n8n_ready` and `Category = Product`.

### 1.2 Processing State Update (Notion status lock)
As soon as a valid item is detected, update its Notion status to “running” to reduce double-processing.

### 1.3 AI Outline Generation (GPT‑5 Mini)
Use Notion fields to build a structured outline optimized for SEO + “AI search / LLM visibility”.

### 1.4 AI Draft Generation (Claude Sonnet 4.5)
Use GPT output + Notion fields to produce a full Markdown article that starts with YAML front matter.

### 1.5 Document Creation & Publishing Loopback (Google Docs → Notion)
Create a Google Doc titled from Notion, insert the generated text, write the Doc URL back into Notion, then set status to completed.

---

## 2. Block-by-Block Analysis

### Block 1 — Notion polling & eligibility checks
**Overview:** Detects updated pages in the target Notion database and prevents execution unless the entry is explicitly approved for generation (status + category checks).

**Nodes involved:** `Notion Trigger`, `If_n8n_ready`, `If_Product`

#### Node: Notion Trigger
- **Type / role:** Notion Trigger (`n8n-nodes-base.notionTrigger`) — polling trigger for database page updates.
- **Configuration (interpreted):**
  - Event: *page updated in database*.
  - Polling schedule: **every hour**.
  - Database: selected from list (cached name: **Content Plan**).
- **Key variables used:** Outputs the updated database page fields in `$json` (e.g., `$json.Status`, `$json.Category`, `$json.id`, `$json['Project name']`, `$json.Notes`).
- **Connections:**
  - **Output →** `If_n8n_ready`
- **Version requirements:** TypeVersion **1**.
- **Edge cases / failures:**
  - Notion auth/token expiration; database permissions missing.
  - Trigger returns pages updated for reasons unrelated to content readiness; filtering relies on subsequent IF nodes.
  - If Notion property names differ (e.g., “Project name” renamed), downstream expressions will fail.

#### Node: If_n8n_ready
- **Type / role:** IF (`n8n-nodes-base.if`) — status gate.
- **Configuration (interpreted):**
  - Condition: `{{$json.Status}}` **equals** `"n8n_ready"`.
  - Strict type validation enabled (n8n IF v2 conditions).
- **Connections:**
  - **True →** `If_Product`
  - **False →** (no downstream; workflow effectively stops)
- **Version requirements:** TypeVersion **2.2** (newer condition builder).
- **Edge cases / failures:**
  - If `Status` is a Notion **status** type, Notion may output an object rather than a plain string depending on node configuration/mapping; this expression assumes a string.
  - If `Status` is missing/null, strict validation may cause the condition to evaluate unexpectedly (usually false).

#### Node: If_Product
- **Type / role:** IF — category gate.
- **Configuration (interpreted):**
  - Condition: `{{ $('Notion Trigger').item.json.Category[0] }}` **equals** `"Product"`.
  - Assumes `Category` is an array-like field and the first entry is the label.
- **Connections:**
  - **True →** `Update a database page2`
  - **False →** (no downstream; workflow stops)
- **Version requirements:** TypeVersion **2.2**.
- **Edge cases / failures:**
  - If `Category` is a multi-select, array shape might be objects (`[{name:...}]`) rather than strings; `Category[0]` may be an object, causing comparison to fail.
  - If `Category` is empty, `Category[0]` is undefined.

---

### Block 2 — Mark item as running (lock)
**Overview:** Prevents duplicate processing by updating the Notion page status as soon as it passes gating.

**Nodes involved:** `Update a database page2`

#### Node: Update a database page2
- **Type / role:** Notion (`n8n-nodes-base.notion`) — update database page properties.
- **Configuration (interpreted):**
  - Resource: **Database Page**
  - Operation: **Update**
  - Page ID: `{{ $('Notion Trigger').item.json.id }}`
  - Property update: `Status` (status-type) set to **`n8n_runing`** (note spelling).
- **Connections:**
  - **Input ←** `If_Product` (true path)
  - **Output →** `Outline - GPT-5`
- **Version requirements:** TypeVersion **2.2**.
- **Edge cases / failures:**
  - Status option must exist in the Notion database (exact spelling). Here it uses **`n8n_runing`** (missing second “n” in “running”)—if Notion doesn’t have this exact status value, the update fails.
  - Permission issues updating the page.
  - Race condition: multiple workflow executions can still occur if multiple updates happen within the polling window (status update mitigates but does not fully eliminate in all cases).

---

### Block 3 — Outline generation with GPT‑5 Mini
**Overview:** Builds an announcement outline from Notion’s “Project name” and “Notes”, guided by a structured system prompt optimized for scannable product announcements + SEO/AI search.

**Nodes involved:** `Outline - GPT-5`

#### Node: Outline - GPT-5
- **Type / role:** OpenAI (LangChain) (`@n8n/n8n-nodes-langchain.openAi`) — LLM call to generate an outline.
- **Configuration (interpreted):**
  - Model: **gpt-5-mini**
  - Messages:
    - **System prompt:** long instruction set defining tone, structure, and output expectations for product announcements (headline/summary, feature blocks, benefits, FAQ, CTA, optional metadata).
    - **User message:** provides:
      - `Topic: {{ $('Notion Trigger').item.json['Project name'] }}`
      - `Notes: {{ $('Notion Trigger').item.json.Notes }}`
      - Task: produce SEO + AI-search friendly outline.
  - Output simplification: **disabled** (`simplify: false`) → downstream must read provider-native structure.
- **Connections:**
  - **Input ←** `Update a database page2`
  - **Output →** `Claude_text_writer`
- **Version requirements:** TypeVersion **1.8** (node package-specific).
- **Key expressions / variables:**
  - Pulls Notion fields via `$('Notion Trigger').item.json[...]`.
- **Edge cases / failures:**
  - OpenAI credential/auth/quota issues.
  - Because `simplify` is off, the outline is referenced later as `{{$json.choices[0].message.content}}`; if the OpenAI node returns a different schema (model/provider change), Claude step may break.
  - Notes field may be empty; prompt instructs model to infer and continue.

---

### Block 4 — Full draft generation with Claude
**Overview:** Produces the complete Markdown article (starting with YAML front matter) based on GPT outline + Notion metadata and the writing system instructions.

**Nodes involved:** `Claude_text_writer`

#### Node: Claude_text_writer
- **Type / role:** Anthropic (LangChain) (`@n8n/n8n-nodes-langchain.anthropic`) — LLM call to write final article.
- **Configuration (interpreted):**
  - Model: **claude-sonnet-4-5-20250929**
  - Max tokens: **64,000**
  - Messages:
    - Assistant message contains extensive “Role & Objective / Guidelines / Template / Agent behavior” for product announcements.
    - Second message (user-style content, but stored as `content` with expressions) supplies:
      - **Outline:** `{{ $json.choices[0].message.content }}` (from GPT step)
      - **Topic:** `{{ $('Notion Trigger').item.json['Project name'] }}`
      - **Content Type:** `{{ $('Notion Trigger').item.json.Notes }}`
      - Requires Markdown output with **YAML front matter**.
  - Output simplification: **disabled**.
- **Connections:**
  - **Input ←** `Outline - GPT-5`
  - **Output →** `Create a document`
- **Version requirements:** TypeVersion **1**.
- **Key expressions / variables:**
  - Reads GPT outline from `$json.choices[0].message.content`.
  - Reads Notion properties from `$('Notion Trigger').item.json`.
- **Edge cases / failures:**
  - Anthropic credential/quota issues or large output truncation if provider enforces stricter limits.
  - Conflicting instructions: the “assistant” message defines one structure, and the second message demands YAML front matter for a “blog article”; model generally complies but output may mix formats.
  - Downstream Google Docs insertion expects `$('Claude_text_writer').item.json.content[0].text`; if Anthropic node output format differs, insertion fails.

---

### Block 5 — Google Docs creation, insertion, and Notion backfill
**Overview:** Creates a new Google Doc, inserts the generated Markdown content, then writes the doc link back into Notion and marks the Notion item completed.

**Nodes involved:** `Create a document`, `Update a document`, `Update a database page`, `Update a database page1`

#### Node: Create a document
- **Type / role:** Google Docs (`n8n-nodes-base.googleDocs`) — creates a new document.
- **Configuration (interpreted):**
  - Operation: **Create**
  - Title: `{{ $('Notion Trigger').item.json['Project name'] }}`
  - Folder: not set (empty `folderId`) → document lands in default Drive location for the authenticated account.
- **Connections:**
  - **Input ←** `Claude_text_writer`
  - **Output →** `Update a document`
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - OAuth scope/consent issues; missing Drive/Docs permissions.
  - If `Project name` is empty, doc title may be blank or defaulted.

#### Node: Update a document
- **Type / role:** Google Docs — inserts content into the created document.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Document URL/ID field: `{{ $json.id }}` (from “Create a document” output; despite the label “documentURL”, it’s being used as the doc ID).
  - Action: **Insert** text
  - Insert text: `{{ $('Claude_text_writer').item.json.content[0].text }}`
- **Connections:**
  - **Input ←** `Create a document`
  - **Output →** `Update a database page`
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - If Google Docs node truly requires a full URL in this field, providing only an ID may fail depending on node behavior/version.
  - Large text insertion can hit Google API limits/timeouts.
  - If Claude output path is different (e.g., `text` stored elsewhere), the inserted content will be blank or expression will error.

#### Node: Update a database page
- **Type / role:** Notion — writes the Google Doc link to the Notion page.
- **Configuration (interpreted):**
  - Resource: **Database Page**, Operation: **Update**
  - Page ID: `{{ $('Notion Trigger').item.json.id }}`
  - Property update: `Google_Docs_Link` (URL property) set to:
    - `https://docs.google.com/document/d/{{ $json.id }}`
    - Here `$json.id` refers to the document id coming from the prior Google Docs node.
- **Connections:**
  - **Input ←** `Update a document`
  - **Output →** `Update a database page1`
- **Version requirements:** TypeVersion **2.2**.
- **Edge cases / failures:**
  - Notion property key must exactly match `Google_Docs_Link` and be URL type.
  - If `$json.id` is not present (upstream schema change), link becomes malformed.

#### Node: Update a database page1
- **Type / role:** Notion — marks completion.
- **Configuration (interpreted):**
  - Resource: **Database Page**, Operation: **Update**
  - Page ID: `{{ $('Notion Trigger').item.json.id }}`
  - Property update (as configured): key shown as `=Status|status` and value as `=n8n_completed`
    - This appears to be incorrectly prefixed with `=` in both key and value.
- **Connections:**
  - **Input ←** `Update a database page`
  - **Output:** none (end)
- **Version requirements:** TypeVersion **2.2**.
- **Edge cases / failures:**
  - Likely misconfiguration: the property key should be `Status|status` (without `=`) and status value should be `n8n_completed` (without `=`). As-is, Notion update may fail or silently do nothing depending on n8n validation.
  - Status option `n8n_completed` must exist in Notion.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Notion Trigger | notionTrigger | Poll Notion database for updated pages | — | If_n8n_ready | # Automated Product Blog Article Generation: Notion → AI → Google Docs… (Monitors Content Plan hourly; generates outline + article; creates GDocs; updates Notion; requirements + customization tips) |
| If_n8n_ready | if | Gate execution to only items with Status = n8n_ready | Notion Trigger | If_Product | ## Step 1: Monitor Notion, Check Status Verify Content type… (checks Status then Category; prevents unapproved runs; update status to prevent double processing) |
| If_Product | if | Gate execution to only Category = Product | If_n8n_ready (true) | Update a database page2 | ## Step 1: Monitor Notion, Check Status Verify Content type… |
| Update a database page2 | notion | Set Notion status to running/processing | If_Product (true) | Outline - GPT-5 | ## Step 1: Monitor Notion, Check Status Verify Content type… |
| Outline - GPT-5 | @n8n/n8n-nodes-langchain.openAi | Generate SEO/AI-search-friendly outline | Update a database page2 | Claude_text_writer | ## Step 2. Create outline and final text… (add product specs to prompts; GPT makes outline from Project name + Notes) |
| Claude_text_writer | @n8n/n8n-nodes-langchain.anthropic | Generate full Markdown article with YAML front matter | Outline - GPT-5 | Create a document | ## Step 2. Create outline and final text… (Claude generates full article from outline + Notion details) |
| Create a document | googleDocs | Create Google Doc for the article | Claude_text_writer | Update a document | ## Step 3 Save text and report back to your Notion… (creates doc titled from Project name; folder configurable) |
| Update a document | googleDocs | Insert generated text into the Google Doc | Create a document | Update a database page |  |
| Update a database page | notion | Write Google Docs URL back to Notion | Update a document | Update a database page1 | ## Step 3 Save text and report back to your Notion… (adds Google_Docs_Link url for one-click access) |
| Update a database page1 | notion | Mark Notion item as completed | Update a database page | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add trigger: “Notion Trigger”**
   - Node: **Notion Trigger**
   - Event: **Paged Updated in Database**
   - Poll interval: **Every hour**
   - Database: select your **Content Plan** database
   - Credentials: connect **Notion API** integration with access to that database.
3. **Add IF node: “If_n8n_ready”**
   - Condition: `Status` equals `n8n_ready`
   - Connect: `Notion Trigger → If_n8n_ready (main)`
4. **Add IF node: “If_Product”**
   - Condition: Category equals `Product`
   - Expression used in workflow: `{{ $('Notion Trigger').item.json.Category[0] }}`
   - Connect: `If_n8n_ready (true) → If_Product`
5. **Add Notion node: “Update a database page2” (set running)**
   - Resource: **Database Page**
   - Operation: **Update**
   - Page ID: `{{ $('Notion Trigger').item.json.id }}`
   - Properties: set **Status** to `n8n_runing` (or correct to your preferred `n8n_running`, but ensure the Notion status option exists)
   - Connect: `If_Product (true) → Update a database page2`
6. **Add OpenAI node: “Outline - GPT-5”**
   - Node: **OpenAI (LangChain)**
   - Model: **gpt-5-mini**
   - Messages:
     - System: your product announcement structure + brand voice + SEO/AI-search guidance
     - User: include Notion fields:
       - Topic: `{{ $('Notion Trigger').item.json['Project name'] }}`
       - Notes: `{{ $('Notion Trigger').item.json.Notes }}`
   - Keep **Simplify OFF** if you want to reuse `choices[0].message.content` downstream exactly as in this workflow.
   - Credentials: **OpenAI API**
   - Connect: `Update a database page2 → Outline - GPT-5`
7. **Add Anthropic node: “Claude_text_writer”**
   - Node: **Anthropic (LangChain)**
   - Model: **Claude Sonnet 4.5** (select your available Sonnet model)
   - Max tokens: set high enough for full article (workflow uses **64000**)
   - Messages:
     - Instruction message defining writing rules (tone, structure, “never error” behavior)
     - Content message that embeds:
       - Outline: `{{ $json.choices[0].message.content }}`
       - Topic: `{{ $('Notion Trigger').item.json['Project name'] }}`
       - Content type: `{{ $('Notion Trigger').item.json.Notes }}`
     - Require Markdown output with YAML front matter.
   - Credentials: **Anthropic API**
   - Connect: `Outline - GPT-5 → Claude_text_writer`
8. **Add Google Docs node: “Create a document”**
   - Operation: **Create**
   - Title: `{{ $('Notion Trigger').item.json['Project name'] }}`
   - Folder: pick a Drive folder (optional but recommended)
   - Credentials: **Google Docs OAuth2**
   - Connect: `Claude_text_writer → Create a document`
9. **Add Google Docs node: “Update a document”**
   - Operation: **Update**
   - Document identifier: use the created document’s ID (this workflow uses `{{ $json.id }}`)
   - Action: **Insert** text
   - Text: `{{ $('Claude_text_writer').item.json.content[0].text }}`
   - Connect: `Create a document → Update a document`
10. **Add Notion node: “Update a database page” (write doc link)**
    - Resource: **Database Page**, Operation: **Update**
    - Page ID: `{{ $('Notion Trigger').item.json.id }}`
    - Property: `Google_Docs_Link` (URL) set to `https://docs.google.com/document/d/{{ $json.id }}`
    - Connect: `Update a document → Update a database page`
11. **Add Notion node: “Update a database page1” (mark completed)**
    - Resource: **Database Page**, Operation: **Update**
    - Page ID: `{{ $('Notion Trigger').item.json.id }}`
    - Property: `Status` set to `n8n_completed`
    - (Ensure you do **not** include the leading `=` in the property key/value.)
    - Connect: `Update a database page → Update a database page1`
12. **Validate required Notion properties**
    - Database must contain at least: **Status**, **Project name**, **Notes**, **Category**, **Google_Docs_Link**
    - Status options should include: `n8n_ready`, `n8n_runing` (or your corrected value), `n8n_completed`.
13. **Activate the workflow**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Automated Product Blog Article Generation: Notion → AI → Google Docs” | Workflow sticky note summary: monitors Notion Content Plan hourly; generates outline + article; creates Google Docs; updates Notion with links and status; lists requirements and customization tips. |
| Requirements listed in sticky note | Notion database fields: Status, Project name, Notes, Category, Google_Docs_Link; Google Drive folder; Anthropic API (Claude Sonnet 4.5); OpenAI API (GPT‑5 Mini). |
| Customization guidance | Update system prompts with your brand/product specifics; adjust trigger conditions to match your Notion properties; optionally connect to a product backlog to draft announcements automatically. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.