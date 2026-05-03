Curate and send weekly AI newsletters with Tavily, Gemini, and Gmail

https://n8nworkflows.xyz/workflows/curate-and-send-weekly-ai-newsletters-with-tavily--gemini--and-gmail-12714


# Curate and send weekly AI newsletters with Tavily, Gemini, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow curates a weekly AI-themed newsletter by collecting (1) internal company updates from a specified blog domain and (2) external market news via Tavily search, then uses **Gemini** to write a structured newsletter and finally sends it as a styled **HTML email via Gmail**.

**Primary use cases:**
- Automated weekly “what happened this week” emails for a product/company.
- Combining internal release notes/blog posts with industry news for a consistent newsletter format.
- Generating narrative editorial content from raw search results.

### 1.1 Scheduling & Configuration
Runs weekly and defines newsletter variables (topic, brand, URLs, headers).

### 1.2 Research (Parallel Tavily Searches)
Searches external news and the company blog in parallel for the last 7 days.

### 1.3 Aggregation & Normalization
Aggregates both result sets into a single `research_data` array (robust to missing branches).

### 1.4 AI Writing (Gemini via LangChain Agent)
Gemini receives the aggregated search results and produces a **single JSON object** containing `intro_text`, `product_section`, and `news_section` (HTML strings).

### 1.5 Email Rendering & Delivery
Loads an HTML template, injects AI + config variables, then sends the final HTML with Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Newsletter Configuration
**Overview:** Triggers the workflow weekly and sets all reusable configuration fields (topic, branding, URLs, section headers) used downstream in search queries, AI prompts, and the email template.

**Nodes Involved:**
- Weekly schedule trigger
- Set newsletter config

#### Node: Weekly schedule trigger
- **Type / role:** `Schedule Trigger` — entry point; runs on a fixed interval.
- **Configuration choices:**
  - Runs every **7 days** at **09:30** (instance timezone applies).
- **Inputs/Outputs:**
  - No inputs (trigger).
  - Output → `Set newsletter config`.
- **Version notes:** typeVersion **1.3** (standard scheduling node behavior).
- **Edge cases / failures:**
  - Timezone surprises (server vs expected local timezone).
  - Workflow inactive (`active: false` in the provided workflow), so it will not run until enabled.

#### Node: Set newsletter config
- **Type / role:** `Set` — defines constants used across the workflow.
- **Key fields created:**
  - `topic`: “AI Sales Agents”
  - `company_blog_url`: `https://aisalesagenthq.scoot.app/`
  - `logo_url`: `https://aisalesagenthq.scoot.app/content/images/size/w256h256/2025/10/Preciate-AppIcon-256x256.png`
  - `subscribe_url`: `https://aisalesagenthq.scoot.app/`
  - `newsletter_name`: “AI Sales Agent HQ”
  - `author_name`: “The AI Team”
  - `email_title`: “Weekly Update: AI Sales Agents”
  - `header_internal`: “New this Week from AI Sales Agent HQ”
  - `header_external`: “AI News”
- **Expressions/variables used:** none internally; provides values referenced later via expressions like `$('Set newsletter config').item.json.topic`.
- **Inputs/Outputs:**
  - Input ← `Weekly schedule trigger`
  - Outputs → `Search external news (Tavily)` and `Search company blog (Tavily)` (fan-out).
- **Version notes:** typeVersion **3.4**.
- **Edge cases / failures:**
  - Invalid URLs (logo/blog) can yield poor search results or broken email images.
  - Trailing slash/domain matching: later AI logic checks “results from the domain `company_blog_url`”; if the returned URLs don’t exactly match (subdomains, http/https variants), internal/external classification may be imperfect.

---

### Block 2 — Research Phase (Tavily Searches in Parallel)
**Overview:** Uses Tavily to gather two sets of results over the last 7 days: external news about the topic, and internal updates constrained to the company blog domain.

