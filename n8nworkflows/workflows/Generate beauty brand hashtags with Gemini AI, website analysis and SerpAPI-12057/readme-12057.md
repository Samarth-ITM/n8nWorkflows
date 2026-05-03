Generate beauty brand hashtags with Gemini AI, website analysis and SerpAPI

https://n8nworkflows.xyz/workflows/generate-beauty-brand-hashtags-with-gemini-ai--website-analysis-and-serpapi-12057


# Generate beauty brand hashtags with Gemini AI, website analysis and SerpAPI

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *7 AI Beauty Brand Hashtag Generator (Website-Aware + Multi-Platform)*  
**Stated title:** *Generate beauty brand hashtags with Gemini AI, website analysis and SerpAPI*

**Purpose:**  
Collects beauty brand inputs via a form, scrapes the brand website for context, uses Google Gemini to analyze brand positioning, then uses a LangChain Agent (Gemini) to generate structured hashtags (categorized + popularity). A second LangChain Agent uses SerpAPI live search to fetch “currently trending” platform-specific hashtags (LinkedIn/Instagram/Facebook/X). Finally, it saves the generated hashtags to Google Sheets.

**Target use cases:**
- Beauty/skincare marketers generating brand-aware hashtag sets quickly
- Agencies creating repeatable hashtag research pipelines
- Multi-platform social planning with trend validation via search

### 1.1 Input Reception & Website Scraping
Collect form data, then fetch HTML from the provided website URL.

### 1.2 Brand Analysis (Website-Aware)
Send scraped site content + form context into Gemini to extract structured brand insights (positioning, ingredients, values, tone, etc.).

### 1.3 Hashtag Generation (Structured)
Generate a balanced set of hashtags (general/niche/trending/branded) with popularity estimates, returned as strict JSON.

### 1.4 Platform Trend Research (Live Search + Agent)
Use SerpAPI as a search tool so the agent can discover platform-relevant trending hashtags “right now”.

### 1.5 Persistence
Append final results to Google Sheets with metadata (brand/category/timestamp).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Website Scraping

**Overview:**  
This block gathers brand requirements from a user-facing form and downloads the website HTML content. It provides the raw textual basis for later AI analysis.

**Nodes involved:**
- **Beauty Brand Hashtag Form**
- **Scrape Skincare Website**

#### Node: Beauty Brand Hashtag Form
- **Type / role:** `n8n-nodes-base.formTrigger` — Entry point; hosts a form and triggers workflow on submission.
- **Key configuration (interpreted):**
  - Form title: “Beauty Brand Hashtag Generator”
  - Description: “Generate trending hashtags for your beauty brand”
  - Required fields:
    - “Skincare Brand Website URL”
    - “Brand Name”
    - “Product Category (e.g., skincare, makeup, haircare)”
    - “Target Audience”
    - “Brand Tone (e.g., luxury, natural, playful)”
  - Optional/number field:
    - “Number of Hashtags (5-10)” default `"10"`
- **Key data/variables produced:**
  - Form fields appear in the output JSON; later nodes reference them by label, e.g.:
    - `$json['Skincare Brand Website URL']`
    - `$('Beauty Brand Hashtag Form').item.json['Brand Name']`
  - Submission timestamp: `submittedAt` (used later for formatting).
- **Connections:**
  - **Outgoing (main):** to **Scrape Skincare Website**
- **Edge cases / failures:**
  - Users can enter non-URL text; the HTTP request may fail or return unexpected content.
  - “Number of Hashtags (5-10)” is not validated in workflow logic; users can enter values outside range.
- **Version notes:** Form Trigger `typeVersion 2.3` (features/field handling can differ vs earlier 1.x).

#### Node: Scrape Skincare Website
- **Type / role:** `n8n-nodes-base.httpRequest` — Fetches website content for analysis.
- **Key configuration (interpreted):**
  - URL: expression `={{ $json['Skincare Brand Website URL'] }}`
  - Sends headers with a modern browser-like User-Agent + Accept headers.
  - `allowUnauthorizedCerts: true` (will ignore invalid TLS certificates)
  - Response option: `neverError: true` (prevents node from throwing on non-2xx in some cases; still may produce empty/partial data)
