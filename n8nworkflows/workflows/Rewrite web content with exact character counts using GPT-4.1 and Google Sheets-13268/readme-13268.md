Rewrite web content with exact character counts using GPT-4.1 and Google Sheets

https://n8nworkflows.xyz/workflows/rewrite-web-content-with-exact-character-counts-using-gpt-4-1-and-google-sheets-13268


# Rewrite web content with exact character counts using GPT-4.1 and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Rewrite web content with exact character counts using GPT-4.1 and Google Sheets

**Purpose:**  
This workflow collects a rewriting request via an n8n Form, fetches a reference webpage, converts its HTML to Markdown, rewrites the content using GPT‑4.1 with *ultra-strict per-line exact character count constraints* (skip URLs and footers, keep form labels), then runs a second GPT‑4.1 agent to compare original vs rewritten text and finally logs a structured comparison (including character counts) into Google Sheets.

**Typical use cases:**
- Rewriting service pages while preserving line lengths to keep layout/design stable.
- Producing auditable rewrite diffs (original vs rewritten) for QA.
- Enforcing strict “do not touch” content rules (form labels, URLs, footers).

### Logical Blocks
**1.1 Input Reception (Form) → URL fetch**  
Collect request fields, then retrieve raw HTML from the reference URL.

**1.2 Content Normalization (HTML → Markdown)**  
Convert fetched HTML into Markdown so the rewriting agent operates on cleaner, line-based content.

**1.3 AI Rewriting (GPT‑4.1)**  
Rewrite line-by-line with exact character count matching and strict skip/keep rules.

**1.4 AI Comparison + Structured Parsing (GPT‑4.1 + parser)**  
Compare original Markdown vs rewritten output, force JSON output, and parse/fix into a predictable schema.

**1.5 Results Logging (Split + Google Sheets append)**  
Split comparisons into rows and append to a Google Sheet with old/new text and length fields.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Form) → URL fetch
**Overview:**  
Accepts a human request (client + instruction + reference URL), then fetches the referenced page content for rewriting.

**Nodes involved:**
- Submit Content Rewriting Request
- Fetch Reference URL Content

#### Node: Submit Content Rewriting Request
- **Type / role:** `Form Trigger` (n8n form webhook entrypoint)
- **Configuration (interpreted):**
  - Form title: “Service Page Content Creation”
  - Description: “As soon as you fill the form, then we can start it.”
  - Fields:
    - Client ID (required)
    - Service Page Keyword (optional)
    - Instruction (required)
    - Reference Url (optional in form config, but required logically for downstream HTTP)
- **Key variables/fields produced:**
  - `$json["Client ID"]`, `$json["Instruction"]`, `$json["Reference Url"]`, `$json["Service Page Keyword"]`
- **Connections:**
  - Output → **Fetch Reference URL Content**
- **Edge cases / failures:**
  - Missing/blank “Reference Url” leads to invalid HTTP request URL downstream.
  - User-entered URL may be blocked, require cookies, or return non-HTML.
- **Version notes:** typeVersion `2.3` (formTrigger feature set depends on n8n version; ensure your n8n supports Form Trigger nodes).

#### Node: Fetch Reference URL Content
- **Type / role:** `HTTP Request` (fetches page content)
- **Configuration (interpreted):**
  - URL: expression `={{ $json["Reference Url"] }}`
  - Method: **POST** (not typical for fetching a webpage; many sites expect GET)
  - Response is expected in a field like `$json.data` (as used later)
- **Connections:**
  - Input ← Submit Content Rewriting Request
  - Output → **Convert HTML to Markdown**
- **Edge cases / failures:**
  - **405 Method Not Allowed / 403 Forbidden** if the server disallows POST.
  - Anti-bot protections (Cloudflare), redirects, login walls.
  - Large HTML pages may exceed memory or timeouts.
  - If the response does not populate `data`, the Markdown node will fail/produce empty output.
- **Version notes:** typeVersion `4.3` (HTTP Request node behavior differs across versions; confirm response shape and default “response format” settings).

**Sticky note context (applies to this block):**  
“## Content Fetching … Receives rewriting request via form, fetches the reference URL's HTML content, and converts it to clean Markdown for AI processing.”

---

### 2.2 Content Normalization (HTML → Markdown)
**Overview:**  
Transforms fetched HTML into Markdown to make content more consistent for line-by-line rewriting rules.

**Nodes involved:**
- Convert HTML to Markdown

