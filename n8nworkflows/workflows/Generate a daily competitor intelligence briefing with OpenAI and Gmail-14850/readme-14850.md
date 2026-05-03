Generate a daily competitor intelligence briefing with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/generate-a-daily-competitor-intelligence-briefing-with-openai-and-gmail-14850


# Generate a daily competitor intelligence briefing with OpenAI and Gmail

## 1. Workflow Overview

This workflow automates daily competitive intelligence monitoring. Every morning at 8 AM (based on the n8n server timezone), it:

1. Iterates over a user-defined list of competitor websites.
2. Fetches each competitor's web page, strips it down to readable text, and sends it to OpenAI GPT-4o-mini for per-competitor signal extraction.
3. Aggregates all individual analyses into a single prompt for GPT-4o, which writes a polished executive briefing in HTML.
4. Formats the briefing into a styled email, delivers it via Gmail, and appends a log entry to Google Sheets.

The logic is organized into three blocks:

| Block | Purpose | Nodes |
|---|---|---|
| **1 – Schedule & Configure** | Fires on a 24-hour schedule and emits one item per competitor with all configuration metadata. | Daily 8AM Trigger, Configure Competitors and Settings |
| **2 – Per-Competitor Fetch, Clean & Analyze** | For each competitor: fetches the page, extracts text, calls GPT-4o-mini for analysis, parses the AI response. | Fetch Competitor Page, Extract Text and Build AI Prompt, AI Analyze Competitor, Parse AI Response |
| **3 – Compile, Email & Log** | Aggregates all analyses, calls GPT-4o for the final briefing, formats the HTML email, sends it via Gmail, and logs to Google Sheets. | Aggregate and Build Briefing Prompt, AI Generate Final Briefing, Format HTML Email, Send Email Briefing, Log to Google Sheets |

---

## 2. Block-by-Block Analysis

### Block 1 – Schedule & Configure

**Overview:** A scheduled trigger fires once every 24 hours. The Configure node then expands a static competitor list into one output item per competitor, attaching all shared settings (company name, focus area, recipient email, run date) to each item.

**Nodes Involved:**
- Daily 8AM Trigger
- Configure Competitors and Settings

---

#### Daily 8AM Trigger

- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)
- **Technical Role:** Workflow entry point; produces a single empty item every 24 hours.
- **Configuration:**
  - Interval field: `hours`
  - Interval value: `24` (fires once daily)
  - The actual clock time depends on the n8n instance's default timezone; it is not configurable per-node in v1.2.
- **Key Expressions / Variables:** None.
- **Input Connections:** None (trigger node).
- **Output Connections:** → `Configure Competitors and Settings` (main, index 0).
- **Version-Specific Notes:** In v1.2 the trigger fires every N hours from the time the workflow is activated. To guarantee 8 AM specifically, the n8n server timezone or the `cron` mode (available in v1.1+) may be preferred.
- **Edge Cases / Failure Types:**
  - If the n8n instance is down at the scheduled time, the run is missed entirely (no retry).
  - Changing the workflow's active state resets the interval timer.

---

#### Configure Competitors and Settings

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Central configuration hub. Emits one item per competitor, each carrying full run metadata.
- **Configuration:**
  - **JavaScript code** defines:
    - `REPORT_EMAIL` — recipient email address (default: `user@example.com`).
    - `YOUR_COMPANY` — company name used in AI prompts (default: `Acme Corp`).
    - `YOUR_FOCUS` — market/product category (default: `B2B SaaS project management tools`).
    - `COMPETITORS` — array of objects, each with `name`, `url`, and `watch_for` fields. Three placeholder competitors are included.
  - The code below the "DO NOT EDIT" line computes `today` (ISO date) and maps the array to output items.
- **Key Expressions / Variables:**
  - Output fields per item: `competitor_name`, `competitor_url`, `watch_for`, `report_email`, `your_company`, `your_focus`, `run_date`.
- **Input Connections:** ← `Daily 8AM Trigger` (main, index 0).
- **Output Connections:** → `Fetch Competitor Page` (main, index 0).
- **Edge Cases / Failure Types:**
  - If the `COMPETITORS` array is empty, downstream nodes receive zero items and the workflow effectively does nothing (the Aggregate node later throws an error).
  - Invalid URLs will cause the downstream HTTP Request to fail (mitigated by `continueOnFail`).
  - Changing the email recipient here propagates automatically to both the Gmail and Sheets nodes.