- **Inputs / outputs:**
  - **Input:** form submission item
  - **Output:** response data under the node’s output (referenced later as `{{ $json.data }}` in analysis prompt)
- **Connections:**
  - **Incoming:** Beauty Brand Hashtag Form
  - **Outgoing:** Analyze Website Content
- **Edge cases / failures:**
  - Anti-bot/WAF blocks → returns CAPTCHA/403 HTML; analysis quality degrades.
  - Large HTML pages → token limits in downstream AI node.
  - Non-HTML responses (PDF, redirects, scripts) → irrelevant content.
  - Allowing unauthorized certs can hide real TLS misconfiguration/security issues.
- **Version notes:** HTTP Request `typeVersion 4.3` (field names/options differ across versions).

**Sticky note context applied to this block:**  
“## User Input & Website Scraping … Collects brand details through a form and fetches the brand’s website content…”

---

### Block 2 — Brand Analysis (Website-Aware)

**Overview:**  
Uses Gemini to summarize the scraped website into a structured JSON “brand_analysis” object that guides downstream hashtag generation.

**Nodes involved:**
- **Analyze Website Content**

#### Node: Analyze Website Content
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — Gemini completion with explicit JSON output enabled.
- **Key configuration (interpreted):**
  - Model: `models/gemini-flash-lite-latest`
  - `jsonOutput: true` (node expects JSON)
  - Prompt includes:
    - Raw scraped content: `{{ $json.data }}`
    - Form context: Brand Name, Category, Audience, Tone (pulled from the Form Trigger node)
  - Asks for JSON structure:
    ```json
    {
      "brand_analysis": {
        "positioning": "",
        "key_ingredients": [],
        "brand_values": [],
        "messaging_tone": "",
        "target_demographics": "",
        "unique_selling_points": [],
        "content_themes": []
      }
    }
    ```
- **Inputs / outputs:**
  - **Input:** HTTP response item (contains `data`)
  - **Output:** Gemini response object; downstream node references `{{ $json.content.parts[0].text }}` (note: that is the *text field of Gemini response*, not necessarily already parsed JSON).
- **Connections:**
  - **Incoming:** Scrape Skincare Website
  - **Outgoing (main):** Generate Beauty Hashtags
- **Credentials:**
  - Google Gemini / PaLM credential: `googlePalmApi`
- **Edge cases / failures:**
  - Token overflow if `data` is huge (full HTML). Consider stripping tags or truncating before AI.
  - If website returns noisy HTML, Gemini may output weak/noisy insights.
  - Even with “Return ONLY valid JSON”, models sometimes produce extra text; downstream is referencing the text content, not a parsed object, which may include formatting artifacts.
- **Version notes:** Node `typeVersion 1` (LangChain Gemini node behavior may change across releases).

**Sticky note context applied to this block:**  
“## Brand Analysis … extracts brand positioning, tone, key ingredients, values, and target audience…”

---

### Block 3 — Hashtag Generation (Structured)

**Overview:**  
A LangChain Agent backed by Gemini generates a requested number of hashtags using the brand inputs plus the website-derived analysis, returning structured JSON that is validated/fixed by a structured output parser.

**Nodes involved:**
- **Generate Beauty Hashtags**
- **Gemini Model for Hashtags**
- **Parse Hashtag Output**
- **Google Gemini Chat Model** (present but not actually used by the agent in this wiring)

#### Node: Generate Beauty Hashtags
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Agent that composes a prompt and calls a connected chat model; outputs parsed structure via connected output parser.
- **Key configuration (interpreted):**
  - Prompt builds from form fields and website analysis:
    - Hashtag count: `{{ $('Beauty Brand Hashtag Form').item.json.num_hashtags || 10 }}`
      - Note: the form field label is “Number of Hashtags (5-10)”. Unless n8n maps it to `num_hashtags`, this may always fallback to 10.
    - Website analysis inserted as: `{{ $json.content.parts[0].text }}`
  - System message enforces:
    - Mix of high-volume + niche
    - Include branded hashtags
    - Trend/season relevance
    - Avoid banned/spam-flagged tags
    - Categorize each hashtag
    - Popularity estimate
  - **Return ONLY structured JSON** (relies on output parser)
