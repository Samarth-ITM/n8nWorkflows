Generate research-backed cold email icebreakers with web scraping, OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/generate-research-backed-cold-email-icebreakers-with-web-scraping--openai-and-google-sheets-13282


# Generate research-backed cold email icebreakers with web scraping, OpenAI and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate research-backed cold email icebreakers with web scraping, OpenAI and Google Sheets  
**Workflow name (in JSON):** Deep Multiline Icebreaker

**Purpose:**  
Collect campaign/product inputs via an n8n Form, fetch targeted leads from an external scraper (Apify Actor), scrape each company’s website (homepage + a few internal links), summarize each scraped page with OpenAI (GPT), aggregate the summaries, generate a personalized multi-line cold email in JSON, and append the final outreach content to Google Sheets.

**Target use cases:**
- Sales/BD outbound personalization at scale
- Generating “researched” email bodies that reference real on-site details
- Building a review queue in Google Sheets for outbound operations

### Logical blocks
**1.1 Input Reception & Normalization**
- Form collects product + targeting details and URL-encodes the strings for safe downstream API usage.

**1.2 Lead Collection (Apify) + Lead Filtering**
- Calls an Apify Actor to fetch leads; filters to only those with both company website and email.

**1.3 Website Research: Scrape homepage → extract internal links → normalize → pick top N**
- Scrapes homepage HTML, extracts `<a href>` links, splits them into items, keeps relative links, normalizes absolute/relative paths, removes duplicates, and limits to a few pages.

**1.4 Page Fetch → HTML-to-Markdown → AI Summarization**
- Fetches each chosen page, converts HTML to Markdown, and summarizes it into a JSON abstract using GPT.

**1.5 Aggregate Summaries → Generate Email → Store Output**
- Aggregates multiple page abstracts for the current lead, generates a tailored email JSON with GPT, and appends results to Google Sheets; loops to the next lead.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Normalization
**Overview:**  
Captures campaign details (product description, target designation, location, etc.) via a Form Trigger and URL-encodes all string fields before sending them to the lead scraper API.

**Nodes involved:**
- **Details** (Form Trigger)
- **formatting** (Code)

#### Node: Details
- **Type / role:** `n8n-nodes-base.formTrigger` — Entry point collecting user inputs via hosted form/webhook.
- **Configuration (interpreted):**
  - Form title: **“Outreach”**
  - Required fields:
    - About your Product (textarea)
    - Designation
    - Location
    - Your Name
    - Product URL
  - Optional field: Keywords
- **Key variables used downstream:**
  - `$('Details').item.json["About your Product"]`
  - `$('Details').item.json.Designation`
  - `$('Details').item.json.Location`
  - `$('Details').item.json["Your Name"]`
  - `$('Details').item.json["Product URL"]`
- **Connections:**
  - Output → **formatting**
- **Potential failure / edge cases:**
  - Missing required fields prevents submission (expected behavior).
  - Location/designation values may not match the lead provider’s taxonomy, leading to low/empty results.

#### Node: formatting
- **Type / role:** `n8n-nodes-base.code` — Normalizes/encodes strings for safer API consumption.
- **Configuration choices:**
  - JavaScript loops over all keys; if value is string, applies `encodeURIComponent`.
  - Note: despite the comment “Convert string to lowercase”, it does **not** lower-case; it only URL-encodes.
- **Input/Output:**
  - Input: form JSON
  - Output: same keys, URL-encoded
- **Connections:**
  - Output → **Leads Scraper1**
- **Edge cases / failures:**
  - URL-encoding the **Product URL** will turn `https://...` into encoded form. In this workflow, the Product URL is later referenced directly from **Details** (not from `formatting`), so this is mostly safe. But if you later reuse encoded values elsewhere, you may need `decodeURIComponent`.

---

### 2.2 Lead Collection (Apify) + Lead Filtering
**Overview:**  
Calls an Apify Actor endpoint to retrieve a dataset of leads, then filters to keep only those leads that have both a company website and an email.

**Nodes involved:**
- **Leads Scraper1** (HTTP Request)
- **Only Websites & Emails1** (Filter)