---

### Block 2 – Per-Competitor Fetch, Clean & Analyze

**Overview:** For each competitor emitted by Block 1, the workflow fetches the target URL, cleans the raw HTML into plain text (capped at 4,500 characters), constructs a structured OpenAI prompt, calls GPT-4o-mini, and parses the AI response back into structured data. Because the Configure node outputs multiple items, n8n processes this block once per competitor (effectively in parallel, depending on n8n execution settings).

**Nodes Involved:**
- Fetch Competitor Page
- Extract Text and Build AI Prompt
- AI Analyze Competitor
- Parse AI Response

---

#### Fetch Competitor Page

- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** HTTP GET to the competitor URL to retrieve raw page content.
- **Configuration:**
  - URL: `={{ $json.competitor_url }}` (dynamically set per item).
  - Method: GET (default).
  - Timeout: 20,000 ms (20 seconds).
  - Max redirects: 5.
  - Response format: default (full response object).
  - `continueOnFail`: **true** — the workflow proceeds even if the request fails, enabling downstream fallback handling.
- **Key Expressions / Variables:** `{{ $json.competitor_url }}`.
- **Input Connections:** ← `Configure Competitors and Settings` (main, index 0).
- **Output Connections:** → `Extract Text and Build AI Prompt` (main, index 0).
- **Edge Cases / Failure Types:**
  - **DNS errors / timeouts / 403 / CAPTCHA walls** — the node outputs an error object; the `continueOnFail` flag prevents workflow termination.
  - **Heavy pages** — the full response body is passed downstream; no size limit is enforced at this node.
  - **Redirects beyond 5** — the request fails.
  - **SSL certificate issues** — the request fails unless the n8n instance has `NODE_TLS_REJECT_UNAUTHORIZED=0` (not recommended for production).

---

#### Extract Text and Build AI Prompt

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Cleans raw HTML into plain text and builds the OpenAI chat-completion request body for per-competitor analysis.
- **Configuration:**
  - JavaScript code performs:
    1. Reads the HTTP response from `Fetch Competitor Page` (`$input.item.json`).
    2. Reads configuration metadata from `Configure Competitors and Settings` via `$('Configure Competitors and Settings').item.json`.
    3. Attempts to extract the HTML string from `htmlData.data` or falls back to stringifying the object.
    4. Strips `<script>`, `<style>`, `<svg>`, `<nav>`, `<footer>`, `<head>` blocks and all remaining HTML tags, decodes common HTML entities, collapses whitespace, and slices to 4,500 characters.
    5. Builds `systemMsg` and `userMsg` strings instructing GPT-4o-mini to return a structured analysis (competitor name, up to 4 key signals, threat level with reason, recommended action).
    6. Assembles the `openaiBody` object (model: `gpt-4o-mini`, max_tokens: 500, temperature: 0.3) and stringifies it.
- **Key Expressions / Variables:**
  - References `$('Configure Competitors and Settings')` to pull configuration metadata.
  - Output: `competitor_name`, `competitor_url`, `watch_for`, `report_email`, `your_company`, `your_focus`, `run_date`, `openai_body` (JSON string).
- **Input Connections:** ← `Fetch Competitor Page` (main, index 0).
- **Output Connections:** → `AI Analyze Competitor` (main, index 0).
- **Edge Cases / Failure Types:**
  - If the HTML is empty or the fetch failed, `pageText` resolves to `''` and the user prompt includes `ERROR: Page could not be loaded.`, allowing the AI to report the issue.
  - Pages with primarily JavaScript-rendered content may yield very thin text (no headless browser is used).
  - The 4,500-character cap may truncate meaningful content from long pages.

---

#### AI Analyze Competitor

- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Sends the per-competitor prompt to the OpenAI Chat Completions API.
- **Configuration:**
  - URL: `https://api.openai.com/v1/chat/completions`
  - Method: POST
  - Authentication: `predefinedCredentialType` → `openAiApi` (requires an OpenAI API key credential configured in n8n).
  - Body specification: `json`
  - JSON Body: `={{ $json.openai_body }}` (the stringified request built by the previous node).
  - Timeout: 45,000 ms (45 seconds).