- **Inputs / outputs:**
  - **Input (main):** output from Analyze Website Content
  - **Output (main):** object containing something like `$json.output.hashtags` (used later)
- **Connections:**
  - **Incoming (main):** Analyze Website Content
  - **AI language model:** Gemini Model for Hashtags
  - **AI output parser:** Parse Hashtag Output
  - **Outgoing (main):** Get Trending Platform Hashtags
- **Edge cases / failures:**
  - If `num_hashtags` variable is missing, count defaults to 10 regardless of user input.
  - If website analysis text is not valid/too long, agent response quality drops.
  - Hashtag policy: “banned hashtags” cannot be reliably validated without an external up-to-date list.
- **Version notes:** Agent `typeVersion 2.2` (agent I/O and parser wiring differs across older versions).

#### Node: Gemini Model for Hashtags
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — Chat LLM used by the agent.
- **Key configuration:**
  - Model: `models/gemini-2.0-flash-exp`
- **Connections:**
  - **Outgoing (ai_languageModel):** to Generate Beauty Hashtags
- **Credentials:** `googlePalmApi`
- **Edge cases:**
  - Experimental model name (`-exp`) may change or be deprecated.

#### Node: Parse Hashtag Output
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Validates and auto-fixes the agent response into structured JSON.
- **Key configuration:**
  - `autoFix: true`
  - Example schema:
    - `hashtags`: array of `{hashtag, category, popularity}`
- **Connections:**
  - **Outgoing (ai_outputParser):** to Generate Beauty Hashtags
- **Edge cases:**
  - If model returns wildly off-format output, auto-fix may fail or silently coerce incorrect structure.