#### Node: Leads Scraper1
- **Type / role:** `n8n-nodes-base.httpRequest` — Calls Apify Actor `run-sync-get-dataset-items` to fetch leads.
- **Configuration choices:**
  - Method: **POST**
  - URL: `https://api.apify.com/v2/acts/IoSHqwTR9YGhzccez/run-sync-get-dataset-items`
  - Body: JSON with:
    - `contact_job_title`: `[ "{{ $json.Designation }}" ]` (Designation from the *encoded* form output)
    - `contact_location`: `[ "{{ $('Details').item.json.Location.toLowerCase() }}" ]` (lowercased original input, not encoded)
    - `email_status`: `["validated"]`
    - `fetch_count`: `100`
  - Headers:
    - `Authorization: Bearer <API Key>` (must be replaced with real Apify token)
    - `Accept: application/json`
- **Connections:**
  - Output → **Only Websites & Emails1**
- **Edge cases / failures:**
  - Invalid/expired Apify token → 401/403.
  - Actor ID changed/removed → 404.
  - `run-sync-get-dataset-items` can time out if actor run is slow.
  - Returned schema differences (field names not matching expected downstream: `company_website`, `email`, etc.).

#### Node: Only Websites & Emails1
- **Type / role:** `n8n-nodes-base.filter` — Ensures only actionable leads proceed.
- **Configuration choices:**
  - Condition 1: `company_website` **exists**
  - Condition 2: `email` **exists**
- **Connections:**
  - Output → **Scrape Home1**
- **Edge cases / failures:**
  - If Apify returns fields under different keys (e.g., `website` instead of `company_website`), all leads will be filtered out.
  - “Exists” check doesn’t validate URL/email format; malformed values still pass.

---

### 2.3 Website Research: Scrape homepage → extract links → normalize → select top N
**Overview:**  
For each lead, fetches the homepage, extracts all anchor links, splits them into individual items, keeps relative links, normalizes absolute links into paths, removes duplicates, and limits the number of pages to scrape/summarize.

**Nodes involved:**
- **Scrape Home1** (HTTP Request)
- **HTML1** (HTML Extract)
- **Edit Fields1** (Set)
- **Loop Over Items1** (Split In Batches)
- **Split Out1** (Split Out)
- **Filter1** (Filter)
- **Code1** (Code normalization)
- **Remove Duplicate URLs1** (Remove Duplicates)
- **Limit1** (Limit)

#### Node: Scrape Home1
- **Type / role:** `n8n-nodes-base.httpRequest` — Fetches company homepage HTML.
- **Configuration choices:**
  - URL: `={{ $json.company_website }}`
  - Redirect handling enabled (default redirect options present).
  - `allowUnauthorizedCerts: false`
  - `onError: continueErrorOutput` (workflow continues even if fetch fails; an error output item can be produced)
- **Connections:**
  - Output → **HTML1**
- **Edge cases / failures:**
  - 403/anti-bot, Cloudflare, or blocked user agents.
  - Timeout / large pages.
  - Non-HTML responses (PDF, JSON) will reduce extraction quality.
  - If request fails, HTML1 may receive no `data` depending on n8n’s error output behavior.

#### Node: HTML1
- **Type / role:** `n8n-nodes-base.html` — Extracts link URLs from HTML.
- **Configuration choices:**
  - Operation: Extract HTML content
  - CSS selector: `a`
  - Attribute: `href`
  - Return: array under `links`
  - Options: trim values + clean up text
- **Connections:**
  - Output → **Edit Fields1**
- **Edge cases / failures:**
  - Missing `data` field if HTTP node output differs; extraction may yield empty `links`.
  - Links can include `mailto:`, `tel:`, `#fragment`, JS pseudo-links, absolute URLs to other domains.

#### Node: Edit Fields1
- **Type / role:** `n8n-nodes-base.set` — Normalizes lead fields and attaches extracted links to the lead context.
- **Configuration choices (key assignments):**
  - `first_name`, `last_name`, `email`, `headline`, etc. copied from **Only Websites & Emails1**
  - `website_url` = `company_website`
  - `location` = `city + country`
  - `phone_number` = `company_phone`
  - `designation` = `headline`
  - `links` = `={{ $json.links }}` (from HTML1)
