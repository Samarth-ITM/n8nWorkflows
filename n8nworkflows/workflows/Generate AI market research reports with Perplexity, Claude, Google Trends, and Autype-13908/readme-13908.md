Generate AI market research reports with Perplexity, Claude, Google Trends, and Autype

https://n8nworkflows.xyz/workflows/generate-ai-market-research-reports-with-perplexity--claude--google-trends--and-autype-13908


# Generate AI market research reports with Perplexity, Claude, Google Trends, and Autype

# 1. Workflow Overview

This workflow generates a market research report PDF from a simple user-submitted form. A user enters product details, the workflow automatically gathers market signals from Google Trends and Google Search via SerpAPI, performs AI-assisted market research with Perplexity through OpenRouter, writes a polished report in Autype Extended Markdown with Anthropic Claude through OpenRouter, renders the report as a PDF with Autype, and finally saves the PDF to Google Drive.

Typical use cases include product strategy, startup competitive analysis, investor preparation, internal planning, and fast generation of board-ready market research documents.

## 1.1 Input Reception

The workflow starts with an n8n Form Trigger that collects the product name, industry, product description, and desired report language.

## 1.2 External Data Gathering

After form submission, three parallel operations run:
- Google Trends query for industry interest over the last 12 months
- Google Search query to identify competitors
- HTTP download of Autype Markdown syntax guidance

These outputs are merged into one consolidated context object.

## 1.3 AI Market Research

A LangChain Agent uses the Perplexity Sonar Pro model through OpenRouter to transform the collected search/trend context into a structured research analysis, including market overview, competitors, trends, and product positioning.

## 1.4 AI Report Writing

A second LangChain Agent uses Anthropic Claude Sonnet 4.6 through OpenRouter to write a full professional report in Autype Extended Markdown, following strict formatting and rendering rules.

## 1.5 PDF Rendering and Storage

The generated markdown is cleaned, given a title and filename, rendered as a styled PDF by Autype, and uploaded to Google Drive.

---

# 2. Block-by-Block Analysis

## Block 1 — Form Input & Trigger

### Overview
This block collects all user input required to generate the report and serves as the workflow entry point. It also controls the user-facing submission confirmation and waits for the final node response mode.

### Nodes Involved
- Market Research Form

### Node Details

#### Market Research Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger that exposes a hosted form and starts the workflow when submitted.
- **Configuration choices:**
  - Form title: **Market Research Report Generator**
  - Description explains that the workflow researches the market, identifies competitors, analyzes trends, and generates a PDF
  - Submission response text: **Your market research report is being generated. This may take a minute...**
  - Response mode: **lastNode**
- **Form fields:**
  - `productName` — required text
  - `industry` — required text
  - `productDescription` — required textarea
  - `language` — required dropdown with English, German, French, Spanish
- **Key expressions or variables used:**
  - Downstream nodes reference values such as:
    - `$('Market Research Form').first().json.productName`
    - `$('Market Research Form').first().json.industry`
    - `$('Market Research Form').first().json.productDescription`
    - `$('Market Research Form').first().json.language`
- **Input and output connections:**
  - No input; it is the entry point
  - Outputs to:
    - Google Trends
    - Search Competitors
    - Download Markdown Syntax
- **Version-specific requirements:**
  - Uses `typeVersion: 2.5`
  - Requires n8n support for Form Trigger nodes
- **Edge cases or potential failure types:**
  - Form not deployed or not reachable publicly
  - Required field validation blocks execution
  - If `responseMode = lastNode`, downstream failures may affect user experience
- **Sub-workflow reference:** None

---

## Block 2 — Data Gathering from Google Trends, Search, and Markdown Reference

### Overview
This block enriches the form input with public market signals and technical formatting instructions. It collects trend data, competitor discovery results, and the Autype markdown syntax reference required later for report generation.

### Nodes Involved
- Google Trends
- Search Competitors
- Download Markdown Syntax
- Merge Trends + Competitors
- Merge With Syntax
- Prepare Research Context

### Node Details

#### Google Trends
- **Type and technical role:** `n8n-nodes-serpapi.serpApi`  
  Community SerpAPI node used to fetch Google Trends data.
- **Configuration choices:**
  - Operation: `google_trends`
  - Date range: `today 12-m`
- **Key expressions or variables used:**
  - No explicit query expression is visible in the JSON; this implies the node likely depends on the incoming item or manual setup in the node UI. In practice, this should query the submitted `industry`.
- **Input and output connections:**
  - Input from: Market Research Form
  - Output to: Merge Trends + Competitors
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires community node package `n8n-nodes-serpapi`
- **Edge cases or potential failure types:**
  - Missing or invalid SerpAPI credentials
  - Quota exhaustion on SerpAPI free plan
  - Query misconfiguration if the search term is not explicitly mapped to `industry`
  - Unexpected response shape, causing downstream code extraction failure
- **Sub-workflow reference:** None