**Nodes Involved:**
- Search external news (Tavily)
- Search company blog (Tavily)
- Extract news results
- Extract blog results
- Combine search results

#### Node: Search external news (Tavily)
- **Type / role:** Tavily search node (`@tavily/n8n-nodes-tavily.tavily`) — finds market news about the configured topic.
- **Configuration choices:**
  - Query: `{{$json.topic}}` (from `Set newsletter config`)
  - Options:
    - `days: 7` (lookback window)
    - `topic: "news"` (Tavily category)
    - `max_results: 5`
    - `search_depth: "advanced"`
- **Credentials:** Tavily API credential required.
- **Inputs/Outputs:**
  - Input ← `Set newsletter config`
  - Output → `Extract news results`
- **Version notes:** typeVersion **1** (Tavily community/partner node version).
- **Edge cases / failures:**
  - Auth errors / quota exhaustion / 429 rate limits.
  - Empty results if topic too narrow or Tavily has limited coverage that week.
  - Network timeouts.

#### Node: Search company blog (Tavily)
- **Type / role:** Tavily search — finds internal posts (company blog) related to “news/update/release…” within last 7 days.
- **Configuration choices:**
  - Query: `site:{company_blog_url} "news" OR "update" OR "launch" OR "release" OR "template" OR "New"`
  - Options:
    - `days: 7`
    - `topic: "general"`
    - `max_results: 5`
    - `search_depth: "advanced"`
  - `alwaysOutputData: true` (important): keeps the branch producing output even if it finds nothing (depending on node behavior).
- **Credentials:** Tavily API credential required.
- **Inputs/Outputs:**
  - Input ← `Set newsletter config`
  - Output → `Extract blog results`
- **Edge cases / failures:**
  - The `site:` operator depends on Tavily’s handling; results may be sparse if pages are not indexed.
  - If blog uses unusual routing or blocks crawlers, results may be missing.
  - Despite `alwaysOutputData`, a hard failure (auth/network) can still prevent usable results.

#### Node: Extract news results
- **Type / role:** `Set` — maps Tavily `results` to a field `news_results`.
- **Configuration choices:**
  - `news_results = {{$json.results}}` stored as **string** type (note: this is likely not ideal because results are an array/object).
- **Inputs/Outputs:**
  - Input ← `Search external news (Tavily)`
  - Output → `Combine search results` (input index 0)
- **Edge cases / failures:**
  - Declaring the field as `string` while assigning an array can cause type coercion or unexpected serialization.
  - This node is effectively bypassed later by `Aggregate all articles` (which reads directly from the Tavily nodes), so issues here usually won’t break final output unless you later refactor to rely on the merge output.

#### Node: Extract blog results
- **Type / role:** `Set` — maps Tavily `results` to `blog_results`.
- **Configuration choices:**
  - `blog_results = {{$json.results}}` stored as **string** type (same caveat as above).
- **Inputs/Outputs:**
  - Input ← `Search company blog (Tavily)`
  - Output → `Combine search results` (input index 1)
- **Edge cases / failures:** same as `Extract news results`.

#### Node: Combine search results
- **Type / role:** `Merge` — attempts to combine items from the two branches.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `position` (combineByPosition)
  - `includeUnpaired: false` (requires both sides to have paired items)
- **Inputs/Outputs:**
  - Inputs ← `Extract news results` (0), `Extract blog results` (1)
  - Output → `Aggregate all articles`
- **Edge cases / failures:**
  - If one side produces fewer items, `includeUnpaired: false` can drop data.
  - In this workflow, the next code node *does not actually rely on the merged payload*; it reads from the original search nodes directly, reducing risk.

---

### Block 3 — Aggregation & Canonical Research Data
**Overview:** Produces a single `research_data` array by concatenating results from both Tavily search nodes, intentionally ignoring the merge/set outputs to ensure clean access to the original results arrays.

**Nodes Involved:**
- Aggregate all articles