- **Connections:**
  - Output → **Loop Over Items1**
- **Edge cases / failures:**
  - If upstream fields (`city`, `country`, `company_phone`) are absent, expressions may yield `"undefined undefined"` or `undefined`.
  - If `links` isn’t an array, later split nodes may behave unexpectedly.

#### Node: Loop Over Items1
- **Type / role:** `n8n-nodes-base.splitInBatches` — Controls iteration over leads (batching/loop).
- **Configuration choices:**
  - Batch size not specified (defaults apply).
  - Two outputs used:
    - **Output 0:** loop continuation (used after writing to Sheets)
    - **Output 1:** current batch items (drives scraping pipeline)
- **Connections:**
  - Output 1 → **Split Out1**
  - (Later) **Add Row1** → Output 0 to continue loop
- **Edge cases / failures:**
  - If no items pass filtering, loop produces no iterations.
  - Miswiring can cause infinite loops; here it is intentionally wired for per-lead processing.

#### Node: Split Out1
- **Type / role:** `n8n-nodes-base.splitOut` — Converts `links[]` into one item per link.
- **Configuration choices:**
  - Field to split: `links`
  - `onError: continueErrorOutput`
- **Connections:**
  - Output → **Filter1**
- **Edge cases / failures:**
  - If `links` is empty or not an array, produces zero items or error output.
  - Large number of links can greatly expand item count (performance/API cost).

#### Node: Filter1
- **Type / role:** `n8n-nodes-base.filter` — Keeps only relative links.
- **Configuration choices:**
  - Condition: `links` starts with `/`
- **Connections:**
  - Output → **Code1**
- **Edge cases / failures:**
  - This drops absolute links entirely before normalization. If a site uses only absolute internal links, you may lose important pages.
  - Also drops `#...`, `mailto:`, and non-standard paths (which is often good).

#### Node: Code1
- **Type / role:** `n8n-nodes-base.code` — Normalizes links into relative paths.
- **Logic (interpreted):**
  - If `links` starts with `/`: keep as-is.
  - Else if starts with `http://` or `https://`: parse and keep only pathname, strip trailing slash (except root).
  - Else: leave unchanged.
- **Connections:**
  - Output → **Remove Duplicate URLs1**
- **Edge cases / failures:**
  - Because **Filter1** already enforces `/`, the “absolute URL” branch rarely runs in current wiring. If you want absolute internal links normalized, you’d typically move Code1 **before** Filter1 or adjust Filter1.
  - URL parsing failures are caught; original link preserved.

#### Node: Remove Duplicate URLs1
- **Type / role:** `n8n-nodes-base.removeDuplicates` — Deduplicates link items.
- **Configuration choices:**
  - Uses node defaults (in practice, dedupe by full JSON or selected fields depending on node version defaults).
  - `alwaysOutputData: true`, `onError: continueErrorOutput`
- **Connections:**
  - Output → **Limit1**
- **Edge cases / failures:**
  - If defaults dedupe by entire JSON, two items with same `links` but different metadata might not dedupe as intended. Consider explicitly selecting the `links` field if node supports it in your version.

#### Node: Limit1
- **Type / role:** `n8n-nodes-base.limit` — Caps number of pages to summarize per lead.
- **Configuration choices:**
  - `maxItems: 3`
- **Connections:**
  - Output → **Request web page for URL1**
- **Edge cases / failures:**
  - Only 3 internal pages are used (and note: homepage itself is not separately summarized in this workflow unless it appears in extracted links).

---

### 2.4 Page Fetch → Markdown Conversion → AI Summarization
**Overview:**  
Fetches each selected internal page, converts HTML to Markdown for cleaner LLM input, and asks GPT to return a two-paragraph abstract as JSON.

**Nodes involved:**
- **Request web page for URL1** (HTTP Request)
- **Markdown1** (Markdown)
- **Summarize Website Page1** (OpenAI)

#### Node: Request web page for URL1
- **Type / role:** `n8n-nodes-base.httpRequest` — Fetches each internal page.
- **Configuration choices:**
  - URL expression:  
    `={{ $('Loop Over Items1').item.json.website_url }}{{ $json.links }}`
    - Base URL from lead (`website_url`)
    - Path from current link item (`links`)
  - `onError: continueRegularOutput` (continues on error and sends something forward)