#### Search Competitors
- **Type and technical role:** `n8n-nodes-serpapi.serpApi`  
  Community SerpAPI node used for Google Search results that help identify competitors.
- **Configuration choices:**
  - Standard search operation
  - Number of results requested: `10`
- **Key expressions or variables used:**
  - No explicit query expression is shown in the JSON; practically this should search based on `productName`, `industry`, or a competitor-focused query string.
- **Input and output connections:**
  - Input from: Market Research Form
  - Output to: Merge Trends + Competitors
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires `n8n-nodes-serpapi`
- **Edge cases or potential failure types:**
  - Invalid API key or exhausted quota
  - Poor query formulation leading to irrelevant competitors
  - Missing `organic_results` in response
- **Sub-workflow reference:** None

#### Download Markdown Syntax
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the Autype markdown syntax guide as plain text for prompt injection into the report-writing AI.
- **Configuration choices:**
  - URL: `https://autype.com/llm-resources/markdown-syntax.md`
  - Response format: `text`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: Market Research Form
  - Output to: Merge With Syntax
- **Version-specific requirements:**
  - Uses `typeVersion: 4.2`
- **Edge cases or potential failure types:**
  - URL unavailable or returns non-text content
  - Timeout or network issue
  - Syntax file changes in ways that break assumptions in downstream code
- **Sub-workflow reference:** None

#### Merge Trends + Competitors
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the Google Trends and competitor-search outputs into one item stream.
- **Configuration choices:**
  - Mode: `combine`
  - Combine strategy: `combineAll`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs from:
    - Google Trends
    - Search Competitors
  - Output to: Merge With Syntax
- **Version-specific requirements:**
  - Uses `typeVersion: 3.1`
- **Edge cases or potential failure types:**
  - If one branch returns no item, merged output may be incomplete
  - Mismatched item counts can create confusing merged objects depending on node behavior
- **Sub-workflow reference:** None

#### Merge With Syntax
- **Type and technical role:** `n8n-nodes-base.merge`  
  Adds the downloaded markdown syntax data to the already merged trend and competitor data.
- **Configuration choices:**
  - Mode: `combine`
  - Combine strategy: `combineAll`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs from:
    - Merge Trends + Competitors
    - Download Markdown Syntax
  - Output to: Prepare Research Context
- **Version-specific requirements:**
  - Uses `typeVersion: 3.1`
- **Edge cases or potential failure types:**
  - Missing syntax file leaves report-writer prompt with incomplete formatting rules
- **Sub-workflow reference:** None

#### Prepare Research Context
- **Type and technical role:** `n8n-nodes-base.code`  
  Normalizes all gathered data into a single structured object for the research agent.
- **Configuration choices:**
  - JavaScript code manually inspects all merged input items
  - Pulls original form values from `Market Research Form`
  - Builds:
    - `trendsSummary`
    - `competitorSearchResults`
    - `markdownSyntax`
- **Key expressions or variables used:**
  - `const items = $input.all();`
  - `const form = $('Market Research Form').first().json;`
  - Detects Google Trends via `d.interest_over_time?.timeline_data`
  - Detects search results via `d.organic_results`
  - Detects markdown syntax via `d.data`
- **Input and output connections:**
  - Input from: Merge With Syntax
  - Output to: AI Research Agent
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If Google Trends schema changes, `timeline_data` extraction may fail
  - If search results do not include `organic_results`, competitor list becomes blank
  - If syntax document no longer includes the word `Markdown`, detection may fail
  - Assumes form node exists and produced one item
- **Sub-workflow reference:** None

---

## Block 3 — AI Market Research with Perplexity

### Overview
This block turns the normalized context into a high-value market analysis. The LangChain agent uses a connected OpenRouter chat model configured for Perplexity Sonar Pro.

### Nodes Involved
- OpenRouter Perplexity
- AI Research Agent

### Node Details

#### OpenRouter Perplexity
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Language model connector for OpenRouter, configured to use Perplexity Sonar Pro.
- **Configuration choices:**
  - Model: `perplexity/sonar-pro`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model connection to: AI Research Agent
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires OpenRouter API credential
  - Requires LangChain-compatible n8n nodes
- **Edge cases or potential failure types:**
  - Invalid OpenRouter credential
  - Model unavailability or rate limiting
  - Higher latency from web-enabled model behavior
- **Sub-workflow reference:** None

#### AI Research Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Agent node that prompts the model to generate a factual market-research output using gathered inputs.
- **Configuration choices:**
  - Prompt includes:
    - product name
    - industry
    - product description
    - competitor search results
    - Google Trends summary
    - requested output language
  - Requires output covering:
    1. Market overview
    2. Top 5–8 competitors
    3. Current trends and 2-year predictions
    4. Product positioning
  - System message instructs factual, specific, data-driven analysis with real companies and real data points
  - Max iterations: `3`
- **Key expressions or variables used:**
  - `{{ $json.productName }}`
  - `{{ $json.industry }}`
  - `{{ $json.productDescription }}`
  - `{{ $json.competitorSearchResults }}`
  - `{{ $json.trendsSummary }}`
  - `{{ $json.language }}`