#### Node: Aggregate all articles
- **Type / role:** `Code` — consolidates research data.
- **Configuration choices (interpreted):**
  - Uses n8n node-data accessors:
    - `$('Search external news (Tavily)').all()`
    - `$('Search company blog (Tavily)').all()`
  - For each item, if `item.json.results` is an array, concatenates into `allArticles`.
  - Wraps each access in a `try/catch` to tolerate missing branches.
- **Output:**
  - Returns one item: `{ research_data: allArticles }`
- **Inputs/Outputs:**
  - Input ← `Combine search results` (but logically it just ensures ordering; data read is direct)
  - Output → `Load email template`
- **Edge cases / failures:**
  - If Tavily node output schema changes (e.g., `results` renamed), aggregation will silently produce empty `research_data`.
  - If Tavily nodes error hard and produce no execution data, the catch block logs and proceeds, resulting in an empty newsletter.
  - Very large result payloads can inflate prompt size; here capped by `max_results: 5` per search, so manageable.

---

### Block 4 — AI Generation (Gemini + Agent)
**Overview:** Gemini (as the LLM) is attached to a LangChain Agent node that receives all aggregated search results and produces a single JSON object with HTML fragments for the newsletter.

**Nodes Involved:**
- Load email template (feeds agent execution chain)
- Generate newsletter content
- Gemini 1.5 Flash

#### Node: Gemini 1.5 Flash
- **Type / role:** `lmChatGoogleGemini` — language model provider node used by the agent.
- **Configuration choices:**
  - Default options (none explicitly set).
- **Credentials:** Google Gemini/PaLM API credential required.
- **Connections:**
  - Connected to `Generate newsletter content` via `ai_languageModel`.
- **Edge cases / failures:**
  - Invalid API key / project restrictions.
  - Safety filters may refuse some content depending on prompts/news content.
  - Token/context limits: mitigated by small result count but still possible if Tavily returns long snippets.

#### Node: Generate newsletter content
- **Type / role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt + model to produce structured output.
- **Configuration choices:**
  - **User prompt** includes:
    - `Raw Search Data: {{ JSON.stringify($('Aggregate all articles').first().json.research_data) }}`
    - Task: classify results from the company domain into internal updates; all others as market news.
  - **System message** enforces strict output:
    - “Return a SINGLE valid JSON object. Do not wrap in markdown.”
    - JSON keys: `intro_text`, `product_section`, `news_section`
    - HTML formatting requirements for each section.
  - Prompt type: `define` (explicit prompt fields).
- **Inputs/Outputs:**
  - Main input ← `Load email template` (execution dependency; content is not directly used here)
  - Reads research data from `Aggregate all articles` via expression.
  - Output → `Build final email` with field `output` (agent output).
- **Edge cases / failures:**
  - Model may still output invalid JSON (extra commentary, markdown fences). Downstream code attempts to clean/parse.
  - Domain matching: the instruction uses `company_blog_url` string; model may misclassify if URLs vary.
  - If `research_data` is empty, model should produce intro + external news; but might produce minimal content.

#### Node: Load email template
- **Type / role:** `Code` — provides a reusable HTML email template with placeholders.
- **Configuration choices:**
  - Returns `{ html_template: template }` where template includes placeholders:
    - `{{LOGO_URL}}`, `{{NEWSLETTER_NAME}}`, `{{EMAIL_TITLE}}`, `{{AUTHOR}}`, `{{DATE}}`
    - `{{INTRO_TEXT}}`, `{{PRODUCT_SECTION}}`, `{{NEWS_SECTION}}`, `{{SUBSCRIBE_URL}}`
- **Inputs/Outputs:**
  - Input ← `Aggregate all articles`
  - Output → `Generate newsletter content`
- **Edge cases / failures:**
  - Placeholders must match exactly what `Build final email` replaces.
  - Email client compatibility: CSS is embedded; most clients are OK, but some (notably Outlook desktop) can be inconsistent with modern CSS.

---