- **Connections:**
  - Output → **Markdown1**
- **Edge cases / failures:**
  - If `website_url` ends with `/` and `links` starts with `/`, you may get `https://site.com//path` (often tolerated by servers).
  - Relative paths without leading slash would break; Filter1 currently ensures leading `/`.
  - Fetching pages may return non-HTML; conversion/summarization quality drops.

#### Node: Markdown1
- **Type / role:** `n8n-nodes-base.markdown` — Converts HTML into Markdown.
- **Configuration choices:**
  - Input HTML: `={{ $json.data ? $json.data : "<div>empty</div>" }}`
  - If request failed/no `data`, uses `<div>empty</div>` as fallback.
- **Connections:**
  - Output → **Summarize Website Page1**
- **Edge cases / failures:**
  - If HTTP node returns content under a different key than `data`, you may always hit “empty”.
  - Very large HTML pages may produce huge Markdown; consider truncation to reduce token usage.

#### Node: Summarize Website Page1
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM summarization into structured JSON.
- **Configuration choices:**
  - Model: **gpt-4.1**
  - System prompt: website scraping assistant
  - User instructions: return `{"abstract":"..."}`; “no content” if empty; two-paragraph abstract, spartan tone
  - Third message content: `={{ $json.data }}`
  - `jsonOutput: true` (expects valid JSON)
- **Connections:**
  - Output → **Aggregate1**
- **Edge cases / failures:**
  - The node passes `{{$json.data}}` as content, but **Markdown1** typically outputs Markdown in a `data` field (depending on node version). If Markdown1 outputs to another field (e.g., `markdown`), summarization may receive wrong/empty input.
  - JSON formatting failures from the model can break downstream access to `message.content.abstract`. Using `jsonOutput: true` helps but does not guarantee perfection under all conditions.
  - Rate limits / token limits if pages are long.

---

### 2.5 Aggregate Summaries → Generate Email → Store Output (Loop)
**Overview:**  
Aggregates multiple page abstracts for the current lead, generates a personalized cold email JSON, appends it to Google Sheets, then triggers the next batch iteration.

**Nodes involved:**
- **Aggregate1** (Aggregate)
- **Generate Multiline Icebreaker1** (OpenAI)
- **Add Row1** (Google Sheets)

#### Node: Aggregate1
- **Type / role:** `n8n-nodes-base.aggregate` — Collects multiple summaries into one array for generation.
- **Configuration choices:**
  - Aggregates field: `message.content.abstract`
  - Output becomes an array (referenced later as `$json.abstract`)
- **Connections:**
  - Output → **Generate Multiline Icebreaker1**
- **Edge cases / failures:**
  - If summarization output schema differs (no `message.content.abstract`), aggregation will be empty.
  - Aggregation behavior depends on execution structure; ensure it aggregates per lead (not across multiple leads). With the current looping, verify batching doesn’t mix items from different leads.

#### Node: Generate Multiline Icebreaker1
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — Generates final email JSON from profile + website abstracts + product info.
- **Configuration choices:**
  - Model: **gpt-4.1**
  - Temperature: **0.5**
  - `jsonOutput: true`
  - Prompt includes a strict JSON format with an `"email"` field containing full email (subject + body).
  - Inputs injected:
    - Lead profile fields: `first_name`, `last_name`, `headline` from `Loop Over Items1`
    - Website abstracts: `={{ $json.abstract.join("/n") }}`
      - Note: this uses `"/n"` not `"\n"`, so it joins with literal `/n` instead of a newline. Likely a bug; should be `"\n"`.
    - Product description and product URL and your name from **Details**
- **Connections:**
  - Output → **Add Row1**
- **Edge cases / failures:**
  - If `$json.abstract` is not an array, `.join()` fails.
  - If the model returns JSON that doesn’t include `message.content.email` (or returns `icebreaker` instead), Google Sheets mapping breaks.
  - Prompt contains an embedded example assistant response with `{"icebreaker": ...}` which conflicts with the required output schema (`{"email": ...}`). This increases the chance the model outputs the wrong key. Consider removing that embedded assistant example or aligning it to `"email"`.