- **Input and output connections:**
  - Main input from: Prepare Research Context
  - AI model input from: OpenRouter Perplexity
  - Main output to: Prepare Report Input
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Hallucinated competitor details despite instructions
  - Output may be too generic if upstream search query was weak
  - Token/context issues if syntax or search content becomes too large
  - Language mismatch if model ignores requested language
- **Sub-workflow reference:** None

---

## Block 4 — Report Preparation and AI Writing with Claude

### Overview
This block reformats research output for report generation, then asks Claude to produce a polished report in Autype Extended Markdown under strict rendering constraints.

### Nodes Involved
- Prepare Report Input
- OpenRouter Anthropic Claude
- AI Report Writer

### Node Details

#### Prepare Report Input
- **Type and technical role:** `n8n-nodes-base.code`  
  Collects the research output and combines it with original form data and markdown syntax before passing everything to the writer agent.
- **Configuration choices:**
  - Reads:
    - research output from current item
    - original form values from Market Research Form
    - normalized context from Prepare Research Context
  - Emits:
    - `researchData`
    - `productName`
    - `industry`
    - `productDescription`
    - `language`
    - `trendsSummary`
    - `markdownSyntax`
- **Key expressions or variables used:**
  - `const researchOutput = $json.output;`
  - `const form = $('Market Research Form').first().json;`
  - `const context = $('Prepare Research Context').first().json;`
- **Input and output connections:**
  - Input from: AI Research Agent
  - Output to: AI Report Writer
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If AI Research Agent does not return an `output` field, report generation prompt becomes incomplete
  - Cross-node references fail if node names are changed without updating code
- **Sub-workflow reference:** None

#### OpenRouter Anthropic Claude
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  OpenRouter model connector configured for Claude Sonnet 4.6.
- **Configuration choices:**
  - Model: `anthropic/claude-sonnet-4.6`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model connection to: AI Report Writer
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires OpenRouter credentials
- **Edge cases or potential failure types:**
  - Model access restrictions on OpenRouter account
  - Rate limiting or long generation times
- **Sub-workflow reference:** None

#### AI Report Writer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Generates the final markdown report according to a prescribed title page/template and formatting rules.
- **Configuration choices:**
  - Prompt asks for a professional report for the given product and industry
  - Includes:
    - Perplexity research output
    - Google Trends summary
    - requested language
    - title-page / TOC template snippet
  - System prompt injects full Autype markdown syntax reference
  - Enforces rendering constraints:
    - no `&nbsp;`
    - no manual page breaks before headings
    - blockquotes may use only `color` and `borderColor`
    - specific report section structure
    - table usage for competitor comparison and SWOT
    - output only markdown
    - 1500–3000 words
  - Max iterations: `3`
- **Key expressions or variables used:**
  - `{{ $json.productName }}`
  - `{{ $json.industry }}`
  - `{{ $json.productDescription }}`
  - `{{ $json.researchData }}`
  - `{{ $json.trendsSummary }}`
  - `{{ $json.language }}`
  - `{{ $json.markdownSyntax }}`
- **Input and output connections:**
  - Main input from: Prepare Report Input
  - AI model input from: OpenRouter Anthropic Claude
  - Main output to: Prepare Render Payload
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Model may still return code fences, handled later by cleanup
  - Markdown may violate Autype syntax despite prompt instructions
  - Long prompt due to injected syntax reference may increase token usage
  - The TOC title is hardcoded as `Inhaltsverzeichnis`, which is German; this may be undesirable for non-German reports
- **Sub-workflow reference:** None

---

## Block 5 — Render Payload Preparation, PDF Rendering, and Storage

### Overview
This block converts the markdown output into a renderable payload, produces a PDF via Autype, and stores the final file in Google Drive.

### Nodes Involved
- Prepare Render Payload
- Render Report PDF
- Save Report to Drive

### Node Details

#### Prepare Render Payload
- **Type and technical role:** `n8n-nodes-base.code`  
  Sanitizes markdown output and derives the final PDF title and filename.
- **Configuration choices:**
  - Reads markdown from `AI Report Writer` output field
  - Removes opening/closing markdown code fences if present
  - Generates slug from product name
  - Creates:
    - `content`
    - `title`
    - `filename`
- **Key expressions or variables used:**
  - `const markdown = $json.output;`
  - `const form = $('Market Research Form').first().json;`
  - `const today = new Date().toISOString().split('T')[0];`
  - slug logic: `form.productName.toLowerCase().replace(/[^a-z0-9]+/g, '-')`
- **Input and output connections:**
  - Input from: AI Report Writer
  - Output to: Render Report PDF
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Slugging removes non-ASCII characters, which may produce awkward filenames
  - If `output` is empty, render node receives blank content
- **Sub-workflow reference:** None

#### Render Report PDF
- **Type and technical role:** `n8n-nodes-autype.autype`  
  Community Autype node that renders markdown to PDF and downloads the generated binary.