#### Node: Google Gemini Chat Model (unused in practice)
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`
- **Key configuration:** defaults (no explicit modelName set)
- **Connections:**
  - Connected **to Parse Hashtag Output** as `ai_languageModel`
  - However, **Parse Hashtag Output** is an output parser; it typically does not require its own LLM connection unless using LLM-based repair. Here it’s wired, but the agent already has its own model and the parser is configured with `autoFix`. This extra model is likely redundant or miswired.
- **Edge cases:**
  - Unnecessary model calls (cost/latency) if the parser uses it for fixing.
  - Confusion during maintenance (two Gemini chat model nodes).

**Sticky note context applied to this block:**  
“## Hashtag Generation … structured hashtags aligned with brand identity…”

---

### Block 4 — Platform Trend Research (Live Search + Agent)

**Overview:**  
Uses another LangChain Agent with SerpAPI as a tool to research and propose 6 trending hashtags per platform (LinkedIn, Instagram, Facebook, X). Intended to be “current” as of execution date.

**Nodes involved:**
- **Get Trending Platform Hashtags**
- **Gemini Model for Trending**
- **SerpAPI Search Tool**
- **Parse Hashtag Output1**
- **Google Gemini Chat Model1** (similarly appears redundant/miswired)

#### Node: Get Trending Platform Hashtags
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Agent with a search tool.
- **Key configuration (interpreted):**
  - Prompt includes previously generated hashtags:
    - `{{ JSON.stringify($json.output.hashtags, null, 2) }}`
  - Asks agent to use search tool to find **6 trending hashtags per platform**, “RIGHT NOW” in `{{ $now.format('dd MMMM yyyy') }}`
  - Output requested: “simple list grouped by platform”
  - `hasOutputParser: true` (but parser schema example is *hashtags array*, which doesn’t match “grouped by platform list”)
- **Connections:**
  - **Incoming (main):** Generate Beauty Hashtags
  - **AI language model:** Gemini Model for Trending
  - **AI tool:** SerpAPI Search Tool
  - **AI output parser:** Parse Hashtag Output1
  - **Outgoing (main):** Save Hashtags to Sheet
- **Edge cases / failures:**
  - **Schema mismatch risk:** requested output is grouped-by-platform list, but parser example expects `{ "hashtags": [ ... ] }`. This can cause parsing failures or forced/incorrect coercion.
  - “Trending RIGHT NOW” is hard to guarantee via web search; results may be blog posts or outdated pages.
  - Tool usage depends on SerpAPI quota and query relevance.
- **Version notes:** Agent `typeVersion 2.2`.

#### Node: Gemini Model for Trending
- **Type / role:** `lmChatGoogleGemini` chat model for the trending agent.
- **Key configuration:**
  - Model: `models/gemini-2.0-flash-exp`
- **Credentials:** `googlePalmApi`

#### Node: SerpAPI Search Tool
- **Type / role:** `toolSerpApi` — Enables the agent to run live web searches.
- **Key configuration:** defaults (no special options set)
- **Credentials:** `serpApi`
- **Edge cases:**
  - Invalid API key, exhausted credits, rate limits.
  - Search results may not reflect actual in-platform trending (especially LinkedIn/Instagram).

#### Node: Parse Hashtag Output1
- **Type / role:** Structured output parser (same schema example as hashtags array).
- **Key configuration:** `autoFix: true`
- **Concern:** Does not match the requested grouped-by-platform output unless the agent is actually returning a hashtags array (not shown).
- **Maintenance note:** consider changing schema to:
  - `{ "platforms": { "linkedin": [], "instagram": [], "facebook": [], "x": [] } }`

#### Node: Google Gemini Chat Model1 (likely redundant)
- **Type / role:** Another Gemini chat model node connected to the output parser.
- **Risk:** unnecessary complexity/cost, same as the earlier “Google Gemini Chat Model”.

**Sticky note context applied to this block:**  
“## Platform Trend Research … trending hashtags for LinkedIn, Instagram, Facebook, and X using live search data…”

---

### Block 5 — Save Results

**Overview:**  
Appends a row to Google Sheets containing brand name, category, timestamp, and a comma-separated list of generated hashtags.

**Nodes involved:**
- **Save Hashtags to Sheet**

#### Node: Save Hashtags to Sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — Append row to a spreadsheet.
- **Operation:** `append`
- **Key configuration (interpreted):**
  - Spreadsheet: “Hashtags for My Brand” (documentId points to a specific file)
  - Sheet tab: “Sheet1” (`gid=0`)
  - Mapping mode: “defineBelow” with explicit column mapping:
    - **Hashtags:** `={{ $json.output.hashtags.map(h => h.hashtag).join(', ') }}`
    - **Brand Name:** from form
    - **Generated At:** formatted from `submittedAt`:
      - `toDateTime().format("dd MMM yyyy HH 'hours' mm 'minutes'")`
    - **Product Category:** from form
- **Connections:**
  - **Incoming (main):** Get Trending Platform Hashtags
  - **Outgoing:** none
- **Credentials:** `googleSheetsOAuth2Api`
- **Edge cases / failures:**
  - If the incoming item from “Get Trending Platform Hashtags” does **not** contain `$json.output.hashtags`, this mapping fails. (Currently, “Get Trending…” prompt is about platform trends; it may not output the original `output.hashtags` unless the agent preserves it.)
  - Spreadsheet permissions, wrong sheet name/gid, or missing columns.
  - Date formatting depends on `submittedAt` being present and parseable.
- **Version notes:** Google Sheets node `typeVersion 4.7`.

**Sticky note context applied to this block:**  
“## Save Results … saves it to Google Sheets with brand name, category, and timestamp…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Beauty Brand Hashtag Form | n8n-nodes-base.formTrigger | Collect user inputs and start workflow | — | Scrape Skincare Website | ## User Input & Website Scraping<br>Collects brand details through a form and fetches the brand’s website content using an HTTP request. This provides the raw data needed for brand analysis and ensures hashtags are based on real brand context. |
| Scrape Skincare Website | n8n-nodes-base.httpRequest | Fetch website HTML/content | Beauty Brand Hashtag Form | Analyze Website Content | ## User Input & Website Scraping<br>Collects brand details through a form and fetches the brand’s website content using an HTTP request. This provides the raw data needed for brand analysis and ensures hashtags are based on real brand context. |
| Analyze Website Content | @n8n/n8n-nodes-langchain.googleGemini | Extract structured brand insights from website content | Scrape Skincare Website | Generate Beauty Hashtags | ## Brand Analysis<br>Analyzes website content using AI to extract brand positioning, tone, key ingredients, values, and target audience. These insights guide the hashtag strategy instead of relying on generic prompts. |
| Generate Beauty Hashtags | @n8n/n8n-nodes-langchain.agent | Generate structured hashtags from brand inputs + analysis | Analyze Website Content | Get Trending Platform Hashtags | ## Hashtag Generation<br>Generates structured hashtags aligned with the brand’s identity. Each hashtag includes category and popularity, ensuring a mix of branded, niche, and high-reach tags. |
| Gemini Model for Hashtags | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backing the hashtag generation agent | — | Generate Beauty Hashtags (ai_languageModel) | ## Hashtag Generation<br>Generates structured hashtags aligned with the brand’s identity. Each hashtag includes category and popularity, ensuring a mix of branded, niche, and high-reach tags. |
| Parse Hashtag Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce/repair structured JSON output for hashtags | — | Generate Beauty Hashtags (ai_outputParser) | ## Hashtag Generation<br>Generates structured hashtags aligned with the brand’s identity. Each hashtag includes category and popularity, ensuring a mix of branded, niche, and high-reach tags. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | (Possibly redundant) model wired to output parser | — | Parse Hashtag Output (ai_languageModel) | ## Hashtag Generation<br>Generates structured hashtags aligned with the brand’s identity. Each hashtag includes category and popularity, ensuring a mix of branded, niche, and high-reach tags. |
| Get Trending Platform Hashtags | @n8n/n8n-nodes-langchain.agent | Research current trending platform hashtags via search tool | Generate Beauty Hashtags | Save Hashtags to Sheet | ## Platform Trend Research<br>Finds currently trending hashtags for LinkedIn, Instagram, Facebook, and X using live search data. This ensures the output stays relevant and timely. |
| Gemini Model for Trending | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backing the trending agent | — | Get Trending Platform Hashtags (ai_languageModel) | ## Platform Trend Research<br>Finds currently trending hashtags for LinkedIn, Instagram, Facebook, and X using live search data. This ensures the output stays relevant and timely. |
| SerpAPI Search Tool | @n8n/n8n-nodes-langchain.toolSerpApi | Tool for live web search used by agent | — | Get Trending Platform Hashtags (ai_tool) | ## Platform Trend Research<br>Finds currently trending hashtags for LinkedIn, Instagram, Facebook, and X using live search data. This ensures the output stays relevant and timely. |
| Parse Hashtag Output1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce/repair structured output of trending step (schema may not match) | — | Get Trending Platform Hashtags (ai_outputParser) | ## Platform Trend Research<br>Finds currently trending hashtags for LinkedIn, Instagram, Facebook, and X using live search data. This ensures the output stays relevant and timely. |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | (Possibly redundant) model wired to output parser | — | Parse Hashtag Output1 (ai_languageModel) | ## Platform Trend Research<br>Finds currently trending hashtags for LinkedIn, Instagram, Facebook, and X using live search data. This ensures the output stays relevant and timely. |
| Save Hashtags to Sheet | n8n-nodes-base.googleSheets | Append hashtags + metadata to Google Sheets | Get Trending Platform Hashtags | — | ## Save Results<br>Formats the final hashtag list and saves it to Google Sheets with brand name, category, and timestamp for easy reuse and collaboration. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / context | — | — | ## AI Beauty Brand Hashtag Generator<br>This workflow helps beauty and skincare brands generate highly relevant, high-performing hashtags using AI, real website content, and current social media trends.<br><br>Instead of relying on generic hashtag lists, the workflow analyzes the brand’s own website, understands its positioning, tone, and audience, and then generates a balanced mix of branded, niche, and trending hashtags. It also researches platform-specific trends to ensure the hashtags are timely and appropriate for LinkedIn, Instagram, Facebook, and X (Twitter).<br><br>All inputs are collected through a simple form, making this workflow easy to use for non-technical users, agencies, and marketers. The final hashtag set is automatically saved to Google Sheets for reuse, planning, or collaboration.<br><br>### How it works<br>The workflow starts with a form where the user provides brand details and a website URL. The website is scraped and analyzed using AI to extract brand insights. Based on this analysis, AI generates structured hashtags with categories and popularity levels. A second AI step fetches currently trending hashtags per platform. Finally, the results are formatted and saved to Google Sheets.<br><br>### Setup steps<br>1. Connect Google Gemini credentials<br>2. Connect Google Sheets credentials. Make a copy of https://docs.google.com/spreadsheets/d/1oY7H9l2odzTL73LiJDbQ2lkned_YRtS1__ORfluZtwg/edit?gid=0#gid=0.<br>3. Add your Google Sheet ID and sheet name<br>4. Update SerpAPI credentials for trend research<br>5. Activate the workflow and open the form URL |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / context | — | — | ## User Input & Website Scraping<br>Collects brand details through a form and fetches the brand’s website content using an HTTP request. This provides the raw data needed for brand analysis and ensures hashtags are based on real brand context. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / context | — | — | ## Brand Analysis<br>Analyzes website content using AI to extract brand positioning, tone, key ingredients, values, and target audience. These insights guide the hashtag strategy instead of relying on generic prompts. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / context | — | — | ## Hashtag Generation<br>Generates structured hashtags aligned with the brand’s identity. Each hashtag includes category and popularity, ensuring a mix of branded, niche, and high-reach tags. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / context | — | — | ## Platform Trend Research<br>Finds currently trending hashtags for LinkedIn, Instagram, Facebook, and X using live search data. This ensures the output stays relevant and timely. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation / context | — | — | ## Save Results<br>Formats the final hashtag list and saves it to Google Sheets with brand name, category, and timestamp for easy reuse and collaboration. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: `7 AI Beauty Brand Hashtag Generator (Website-Aware + Multi-Platform)`
- Ensure workflow setting “Execution Order” is `v1` (default is usually fine).

2) **Add Form Trigger**
- Node: **Form Trigger**
- Title: “Beauty Brand Hashtag Generator”
- Description: “Generate trending hashtags for your beauty brand”
- Add fields (mark required as indicated):
  - Text (required): “Skincare Brand Website URL”
  - Text (required): “Brand Name”
  - Text (required): “Product Category (e.g., skincare, makeup, haircare)”
  - Text (required): “Target Audience”
  - Text (required): “Brand Tone (e.g., luxury, natural, playful)”
  - Number: “Number of Hashtags (5-10)” default `10`
- Save the node.

3) **Add HTTP Request to scrape the website**
- Node: **HTTP Request**
- Method: GET
- URL (expression): `{{$json['Skincare Brand Website URL']}}`
- Options:
  - Allow unauthorized certs: enabled
  - Response: configure to not hard-fail on non-2xx (equivalent of `neverError: true` if available in your version)
- Headers:
  - `User-Agent`: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36`
  - `Accept`: `text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`
  - `Accept-Language`: `en-US,en;q=0.9`