#### Node: Add Row1
- **Type / role:** `n8n-nodes-base.googleSheets` — Appends results to a sheet.
- **Configuration choices:**
  - Operation: **Append**
  - Document: “Multiline Icebreaker Generator” (Google Spreadsheet ID provided)
  - Sheet/tab: “Leads”
  - Columns mapped:
    - Lead metadata (first/last/email/location/designation/website_url/phone_number)
    - Outreach Content: `={{ $json.message.content.email }}`
  - `phone_number` stored as JSON string: `={{ JSON.stringify($('Loop Over Items1').item.json.phone_number) }}`
- **Connections:**
  - Output → **Loop Over Items1** (loop continuation output 0)
- **Credentials:**
  - Google Sheets OAuth2 required (`googleSheetsOAuth2Api`)
- **Edge cases / failures:**
  - Sheet schema mismatch (missing columns, different header names) leads to append errors.
  - If GPT output path differs (e.g., `$json.email` instead of `$json.message.content.email`), Outreach Content will be blank or expression error.
  - Google API rate limits with high volumes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note1 | Sticky Note | Workflow description & setup links | — | — | # Deep Multiline Icebreaker  \n**Generate research-backed icebreakers with web scraping, OpenAI, and Google Sheets**  \nDemo & Setup Video: https://drive.google.com/file/d/1HpkWNCC0YC_z2Yag41Hf68o54jQTnZJy/view?usp=sharing  \nSheet Template: https://docs.google.com/spreadsheets/d/1X9Y1VR-aqzoEV56jlCsPVMlOt5zCl_MUAvs1TwgnZic/edit?usp=sharing  \nCourse: https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |
| Details | Form Trigger | Collect outreach/product inputs | — | formatting | # Input & Lead Collection  \nCollects campaign details via form input and scrapes targeted leads using an external source. Only valid leads with emails and company websites move forward for personalization. |
| formatting | Code | URL-encode form strings before API call | Details | Leads Scraper1 | # Input & Lead Collection  \nCollects campaign details via form input and scrapes targeted leads using an external source. Only valid leads with emails and company websites move forward for personalization. |
| Leads Scraper1 | HTTP Request | Fetch leads from Apify Actor | formatting | Only Websites & Emails1 | # Input & Lead Collection  \nCollects campaign details via form input and scrapes targeted leads using an external source. Only valid leads with emails and company websites move forward for personalization. |
| Only Websites & Emails1 | Filter | Keep leads with website + email | Leads Scraper1 | Scrape Home1 | # Input & Lead Collection  \nCollects campaign details via form input and scrapes targeted leads using an external source. Only valid leads with emails and company websites move forward for personalization. |
| Scrape Home1 | HTTP Request | Fetch company homepage HTML | Only Websites & Emails1 | HTML1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| HTML1 | HTML | Extract all `<a href>` links | Scrape Home1 | Edit Fields1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Edit Fields1 | Set | Map lead fields + attach links array | HTML1 | Loop Over Items1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Loop Over Items1 | Split In Batches | Iterate through leads (loop control) | Edit Fields1 (start), Add Row1 (continue) | Split Out1 (output 1) | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Split Out1 | Split Out | One item per extracted link | Loop Over Items1 | Filter1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Filter1 | Filter | Keep relative links starting with `/` | Split Out1 | Code1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Code1 | Code | Normalize links (absolute → path, trim trailing slash) | Filter1 | Remove Duplicate URLs1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Remove Duplicate URLs1 | Remove Duplicates | Deduplicate link items | Code1 | Limit1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Limit1 | Limit | Cap number of pages per lead | Remove Duplicate URLs1 | Request web page for URL1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Request web page for URL1 | HTTP Request | Fetch each internal page HTML | Limit1 | Markdown1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Markdown1 | Markdown | Convert HTML → Markdown | Request web page for URL1 | Summarize Website Page1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Summarize Website Page1 | OpenAI (LangChain) | Summarize each page into JSON abstract | Markdown1 | Aggregate1 | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Aggregate1 | Aggregate | Combine page abstracts per lead | Summarize Website Page1 | Generate Multiline Icebreaker1 | # Icebreaker Generation & Output  \nGenerates personalized multi-line cold email icebreakers using AI and saves the final results to Google Sheets for review and outbound use. |
| Generate Multiline Icebreaker1 | OpenAI (LangChain) | Generate final email JSON | Aggregate1 | Add Row1 | # Icebreaker Generation & Output  \nGenerates personalized multi-line cold email icebreakers using AI and saves the final results to Google Sheets for review and outbound use. |
| Add Row1 | Google Sheets | Append lead + outreach content to sheet | Generate Multiline Icebreaker1 | Loop Over Items1 | # Icebreaker Generation & Output  \nGenerates personalized multi-line cold email icebreakers using AI and saves the final results to Google Sheets for review and outbound use. |
| Sticky Note | Sticky Note | Block description | — | — | # Input & Lead Collection  \nCollects campaign details via form input and scrapes targeted leads using an external source. Only valid leads with emails and company websites move forward for personalization. |
| Sticky Note2 | Sticky Note | Block description | — | — | # Website Research & AI Analysis  \nScrapes company websites, extracts key internal pages, and uses AI to summarize each page. All insights are combined to build deep, research-backed context for outreach. |
| Sticky Note3 | Sticky Note | Block description | — | — | # Icebreaker Generation & Output  \nGenerates personalized multi-line cold email icebreakers using AI and saves the final results to Google Sheets for review and outbound use. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it (e.g.) **Deep Multiline Icebreaker**
- Ensure workflow setting **Execution Order** is set to **v1** (matches JSON).