- **Configuration choices:**
  - Operation: `renderMarkdown`
  - Content from `{{ $json.content }}`
  - Download output enabled
  - Paper size: `A4`
  - Dynamic title from `{{ $json.title }}`
  - Extensive default styling JSON:
    - Font family: Open Sans
    - Font size: 11
    - Base color: `#1a1a1a`
    - Line height: 1.6
    - Heading styles:
      - h1: 28pt, black, bold, page break before
      - h2: 22pt, dark gray, bold, page break before
      - h3: 18pt
    - Blockquote left/right indentation reset
    - Chart palette configured
    - Header:
      - left text: `SearchIT Inc.`
      - right logo image from Icons8
      - excluded on first page
    - Footer:
      - centered `Page {{pageNumber}}`
      - excluded on first page
- **Key expressions or variables used:**
  - `={{ $json.content }}`
  - `={{ $json.title }}`
- **Input and output connections:**
  - Input from: Prepare Render Payload
  - Output to: Save Report to Drive
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires community node `n8n-nodes-autype`
  - Requires Autype API credentials
  - Best suited for self-hosted n8n where community nodes are available
- **Edge cases or potential failure types:**
  - Invalid Autype credential
  - Rendering failure due to malformed markdown or unsupported syntax
  - External images in header/title page may fail to load
  - PDF binary output handling depends on n8n binary mode; this workflow uses `binaryMode: separate`
- **Sub-workflow reference:** None

#### Save Report to Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the rendered PDF to a configured Google Drive folder.
- **Configuration choices:**
  - File name from `{{ $('Prepare Render Payload').item.json.filename }}`
  - Drive: `My Drive`
  - Folder ID: explicit configured folder ID placeholder
- **Key expressions or variables used:**
  - `={{ $('Prepare Render Payload').item.json.filename }}`