### Block 5 — Email Rendering & Delivery
**Overview:** Parses and sanitizes the AI output, injects it into the HTML template along with config values and the current date, then sends via Gmail.

**Nodes Involved:**
- Build final email
- Send newsletter (Gmail)

#### Node: Build final email
- **Type / role:** `Code` — transforms template + config + AI JSON into final `html_email`.
- **Configuration choices (interpreted):**
  1. Reads:
     - Template from `Load email template`
     - Config from `Set newsletter config`
     - AI raw output from `Generate newsletter content` → `.json.output`
  2. Cleans AI output if it is a string:
     - Removes ```json fences and ``` blocks.
  3. Attempts JSON parse:
     - On parse failure, falls back to:
       - `intro_text = "<p>Here is your weekly update.</p>"`
       - `product_section = ""`
       - `news_section = aiRaw` (raw string)
  4. Generates date like “January 25, 2026” using `en-US` locale.
  5. Replaces placeholders:
     - Uses global replace for `{{NEWSLETTER_NAME}}` (`/g`) because it appears in header and footer.
- **Output:** `{ html_email: finalEmail }`
- **Inputs/Outputs:**
  - Input ← `Generate newsletter content`
  - Output → `Send newsletter (Gmail)`
- **Edge cases / failures:**
  - If AI returns JSON but missing keys, replacements default to empty string.
  - If AI returns HTML with unsafe tags, this workflow does not sanitize; Gmail will still send, but recipients’ clients may strip content.
  - If template placeholders are altered, replacements may fail silently (resulting in unreplaced tokens in emails).

#### Node: Send newsletter (Gmail)
- **Type / role:** `Gmail` — sends the final HTML email.
- **Configuration choices:**
  - To: `user@example.com` (must be replaced with real recipients / list strategy)
  - Subject: `Weekly AI Update: {{ topic }}`
  - Message body: `{{$json.html_email}}` (HTML)
- **Credentials:** Gmail OAuth2 credential (“Daniel Gmail account” in the workflow).
- **Inputs/Outputs:**
  - Input ← `Build final email`
  - No downstream outputs.
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:**
  - OAuth token expiration / revoked consent.
  - Gmail sending limits (daily quotas) if used for a list.
  - Deliverability: sending to many recipients should use proper list management (BCC, dedicated ESP, unsubscribe handling). Current template includes an “Unsubscribe” link placeholder `href="#"` which is not functional.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly schedule trigger | n8n-nodes-base.scheduleTrigger | Weekly entry trigger | — | Set newsletter config | ## How it works<br>This workflow automatically curates and sends a weekly AI newsletter by combining internal blog posts with external news.<br><br>1. **Trigger** - Runs weekly on a schedule (configurable)<br>2. **Research** - Tavily searches your company blog and external news for your topic<br>3. **AI Writing** - Gemini generates a professional newsletter with intro, internal updates, and market news<br>4. **Send** - Gmail delivers the formatted HTML email to your subscribers<br><br>## Setup steps<br>1. Open **Set newsletter config** node to customize: topic, newsletter name, logo URL, blog URL<br>2. Add your **Tavily API** credentials (get key at tavily.com)<br>3. Add your **Google Gemini API** credentials<br>4. Add your **Gmail OAuth** credentials<br>5. Update the recipient email in **Send newsletter** node<br>6. Test with manual execution before enabling the schedule |