2) **Add Form Trigger (Details)**
- Node: **Form Trigger**
- Form title: `Outreach`
- Add fields:
  - “About your Product” (Textarea, required)
  - “Designation” (Text, required)
  - “Location” (Text, required)
  - “Keywords” (Text, optional)
  - “Your Name” (Text, required)
  - “Product URL” (Text, required)
- Save so n8n generates a form URL/webhook.

3) **Add Code node (formatting)**
- Node: **Code**
- Paste logic to URL-encode all string fields (encodeURIComponent).
- Connect: **Details → formatting**

4) **Add HTTP Request node (Leads Scraper1)**
- Node: **HTTP Request**
- Method: `POST`
- URL: `https://api.apify.com/v2/acts/IoSHqwTR9YGhzccez/run-sync-get-dataset-items`
- Body type: JSON
- JSON body (expressions):
  - `contact_job_title`: array containing `{{ $json.Designation }}`
  - `contact_location`: array containing `{{ $('Details').item.json.Location.toLowerCase() }}`
  - `email_status`: `["validated"]`
  - `fetch_count`: `100`
- Headers:
  - `Authorization`: `Bearer <YOUR_APIFY_API_KEY>`
  - `Accept`: `application/json`
- Connect: **formatting → Leads Scraper1**

5) **Add Filter node (Only Websites & Emails1)**
- Node: **Filter**
- Conditions:
  - `{{$json.company_website}}` exists
  - `{{$json.email}}` exists
- Connect: **Leads Scraper1 → Only Websites & Emails1**

6) **Add HTTP Request node (Scrape Home1)**
- Node: **HTTP Request**
- URL: `={{ $json.company_website }}`
- Enable redirects (defaults ok)
- Error handling: set **On Error** to “Continue (error output)” if you want the same resilience.
- Connect: **Only Websites & Emails1 → Scrape Home1**

7) **Add HTML node (HTML1)**
- Node: **HTML**
- Operation: Extract HTML Content
- Extract:
  - Key: `links`
  - Selector: `a`
  - Return value: attribute `href`
  - Return as array: enabled
- Connect: **Scrape Home1 → HTML1**

8) **Add Set node (Edit Fields1)**
- Node: **Set**
- Add fields (use expressions pointing to Only Websites & Emails1 item):
  - `first_name`, `last_name`, `email`, `headline`
  - `website_url` = company_website
  - `location` = `city + " " + country`
  - `phone_number` = company_phone
  - `designation` = headline
  - `links` = `{{$json.links}}`
- Connect: **HTML1 → Edit Fields1**