- **Input and output connections:**
  - Input from: Render Report PDF
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion: 3`
  - Requires Google Drive OAuth2 credentials
- **Edge cases or potential failure types:**
  - Invalid or expired Google OAuth token
  - Folder ID not found or inaccessible
  - Binary property mismatch if upload configuration is incomplete in practice
  - Permission issues for target drive/folder
- **Sub-workflow reference:** None

---

## Block 6 — Documentation Sticky Notes

### Overview
These nodes do not execute business logic but document the workflow visually in the n8n canvas. Their content is important because it explains prerequisites, architecture, and intended behavior.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation for the full workflow.
- **Configuration choices:**
  - Describes the end-to-end process, report sections, use case, and prerequisites
  - Includes links:
    - [serpapi.com](https://serpapi.com)
    - [openrouter.ai](https://openrouter.ai)
    - [app.autype.com](https://app.autype.com)
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents form and data-gathering block
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents Perplexity/OpenRouter research block
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents Claude/OpenRouter writing block and markdown constraints
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**
  - Documents rendering and output block
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Market Research Form | n8n-nodes-base.formTrigger | Entry form collecting product and language inputs |  | Google Trends; Search Competitors; Download Markdown Syntax | ### 1. Form Input & Data Gathering<br>The **Market Research Form** collects product name, industry, description, and report language. Three parallel requests then run:<br>- **Google Trends** (SerpAPI Official node) -- 12-month search interest for the industry<br>- **Search Competitors** (SerpAPI Google Search) -- discovers competitors automatically from web results<br>- **Download Markdown Syntax** -- fetches Autype's extended markdown reference for the report writer<br><br>All results are merged and prepared as a single context object. |
| Google Trends | n8n-nodes-serpapi.serpApi | Fetches Google Trends data | Market Research Form | Merge Trends + Competitors | ### 1. Form Input & Data Gathering<br>The **Market Research Form** collects product name, industry, description, and report language. Three parallel requests then run:<br>- **Google Trends** (SerpAPI Official node) -- 12-month search interest for the industry<br>- **Search Competitors** (SerpAPI Google Search) -- discovers competitors automatically from web results<br>- **Download Markdown Syntax** -- fetches Autype's extended markdown reference for the report writer<br><br>All results are merged and prepared as a single context object. |
| Search Competitors | n8n-nodes-serpapi.serpApi | Searches the web for likely competitors | Market Research Form | Merge Trends + Competitors | ### 1. Form Input & Data Gathering<br>The **Market Research Form** collects product name, industry, description, and report language. Three parallel requests then run:<br>- **Google Trends** (SerpAPI Official node) -- 12-month search interest for the industry<br>- **Search Competitors** (SerpAPI Google Search) -- discovers competitors automatically from web results<br>- **Download Markdown Syntax** -- fetches Autype's extended markdown reference for the report writer<br><br>All results are merged and prepared as a single context object. |
| Download Markdown Syntax | n8n-nodes-base.httpRequest | Downloads Autype markdown syntax reference | Market Research Form | Merge With Syntax | ### 1. Form Input & Data Gathering<br>The **Market Research Form** collects product name, industry, description, and report language. Three parallel requests then run:<br>- **Google Trends** (SerpAPI Official node) -- 12-month search interest for the industry<br>- **Search Competitors** (SerpAPI Google Search) -- discovers competitors automatically from web results<br>- **Download Markdown Syntax** -- fetches Autype's extended markdown reference for the report writer<br><br>All results are merged and prepared as a single context object. |
| Merge Trends + Competitors | n8n-nodes-base.merge | Combines trends and search data | Google Trends; Search Competitors | Merge With Syntax | ### 1. Form Input & Data Gathering<br>The **Market Research Form** collects product name, industry, description, and report language. Three parallel requests then run:<br>- **Google Trends** (SerpAPI Official node) -- 12-month search interest for the industry<br>- **Search Competitors** (SerpAPI Google Search) -- discovers competitors automatically from web results<br>- **Download Markdown Syntax** -- fetches Autype's extended markdown reference for the report writer<br><br>All results are merged and prepared as a single context object. |
| Merge With Syntax | n8n-nodes-base.merge | Adds markdown syntax content to gathered research inputs | Merge Trends + Competitors; Download Markdown Syntax | Prepare Research Context | ### 1. Form Input & Data Gathering<br>The **Market Research Form** collects product name, industry, description, and report language. Three parallel requests then run:<br>- **Google Trends** (SerpAPI Official node) -- 12-month search interest for the industry<br>- **Search Competitors** (SerpAPI Google Search) -- discovers competitors automatically from web results<br>- **Download Markdown Syntax** -- fetches Autype's extended markdown reference for the report writer<br><br>All results are merged and prepared as a single context object. |
| Prepare Research Context | n8n-nodes-base.code | Normalizes trend, search, syntax, and form data into one object | Merge With Syntax | AI Research Agent | ### 1. Form Input & Data Gathering<br>The **Market Research Form** collects product name, industry, description, and report language. Three parallel requests then run:<br>- **Google Trends** (SerpAPI Official node) -- 12-month search interest for the industry<br>- **Search Competitors** (SerpAPI Google Search) -- discovers competitors automatically from web results<br>- **Download Markdown Syntax** -- fetches Autype's extended markdown reference for the report writer<br><br>All results are merged and prepared as a single context object. |
| OpenRouter Perplexity | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides Perplexity Sonar Pro model to the research agent |  | AI Research Agent | ### 2. AI Research (Perplexity via OpenRouter)<br>The **AI Research Agent** uses Perplexity Sonar Pro (via OpenRouter) to conduct deep market research with real-time web access. It receives the Google Search results and Trends data as context, then provides:<br>- Comprehensive market overview with data points<br>- Detailed competitor profiles (features, pricing, positioning)<br>- Market trends and predictions<br>- Product positioning analysis<br><br>Perplexity's web search capability ensures the data is current and cited. |
| AI Research Agent | @n8n/n8n-nodes-langchain.agent | Generates factual market research from gathered inputs | Prepare Research Context; OpenRouter Perplexity (AI model) | Prepare Report Input | ### 2. AI Research (Perplexity via OpenRouter)<br>The **AI Research Agent** uses Perplexity Sonar Pro (via OpenRouter) to conduct deep market research with real-time web access. It receives the Google Search results and Trends data as context, then provides:<br>- Comprehensive market overview with data points<br>- Detailed competitor profiles (features, pricing, positioning)<br>- Market trends and predictions<br>- Product positioning analysis<br><br>Perplexity's web search capability ensures the data is current and cited. |
| Prepare Report Input | n8n-nodes-base.code | Repackages research output and context for the report writer | AI Research Agent | AI Report Writer | ### 3. AI Report Writer (Anthropic Claude)<br>The **AI Report Writer** uses Anthropic Claude (via OpenRouter) to take all research data and write a structured market research report using Autype Extended Markdown. The system prompt includes the full syntax reference and enforces:<br>- No &nbsp; entities (unsupported)<br>- Blockquotes: only color + borderColor attributes<br>- Page breaks before h1/h2 are automatic (no manual ---  needed)<br><br>The report is then cleaned, rendered to a styled PDF via Autype (Open Sans font, professional heading hierarchy, comfortable spacing, header/footer), and saved to Google Drive. |
| OpenRouter Anthropic Claude | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides Claude Sonnet 4.6 model to the report writer |  | AI Report Writer | ### 3. AI Report Writer (Anthropic Claude)<br>The **AI Report Writer** uses Anthropic Claude (via OpenRouter) to take all research data and write a structured market research report using Autype Extended Markdown. The system prompt includes the full syntax reference and enforces:<br>- No &nbsp; entities (unsupported)<br>- Blockquotes: only color + borderColor attributes<br>- Page breaks before h1/h2 are automatic (no manual ---  needed)<br><br>The report is then cleaned, rendered to a styled PDF via Autype (Open Sans font, professional heading hierarchy, comfortable spacing, header/footer), and saved to Google Drive. |
| AI Report Writer | @n8n/n8n-nodes-langchain.agent | Writes the final structured markdown report | Prepare Report Input; OpenRouter Anthropic Claude (AI model) | Prepare Render Payload | ### 3. AI Report Writer (Anthropic Claude)<br>The **AI Report Writer** uses Anthropic Claude (via OpenRouter) to take all research data and write a structured market research report using Autype Extended Markdown. The system prompt includes the full syntax reference and enforces:<br>- No &nbsp; entities (unsupported)<br>- Blockquotes: only color + borderColor attributes<br>- Page breaks before h1/h2 are automatic (no manual ---  needed)<br><br>The report is then cleaned, rendered to a styled PDF via Autype (Open Sans font, professional heading hierarchy, comfortable spacing, header/footer), and saved to Google Drive. |
| Prepare Render Payload | n8n-nodes-base.code | Cleans markdown and builds title/filename for rendering | AI Report Writer | Render Report PDF | ### 4. Render PDF & Save<br>The **Prepare Render Payload** strips code fences and sets the title/filename. **Render Report PDF** uses Autype Render from Markdown with professional defaults:<br>- Font: Open Sans, 11pt<br>- Headings: h1 28pt, h2 22pt, h3 18pt with automatic page breaks<br>- Chart colors: blue, red, green, amber, purple palette<br>- Header: company name + logo (excluded on first page)<br>- Footer: centered page number (excluded on first page)<br>- Generous spacing for readability<br><br>Replace the Google Drive node with Email, S3, Slack, or any other output. |
| Render Report PDF | n8n-nodes-autype.autype | Renders markdown into a styled PDF | Prepare Render Payload | Save Report to Drive | ### 4. Render PDF & Save<br>The **Prepare Render Payload** strips code fences and sets the title/filename. **Render Report PDF** uses Autype Render from Markdown with professional defaults:<br>- Font: Open Sans, 11pt<br>- Headings: h1 28pt, h2 22pt, h3 18pt with automatic page breaks<br>- Chart colors: blue, red, green, amber, purple palette<br>- Header: company name + logo (excluded on first page)<br>- Footer: centered page number (excluded on first page)<br>- Generous spacing for readability<br><br>Replace the Google Drive node with Email, S3, Slack, or any other output. |
| Save Report to Drive | n8n-nodes-base.googleDrive | Uploads the generated PDF to Google Drive | Render Report PDF |  | ### 4. Render PDF & Save<br>The **Prepare Render Payload** strips code fences and sets the title/filename. **Render Report PDF** uses Autype Render from Markdown with professional defaults:<br>- Font: Open Sans, 11pt<br>- Headings: h1 28pt, h2 22pt, h3 18pt with automatic page breaks<br>- Chart colors: blue, red, green, amber, purple palette<br>- Header: company name + logo (excluded on first page)<br>- Footer: centered page number (excluded on first page)<br>- Generous spacing for readability<br><br>Replace the Google Drive node with Email, S3, Slack, or any other output. |
| Sticky Note | n8n-nodes-base.stickyNote | General workflow documentation |  |  | ## Generate AI Market Research Reports with Perplexity, Google Trends, and Autype<br><br>### Submit a simple form with your product name, industry, and description. The workflow automatically researches your market, discovers competitors, analyzes trends, and generates a professionally styled PDF report.<br><br>**Fully automated -- no manual competitor input required.**<br><br>**Data pipeline:**<br>1. You fill out a form (product name, industry, description)<br>2. SerpAPI fetches Google Trends data and Google Search results for competitors<br>3. Perplexity AI (via OpenRouter) conducts deep market and competitor research<br>4. Anthropic Claude (via OpenRouter) writes a structured report in Autype Extended Markdown<br>5. Autype renders a styled PDF and saves it to Google Drive<br><br>**Report sections:**<br>1. Executive Summary<br>2. Market Overview<br>3. Trend Analysis (with Google Trends data)<br>4. Competitive Landscape (table)<br>5. SWOT Analysis (table)<br>6. Key Findings & Recommendations<br>7. Sources<br><br>**Use Case:** Product managers, startup founders, strategists, and consultants who need quick, automated market research reports for investor decks, board meetings, or strategic planning.<br><br>### Requirements<br>* **SerpAPI account** -- Free tier: 100 searches/month ([serpapi.com](https://serpapi.com)). Install community node: `n8n-nodes-serpapi`.<br>* **OpenRouter account** -- For Perplexity Sonar model ([openrouter.ai](https://openrouter.ai))<br>* **Autype account** -- [app.autype.com](https://app.autype.com) > Settings > API Keys<br>* **Google Drive** -- OAuth2 credential (optional)<br>* **n8n-nodes-autype** -- Install via Settings > Community Nodes<br><br>> Community nodes required: `n8n-nodes-autype`, `n8n-nodes-serpapi`. Only available on self-hosted n8n. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for input/data gathering block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for AI research block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for AI report writing block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note for rendering/output block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Install required community nodes on self-hosted n8n**
   - Install:
     - `n8n-nodes-serpapi`
     - `n8n-nodes-autype`
   - This workflow relies on community nodes not typically available in n8n Cloud.

2. **Prepare credentials**
   - Create a **SerpAPI** credential.
   - Create an **OpenRouter API** credential.
   - Create an **Autype API** credential.
   - Create a **Google Drive OAuth2** credential if you want to save the PDF to Drive.

3. **Create the trigger form**
   - Add a **Form Trigger** node named **Market Research Form**.
   - Set:
     - Form title: `Market Research Report Generator`
     - Description: explain that the workflow researches the market and generates a professional PDF report
     - Response mode: `Last node`
     - Submitted text: `Your market research report is being generated. This may take a minute...`
   - Add fields:
     1. `productName` — Text — required
     2. `industry` — Text — required
     3. `productDescription` — Textarea — required
     4. `language` — Dropdown — required — options:
        - English
        - German
        - French
        - Spanish

4. **Create the Google Trends request**
   - Add a **SerpAPI** node named **Google Trends**.
   - Choose the operation for **Google Trends**.
   - Set the date range to `today 12-m`.
   - Map the query/search term to the submitted industry in the node UI.
   - Attach the SerpAPI credential.
   - Connect **Market Research Form → Google Trends**.

5. **Create the competitor search request**
   - Add another **SerpAPI** node named **Search Competitors**.
   - Configure it for normal Google search results.
   - Set number of results to `10`.
   - Use a query based on the product and market. A practical example:
     - `top competitors in {{$json.industry}}`
     - or `{{$json.productName}} competitors {{$json.industry}}`
   - Attach the same SerpAPI credential.
   - Connect **Market Research Form → Search Competitors**.

6. **Create the markdown syntax download**
   - Add an **HTTP Request** node named **Download Markdown Syntax**.
   - Set URL to:
     - `https://autype.com/llm-resources/markdown-syntax.md`
   - Set response format to **Text**.
   - Connect **Market Research Form → Download Markdown Syntax**.