#### Node: Convert HTML to Markdown
- **Type / role:** `Markdown` node (HTML → Markdown conversion)
- **Configuration (interpreted):**
  - HTML input: `={{ $json.data }}`
  - Produces Markdown into `data` (used downstream as `$('Convert HTML to Markdown').item.json.data`)
- **Connections:**
  - Input ← Fetch Reference URL Content
  - Output → Rewrite Content with Exact Character Count
- **Edge cases / failures:**
  - If HTML is empty or binary, conversion yields empty/garbled Markdown.
  - Complex pages may convert into noisy Markdown (menus, repeated nav, scripts removed inconsistently), impacting rewrite quality and line boundaries.
- **Version notes:** typeVersion `1` (ensure the Markdown node is available in your n8n build).

---

### 2.3 AI Content Rewriting (GPT‑4.1)
**Overview:**  
Uses a LangChain Agent with GPT‑4.1 to rewrite content while enforcing **exact character count per line**, preserving form labels, and skipping URLs/footers.

**Nodes involved:**
- Rewrite Content with Exact Character Count
- OpenAI GPT-4.1 Rewriting Model

#### Node: OpenAI GPT-4.1 Rewriting Model
- **Type / role:** `lmChatOpenAi` (LangChain Chat Model provider)
- **Configuration (interpreted):**
  - Model: `gpt-4.1`
  - Timeout: `100000` ms
  - Credential: “Open AI - Misc for testing”
- **Connections:**
  - Provides `ai_languageModel` input to **Rewrite Content with Exact Character Count**
- **Edge cases / failures:**
  - Invalid/expired OpenAI key, quota exceeded, model access denied.
  - Timeouts on large pages.
- **Version notes:** typeVersion `1.2` for the OpenAI LangChain node (parameters and model list can differ across n8n versions).