| Set newsletter config | n8n-nodes-base.set | Central config/constants | Weekly schedule trigger | Search external news (Tavily); Search company blog (Tavily) | ## How it works<br>…(same as above)… |
| Search external news (Tavily) | @tavily/n8n-nodes-tavily.tavily | External research search | Set newsletter config | Extract news results | **Research Phase**<br>Searches your blog and external news sources in parallel using Tavily |
| Search company blog (Tavily) | @tavily/n8n-nodes-tavily.tavily | Internal blog search | Set newsletter config | Extract blog results | **Research Phase**<br>Searches your blog and external news sources in parallel using Tavily |
| Extract news results | n8n-nodes-base.set | Maps Tavily results (news) | Search external news (Tavily) | Combine search results | **Research Phase**<br>Searches your blog and external news sources in parallel using Tavily |
| Extract blog results | n8n-nodes-base.set | Maps Tavily results (blog) | Search company blog (Tavily) | Combine search results | **Research Phase**<br>Searches your blog and external news sources in parallel using Tavily |
| Combine search results | n8n-nodes-base.merge | Attempts to combine both branches | Extract news results; Extract blog results | Aggregate all articles | **Research Phase**<br>Searches your blog and external news sources in parallel using Tavily |
| Aggregate all articles | n8n-nodes-base.code | Canonical aggregation into `research_data` | Combine search results | Load email template |  |
| Load email template | n8n-nodes-base.code | Provides HTML template | Aggregate all articles | Generate newsletter content |  |
| Generate newsletter content | @n8n/n8n-nodes-langchain.agent | Produces structured JSON newsletter sections | Load email template | Build final email | **AI Generation**<br>Gemini writes the newsletter content in structured HTML format |
| Gemini 1.5 Flash | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for agent | — | Generate newsletter content (ai_languageModel) | **AI Generation**<br>Gemini writes the newsletter content in structured HTML format |
| Build final email | n8n-nodes-base.code | Merges template + config + AI output | Generate newsletter content | Send newsletter (Gmail) | **Email Output**<br>Merges AI content with HTML template and sends via Gmail |
| Send newsletter (Gmail) | n8n-nodes-base.gmail | Sends final HTML email | Build final email | — | **Email Output**<br>Merges AI content with HTML template and sends via Gmail |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment | — | — | (This node is itself the comment content shown in its row) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | (This node is itself the comment content shown in its row) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | (This node is itself the comment content shown in its row) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | (This node is itself the comment content shown in its row) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it (example): “Curate and send weekly AI newsletters with Tavily, Gemini, and Gmail”.
   - Keep it inactive until testing is complete.

2. **Add `Schedule Trigger`**
   - Node: **Schedule Trigger**
   - Configure: Interval every **7 days**, time **09:30** (adjust timezone if needed).
   - Connect → next node.

3. **Add `Set` node for configuration**
   - Node: **Set**
   - Add fields (String):
     - `topic` (e.g., “AI Sales Agents”)
     - `company_blog_url` (e.g., `https://aisalesagenthq.scoot.app/`)
     - `logo_url` (public image URL)
     - `subscribe_url`
     - `newsletter_name`
     - `author_name`
     - `email_title`
     - `header_internal`
     - `header_external`
   - Connect from trigger → this node.