7. **Merge trends and competitor results**
   - Add a **Merge** node named **Merge Trends + Competitors**.
   - Set:
     - Mode: `Combine`
     - Combine by: `Combine All`
   - Connect:
     - **Google Trends → Merge Trends + Competitors** input 1
     - **Search Competitors → Merge Trends + Competitors** input 2

8. **Merge syntax into the combined data**
   - Add another **Merge** node named **Merge With Syntax**.
   - Set:
     - Mode: `Combine`
     - Combine by: `Combine All`
   - Connect:
     - **Merge Trends + Competitors → Merge With Syntax** input 1
     - **Download Markdown Syntax → Merge With Syntax** input 2

9. **Create the research context normalizer**
   - Add a **Code** node named **Prepare Research Context**.
   - Paste logic equivalent to:
     - read all merged items
     - pull original form data from `Market Research Form`
     - extract Google Trends timeline data
     - summarize the last 6 points
     - extract `organic_results` and format top 10 competitor search results
     - extract markdown syntax text from HTTP response
     - return a single JSON object with:
       - `productName`
       - `industry`
       - `productDescription`
       - `language`
       - `trendsSummary`
       - `competitorSearchResults`
       - `markdownSyntax`
   - Connect **Merge With Syntax → Prepare Research Context**.

10. **Create the Perplexity model node**
    - Add an **OpenRouter Chat Model** node named **OpenRouter Perplexity**.
    - Select model:
      - `perplexity/sonar-pro`
    - Attach the OpenRouter credential.