9) **Add Split In Batches (Loop Over Items1)**
- Node: **Split In Batches**
- Keep default batch size or set explicitly.
- Connect: **Edit Fields1 → Loop Over Items1**
- You will later connect **Add Row1 → Loop Over Items1** to continue the loop.

10) **Add Split Out (Split Out1)**
- Node: **Split Out**
- Field to split: `links`
- Connect from **Loop Over Items1** using **output 1** (the “items” output) → **Split Out1**

11) **Add Filter (Filter1)**
- Node: **Filter**
- Condition: `={{ $json.links }}` starts with `/`
- Connect: **Split Out1 → Filter1**

12) **Add Code node (Code1)**
- Node: **Code**
- Paste link normalization logic (relative keep; absolute URL → pathname; trim trailing slash).
- Connect: **Filter1 → Code1**

13) **Add Remove Duplicates (Remove Duplicate URLs1)**
- Node: **Remove Duplicates**
- Configure dedupe field if available in your version (prefer `links`).
- Set **On Error** to continue if desired.
- Connect: **Code1 → Remove Duplicate URLs1**

14) **Add Limit (Limit1)**
- Node: **Limit**
- Max items: `3`
- Connect: **Remove Duplicate URLs1 → Limit1**

15) **Add HTTP Request (Request web page for URL1)**
- Node: **HTTP Request**
- URL expression:
  - `={{ $('Loop Over Items1').item.json.website_url }}{{ $json.links }}`
- Set **On Error** to “Continue (regular output)” if you want to proceed even if a page fails.
- Connect: **Limit1 → Request web page for URL1**

16) **Add Markdown node (Markdown1)**
- Node: **Markdown**
- HTML field:
  - `={{ $json.data ? $json.data : "<div>empty</div>" }}`
- Connect: **Request web page for URL1 → Markdown1**

17) **Add OpenAI node (Summarize Website Page1)**
- Node: **OpenAI (LangChain)**
- Credentials: configure **OpenAI API** credential in n8n.
- Model: `gpt-4.1`
- Enable structured JSON output (`jsonOutput: true`)
- Messages:
  - System: website scraping assistant
  - User: instructions to output `{"abstract":"..."}` with “no content” if empty
  - User content: `={{ $json.data }}`
- Connect: **Markdown1 → Summarize Website Page1**

18) **Add Aggregate node (Aggregate1)**
- Node: **Aggregate**
- Aggregate field: `message.content.abstract`
- Connect: **Summarize Website Page1 → Aggregate1**

19) **Add OpenAI node (Generate Multiline Icebreaker1)**
- Node: **OpenAI (LangChain)**
- Model: `gpt-4.1`
- Temperature: `0.5`
- `jsonOutput: true`
- Prompt: provide required JSON schema with `"email"` field and embed:
  - Lead profile from `$('Loop Over Items1').item.json...`
  - Website abstracts from `{{$json.abstract.join("\n")}}` (recommended fix vs `/n`)
  - Product description, your name, product URL from `$('Details').item.json[...]`
- Connect: **Aggregate1 → Generate Multiline Icebreaker1**

20) **Add Google Sheets node (Add Row1)**
- Node: **Google Sheets**
- Credentials: configure **Google Sheets OAuth2** in n8n.
- Operation: **Append**
- Select Spreadsheet ID and Sheet name (use the provided template if desired).
- Map columns:
  - `first_name`, `last_name`, `designation`, `email`, `website_url`, `location`, `phone_number`
  - `Outreach Content` = `={{ $json.message.content.email }}`
- Connect: **Generate Multiline Icebreaker1 → Add Row1**

21) **Close the loop**
- Connect: **Add Row1 → Loop Over Items1** using **output 0** (continue to next batch).
- This makes the workflow process each lead sequentially.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | https://drive.google.com/file/d/1HpkWNCC0YC_z2Yag41Hf68o54jQTnZJy/view?usp=sharing |
| Google Sheet Template | https://docs.google.com/spreadsheets/d/1X9Y1VR-aqzoEV56jlCsPVMlOt5zCl_MUAvs1TwgnZic/edit?usp=sharing |
| Course link (author-provided) | https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |

