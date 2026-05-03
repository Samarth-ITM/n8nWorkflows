Extract high-intent SEO keywords with Decodo from web pages with AI + SERP data

https://n8nworkflows.xyz/workflows/extract-high-intent-seo-keywords-with-decodo-from-web-pages-with-ai---serp-data-11491


# Extract high-intent SEO keywords with Decodo from web pages with AI + SERP data

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow performs automated SEO auditing (“SEO Watchdog”) on a list of URLs. It reads URLs from Google Sheets, scrapes each page via **Decodo**, extracts key SEO signals from raw HTML using JavaScript, sends the compact extract to an **AI Agent** (OpenAI model) to produce an **executive-friendly SEO summary in JSON**, then saves results to another Google Sheet and emails an HTML report via Gmail.

**Target use cases:**
- Periodic SEO monitoring of landing pages or key product pages
- Automated executive reports for stakeholders (non-technical)
- Lightweight on-page SEO diagnostics (title/meta/canonical/headings + text excerpt)

### 1.1 Entry & URL Intake
Manual execution starts the workflow, then Google Sheets provides the list of URLs to process.

### 1.2 Iteration / Batching
URLs are processed item-by-item through a batching loop to avoid overloading scraping/LLM calls.

### 1.3 Web Scraping (HTML retrieval)
Each URL is fetched using Decodo to reliably extract HTML content.

### 1.4 SEO Signal Extraction (HTML → compact text)
A JavaScript Code node reduces HTML into structured SEO fields and a compact text payload for the LLM.

### 1.5 AI Executive SEO Analysis (LLM)
The AI Agent consumes the compact payload and returns **only JSON** describing status, issues, quick wins, and recommendations.

### 1.6 Output Normalization (Parse/Repair JSON)
A JavaScript Code node removes code fences, extracts the first JSON object, and attempts simple repairs before parsing.

### 1.7 Reporting Outputs (Email + Sheets)
The workflow sends an HTML email (Gmail) and appends the structured results into an output Google Sheet, then continues the loop.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Input Reception
**Overview:** Starts manually and loads the URL list from a Google Sheet.  
**Nodes involved:**  
- When clicking ‘Execute workflow’
- Get row(s) in sheet

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger — entry point for ad-hoc runs.
- **Configuration:** No parameters.
- **Connections:** Outputs to **Get row(s) in sheet**.
- **Failure modes / edge cases:** None (only requires manual execution).
- **Version notes:** `typeVersion: 1` standard.

#### Node: Get row(s) in sheet
- **Type / role:** Google Sheets node — reads input URLs.
- **Configuration (interpreted):**
  - Operation: *Get row(s)* (read rows from a sheet).
  - Document: Spreadsheet named (cached) `urls_to_scrape` (ID: `1mIVy...`).
  - Sheet: `Hoja 1` (gid=0).
  - Range handling: uses “specifyRange” mode (range not explicitly visible in the JSON; relies on node UI settings).
- **Key fields/variables:**
  - Downstream expression expects a column named **`urls`**: `$('Get row(s) in sheet').item.json.urls`
- **Connections:**
  - In: Manual Trigger
  - Out: Loop Over Items
- **Failure modes / edge cases:**
  - OAuth permission issues / expired token
  - Sheet structure mismatch (missing `urls` column)
  - Empty result set → loop receives zero items
- **Version notes:** `typeVersion: 4.7` (Google Sheets node behavior may differ vs older versions for ranges and outputs).

---

### Block 2 — Looping / Batch Processing
**Overview:** Iterates through rows so each URL is scraped and analyzed individually.  
**Nodes involved:**  
- Loop Over Items (Split in Batches)

#### Node: Loop Over Items
- **Type / role:** SplitInBatches — controls iteration.
- **Configuration:**
  - Batch options are default (batch size not explicitly set in JSON; n8n defaults apply unless set in UI).
- **Connections (important detail):**
  - In: Get row(s) in sheet; also receives a “continue” signal from Append row in sheet
  - Out:
    - **Output index 1** → Decodo (this is the “current batch/items” output)
    - Output index 0 is unused in this workflow
- **Failure modes / edge cases:**
  - If batch size is too large, Decodo/LLM rate limits may be hit.
  - If downstream nodes error, the loop will stop mid-run unless error handling is added.
- **Version notes:** `typeVersion: 3`.

---

### Block 3 — Content Extraction (Scraping)
**Overview:** Fetches HTML for each URL using Decodo.  
**Nodes involved:**  
- Decodo