11. **Create the research agent**
    - Add a **LangChain Agent** node named **AI Research Agent**.
    - Set prompt type to define your own prompt.
    - Main prompt should request:
      - market overview
      - top 5–8 competitors with details
      - current trends and 2-year predictions
      - product positioning
    - Include expressions:
      - product name
      - industry
      - product description
      - competitor search results
      - trend summary
      - language
    - Set a strong system message instructing the model to be factual, specific, and data-driven.
    - Set max iterations to `3`.
    - Connect:
      - **Prepare Research Context → AI Research Agent** main input
      - **OpenRouter Perplexity → AI Research Agent** AI model input

12. **Prepare the report input**
    - Add a **Code** node named **Prepare Report Input**.
    - Configure it to:
      - read research output from the previous node’s `output` field
      - re-read form input from `Market Research Form`
      - re-read context from `Prepare Research Context`
      - return:
        - `researchData`
        - `productName`
        - `industry`
        - `productDescription`
        - `language`
        - `trendsSummary`
        - `markdownSyntax`
    - Connect **AI Research Agent → Prepare Report Input**.

13. **Create the Claude model node**
    - Add an **OpenRouter Chat Model** node named **OpenRouter Anthropic Claude**.
    - Select model:
      - `anthropic/claude-sonnet-4.6`
    - Attach the OpenRouter credential.

14. **Create the report-writing agent**
    - Add a **LangChain Agent** node named **AI Report Writer**.
    - Feed it the report input from the previous node.
    - In the user prompt, ask it to write a professional report for the product and industry, using:
      - research data
      - trends summary
      - desired language
      - a title-page/TOC template
    - In the system message:
      - inject the full markdown syntax reference
      - enforce Autype formatting constraints
      - require these exact sections:
        1. Executive Summary
        2. Market Overview
        3. Trend Analysis
        4. Competitive Landscape
        5. SWOT Analysis
        6. Key Findings & Recommendations
        7. Sources
      - require tables for competitor comparison and SWOT
      - prohibit code fences in final output
    - Set max iterations to `3`.
    - Connect:
      - **Prepare Report Input → AI Report Writer** main input
      - **OpenRouter Anthropic Claude → AI Report Writer** AI model input

15. **Create the render payload preparation**
    - Add a **Code** node named **Prepare Render Payload**.
    - Configure it to:
      - read markdown from `AI Report Writer` output
      - remove surrounding triple backticks if present
      - create today’s date in `YYYY-MM-DD`
      - slugify product name
      - output:
        - `content`
        - `title` like `Market Research Report: {productName} - {industry}`
        - `filename` like `market-research-{slug}-{date}.pdf`
    - Connect **AI Report Writer → Prepare Render Payload**.

16. **Create the PDF renderer**
    - Add an **Autype** node named **Render Report PDF**.
    - Select operation:
      - `Render Markdown`
    - Set:
      - Content = `{{$json.content}}`
      - Download output = true
      - Size = `A4`
      - Title = `{{$json.title}}`
    - In defaults/style JSON, configure:
      - Open Sans, 11pt
      - line height 1.6
      - heading hierarchy with automatic page breaks on h1/h2
      - chart colors
      - blockquote indentation reset
      - header with company name and logo
      - footer with centered page number
    - Attach the Autype credential.
    - Connect **Prepare Render Payload → Render Report PDF**.

17. **Create the Google Drive upload**
    - Add a **Google Drive** node named **Save Report to Drive**.
    - Configure it to upload the binary file produced by Autype.
    - Set file name expression to:
      - `{{$('Prepare Render Payload').item.json.filename}}`
    - Select:
      - Drive: `My Drive`
      - Target folder: your chosen folder ID
    - Attach the Google Drive OAuth2 credential.
    - Connect **Render Report PDF → Save Report to Drive**.

18. **Add optional canvas notes**
    - Create sticky notes describing:
      - workflow purpose and prerequisites
      - form/data gathering block
      - AI research block
      - AI writing block
      - rendering/output block

19. **Test each stage independently**
    - Submit the form with a known company/product.
    - Verify:
      - Google Trends returns timeline data
      - competitor search returns `organic_results`
      - markdown syntax file is downloaded
      - research agent returns structured text in `output`
      - report writer returns markdown only
      - Autype returns a binary PDF
      - Google Drive upload succeeds

20. **Recommended hardening before production**
    - Explicitly define SerpAPI query expressions in both search nodes.
    - Add error handling paths for:
      - failed HTTP fetch
      - missing search results
      - model timeout/rate limit
      - PDF render failure
      - Google Drive upload failure
    - Consider replacing Google Drive with email, S3, Slack, or a webhook response depending on your delivery model.

### Credential Configuration Summary
- **SerpAPI**
  - API key from SerpAPI account
- **OpenRouter**
  - API key from OpenRouter account
- **Autype**
  - API key from Autype account settings
- **Google Drive OAuth2**
  - OAuth client with Drive access and permission to target folder

### Input / Output Expectations
- **Workflow input:** one form submission with 4 required fields
- **Intermediate outputs:**
  - Google Trends JSON
  - Search result JSON
  - Markdown syntax text
  - Research analysis text
  - Final markdown report
- **Final output:** rendered PDF uploaded to Google Drive

### Sub-workflow Setup
- This workflow does **not** invoke any sub-workflows.
- It has a single entry point: **Market Research Form**.
- It has one AI-model branch for research and one AI-model branch for report writing.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SerpAPI account required; free tier mentioned as 100 searches/month | [https://serpapi.com](https://serpapi.com) |
| OpenRouter account required for Perplexity Sonar and Claude Sonnet models | [https://openrouter.ai](https://openrouter.ai) |
| Autype account required; API keys available in account settings | [https://app.autype.com](https://app.autype.com) |
| Autype markdown syntax reference is downloaded dynamically by the workflow | `https://autype.com/llm-resources/markdown-syntax.md` |
| Community nodes required: `n8n-nodes-autype`, `n8n-nodes-serpapi` | Self-hosted n8n only |
| Workflow branding in PDF header uses `SearchIT Inc.` and an Icons8 logo | Header configuration inside Autype node |
| Report output is intended for product managers, startup founders, strategists, and consultants | Described in workflow notes |
| The report TOC template includes the German label `Inhaltsverzeichnis`; adjust if multilingual consistency is needed | AI Report Writer prompt |
| Google Drive is optional; output destination can be replaced with Email, S3, Slack, or another integration | Mentioned in sticky notes |