- Connect: **Form Trigger → HTTP Request**

4) **Add Gemini node to analyze website content**
- Node: **Google Gemini** (LangChain Gemini node in n8n)
- Credential: **Google Gemini / PaLM API** (`googlePalmApi`)
- Model: `models/gemini-flash-lite-latest`
- Enable JSON output (toggle “JSON Output” on).
- Prompt/message content should include:
  - `{{$json.data}}` (scraped HTML/text)
  - Form fields from the Form Trigger node (Brand Name, Category, Audience, Tone)
  - Ask it to output the specified `brand_analysis` JSON object only.
- Connect: **HTTP Request → Analyze Website Content**

5) **Add LangChain Agent for hashtag generation**
- Node: **AI Agent** (LangChain Agent)
- Add **System Message** describing the role (beauty hashtag expert) and constraints (mix types, avoid banned, return only JSON).
- Add **User prompt** that includes:
  - Hashtag count from the form (important: map this correctly)
  - Brand details from the form
  - Website analysis text from the previous Gemini node
- **Important fix suggestion when rebuilding:** use the actual field name produced by your Form Trigger for the number. If n8n uses the label as the JSON key, you may need:
  - `{{$('Beauty Brand Hashtag Form').item.json['Number of Hashtags (5-10)'] || 10}}`
