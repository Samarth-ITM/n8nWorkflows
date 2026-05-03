Repurpose white papers into LinkedIn PDFs and blog posts with BrowserAct and GPT-4o

https://n8nworkflows.xyz/workflows/repurpose-white-papers-into-linkedin-pdfs-and-blog-posts-with-browseract-and-gpt-4o-13376


# Repurpose white papers into LinkedIn PDFs and blog posts with BrowserAct and GPT-4o

## 1. Workflow Overview

**Purpose:**  
This workflow repurposes long-form white papers (web pages or PDFs reachable by URL) into:
1) a **LinkedIn carousel script + caption**,  
2) an **HTML blog post**, and  
3) a **generated carousel PDF** suitable for uploading to LinkedIn—then saves results back into **Google Sheets** and notifies via **Slack**.

**Primary use cases:**
- Marketing/content teams converting technical white papers into distribution-ready assets.
- Automated content operations where URLs are maintained in a spreadsheet “queue”.

### 1.1 Input Reception (Google Sheets)
Reads a list of white paper URLs from a Google Sheet.

### 1.2 Batch Iteration + Completion Notification
Processes the sheet rows in a loop (batching) and posts a Slack message on completion.

### 1.3 Scraping/Extraction (BrowserAct)
For each URL, BrowserAct loads the page/PDF viewer and extracts the raw text.

### 1.4 AI Transformation (OpenRouter GPT-4-class model via LangChain Agent)
An AI agent converts scraped text into structured LinkedIn carousel data and an HTML blog post, enforced by a structured output parser.

### 1.5 PDF Generation + Persistence (APITemplate.io + Google Sheets update)
Creates a PDF from the carousel data, then updates the original row with the PDF link and generated content.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Google Sheets)
**Overview:** Fetches rows from Google Sheets that contain the target white paper URLs. Output becomes the item list to process.

**Nodes involved:**
- **Manual trigger**
- **Get links** (Google Sheets)

#### Node: Manual trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — entry point for manual runs.
- **Configuration choices:** No parameters; user starts execution from the editor.
- **Outputs:** Sends a single trigger item into **Get links**.
- **Edge cases:** None (manual-only). For production, you’d likely replace with Schedule or Webhook.

#### Node: Get links
- **Type / role:** `n8n-nodes-base.googleSheets` — reads rows from a spreadsheet.
- **Configuration (interpreted):**
  - Uses a Google Sheets OAuth2 credential.
  - Targets document **“White Paper to Social Media Converter”** and sheet **Sheet1** (`gid=0`).
  - Operation is not explicitly shown in the JSON snippet, but given the node name and placement it is used to **read/get all rows**.
- **Key fields expected in rows:**
  - `Target Page Url` (used downstream to scrape)
  - `row_number` (used later for updates; n8n Google Sheets nodes often include this for update operations)
- **Outputs / connections:** Outputs items to **Loop Over Items**.
- **Edge cases / failures:**
  - OAuth expiration / missing permissions.
  - Missing `Target Page Url` column or empty cells.
  - Very large sheets may lead to long runtimes/timeouts depending on n8n limits.

---

### Block 2 — Batch Iteration + Completion Notification
**Overview:** Iterates through sheet rows using Split in Batches. It also triggers a Slack notification from the loop node’s “done” path.

**Nodes involved:**
- **Loop Over Items**
- **Notify on completion** (Slack)

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — processes items in batches; enables looping until all items handled.
- **Configuration (interpreted):**
  - Uses default options (batch size not explicitly set in provided JSON; n8n defaults typically apply unless configured).
- **Connections:**
  - **Input:** from **Get links**
  - **Output 0 (main index 0):** goes to **Scrape the data** (process current batch item(s))
  - **Output 1 (main index 1):** goes to **Notify on completion** (runs once when loop finishes)
  - **Loop-back:** **Update Database → Loop Over Items** to continue to next batch item.
- **Edge cases / failures:**
  - If downstream nodes error and “continue on fail” is not enabled, the loop stops early.
  - If `row_number` is missing downstream update matching will fail.

#### Node: Notify on completion
- **Type / role:** `n8n-nodes-base.slack` — posts a completion message.
- **Configuration (interpreted):**
  - Posts text: **“All the white papers extracted from links and updated in Google Sheets.”**
  - Sends to channel ID `C09KLV9DJSX` (cached name: `all-browseract-workflow-test`)
  - `executeOnce: true` (prevents multiple posts in certain multi-branch scenarios)
- **Connections:**
  - **Input:** from **Loop Over Items** completion output.
- **Edge cases / failures:**
  - Slack auth/token revoked.
  - Bot not in channel.
  - Rate limits (unlikely for a single message).

---

### Block 3 — Scraping/Extraction (BrowserAct)
**Overview:** For each row, BrowserAct navigates to the target URL and returns extracted text content (suitable for AI processing).

**Nodes involved:**
- **Scrape the data** (BrowserAct)

#### Node: Scrape the data
- **Type / role:** `n8n-nodes-browseract.browserAct` — runs a BrowserAct workflow to scrape/extract content.
- **Configuration (interpreted):**
  - Mode: `WORKFLOW`
  - BrowserAct workflow ID: `76074693265123883`
  - Passes an input named `White_Paper_Link` into BrowserAct:
    - `input-White_Paper_Link = {{ $json["Target Page Url"] }}`
  - The schema indicates the input is optional; if blank BrowserAct default would be used, but this workflow intends to provide it per row.
- **Expected output shape:**
  - Downstream references `{{ $json.output.string }}`, implying BrowserAct returns an object like:
    - `output: { string: "<extracted text>" }`
- **Connections:**
  - **Input:** from **Loop Over Items**
  - **Output:** to **Convert whitepaper to carousel**
- **Edge cases / failures:**
  - URL unreachable, blocked by bot protection, requires authentication, or is a download with unusual headers.
  - PDF viewer or dynamic content not handled by the BrowserAct workflow.
  - Large documents: extraction may truncate or exceed time limits.
  - BrowserAct credential issues / workflow ID mismatch.

---

### Block 4 — AI Transformation (OpenRouter + Agent + Structured Parser)
**Overview:** Uses an OpenRouter chat model (GPT-4-class) through a LangChain Agent node to generate two assets: a LinkedIn carousel script and a blog post in HTML. A structured output parser enforces JSON format and can auto-fix malformed JSON.

**Nodes involved:**
- **OpenRouter Chat Model**
- **Structured Output Parser**
- **Convert whitepaper to carousel** (LangChain Agent)

#### Node: OpenRouter Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — provides the LLM backend to LangChain nodes.
- **Configuration (interpreted):**
  - Uses OpenRouter API credential.
  - Options not explicitly set; model choice may be in credential defaults or node UI (not shown in JSON).
- **Connections:**
  - Connected via **AI Language Model** ports to:
    - **Convert whitepaper to carousel**
    - **Structured Output Parser** (shared model context for parsing/autofix)
- **Edge cases / failures:**
  - OpenRouter key invalid, quota exceeded.
  - Model unavailable or changed behavior affecting JSON compliance.
  - Latency/timeouts for very long inputs (white papers can be large).

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — validates and parses LLM output into a defined JSON schema.
- **Configuration (interpreted):**
  - `autoFix: true` — attempts to repair near-valid JSON.
  - Provides a schema example with keys:
    - `linkedin_carousel_data`: `{ caption, slides: [{page,title,body}...] }`
    - `blog_post_html`: `"<h1>..."`  
  - **Important mismatch to note:** the agent’s system message demands keys `linkedin` and `blog_post`, while this parser example expects `linkedin_carousel_data` and `blog_post_html`. The downstream nodes reference `linkedin_carousel_data` and `blog_post_html`, so the parser (and/or agent) must ultimately output those keys to avoid failures.
- **Connections:**
  - Output parser is attached to the agent via the **AI Output Parser** port.
- **Edge cases / failures:**
  - If the model returns keys matching the agent prompt (`linkedin`, `blog_post`) but not the parser’s schema, parsing may fail or autoFix may produce unexpected remapping.
  - Very large outputs may exceed parser limits.
- **Version considerations:** Output parser behavior (auto-fix strictness) can differ across node versions; this workflow uses `typeVersion: 1.3`.

#### Node: Convert whitepaper to carousel
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompting and uses the connected model + parser.
- **Configuration (interpreted):**
  - Prompt input:
    - `Input : {{ $json.output.string }}`
  - System message instructs the agent to generate **one JSON object** with:
    - LinkedIn carousel: caption + 5 slides (headline/content)
    - Blog post: HTML body starting with `<h1>`, includes Key Takeaways section
  - `hasOutputParser: true` — uses **Structured Output Parser** for validation.
- **Connections:**
  - **Input:** from **Scrape the data**
  - **AI Language Model input:** from **OpenRouter Chat Model**
  - **AI Output Parser input:** from **Structured Output Parser**
  - **Output:** to **Create a carousel PDF**
- **Edge cases / failures:**
  - Token/context overflow from long extracted text (common with full white papers).
    - Mitigation: summarize in BrowserAct, chunk text, or add a pre-summarization step.
  - JSON non-compliance despite instructions → parser may fail.
  - Content safety policies: model may refuse certain content if the paper includes restricted material.

---

### Block 5 — PDF Generation + Persistence (APITemplate.io + Google Sheets)
**Overview:** Takes the AI-generated carousel data and renders it via APITemplate.io into a downloadable PDF. Then updates the original Google Sheet row with the PDF link and generated assets, and loops to the next item.

**Nodes involved:**
- **Create a carousel PDF** (APITemplate.io)
- **Update Database** (Google Sheets)

#### Node: Create a carousel PDF
- **Type / role:** `n8n-nodes-base.apiTemplateIo` — generates a PDF from a template.
- **Configuration (interpreted):**
  - Resource: `pdf`
  - Template ID: `63677b23c0543794`
  - `download: true` — returns a downloadable asset and a `download_url`.
  - Template data payload is provided as JSON:
    - `propertiesJson = {{ $('Convert whitepaper to carousel').first().json.output.linkedin_carousel_data }}`
  - **Dependency note:** This requires the agent output to contain `output.linkedin_carousel_data` with fields matching your APITemplate placeholders.
- **Connections:**
  - **Input:** from **Convert whitepaper to carousel**
  - **Output:** to **Update Database**
- **Edge cases / failures:**
  - Template mismatch: missing fields cause blank areas or API errors.
  - Large text overflows slide layout.
  - APITemplate rate limits/quota, invalid API key.

#### Node: Update Database
- **Type / role:** `n8n-nodes-base.googleSheets` — updates the corresponding row with generated results.
- **Configuration (interpreted):**
  - Operation: `update`
  - Matches row using `row_number` (`matchingColumns: ["row_number"]`)
  - Writes columns:
    - **PDF Link** = `{{ $json.download_url }}` (from APITemplate output)
    - **Blog Post** = `{{ $('Convert whitepaper to carousel').first().json.output.blog_post_html }}`
    - **Linkdin Post** = `{{ $('Convert whitepaper to carousel').first().json.output.linkedin_carousel_data }}`
    - **row_number** = `{{ $('Loop Over Items').item.json.row_number }}`
    - **Target Page Url** is set to an empty expression (`"="`) which effectively writes an empty string/blank (likely unintended).
- **Connections:**
  - **Input:** from **Create a carousel PDF**
  - **Output:** loops back to **Loop Over Items** to process next row.
- **Edge cases / failures:**
  - If `row_number` is absent or not aligned with the sheet’s row indexing, update will fail or update the wrong row.
  - Writing structured JSON into a single cell (`Linkdin Post`) may exceed cell limits or become hard to read; consider stringifying or splitting into separate columns.
  - The `Target Page Url` overwrite-to-blank is risky (data loss). Prefer leaving it unmapped or mapping the original value.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual trigger | Manual Trigger | Manual workflow start | — | Get links | ### 📥 Step 1: Input & Scraping — The workflow reads a list of white paper URLs from Google Sheets. BrowserAct then navigates to each link and extracts the full text content, handling PDF viewers or web pages automatically. |
| Get links | Google Sheets | Read list of URLs/rows | Manual trigger | Loop Over Items | ### 📥 Step 1: Input & Scraping — The workflow reads a list of white paper URLs from Google Sheets. BrowserAct then navigates to each link and extracts the full text content, handling PDF viewers or web pages automatically. |
| Loop Over Items | Split In Batches | Iterate through sheet rows | Get links; Update Database (loop-back) | Scrape the data; Notify on completion | ### 📥 Step 1: Input & Scraping — The workflow reads a list of white paper URLs from Google Sheets. BrowserAct then navigates to each link and extracts the full text content, handling PDF viewers or web pages automatically. |
| Notify on completion | Slack | Post completion message | Loop Over Items (done output) | — | ### 📥 Step 1: Input & Scraping — The workflow reads a list of white paper URLs from Google Sheets. BrowserAct then navigates to each link and extracts the full text content, handling PDF viewers or web pages automatically. |
| Scrape the data | BrowserAct | Extract white paper text from URL/PDF | Loop Over Items | Convert whitepaper to carousel | ### 📥 Step 1: Input & Scraping — The workflow reads a list of white paper URLs from Google Sheets. BrowserAct then navigates to each link and extracts the full text content, handling PDF viewers or web pages automatically. |
| OpenRouter Chat Model | LangChain Chat Model (OpenRouter) | LLM provider for generation/parsing | — | Convert whitepaper to carousel; Structured Output Parser (AI ports) | ### 🎨 Step 2: Asset Generation — AI drafts a LinkedIn carousel + caption and an HTML blog post. Carousel script is sent to APITemplate.io to generate a PDF; results are saved back to Google Sheets. |
| Structured Output Parser | LangChain Structured Output Parser | Enforce/repair JSON output | OpenRouter Chat Model (AI port) | Convert whitepaper to carousel (AI output parser port) | ### 🎨 Step 2: Asset Generation — AI drafts a LinkedIn carousel + caption and an HTML blog post. Carousel script is sent to APITemplate.io to generate a PDF; results are saved back to Google Sheets. |
| Convert whitepaper to carousel | LangChain Agent | Transform scraped text into carousel + blog HTML | Scrape the data | Create a carousel PDF | ### 🎨 Step 2: Asset Generation — AI drafts a LinkedIn carousel + caption and an HTML blog post. Carousel script is sent to APITemplate.io to generate a PDF; results are saved back to Google Sheets. |
| Create a carousel PDF | APITemplate.io | Render carousel into a PDF | Convert whitepaper to carousel | Update Database | ### 🎨 Step 2: Asset Generation — AI drafts a LinkedIn carousel + caption and an HTML blog post. Carousel script is sent to APITemplate.io to generate a PDF; results are saved back to Google Sheets. |
| Update Database | Google Sheets | Update row with PDF link + generated assets | Create a carousel PDF | Loop Over Items | ### 🎨 Step 2: Asset Generation — AI drafts a LinkedIn carousel + caption and an HTML blog post. Carousel script is sent to APITemplate.io to generate a PDF; results are saved back to Google Sheets. |
| Documentation | Sticky Note | Workflow notes and setup requirements | — | — | ## ⚡ Workflow Overview & Setup — Requirements: BrowserAct, OpenRouter (GPT-4), Google Sheets, APITemplate.io, Slack. Mandatory: BrowserAct template “White Paper to Social Media Converter”. Links: https://docs.browseract.com |
| Step 1 Explanation | Sticky Note | Explains input + scraping stage | — | — | ### 📥 Step 1: Input & Scraping — The workflow reads a list of white paper URLs from Google Sheets. BrowserAct then navigates to each link and extracts the full text content, handling PDF viewers or web pages automatically. |
| Step 3 Explanation | Sticky Note | Explains AI generation + PDF + save-back stage | — | — | ### 🎨 Step 2: Asset Generation — AI drafts LinkedIn carousel + HTML blog post; PDF generated; saved back to Google Sheets. |
| Sticky Note | Sticky Note | Video link | — | — | YouTube: https://www.youtube.com/watch?v=fcWlF0Bza80 |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: *Repurpose white papers from URLs to LinkedIn PDFs and Blog Posts With BrowserAct* (or your preferred name).

2) **Add node: Manual Trigger**  
   - Node type: **Manual Trigger**  
   - No configuration needed.

3) **Add node: Google Sheets (“Get links”)**  
   - Node type: **Google Sheets**  
   - Credentials: configure **Google Sheets OAuth2** (Google Cloud OAuth client; grant Sheets access).  
   - Operation: **Read/Get many rows** (the exact UI label depends on n8n version).  
   - Document: select your spreadsheet (e.g., “White Paper to Social Media Converter”).  
   - Sheet: select `Sheet1` (gid=0).  
   - Ensure the sheet has at least:
     - `Target Page Url` (string URL)
     - a usable row identifier for updates (commonly `row_number` generated by the node or available in output).

4) **Connect**: Manual Trigger → Get links

5) **Add node: Split In Batches (“Loop Over Items”)**  
   - Node type: **Split In Batches**  
   - Set **Batch Size** as desired (e.g., 1–10 depending on rate limits).  
   - Connect: Get links → Loop Over Items

6) **Add node: Slack (“Notify on completion”)**  
   - Node type: **Slack**  
   - Credentials: Slack API (bot token).  
   - Operation: **Post message** (or equivalent).  
   - Channel: choose a channel (e.g., `all-browseract-workflow-test`).  
   - Text: “All the white papers extracted from links and updated in Google Sheets.”  
   - Connect from the **“done/second output”** of Split In Batches to Slack (so it triggers after the loop completes).

7) **Add node: BrowserAct (“Scrape the data”)**  
   - Node type: **BrowserAct**  
   - Credentials: BrowserAct API key.  
   - Mode: **WORKFLOW**  
   - Workflow ID: set to your BrowserAct workflow (in the JSON it is `76074693265123883`).  
   - Configure workflow input mapping:
     - Input field name (example): `White_Paper_Link`
     - Value expression: `{{ $json["Target Page Url"] }}`
   - Connect: Loop Over Items (main output) → Scrape the data

8) **Add node: OpenRouter Chat Model (“OpenRouter Chat Model”)**  
   - Node type: **LangChain Chat Model → OpenRouter**  
   - Credentials: OpenRouter API key.  
   - Choose a model (commonly GPT-4 class, e.g., “gpt-4o” via OpenRouter), set temperature as desired.

9) **Add node: Structured Output Parser (“Structured Output Parser”)**  
   - Node type: **LangChain Output Parser → Structured**  
   - Enable **Auto-fix**.  
   - Define the schema/example that matches what you will use downstream.  
   - Recommendation: align keys with downstream usage, e.g.:
     - `linkedin_carousel_data` (caption, slides, etc.)
     - `blog_post_html` (HTML string)

10) **Add node: LangChain Agent (“Convert whitepaper to carousel”)**  
   - Node type: **LangChain Agent**  
   - Prompt mode: “Define” (custom prompt).  
   - Text input expression: `Input : {{ $json.output.string }}` (assuming BrowserAct returns `output.string`).  
   - System message: paste your instructions for LinkedIn carousel + blog post.  
   - Attach integrations via AI ports:
     - Connect **OpenRouter Chat Model** to the agent’s **Language Model** port.
     - Connect **Structured Output Parser** to the agent’s **Output Parser** port.
   - Connect: Scrape the data → Convert whitepaper to carousel

11) **Add node: APITemplate.io (“Create a carousel PDF”)**  
   - Node type: **APITemplate.io**  
   - Credentials: APITemplate.io API key.  
   - Resource: **PDF**  
   - Template ID: your template (in the JSON: `63677b23c0543794`).  
   - Enable “Download” to get a `download_url`.  
   - Set template properties JSON to the agent output, e.g.:  
     - `{{ $("Convert whitepaper to carousel").first().json.output.linkedin_carousel_data }}`
   - Connect: Convert whitepaper to carousel → Create a carousel PDF

12) **Add node: Google Sheets (“Update Database”)**  
   - Node type: **Google Sheets**  
   - Credentials: same Sheets OAuth2 as earlier.  
   - Operation: **Update**  
   - Document & sheet: same as “Get links”.  
   - Matching column: `row_number`  
   - Map fields to update:
     - `PDF Link` = `{{ $json.download_url }}`
     - `Blog Post` = `{{ $("Convert whitepaper to carousel").first().json.output.blog_post_html }}`
     - `Linkdin Post` = `{{ $("Convert whitepaper to carousel").first().json.output.linkedin_carousel_data }}`
     - `row_number` = `{{ $("Loop Over Items").item.json.row_number }}`
   - Important: **Do not overwrite `Target Page Url`** unless intended; either omit it or map it back to the original value.
   - Connect: Create a carousel PDF → Update Database

13) **Close the loop**  
   - Connect: Update Database → Loop Over Items (to continue with the next batch item).

14) **Test run**  
   - Populate the sheet with at least one `Target Page Url`.  
   - Execute manually.  
   - Verify:
     - BrowserAct returns extracted text.
     - Agent output keys match what APITemplate and Sheets update expect.
     - PDF link is written back and is accessible.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Workflow automatically converts dense white papers (from URLs or PDFs) into engaging social media content… scrapes the text, uses AI… creates a PDF… updates a Google Sheet…” | From sticky note “Documentation” |
| Requirements: BrowserAct, OpenRouter (GPT-4), Google Sheets, APITemplate.io, Slack | From sticky note “Documentation” |
| Mandatory: BrowserAct API (Template: **White Paper to Social Media Converter**) | From sticky note “Documentation” |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | https://www.youtube.com/watch?v=fcWlF0Bza80 |