#### Node: Decodo
- **Type / role:** Decodo scraper — retrieves page content.
- **Configuration:**
  - URL: `={{ $('Get row(s) in sheet').item.json.urls }}`
    - This pulls the current row’s `urls` field (assuming SplitInBatches is iterating items, but note the expression references the Google Sheets node directly).
- **Connections:**
  - In: Loop Over Items (output index 1)
  - Out: Code in JavaScript1
- **Failure modes / edge cases:**
  - Invalid URL / missing scheme (http/https)
  - Decodo auth/credit issues
  - Target site blocks scraping or returns bot challenge
  - Timeouts / large pages
- **Version notes:** Decodo community node `@decodo/n8n-nodes-decodo.decodo` `typeVersion: 1`.

**Sticky note context (applies to this block):**
- “### Content Extraction  
  URLs are read from Google Sheets. Each page is scraped using Decodo to reliably fetch the HTML content.”

---

### Block 4 — HTML → SEO Compact Extract
**Overview:** Converts raw HTML into key SEO fields (title, meta description, canonical, headings, OG tags) and a truncated visible text excerpt; outputs a compact text payload for the AI Agent.  
**Nodes involved:**  
- Code in JavaScript1

#### Node: Code in JavaScript1
- **Type / role:** Code (JavaScript) — SEO signal extractor.
- **Configuration choices:**
  - Reads: `$json.results[0].content` from the Decodo response.
  - Extracts via regex:
    - `<title>`
    - `<meta name="description" ...>`
    - `<link rel="canonical" ...>`
    - OG tags `og:title`, `og:description`
    - `<h1>`, up to 15 `<h2>`
  - Produces:
    - `seo_fields` object with structured values
    - `seo_compact_text` string formatted as:
      - `SEO_PAGE_EXTRACT`
      - TITLE/META_DESCRIPTION/CANONICAL/H1/H2/OG_TITLE/OG_DESCRIPTION
      - `VISIBLE_TEXT_EXCERPT` (stripped tags, scripts/styles removed) truncated to 6000 chars
  - Handles missing HTML:
    - If no HTML string, returns `seo_compact_text: "NO_HTML_INPUT"` and null fields.
- **Key expressions/variables:** Uses `$json?.results?.[0]?.content` (optional chaining).
- **Connections:**
  - In: Decodo
  - Out: AI Agent
- **Failure modes / edge cases:**
  - Unexpected Decodo response shape (e.g., `results` missing) → returns NO_HTML_INPUT
  - Regex-based parsing may miss SEO elements if attributes are unusual or minified
  - Very large HTML is processed (but excerpt is truncated)
- **Version notes:** `typeVersion: 2` Code node.

**Sticky note context (applies to this block):**
- “### AI Analysis  
  JavaScript reduces the HTML to key SEO elements. The AI Agent analyzes the data and generates an executive SEO summary.”

---

### Block 5 — AI Analysis (LLM Agent)
**Overview:** Uses an LLM to produce an executive SEO summary strictly as JSON.  
**Nodes involved:**  
- OpenAI Chat Model
- AI Agent

#### Node: OpenAI Chat Model
- **Type / role:** LangChain OpenAI Chat Model — provides the language model to the agent.
- **Configuration:**
  - Model: `gpt-4.1-mini`
  - Tools: none configured (`builtInTools` empty)
- **Connections:**
  - Provides `ai_languageModel` connection into **AI Agent**
- **Failure modes / edge cases:**
  - OpenAI credential issues
  - Model not available in the account/region
  - Rate limits / token limits if excerpt becomes large (though truncated to 6000 chars)