4. **Add Tavily external search**
   - Node: **Tavily**
   - Operation: Search (default behavior of the Tavily node)
   - Query expression: `{{$json.topic}}`
   - Options:
     - Days: `7`
     - Topic: `news`
     - Max results: `5`
     - Search depth: `advanced`
   - **Credentials:** add Tavily API key (get one at **https://tavily.com**).
   - Connect from `Set newsletter config` → this node.

5. **Add Tavily company blog search**
   - Node: **Tavily**
   - Query expression (example):  
     `site:{{ $('Set newsletter config').item.json.company_blog_url }} "news" OR "update" OR "launch" OR "release" OR "template" OR "New"`
   - Options:
     - Days: `7`
     - Topic: `general`
     - Max results: `5`
     - Search depth: `advanced`
   - Enable **Always Output Data** (so the branch continues even if no results).
   - Use the same Tavily credentials.
   - Connect from `Set newsletter config` → this node.

6. **Add (optional but present) Set nodes to extract results**
   - Node: **Set** after external Tavily:
     - Field: `news_results = {{$json.results}}`
   - Node: **Set** after blog Tavily:
     - Field: `blog_results = {{$json.results}}`
   - Connect each Tavily node → its extractor Set node.

7. **Add `Merge` node**
   - Node: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Include unpaired: **false**
   - Connect:
     - External extractor Set → Merge input 0
     - Blog extractor Set → Merge input 1

8. **Add `Code` node: Aggregate all articles**
   - Node: **Code**
   - Implement logic to concatenate:
     - `$('Search external news (Tavily)').all()` → collect `item.json.results`
     - `$('Search company blog (Tavily)').all()` → collect `item.json.results`
   - Output one item: `{ research_data: allArticles }`
   - Connect Merge → this Code node.

9. **Add `Code` node: Load email template**
   - Node: **Code**
   - Return `{ html_template: "<!doctype html> ... {{INTRO_TEXT}} ... {{NEWS_SECTION}} ..." }`
   - Include placeholders at minimum:
     - `{{LOGO_URL}}`, `{{SUBSCRIBE_URL}}`, `{{NEWSLETTER_NAME}}`, `{{EMAIL_TITLE}}`, `{{AUTHOR}}`, `{{DATE}}`
     - `{{INTRO_TEXT}}`, `{{PRODUCT_SECTION}}`, `{{NEWS_SECTION}}`
   - Connect `Aggregate all articles` → `Load email template`.

10. **Add Gemini LLM node**
   - Node: **Google Gemini Chat Model** (Gemini / PaLM node in n8n: `lmChatGoogleGemini`)
   - **Credentials:** configure Google Gemini/PaLM API key/project in n8n credentials.
   - Keep default options unless you need temperature/model overrides.

11. **Add `AI Agent` node to generate newsletter**
   - Node: **AI Agent** (LangChain Agent)
   - Set **System message** to require “single valid JSON” and define:
     - `intro_text` as HTML paragraphs
     - `product_section` as HTML starting with internal header, empty string if none
     - `news_section` as HTML starting with external header and 3–4 stories as `<ul><li>...`
   - In the **User prompt**, inject aggregated data:  
     `Raw Search Data: {{ JSON.stringify($('Aggregate all articles').first().json.research_data) }}`
   - Also instruct it to treat results from `company_blog_url` as internal.
   - Connect:
     - Main: `Load email template` → `Generate newsletter content`
     - AI language model: `Gemini 1.5 Flash` → Agent (ai_languageModel connection)

12. **Add `Code` node: Build final email**
   - Node: **Code**
   - Read:
     - Template from `Load email template`
     - Config from `Set newsletter config`
     - Agent output from `Generate newsletter content` (typically `.json.output`)
   - Clean markdown fences if present and `JSON.parse` it.
   - Replace placeholders and output `{ html_email: finalEmail }`.
   - Connect Agent → this node.

13. **Add `Gmail` node to send**
   - Node: **Gmail**
   - Operation: Send Email
   - To: set your recipient(s) (replace `user@example.com`)
   - Subject expression: `Weekly AI Update: {{ $('Set newsletter config').item.json.topic }}`
   - Message/body expression: `{{$json.html_email}}`
   - **Credentials:** configure Gmail OAuth2 in n8n and select it here.
   - Connect `Build final email` → `Send newsletter (Gmail)`.

14. **Test and enable**
   - Run manually first and inspect:
     - Tavily outputs (`results`)
     - `Aggregate all articles.research_data`
     - Agent `output` validity (JSON)
     - Rendered HTML in `Build final email`
   - When satisfied, activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Tavily API credentials required; obtain key at tavily.com | https://tavily.com |
| Company blog URL used for internal vs external classification | https://aisalesagenthq.scoot.app/ |
| Logo URL appears in the email header image | https://aisalesagenthq.scoot.app/content/images/size/w256h256/2025/10/Preciate-AppIcon-256x256.png |
| “Unsubscribe” link in the template is a placeholder (`href="#"`) and is not functional | Consider integrating a real list provider or a subscription system |
| Workflow is currently inactive (`active: false`) | Must be enabled for schedule to run |