- **Key Expressions / Variables:** `{{ $json.openai_body }}`.
- **Input Connections:** ← `Extract Text and Build AI Prompt` (main, index 0).
- **Output Connections:** → `Parse AI Response` (main, index 0).
- **Edge Cases / Failure Types:**
  - **401 Unauthorized** — OpenAI credential missing or invalid.
  - **429 Rate Limit** — too many concurrent requests or exceeded quota; may occur if many competitors are configured.
  - **500 / 503 from OpenAI** — transient service errors.
  - **Timeout after 45s** — complex prompts or API latency.
  - `continueOnFail` is NOT set here, so an API failure will halt the workflow for that item (other items in the batch may still process if n8n processes them independently).

---

#### Parse AI Response

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Extracts the AI-generated text from the OpenAI response and carries forward all metadata for aggregation.
- **Configuration:**
  - JavaScript code:
    1. Reads the API response from `$input.item.json`.
    2. Reads configuration from `Extract Text and Build AI Prompt` via `$('Extract Text and Build AI Prompt').item.json`.
    3. Extracts `res.choices[0].message.content` or falls back to an error message string.
    4. Outputs: `competitor_name`, `competitor_url`, `analysis` (the AI text), `report_email`, `your_company`, `your_focus`, `run_date`.
- **Key Expressions / Variables:**
  - `res.choices?.[0]?.message?.content` — safe navigation with optional chaining.
  - Fallback: `**Analysis unavailable** — check your OpenAI credential or the target URL.`
- **Input Connections:** ← `AI Analyze Competitor` (main, index 0).
- **Output Connections:** → `Aggregate and Build Briefing Prompt` (main, index 0).
- **Edge Cases / Failure Types:**
  - If the OpenAI response structure is unexpected (e.g., content moderation block), the optional chaining returns `undefined` and the fallback text is used.
  - If `$('Extract Text and Build AI Prompt')` is not accessible (e.g., in a sub-workflow), the reference will fail.

---

### Block 3 – Compile, Email & Log

**Overview:** All per-competitor analyses are collected into a single item. A second OpenAI call (GPT-4o) synthesizes them into a polished executive briefing in email-safe HTML. The resulting HTML is wrapped in a styled email template, sent via Gmail, and a summary row is appended to Google Sheets.

**Nodes Involved:**
- Aggregate and Build Briefing Prompt
- AI Generate Final Briefing
- Format HTML Email
- Send Email Briefing
- Log to Google Sheets

---

#### Aggregate and Build Briefing Prompt

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Collects all per-competitor analysis items and constructs the final GPT-4o prompt for the consolidated briefing.
- **Configuration:**
  - JavaScript code:
    1. Reads all items via `$input.all()`.
    2. Validates that at least one item exists; throws an error otherwise.
    3. Extracts shared metadata from the first item.
    4. Concatenates each competitor's analysis into labeled blocks (`=== Competitor Name (URL) ===`).
    5. Builds `systemMsg` (VP of Competitive Intelligence persona) and `userMsg` (instructions to write an HTML briefing with: Executive Summary, Competitor Snapshots, Priority Actions, Watch List).
    6. Assembles `openaiBody` (model: `gpt-4o`, max_tokens: 1800, temperature: 0.4) and stringifies it.
  - Output: `report_email`, `your_company`, `your_focus`, `run_date`, `competitor_count`, `openai_body` (JSON string).
- **Key Expressions / Variables:** `$input.all()` to collect all items; `first` variable for shared metadata.
- **Input Connections:** ← `Parse AI Response` (main, index 0).
- **Output Connections:** → `AI Generate Final Briefing` (main, index 0).
- **Edge Cases / Failure Types:**
  - If no items reach this node (e.g., all fetches failed and were filtered), the code throws: `No competitor analyses received -- check the upstream nodes.` This will halt the workflow.
  - Very large numbers of competitors may produce a prompt that exceeds GPT-4o's context window (~128K tokens), though this is unlikely with the 4,500-char cap per competitor.

---

#### AI Generate Final Briefing

- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Sends the aggregated briefing prompt to OpenAI GPT-4o for final synthesis.
- **Configuration:**
  - URL: `https://api.openai.com/v1/chat/completions`
  - Method: POST
  - Authentication: `predefinedCredentialType` → `openAiApi`
  - Body specification: `json`
  - JSON Body: `={{ $json.openai_body }}`
  - Timeout: 90,000 ms (90 seconds).
- **Key Expressions / Variables:** `{{ $json.openai_body }}`.
- **Input Connections:** ← `Aggregate and Build Briefing Prompt` (main, index 0).
- **Output Connections:** → `Format HTML Email` (main, index 0).
- **Edge Cases / Failure Types:**
  - Same auth/rate-limit/timeout risks as `AI Analyze Competitor`.
  - Higher cost per call (GPT-4o) compared to the per-competitor GPT-4o-mini calls.
  - If `max_tokens=1800` is insufficient for many competitors, the output may be truncated.

---

#### Format HTML Email

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Wraps the AI-generated HTML briefing inside a complete, styled HTML email template and produces a plain-text fallback.
- **Configuration:**
  - JavaScript code:
    1. Reads the OpenAI response from `$input.item.json`.
    2. Reads metadata from `Aggregate and Build Briefing Prompt` via `$('Aggregate and Build Briefing Prompt').item.json`.
    3. Extracts `bodyHtml` from `res.choices[0].message.content` with a fallback error paragraph.
    4. Builds a complete HTML document with embedded `<style>` (responsive wrapper, header with dark background, body content area, footer).
    5. Header displays: "Competitor Intelligence Briefing", company name, run date, competitor count.
    6. Footer: "Auto-generated by n8n AI Competitor Intelligence Workflow".
    7. Generates `plainText` by stripping all HTML tags from `bodyHtml`.
  - Output: `email_html`, `email_plain`, `report_email`, `your_company`, `run_date`, `competitor_count`.
- **Key Expressions / Variables:**
  - `res.choices && res.choices[0] && res.choices[0].message && res.choices[0].message.content` — defensive extraction.
  - `$('Aggregate and Build Briefing Prompt').item.json` — metadata reference.
- **Input Connections:** ← `AI Generate Final Briefing` (main, index 0).
- **Output Connections:** → `Send Email Briefing` (main, index 0), → `Log to Google Sheets` (main, index 0).
- **Edge Cases / Failure Types:**
  - If the AI output is not valid HTML (e.g., contains markdown), the email may render poorly. The system prompt explicitly forbids markdown, but model compliance is not guaranteed.
  - The inline `<style>` block may be stripped by some email clients (e.g., Gmail web); however, the styles are simple enough to degrade gracefully.

---

#### Send Email Briefing

- **Type:** `n8n-nodes-base.gmail` (v2.1)
- **Technical Role:** Sends the formatted HTML briefing email to the configured recipient.
- **Configuration:**
  - To: `={{ $json.report_email }}` (dynamically set from configuration).
  - Subject: `={{ 'Competitor Briefing - ' + $json.run_date + ' (' + $json.competitor_count + ' competitors)' }}`
  - Message: `={{ $json.email_html }}` (HTML body).
  - Options: default (no explicit HTML/plain toggle in visible config; the Gmail node sends as HTML when the message contains HTML).
  - Credential: Gmail OAuth2 (must be connected in n8n).
- **Key Expressions / Variables:**
  - `{{ $json.report_email }}`
  - `{{ 'Competitor Briefing - ' + $json.run_date + ' (' + $json.competitor_count + ' competitors)' }}`
  - `{{ $json.email_html }}`
- **Input Connections:** ← `Format HTML Email` (main, index 0).
- **Output Connections:** None (terminal node).
- **Edge Cases / Failure Types:**
  - **OAuth2 token expired or revoked** — the node will fail with an authentication error.
  - **Gmail sending quota exceeded** — Google limits emails per day (typically 500 for consumer accounts, more for Workspace).
  - **Invalid recipient email** — will cause a delivery failure bounce; the node itself may succeed but the email won't be delivered.

---

#### Log to Google Sheets

- **Type:** `n8n-nodes-base.googleSheets` (v4.4)
- **Technical Role:** Appends a summary row to a Google Sheet for historical tracking.
- **Configuration:**
  - Operation: `append`
  - Document ID: `PASTE_YOUR_GOOGLE_SHEET_ID_HERE` (placeholder — must be replaced with a real sheet ID from the Google Sheets URL).
  - Sheet Name: `Intelligence Log` (by name).
  - Column mapping (define below):
    - `Date` ← `={{ $json.run_date }}`
    - `Company` ← `={{ $json.your_company }}`
    - `Briefing Summary` ← `={{ $json.email_plain }}`
    - `Competitors Analyzed` ← `={{ $json.competitor_count }}`
  - Credential: Google Sheets OAuth2 (must be connected in n8n).
- **Key Expressions / Variables:**
  - `{{ $json.run_date }}`, `{{ $json.your_company }}`, `{{ $json.email_plain }}`, `{{ $json.competitor_count }}`
- **Input Connections:** ← `Format HTML Email` (main, index 0).
- **Output Connections:** None (terminal node).
- **Edge Cases / Failure Types:**
  - **Placeholder document ID not replaced** — will cause a "Sheet not found" error.
  - **Sheet name `Intelligence Log` does not exist** — the node will fail unless auto-create is enabled (not configured here).
  - **Missing columns in the sheet** — the node will create them if they don't exist, depending on n8n version behavior.
  - **OAuth2 scope insufficient** — Google credential must have Sheets API scope.
  - **Cell size limit** — the `Briefing Summary` field (plain text of the entire briefing) may exceed Google Sheets' 50,000-character cell limit for very large briefings.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily 8AM Trigger | scheduleTrigger (v1.2) | Fires the workflow every 24 hours | — | Configure Competitors and Settings | Schedule and configure — Fires every morning at 8AM and loops through your list of competitors. All settings -- URLs, company name, email recipient -- live in the Configure node so you only ever need to edit one place. |
| Configure Competitors and Settings | code (v2) | Expands competitor list into items with full metadata | Daily 8AM Trigger | Fetch Competitor Page | Schedule and configure — Fires every morning at 8AM and loops through your list of competitors. All settings -- URLs, company name, email recipient -- live in the Configure node so you only ever need to edit one place. |
| Fetch Competitor Page | httpRequest (v4.2) | HTTP GET for each competitor URL | Configure Competitors and Settings | Extract Text and Build AI Prompt | Fetch, clean and analyse each competitor — Visits each competitor URL, strips the HTML down to readable text, then sends it to GPT-4o-mini to extract key signals, threat level, and a recommended action. Runs once per competitor in parallel. |
| Extract Text and Build AI Prompt | code (v2) | Strips HTML, builds per-competitor OpenAI prompt | Fetch Competitor Page | AI Analyze Competitor | Fetch, clean and analyse each competitor — Visits each competitor URL, strips the HTML down to readable text, then sends it to GPT-4o-mini to extract key signals, threat level, and a recommended action. Runs once per competitor in parallel. |
| AI Analyze Competitor | httpRequest (v4.2) | Calls OpenAI GPT-4o-mini for per-competitor analysis | Extract Text and Build AI Prompt | Parse AI Response | Fetch, clean and analyse each competitor — Visits each competitor URL, strips the HTML down to readable text, then sends it to GPT-4o-mini to extract key signals, threat level, and a recommended action. Runs once per competitor in parallel. |
| Parse AI Response | code (v2) | Extracts AI text from OpenAI response, carries metadata forward | AI Analyze Competitor | Aggregate and Build Briefing Prompt | Fetch, clean and analyse each competitor — Visits each competitor URL, strips the HTML down to readable text, then sends it to GPT-4o-mini to extract key signals, threat level, and a recommended action. Runs once per competitor in parallel. |
| Aggregate and Build Briefing Prompt | code (v2) | Collects all analyses, builds final GPT-4o briefing prompt | Parse AI Response | AI Generate Final Briefing | Compile, email and log — Aggregates all competitor analyses into a single prompt for GPT-4o, which writes the final executive briefing. The briefing is formatted as a clean HTML email, sent to your team, and logged to Google Sheets. |
| AI Generate Final Briefing | httpRequest (v4.2) | Calls OpenAI GPT-4o to synthesize the executive briefing | Aggregate and Build Briefing Prompt | Format HTML Email | Compile, email and log — Aggregates all competitor analyses into a single prompt for GPT-4o, which writes the final executive briefing. The briefing is formatted as a clean HTML email, sent to your team, and logged to Google Sheets. |
| Format HTML Email | code (v2) | Wraps AI HTML in styled email template, produces plain-text fallback | AI Generate Final Briefing | Send Email Briefing, Log to Google Sheets | Compile, email and log — Aggregates all competitor analyses into a single prompt for GPT-4o, which writes the final executive briefing. The briefing is formatted as a clean HTML email, sent to your team, and logged to Google Sheets. |
| Send Email Briefing | gmail (v2.1) | Delivers the HTML briefing email | Format HTML Email | — | Compile, email and log — Aggregates all competitor analyses into a single prompt for GPT-4o, which writes the final executive briefing. The briefing is formatted as a clean HTML email, sent to your team, and logged to Google Sheets. |
| Log to Google Sheets | googleSheets (v4.4) | Appends a summary row to a Google Sheet | Format HTML Email | — | Compile, email and log — Aggregates all competitor analyses into a single prompt for GPT-4o, which writes the final executive briefing. The briefing is formatted as a clean HTML email, sent to your team, and logged to Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it `AI Competitor Intelligence Briefing: Daily Monitoring with OpenAI and Gmail`.

2. **Add a Schedule Trigger node:**
   - Type: `Schedule Trigger`
   - Name: `Daily 8AM Trigger`
   - Interval: `hours`, value `24`
   - This determines the daily execution cadence. The exact clock time aligns with the n8n server timezone when the workflow is activated.

3. **Add a Code node for configuration:**
   - Type: `Code`
   - Name: `Configure Competitors and Settings`
   - Language: JavaScript
   - Paste the configuration code (see below for the logic to replicate):
     - Define `REPORT_EMAIL` (string, your team's email address).
     - Define `YOUR_COMPANY` (string, your company name).
     - Define `YOUR_FOCUS` (string, your market/product category).
     - Define `COMPETITORS` (array of objects with `name`, `url`, `watch_for`).
     - Below the "DO NOT EDIT" line: compute `today` from `new Date().toISOString().split('T')[0]`, then `return COMPETITORS.map(c => ({ json: { competitor_name: c.name, competitor_url: c.url, watch_for: c.watch_for, report_email: REPORT_EMAIL, your_company: YOUR_COMPANY, your_focus: YOUR_FOCUS, run_date: today } }))`.
   - Connect: **Daily 8AM Trigger** → **Configure Competitors and Settings**.

4. **Add an HTTP Request node for fetching competitor pages:**
   - Type: `HTTP Request`
   - Name: `Fetch Competitor Page`
   - Method: GET
   - URL: `={{ $json.competitor_url }}`
   - Timeout: 20000 ms
   - Max Redirects: 5
   - **Continue On Fail**: enabled (critical for resilience)
   - Connect: **Configure Competitors and Settings** → **Fetch Competitor Page**.

5. **Add a Code node to extract text and build the AI prompt:**
   - Type: `Code`
   - Name: `Extract Text and Build AI Prompt`
   - Language: JavaScript
   - Logic to implement:
     - Read HTML from `$input.item.json`; attempt `htmlData.data` or stringify.
     - Strip `<script>`, `<style>`, `<svg>`, `<nav>`, `<footer>`, `<head>` blocks with regex.
     - Remove all remaining HTML tags, decode entities (`&amp;`, `&lt;`, `&gt;`, `&nbsp;`, numeric entities), collapse whitespace, trim, and slice to 4500 characters.
     - Read config from `$('Configure Competitors and Settings').item.json`.
     - Build `systemMsg` (competitive intelligence analyst persona) and `userMsg` (structured analysis request: Competitor name, Key Signals max 4, Threat Level with reason, Recommended Action).
     - Build `openaiBody` with model `gpt-4o-mini`, max_tokens `500`, temperature `0.3`, and two messages (system + user).
     - Output: all config fields plus `openai_body` (stringified).
   - Connect: **Fetch Competitor Page** → **Extract Text and Build AI Prompt**.

6. **Add an HTTP Request node for per-competitor AI analysis:**
   - Type: `HTTP Request`
   - Name: `AI Analyze Competitor`
   - Method: POST
   - URL: `https://api.openai.com/v1/chat/completions`
   - Authentication: Predefined Credential Type → `openAiApi`
   - **Credential**: Create an OpenAI API key credential (see step 13).
   - Body specification: `json`
   - JSON Body: `={{ $json.openai_body }}`
   - Timeout: 45000 ms
   - Connect: **Extract Text and Build AI Prompt** → **AI Analyze Competitor**.

7. **Add a Code node to parse the AI response:**
   - Type: `Code`
   - Name: `Parse AI Response`
   - Language: JavaScript
   - Logic:
     - Extract `choices[0].message.content` from the OpenAI response; fallback to `**Analysis unavailable** — check your OpenAI credential or the target URL.`
     - Read metadata from `$('Extract Text and Build AI Prompt').item.json`.
     - Output: `competitor_name`, `competitor_url`, `analysis`, `report_email`, `your_company`, `your_focus`, `run_date`.
   - Connect: **AI Analyze Competitor** → **Parse AI Response**.

8. **Add a Code node to aggregate analyses and build the briefing prompt:**
   - Type: `Code`
   - Name: `Aggregate and Build Briefing Prompt`
   - Language: JavaScript
   - Logic:
     - Read all items via `$input.all()`.
     - Throw an error if no items exist.
     - Extract shared metadata from `all[0].json`.
     - Build concatenated analysis blocks (`=== Competitor Name (URL) ===\nAnalysis text`).
     - Build `systemMsg` (VP of Competitive Intelligence persona) and `userMsg` (write an HTML briefing with: Executive Summary, Competitor Snapshots, Priority Actions, Watch List; rules: clean HTML only, specific tags, no markdown, no inline CSS).
     - Build `openaiBody` with model `gpt-4o`, max_tokens `1800`, temperature `0.4`.
     - Output: `report_email`, `your_company`, `your_focus`, `run_date`, `competitor_count`, `openai_body` (stringified).
   - Connect: **Parse AI Response** → **Aggregate and Build Briefing Prompt**.

9. **Add an HTTP Request node for the final briefing generation:**
   - Type: `HTTP Request`
   - Name: `AI Generate Final Briefing`
   - Method: POST
   - URL: `https://api.openai.com/v1/chat/completions`
   - Authentication: Predefined Credential Type → `openAiApi`
   - **Credential**: Same OpenAI API key credential as step 6.
   - Body specification: `json`
   - JSON Body: `={{ $json.openai_body }}`
   - Timeout: 90000 ms
   - Connect: **Aggregate and Build Briefing Prompt** → **AI Generate Final Briefing**.

10. **Add a Code node to format the HTML email:**
    - Type: `Code`
    - Name: `Format HTML Email`
    - Language: JavaScript
    - Logic:
      - Extract `choices[0].message.content` from the final briefing response; fallback to an error paragraph.
      - Read metadata from `$('Aggregate and Build Briefing Prompt').item.json`.
      - Wrap the AI HTML inside a full HTML document with embedded `<style>` (wrapper, header with dark background, body, footer).
      - Header: "Competitor Intelligence Briefing" + company name + run date + competitor count.
      - Footer: "Auto-generated by n8n AI Competitor Intelligence Workflow".
      - Generate plain-text version by stripping HTML tags.
      - Output: `email_html`, `email_plain`, `report_email`, `your_company`, `run_date`, `competitor_count`.
    - Connect: **AI Generate Final Briefing** → **Format HTML Email**.

11. **Add a Gmail node to send the briefing:**
    - Type: `Gmail`
    - Name: `Send Email Briefing`
    - To: `={{ $json.report_email }}`
    - Subject: `={{ 'Competitor Briefing - ' + $json.run_date + ' (' + $json.competitor_count + ' competitors)' }}`
    - Message: `={{ $json.email_html }}`
    - **Credential**: Create a Gmail OAuth2 credential (see step 14).
    - Connect: **Format HTML Email** → **Send Email Briefing**.

12. **Add a Google Sheets node to log the run:**
    - Type: `Google Sheets`
    - Name: `Log to Google Sheets`
    - Operation: `append`
    - Document ID: Replace `PASTE_YOUR_GOOGLE_SHEET_ID_HERE` with the actual sheet ID from the Google Sheets URL (the string between `/d/` and `/edit`).
    - Sheet Name: `Intelligence Log`
    - Column mapping (define below):
      - `Date` ← `={{ $json.run_date }}`
      - `Company` ← `={{ $json.your_company }}`
      - `Briefing Summary` ← `={{ $json.email_plain }}`
      - `Competitors Analyzed` ← `={{ $json.competitor_count }}`
    - **Credential**: Create a Google Sheets OAuth2 credential (see step 15).
    - Connect: **Format HTML Email** → **Log to Google Sheets**.
    - **Prerequisite**: Create the Google Sheet with a tab named `Intelligence Log` and columns `Date`, `Company`, `Competitors Analyzed`, `Briefing Summary` (in any order; the node maps by name).

13. **Configure OpenAI API Key credential:**
    - In n8n, go to **Credentials → New Credential → OpenAI API**.
    - Enter your API key obtained from [platform.openai.com](https://platform.openai.com).
    - Name the credential descriptively (e.g., `OpenAI account`).
    - Assign this credential to both the **AI Analyze Competitor** and **AI Generate Final Briefing** nodes.

14. **Configure Gmail OAuth2 credential:**
    - In n8n, go to **Credentials → New Credential → Gmail OAuth2**.
    - Follow the OAuth2 consent flow to authorize n8n to send emails on behalf of the chosen Gmail account.
    - Assign this credential to the **Send Email Briefing** node.

15. **Configure Google Sheets OAuth2 credential:**
    - In n8n, go to **Credentials → New Credential → Google Sheets OAuth2**.
    - Follow the OAuth2 consent flow and ensure the scope includes Sheets API access.
    - Assign this credential to the **Log to Google Sheets** node.

16. **Activate the workflow.** Once active, it will execute every 24 hours from the activation time. To ensure it fires at 8 AM specifically, either:
    - Activate it at 8 AM local server time, or
    - Switch the Schedule Trigger to cron mode (`0 8 * * *`) if using a newer version of the node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow uses two OpenAI models: `gpt-4o-mini` for per-competitor analysis (cheaper, faster) and `gpt-4o` for the final consolidated briefing (higher quality). | Cost optimization strategy |
| OpenAI API keys are available at [platform.openai.com](https://platform.openai.com). | Credential setup |
| Google Sheets sheet ID is the string between `/d/` and `/edit` in the sheet URL. Example: `https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit` → use `YOUR_SHEET_ID`. | Sheet ID extraction |
| The `Fetch Competitor Page` node uses `continueOnFail: true`, meaning if a competitor site is unreachable, the workflow continues. The downstream code node will detect empty/failed content and pass an error indicator to the AI, which is instructed to report the issue. | Resilience design |
| Pages rendered entirely by JavaScript (SPAs) will yield minimal text since this workflow does not use a headless browser. Consider using a ScrapingBee or Puppeteer-based approach for such sites. | Limitation of HTTP-only scraping |
| Gmail has a daily sending limit (500 emails for consumer accounts, higher for Google Workspace). If monitoring many recipients, consider using a dedicated sending service. | Gmail quota constraint |
| The 4,500-character cap per competitor page may truncate content from very long pages. Adjust in the `Extract Text and Build AI Prompt` code node's `.slice(0, 4500)` value if needed. | Text extraction limit |
| To change the briefing delivery time, modify the Schedule Trigger. For precise cron-based scheduling (e.g., `0 8 * * *` for 8 AM daily), use the cron expression mode in the trigger node. | Scheduling flexibility |
| Adding more competitors only requires editing the `COMPETITORS` array in the Configure node. The workflow handles any number of items automatically through n8n's item-based processing. | Scalability |
| The `Briefing Summary` field logged to Google Sheets may approach the 50,000-character cell limit for briefings covering many competitors. Monitor sheet behavior if this occurs. | Google Sheets cell limit |