#### Node: Rewrite Content with Exact Character Count
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` (agent orchestrating rewrite prompt + model)
- **Configuration (interpreted):**
  - Text input: `={{ $json.data }}` (Markdown content)
  - Prompt type: “define”
  - System message: “ULTRA-STRICT SEO Content Rewriter – EXACT Character Count Enforcer” with strict rules:
    - **Every line must match original character count exactly**
    - Keep simple labels (Name/Email/Phone/etc.)
    - Skip URLs entirely
    - Skip footers entirely (explicit “IMPORTANT UPDATE”)
    - Output only rewritten content; no explanations
- **Connections:**
  - Input ← Convert HTML to Markdown
  - `ai_languageModel` ← OpenAI GPT-4.1 Rewriting Model
  - Output → Compare Original vs Rewritten Content (expects `$json.output` to contain rewritten text)
- **Edge cases / failures:**
  - Character-count perfection is extremely hard on long/noisy Markdown; model may violate constraints.
  - Markdown conversion may create inconsistent line breaks; “line” definition becomes ambiguous.
  - Skipping URLs/footers may change total line alignment vs original; downstream comparison expects mapping.
  - If agent output key differs (not `output`), downstream comparison breaks.
- **Version notes:** typeVersion `3` (agent node capabilities vary by n8n release).

**Sticky note context (applies to this block):**  
“## AI Content Rewriting … Rewrites content line-by-line maintaining EXACT character count. Keeps form labels unchanged, skips URLs/footers, rewrites marketing content.”

---

### 2.4 AI Comparison + Structured Parsing (GPT‑4.1 + parser)
**Overview:**  
A second agent compares original Markdown and rewritten output, must return JSON only, then a structured output parser validates and auto-fixes into a predictable schema.

**Nodes involved:**
- Compare Original vs Rewritten Content
- OpenAI GPT-4.1 Comparison Model
- Parse Comparison to Structured Format
- OpenAI GPT-4o-mini Parser Model

#### Node: OpenAI GPT-4.1 Comparison Model
- **Type / role:** `lmChatOpenAi` (chat model for comparison agent)
- **Configuration (interpreted):**
  - Model: `gpt-4.1`
  - Timeout: `100000` ms
  - Credential: “Open AI - Misc for testing”
- **Connections:**
  - Provides `ai_languageModel` input to **Compare Original vs Rewritten Content**
- **Edge cases / failures:**
  - Same OpenAI risks: auth/quota/model access, timeouts.
- **Version notes:** typeVersion `1.2`

#### Node: Compare Original vs Rewritten Content
- **Type / role:** LangChain `agent` (comparison + JSON generation)
- **Configuration (interpreted):**
  - Input text is a composed prompt:
    - ORIGINAL INPUT: `{{ $('Convert HTML to Markdown').item.json.data }}`
    - REWRITTEN OUTPUT: `{{ $json.output }}`
  - System message requires:
    - Line/sentence segmentation, mapping, concise comparison
    - Output **ONLY** valid JSON object with `comparisons` array
    - Verify **character count and word count** match (except skipped lines)
    - For skipped lines, use `"[SKIPPED - Footer/URL]"`
  - `hasOutputParser: true` meaning it relies on an output parser connection.
- **Connections:**
  - Input ← Rewrite Content with Exact Character Count
  - `ai_languageModel` ← OpenAI GPT-4.1 Comparison Model
  - `ai_outputParser` ← Parse Comparison to Structured Format
  - Output (parsed) → Split Comparison into Individual Rows
- **Edge cases / failures:**
  - The agent is asked to “verify exact character and word count” but the rewriting agent only enforces character count; word count equality may often fail.
  - If the model outputs invalid JSON, parser must fix; auto-fix may still fail on badly formatted outputs.
  - Using `$('Convert HTML to Markdown').item.json.data` assumes same item context; if multiple items exist, mapping may mismatch.
- **Version notes:** typeVersion `3`

#### Node: Parse Comparison to Structured Format
- **Type / role:** `outputParserStructured` (forces schema, auto-fixes)
- **Configuration (interpreted):**
  - `autoFix: true` (can call parser model to repair invalid JSON)
  - Schema example expects:
    - `comparisons[]` objects with:
      - `old_text`
      - `ai_suggested_text`
      - `Old_text_count`
      - `ai_suggested_text_Count`
- **Connections:**
  - `ai_languageModel` ← OpenAI GPT-4o-mini Parser Model (used for auto-fix / parsing assistance)
  - Output parser connected to **Compare Original vs Rewritten Content** via `ai_outputParser`
- **Edge cases / failures:**
  - If comparison agent omits the count fields, parser may invent or leave blanks depending on fix behavior.
  - Count field names are inconsistent in casing (e.g., `Old_text_count` vs `ai_suggested_text_Count`)—easy to break mappings later.
- **Version notes:** typeVersion `1.3`

#### Node: OpenAI GPT-4o-mini Parser Model
- **Type / role:** `lmChatOpenAi` (cheap/fast model to repair/normalize JSON)
- **Configuration (interpreted):**
  - Model: `gpt-4o-mini`
  - Credential: “OpenAI - Keyword Research”
- **Connections:**
  - Provides `ai_languageModel` to **Parse Comparison to Structured Format**
- **Edge cases / failures:**
  - Parser model might “fix” JSON but alter content unexpectedly if prompt is ambiguous.
- **Version notes:** typeVersion `1.2`

**Sticky note context (applies to this block):**  
“## Comparison Analysis … Compares original vs rewritten content line-by-line, creates structured JSON comparison with character counts, and parses into proper format.”

---

### 2.5 Results Logging (Split + Google Sheets append)
**Overview:**  
Transforms the comparisons array into individual items and appends each record to Google Sheets for auditing.

**Nodes involved:**
- Split Comparison into Individual Rows
- Log Comparison to Google Sheets

#### Node: Split Comparison into Individual Rows
- **Type / role:** `Split Out` (explode array into items)
- **Configuration (interpreted):**
  - Field to split: `output.comparisons`
  - Produces one item per comparison object.
- **Connections:**
  - Input ← Compare Original vs Rewritten Content
  - Output → Log Comparison to Google Sheets
- **Edge cases / failures:**
  - If parsed output doesn’t contain `output.comparisons`, node outputs nothing or errors.
- **Version notes:** typeVersion `1`

#### Node: Log Comparison to Google Sheets
- **Type / role:** `Google Sheets` (append rows)
- **Configuration (interpreted):**
  - Operation: **Append**
  - Document: `content_comparision` (Spreadsheet ID `1V4MjvK0yN2f7aBBqPG_O3vEisVOalRl2aeNldnS9SIU`)
  - Sheet: `Sheet1` (gid=0)
  - Column mapping (defineBelow):
    - Old Text → `={{ $json.old_text }}`
    - Old Text Length → `={{ $json.Old_text_count }}`
    - Ai Suggested Text → `={{ $json.ai_suggested_text }}`
    - Ai Suggested Text Length → `={{ $json.ai_suggested_text_Count }}`
  - Credential: Google Sheets OAuth2 “AkshSHEET”
- **Connections:**
  - Input ← Split Comparison into Individual Rows
- **Edge cases / failures:**
  - Spreadsheet permissions missing (file not shared with the OAuth user/service).
  - Column name mismatch vs sheet headers.
  - Count fields may be missing due to upstream model output; appended rows may have blanks.
- **Version notes:** typeVersion `4.7` (Google Sheets node options differ across versions).

**Sticky note context (applies to this block):**  
“## Results Logging … Splits comparison data into individual rows and logs each comparison to Google Sheets with old/new text and character counts.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Submit Content Rewriting Request | n8n-nodes-base.formTrigger | Entry point: collect rewriting request via form | — | Fetch Reference URL Content | ## Character-Preserving SEO Content Rewriter… (full workflow description + setup steps). / ## Content Fetching… |
| Fetch Reference URL Content | n8n-nodes-base.httpRequest | Fetch HTML content from Reference URL | Submit Content Rewriting Request | Convert HTML to Markdown | ## Character-Preserving SEO Content Rewriter… / ## Content Fetching… |
| Convert HTML to Markdown | n8n-nodes-base.markdown | Normalize HTML into Markdown | Fetch Reference URL Content | Rewrite Content with Exact Character Count | ## Character-Preserving SEO Content Rewriter… / ## Content Fetching… |
| Rewrite Content with Exact Character Count | @n8n/n8n-nodes-langchain.agent | Rewrite with strict per-line exact character counts; skip URLs/footers | Convert HTML to Markdown | Compare Original vs Rewritten Content | ## Character-Preserving SEO Content Rewriter… / ## AI Content Rewriting… |
| OpenAI GPT-4.1 Rewriting Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for rewriting agent | — | Rewrite Content with Exact Character Count (ai_languageModel) | ## Character-Preserving SEO Content Rewriter… / ## AI Content Rewriting… |
| Compare Original vs Rewritten Content | @n8n/n8n-nodes-langchain.agent | Compare original vs rewritten; produce JSON comparisons | Rewrite Content with Exact Character Count | Split Comparison into Individual Rows | ## Character-Preserving SEO Content Rewriter… / ## Comparison Analysis… |
| OpenAI GPT-4.1 Comparison Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for comparison agent | — | Compare Original vs Rewritten Content (ai_languageModel) | ## Character-Preserving SEO Content Rewriter… / ## Comparison Analysis… |
| Parse Comparison to Structured Format | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce/fix JSON schema for comparisons | OpenAI GPT-4o-mini Parser Model (ai_languageModel) | Compare Original vs Rewritten Content (ai_outputParser) | ## Character-Preserving SEO Content Rewriter… / ## Comparison Analysis… |
| OpenAI GPT-4o-mini Parser Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider to auto-fix/normalize JSON | — | Parse Comparison to Structured Format (ai_languageModel) | ## Character-Preserving SEO Content Rewriter… / ## Comparison Analysis… |
| Split Comparison into Individual Rows | n8n-nodes-base.splitOut | Explode comparisons array into per-row items | Compare Original vs Rewritten Content | Log Comparison to Google Sheets | ## Character-Preserving SEO Content Rewriter… / ## Results Logging… |
| Log Comparison to Google Sheets | n8n-nodes-base.googleSheets | Append comparison rows to a Google Sheet | Split Comparison into Individual Rows | — | ## Character-Preserving SEO Content Rewriter… / ## Results Logging… |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation | — | — | (Contains the full workflow description + setup steps text) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for fetching block | — | — | (Content Fetching note) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for rewriting block | — | — | (AI Content Rewriting note) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for comparison block | — | — | (Comparison Analysis note) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for logging block | — | — | (Results Logging note) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add “Form Trigger” node** named **Submit Content Rewriting Request**  
   - Form title: `Service Page Content Creation`  
   - Description: `As soon as you fill the form, then we can start it.`  
   - Fields:
     - `Client ID` (required)
     - `Service Page Keyword` (optional)
     - `Instruction` (required)
     - `Reference Url` (recommended to set required to avoid downstream failures)

3. **Add “HTTP Request” node** named **Fetch Reference URL Content**  
   - URL: `={{ $json["Reference Url"] }}`
   - Method: **POST** (use **GET** if your target sites reject POST)
   - Ensure the node outputs the response body into a property you will reference next (this workflow expects `data`).

4. **Connect**: Submit Content Rewriting Request → Fetch Reference URL Content.

5. **Add “Markdown” node** named **Convert HTML to Markdown**  
   - Operation: HTML to Markdown
   - HTML field: `={{ $json.data }}`

6. **Connect**: Fetch Reference URL Content → Convert HTML to Markdown.

7. **Add OpenAI chat model node** (`lmChatOpenAi`) named **OpenAI GPT-4.1 Rewriting Model**  
   - Model: `gpt-4.1`
   - Timeout: `100000` ms
   - **Credentials:** create/select an OpenAI API credential with access to `gpt-4.1`.

8. **Add LangChain “Agent” node** named **Rewrite Content with Exact Character Count**  
   - Text input: `={{ $json.data }}`  
   - Prompt type: `define`  
   - System message: paste the “ULTRA-STRICT SEO Content Rewriter – EXACT Character Count Enforcer” instructions (as in the workflow).  
   - Connect the model:
     - From **OpenAI GPT-4.1 Rewriting Model** connect its **ai_languageModel** output to the agent’s **ai_languageModel** input.

9. **Connect**: Convert HTML to Markdown → Rewrite Content with Exact Character Count.

10. **Add OpenAI chat model node** named **OpenAI GPT-4.1 Comparison Model**  
    - Model: `gpt-4.1`
    - Timeout: `100000` ms
    - Credentials: same or another OpenAI credential with access.

11. **Add “Structured Output Parser” node** named **Parse Comparison to Structured Format**  
    - Enable **Auto-fix**
    - Provide a schema example with:
      - `comparisons[]` containing objects with `old_text`, `ai_suggested_text`, `Old_text_count`, `ai_suggested_text_Count`.

12. **Add OpenAI chat model node** named **OpenAI GPT-4o-mini Parser Model**  
    - Model: `gpt-4o-mini`
    - Credentials: an OpenAI credential (can be cheaper) for parsing/repair.

13. **Connect parser model → parser:**  
    - OpenAI GPT-4o-mini Parser Model **ai_languageModel** → Parse Comparison to Structured Format **ai_languageModel**.

14. **Add LangChain “Agent” node** named **Compare Original vs Rewritten Content**  
    - Prompt type: `define`
    - Text prompt should include:
      - Original: `{{ $('Convert HTML to Markdown').item.json.data }}`
      - Rewritten: `{{ $json.output }}`
    - System message: enforce “JSON only” output with `comparisons` array and the skip markers.
    - Enable output parsing by connecting:
      - Parse Comparison to Structured Format **ai_outputParser** → Compare Original vs Rewritten Content **ai_outputParser** input.
    - Connect model:
      - OpenAI GPT-4.1 Comparison Model **ai_languageModel** → Compare Original vs Rewritten Content **ai_languageModel**.

15. **Connect**: Rewrite Content with Exact Character Count → Compare Original vs Rewritten Content.

16. **Add “Split Out” node** named **Split Comparison into Individual Rows**  
    - Field to split out: `output.comparisons`

17. **Connect**: Compare Original vs Rewritten Content → Split Comparison into Individual Rows.

18. **Add “Google Sheets” node** named **Log Comparison to Google Sheets**  
    - Operation: **Append**
    - Set Spreadsheet (Document) and Sheet (tab)
    - Map columns:
      - Old Text → `={{ $json.old_text }}`
      - Old Text Length → `={{ $json.Old_text_count }}`
      - Ai Suggested Text → `={{ $json.ai_suggested_text }}`
      - Ai Suggested Text Length → `={{ $json.ai_suggested_text_Count }}`
    - **Credentials:** connect Google Sheets OAuth2 (ensure the authenticated account has edit access to the spreadsheet).

19. **Connect**: Split Comparison into Individual Rows → Log Comparison to Google Sheets.

20. **Run a test** using the form URL, then verify Google Sheets receives rows.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Character-Preserving SEO Content Rewriter” sticky note includes: workflow purpose, how it works (1–8), and setup steps (Google OAuth, OpenAI keys, sheet URL, share form URL, review in Sheets). | Internal canvas documentation (sticky note text embedded in workflow) |
| Google Sheet used in this workflow: `content_comparision` | https://docs.google.com/spreadsheets/d/1V4MjvK0yN2f7aBBqPG_O3vEisVOalRl2aeNldnS9SIU/edit#gid=0 |