- **Version notes:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` `typeVersion: 1.3`.

#### Node: AI Agent
- **Type / role:** LangChain Agent — runs the analysis prompt and generates JSON output.
- **Configuration:**
  - Input text: `={{ $json.seo_compact_text }}`
  - System message enforces:
    - Audience: non-technical stakeholders
    - Output: **ONLY valid JSON** with fields:
      - `overall_status` ∈ {good, needs_attention, critical}
      - `top_issues` (max 5)
      - `quick_wins` (max 5)
      - title/meta description recommendations (string or null)
      - notes (string or null)
    - No markdown, no emojis
- **Connections:**
  - In (main): Code in JavaScript1
  - In (ai model): OpenAI Chat Model
  - Out: Code in JavaScript (parse/repair)
- **Failure modes / edge cases:**
  - Agent returns non-JSON despite instruction (common) → handled by parse/repair node but not guaranteed
  - Hallucinated fields or wrong enum values
  - Empty/NO_HTML_INPUT may yield vague output; prompt instructs to mention missing info in notes
- **Version notes:** `@n8n/n8n-nodes-langchain.agent` `typeVersion: 3`.

---

### Block 6 — Parse & Repair AI Output
**Overview:** Converts the AI Agent textual output into a parsed JSON object for downstream use, stripping code fences and applying minimal JSON repairs.  
**Nodes involved:**  
- Code in JavaScript

#### Node: Code in JavaScript
- **Type / role:** Code (JavaScript) — output normalization.
- **Configuration choices (interpreted):**
  - Accepts an AI output string from `item.json.output` (or falls back to `item.json`).
  - Removes ```json fences or ``` fences.
  - Extracts first `{ ... }` block by taking substring from first “{” to last “}”.
  - Attempts JSON.parse; if parse fails:
    - removes BOM
    - removes trailing commas before `}` or `]`
  - Returns `{ json: parsed }` for each item.
- **Key variables:**
  - `item.json.output` (expected from AI Agent)
  - `overall_status`, `top_issues`, `quick_wins`, etc. become top-level fields after parsing.
- **Connections:**
  - In: AI Agent
  - Out: Send a message
- **Failure modes / edge cases:**
  - If AI output contains extra braces or multiple JSON objects, “first { … last }” extraction may capture too much.
  - If AI returns arrays or no braces → node throws error (“No JSON object braces found”).
  - Repair logic is basic; won’t fix unquoted keys, single quotes, etc.
- **Version notes:** `typeVersion: 2`.

**Sticky note context (applies to this block):**
- “### Data Output  
  Results are saved to Google Sheets. A formatted SEO summary is sent by email using Gmail.”

---

### Block 7 — Email Reporting + Sheet Storage + Loop Continuation
**Overview:** Sends a formatted HTML email and stores results in an output Google Sheet, then signals the loop to continue.  
**Nodes involved:**  
- Send a message
- Append row in sheet

#### Node: Send a message
- **Type / role:** Gmail — sends HTML email.
- **Configuration:**
  - To: `user@example.com` (placeholder; must be replaced)
  - Subject: `=SEO Watchdog Report — {{$now}}`
  - Message: large HTML template (currently **static sample content**, including emojis in headings)
  - Format: HTML content placed directly in `message` field
- **Connections:**
  - In: Code in JavaScript (parsed AI JSON)
  - Out: Append row in sheet
- **Important implementation note:**  
  The email body shown is not dynamically templated from AI results (no expressions referencing `overall_status`, `top_issues`, etc.). To make it dynamic, you would replace static sections with n8n expressions.
- **Failure modes / edge cases:**
  - Gmail OAuth issues / missing scopes
  - Recipient invalid
  - HTML content size limits (unlikely here)
- **Version notes:** Gmail node `typeVersion: 2.2`.

#### Node: Append row in sheet
- **Type / role:** Google Sheets — writes SEO results.
- **Configuration:**
  - Operation: Append
  - Document: Spreadsheet named (cached) `SEO Analyzer` (ID: `1LPYN...`)
  - Sheet: `Hoja 1` (gid=0)
  - Mapping: “defineBelow” with explicit schema columns:
    - `brand_url` ← `$('Get row(s) in sheet').item.json.urls`
    - `status` ← `$('Code in JavaScript').item.json.overall_status`
    - `top_issues` ← `$('Code in JavaScript').item.json.top_issues`
    - `quick_wins` ← `$('Code in JavaScript').item.json.quick_wins`
    - `title_recommendation` ← `$('Code in JavaScript').item.json.title_recommendation`
    - `meta_description` ← `$('Code in JavaScript').item.json.meta_description_recommendation`
    - `notes` ← `$('Code in JavaScript').item.json.notes`
  - Type conversion disabled (`attemptToConvertTypes: false`)
- **Connections:**
  - In: Send a message
  - Out: Loop Over Items (to continue processing next URL)
- **Failure modes / edge cases:**
  - OAuth issues
  - Missing columns in the destination sheet or mismatched headers
  - Arrays written to cells (`top_issues`, `quick_wins`) may appear as comma-joined or `[object Object]` depending on n8n serialization; often you want `{{$json.top_issues.join("\n")}}`
- **Version notes:** Google Sheets `typeVersion: 4.7`.

---

### Block 8 — Documentation / Canvas Notes (Sticky Notes)
**Overview:** Non-executable nodes that document configuration and expected visuals.  
**Nodes involved:**  
- 📝 OVERVIEW
- 📝 DECODE NOTE
- 📝 AI AGENT NOTE
- 📝 PARSE & REPAIR
- Sticky Note (Input image)
- Sticky Note1 (Email image)
- Sticky Note2 (Google sheet image)

#### Node: 📝 OVERVIEW (Sticky Note)
- **Role:** Documents full workflow purpose + configuration requirements.
- **Key link:** [Decodo – Web Scraper for n8n](https://visit.decodo.com/raqXGD)
- **Failure modes:** None.

#### Node: Sticky Note (Input image)
- **Content:** `## Input![txt](https://ik.imagekit.io/agbb7sr41/input.png)`
- **Role:** Visual hint for input sheet.

#### Node: Sticky Note1 (Email image)
- **Content:**  
  `## Email`  
  `![txt](https://ik.imagekit.io/agbb7sr41/send_email.png)`
- **Role:** Visual hint for email configuration/output.

#### Node: Sticky Note2 (Google sheet image)
- **Content:**  
  `## Google sheet`  
  `![txt](https://ik.imagekit.io/agbb7sr41/output_seo_watachdog.png)`
- **Role:** Visual hint for output sheet.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get row(s) in sheet |  |
| Get row(s) in sheet | Google Sheets | Read URL rows from input spreadsheet | When clicking ‘Execute workflow’ | Loop Over Items | ## AI SEO Watchdog — Overview & Configuration (includes Decodo link) |
| Loop Over Items | Split In Batches | Iterate through URLs | Get row(s) in sheet; Append row in sheet | Decodo (via output 1) |  |
| Decodo | Decodo | Scrape HTML content for each URL | Loop Over Items | Code in JavaScript1 | ### Content Extraction: URLs are read from Google Sheets. Each page is scraped using Decodo to reliably fetch the HTML content. |
| Code in JavaScript1 | Code (JS) | Extract SEO elements + visible text excerpt | Decodo | AI Agent | ### AI Analysis: JavaScript reduces the HTML to key SEO elements. The AI Agent analyzes the data and generates an executive SEO summary. |
| OpenAI Chat Model | LangChain Chat Model (OpenAI) | LLM provider for agent | — | AI Agent (ai_languageModel) |  |
| AI Agent | LangChain Agent | Produce executive SEO JSON from compact extract | Code in JavaScript1; OpenAI Chat Model | Code in JavaScript | ### AI Analysis: JavaScript reduces the HTML to key SEO elements. The AI Agent analyzes the data and generates an executive SEO summary. |
| Code in JavaScript | Code (JS) | Parse/repair AI JSON output | AI Agent | Send a message | ### Data Output: Results are saved to Google Sheets. A formatted SEO summary is sent by email using Gmail. |
| Send a message | Gmail | Send HTML email report | Code in JavaScript | Append row in sheet | ## Email \| ![txt](https://ik.imagekit.io/agbb7sr41/send_email.png) |
| Append row in sheet | Google Sheets | Append results to output spreadsheet | Send a message | Loop Over Items | ## Google sheet \| ![txt](https://ik.imagekit.io/agbb7sr41/output_seo_watachdog.png) |
| 📝 OVERVIEW | Sticky Note | Canvas documentation | — | — | ## AI SEO Watchdog — Overview & Configuration (includes Decodo link) |
| 📝 DECODE NOTE | Sticky Note | Canvas documentation | — | — | ### Content Extraction: URLs are read from Google Sheets. Each page is scraped using Decodo to reliably fetch the HTML content. |
| 📝 AI AGENT NOTE | Sticky Note | Canvas documentation | — | — | ### AI Analysis: JavaScript reduces the HTML to key SEO elements. The AI Agent analyzes the data and generates an executive SEO summary. |
| 📝 PARSE & REPAIR | Sticky Note | Canvas documentation | — | — | ### Data Output: Results are saved to Google Sheets. A formatted SEO summary is sent by email using Gmail. |
| Sticky Note | Sticky Note | Input visual reference | — | — | ## Input![txt](https://ik.imagekit.io/agbb7sr41/input.png) |
| Sticky Note1 | Sticky Note | Email visual reference | — | — | ## Email \| ![txt](https://ik.imagekit.io/agbb7sr41/send_email.png) |
| Sticky Note2 | Sticky Note | Output sheet visual reference | — | — | ## Google sheet \| ![txt](https://ik.imagekit.io/agbb7sr41/output_seo_watachdog.png) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it e.g. “SEO Monitoring Workflow with AI and Email Reporting”.

2) **Add Trigger**
   - Add node: **Manual Trigger**
   - Name: “When clicking ‘Execute workflow’”

3) **Set up the input Google Sheet**
   - Create a Google Sheet (example: `urls_to_scrape`)
   - In `Hoja 1`, add a header column named **`urls`**
   - Put **one URL per row** (include `https://`)