- Connect: **Analyze Website Content → Generate Beauty Hashtags**

6) **Attach a Gemini chat model to the agent**
- Node: **Google Gemini Chat Model** (`lmChatGoogleGemini`)
- Model name: `models/gemini-2.0-flash-exp` (or a stable Gemini model available in your account)
- Credential: `googlePalmApi`
- Connect the model to the agent via the **AI Language Model** connection.

7) **Attach a Structured Output Parser to the agent**
- Node: **Structured Output Parser**
- Enable `Auto-fix`
- Provide a schema example:
  - `{ "hashtags": [ { "hashtag": "#...", "category": "general|niche|trending|branded", "popularity": "high|medium|low" } ] }`
- Connect it to the agent via the **AI Output Parser** connection.

8) **Add LangChain Agent for platform trend research**
- Node: **AI Agent**
- System message: trends expert; must use search tool; return 6 tags per platform.
- User prompt:
  - Include the previously generated hashtags (stringified).
  - Ask for 6 trending hashtags per platform grouped by platform, using current date `{{$now.format('dd MMMM yyyy')}}`.
- Connect: **Generate Beauty Hashtags → Get Trending Platform Hashtags**

9) **Attach Gemini chat model to the trending agent**
- Node: **Google Gemini Chat Model**
- Model: `models/gemini-2.0-flash-exp`
- Credential: `googlePalmApi`
- Connect via **AI Language Model**.

10) **Add SerpAPI tool and connect it to the trending agent**
- Node: **SerpAPI Search Tool**
- Credential: `serpApi`
- Connect tool to agent via **AI Tool** connection.

11) **(Recommended) Fix the trending output schema**
- If you want structured output, set the output parser schema to something like:
  - `{ "platform_trends": { "linkedin": [], "instagram": [], "facebook": [], "x": [] } }`
- Otherwise, you can remove the output parser for the trending agent and accept plain text.
- The provided workflow uses a hashtags-array schema, which may not match the “grouped by platform” requirement.

12) **Add Google Sheets node to save results**
- Node: **Google Sheets**
- Credential: `googleSheetsOAuth2Api`
- Operation: **Append**
- Select Spreadsheet (make a copy of the provided link in the sticky note, or your own)
- Select sheet/tab (e.g., “Sheet1”)
- Map columns:
  - Brand Name → from form
  - Product Category → from form
  - Generated At → format from `submittedAt`
  - Hashtags → join of generated hashtags
- Connect: **Get Trending Platform Hashtags → Save Hashtags to Sheet**
- **Important:** ensure the incoming item to this node still contains the original generated hashtags array. If the trending agent overwrites the item structure, insert a **Merge** node or store hashtags in a separate field before trend research.

13) **Activate workflow and test**
- Open the form URL from the Form Trigger node.
- Submit a real brand site URL.
- Verify:
  - Website scrape returns meaningful content
  - Brand analysis returns valid JSON
  - Hashtags output parses into array
  - Google Sheets row is appended successfully

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Make a copy of the Google Sheet before using | https://docs.google.com/spreadsheets/d/1oY7H9l2odzTL73LiJDbQ2lkned_YRtS1__ORfluZtwg/edit?gid=0#gid=0 |
| Setup steps highlighted in the workflow canvas | Connect Google Gemini credentials; connect Google Sheets credentials; set Sheet ID/name; update SerpAPI credentials; activate workflow and open the form URL |
| Key design intent | Website-aware hashtag generation + live trend research per platform; results stored in Sheets for reuse |