4) **Add “Get row(s) in sheet”**
   - Node type: **Google Sheets**
   - Operation: **Get row(s)** (read rows)
   - Credentials: **Google Sheets OAuth2**
   - Document: select your `urls_to_scrape`
   - Sheet: `Hoja 1`
   - Ensure the output includes a field `urls` for each row
   - Connect: Manual Trigger → Get row(s) in sheet

5) **Add batching loop**
   - Node type: **Split In Batches**
   - Name: “Loop Over Items”
   - Set batch size as desired (e.g., 1–5 to reduce rate-limit risk)
   - Connect: Get row(s) in sheet → Loop Over Items

6) **Add Decodo scraper**
   - Node type: **Decodo**
   - Credentials: **Decodo API**
   - URL field expression:  
     - `{{ $('Get row(s) in sheet').item.json.urls }}`
     - (Preferred improvement: `{{ $json.urls }}` when iterating inside the batch.)
   - Connect: Loop Over Items (output 1) → Decodo

7) **Add HTML → SEO extraction**
   - Node type: **Code** (JavaScript)
   - Name: “Code in JavaScript1”
   - Paste logic that:
     - Reads `$json.results[0].content`
     - Extracts title/meta/canonical/H1/H2/OG tags
     - Strips scripts/styles/tags for visible text excerpt
     - Outputs `seo_compact_text` and `seo_fields`
   - Connect: Decodo → Code in JavaScript1

8) **Add OpenAI model node**
   - Node type: **OpenAI Chat Model** (LangChain)
   - Credentials: **OpenAI API**
   - Model: `gpt-4.1-mini` (or choose an available model)
   - This node will be connected as the Agent’s language model.

9) **Add AI Agent**
   - Node type: **AI Agent** (LangChain)
   - Text input: `{{ $json.seo_compact_text }}`
   - System message: instruct agent to return **ONLY valid JSON** with:
     - overall_status, top_issues, quick_wins, title/meta recommendations, notes
   - Connect:
     - Code in JavaScript1 → AI Agent (main)
     - OpenAI Chat Model → AI Agent (ai_languageModel)

10) **Add Parse/Repair node**
   - Node type: **Code** (JavaScript)
   - Name: “Code in JavaScript”
   - Implement:
     - strip ```json fences
     - extract first `{...}`
     - JSON.parse with minimal repair (trailing commas)
   - Connect: AI Agent → Code in JavaScript

11) **Add Gmail email sender**
   - Node type: **Gmail**
   - Credentials: **Gmail OAuth2**
   - To: set your recipient address
   - Subject: `SEO Watchdog Report — {{$now}}`
   - Message: paste your HTML template  
     - If you want dynamic content, replace static text with expressions like:
       - `{{ $json.overall_status }}`
       - `{{ $json.top_issues.join('<li>...</li>') }}` (build HTML carefully)
   - Connect: Code in JavaScript → Send a message

12) **Set up output Google Sheet**
   - Create another Google Sheet (example: `SEO Analyzer`)
   - In `Hoja 1`, add headers matching:
     - `brand_url`, `status`, `top_issues`, `quick_wins`, `title_recommendation`, `meta_description`, `notes`

13) **Add “Append row in sheet”**
   - Node type: **Google Sheets**
   - Operation: **Append**
   - Credentials: Google Sheets OAuth2
   - Document: `SEO Analyzer`
   - Sheet: `Hoja 1`
   - Map columns using expressions:
     - brand_url: `{{ $('Get row(s) in sheet').item.json.urls }}`
     - status: `{{ $json.overall_status }}` (or `{{ $('Code in JavaScript').item.json.overall_status }}`)
     - top_issues: `{{ $json.top_issues }}` (recommended: join to string)
     - quick_wins: `{{ $json.quick_wins }}` (recommended: join to string)
     - title_recommendation: `{{ $json.title_recommendation }}`
     - meta_description: `{{ $json.meta_description_recommendation }}`
     - notes: `{{ $json.notes }}`
   - Connect: Send a message → Append row in sheet

14) **Close the loop**
   - Connect: Append row in sheet → Loop Over Items (so it continues with the next URL)

15) **(Optional) Add scheduling**
   - Replace Manual Trigger or add a **Schedule Trigger** for continuous monitoring.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI SEO Watchdog — Overview & Configuration” (describes Sheets input/output, Decodo, AI Agent, Gmail) | Included in canvas sticky note “📝 OVERVIEW” |
| Decodo link: “Decodo – Web Scraper for n8n” | https://visit.decodo.com/raqXGD |
| Input illustration | https://ik.imagekit.io/agbb7sr41/input.png |
| Email illustration | https://ik.imagekit.io/agbb7sr41/send_email.png |
| Output sheet illustration | https://ik.imagekit.io/agbb7sr41/output_seo_watachdog.png |

