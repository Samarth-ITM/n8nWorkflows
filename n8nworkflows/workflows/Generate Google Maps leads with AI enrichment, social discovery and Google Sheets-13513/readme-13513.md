Generate Google Maps leads with AI enrichment, social discovery and Google Sheets

https://n8nworkflows.xyz/workflows/generate-google-maps-leads-with-ai-enrichment--social-discovery-and-google-sheets-13513


# Generate Google Maps leads with AI enrichment, social discovery and Google Sheets

## 1. Workflow Overview

**Title:** Generate Google Maps leads with AI enrichment, social discovery and Google Sheets  
**Workflow name (in JSON):** Automate Google Maps lead generation with AI enrichment and social discovery  
**Purpose:** Given one or more Google Maps-style search queries (e.g., “dentists in Pune”), the workflow pulls Google Maps business results via Serper, extracts key business fields, scrapes each website to find emails and social links, performs a secondary Serper web search to discover missing social profiles, validates emails, enriches each lead with LLM-generated business insights and an outreach message, scores/deduplicates leads, then exports everything to Google Sheets.

### Logical blocks
1. **Input & Query Iteration**: Accepts a list of queries and iterates with rate limiting.
2. **Google Maps Harvesting (Serper Maps)**: Pulls multiple pages of Maps results and extracts businesses.
3. **Website Scraping & On-page Contact Extraction**: Visits business websites, extracts socials and email from HTML, handles broken pages.
4. **Social Discovery (Serper Web Search) + Social Confidence Scoring**: Searches social platforms and merges results with scraped data.
5. **Email Validation + Lead Normalization**: Validates emails when present, standardizes lead fields and phone formatting.
6. **Deduplication, Lead Scoring, AI Enrichment & Message Generation**: Dedupes, scores, calls LLM for services/summary and outreach copy.
7. **Export to Google Sheets**: Appends or updates rows in a Google Sheet.

---

## 2. Block-by-Block Analysis

### 2.1 Input & Query Iteration
**Overview:** Starts the workflow manually, reads a list of search queries, splits them into items, and iterates with a throttling mechanism to reduce API rate-limit risks.  
**Nodes involved:** `Start Lead Generation`, `Split Search Queries`, `Process Each Query`, `Rate Limit Protection`

#### Node: Start Lead Generation
- **Type / role:** Manual Trigger — entry point.
- **Configuration:** No parameters. Pinned data shows example input:
  - `queries: ["bakery littlevenice"]`
- **Outputs:** To `Split Search Queries`.
- **Edge cases:** If `queries` is missing or not an array, downstream split will produce no items or error.

#### Node: Split Search Queries
- **Type / role:** Split Out — converts an array into multiple items.
- **Configuration (interpreted):**
  - Splits field `queries`
  - Places each element into field `query`
- **Input:** From `Start Lead Generation`.
- **Output:** To `Process Each Query`.
- **Edge cases:** Non-array `queries` → empty output or runtime error depending on n8n behavior.

#### Node: Process Each Query
- **Type / role:** Split In Batches — controls iteration over queries.
- **Configuration:** Default options (batch size not explicitly set here).
- **Outputs (two paths):**
  - Main output → `Fetch Maps Results (Page 1)` (start query processing)
  - Also triggers → `Rate Limit Protection` (throttle loop)
- **Connections pattern:** A “wait then loop back” pattern exists:
  - `Rate Limit Protection` → `Process Each Query` (to continue next batch)
- **Edge cases:** If a query fails downstream and no item returns, batch iteration can stop unexpectedly.

#### Node: Rate Limit Protection
- **Type / role:** Wait — throttles query processing loop.
- **Configuration:** Wait `amount: 3` (seconds).
- **Input:** From `Process Each Query`.
- **Output:** Back to `Process Each Query` to continue batching.
- **Edge cases:** Long query lists will increase runtime linearly; if hosting has execution time limits, may time out.

---

### 2.2 Google Maps Harvesting (Serper Maps)
**Overview:** For each query, fetches Maps results from Serper (page 1 plus pages 2–12), then extracts businesses with phone numbers into a normalized list.  
**Nodes involved:** `Fetch Maps Results (Page 1)`, `Delay Between Requests`, `Fetch Maps Results (Pages 2–12)`, `Extract Businesses From Maps`

**Sticky note (applies to this block):**
- **Content:** “Maps section: Searches Google Maps using your queries and pulls business name, website, phone number, rating and category for each result.”

#### Node: Fetch Maps Results (Page 1)
- **Type / role:** HTTP Request — calls Serper Maps endpoint for first page.
- **Configuration:**
  - `GET https://google.serper.dev/maps?q={{ $json.query }}`
  - Auth: Generic credential type using **HTTP Header Auth** (Serper API key typically in `X-API-KEY` header).
- **Input:** Query items from `Process Each Query`.
- **Output:** To `Delay Between Requests`.
- **Failure types:**
  - 401/403 if Serper API key missing/invalid
  - 429 if rate limited
  - Unexpected schema changes in response

#### Node: Delay Between Requests
- **Type / role:** Wait — spacing before bulk multi-page call.
- **Configuration:** No explicit amount set in JSON (defaults to node’s standard wait behavior; typically requires an amount—verify in UI).
- **Input:** From `Fetch Maps Results (Page 1)`.
- **Output:** To `Fetch Maps Results (Pages 2–12)`.
- **Edge cases:** If effectively “0 seconds”, may not help rate limiting; if misconfigured, could stall.

#### Node: Fetch Maps Results (Pages 2–12)
- **Type / role:** HTTP Request — bulk request for pages 2–12.
- **Configuration:**
  - POST `https://google.serper.dev/maps`
  - Body is a JSON array of 11 requests (page 2..12), using:
    - `q: {{ $json.searchParameters.q }}`
    - `ll: {{ $json.ll }}`
  - Auth: same Serper header auth.
- **Input:** From `Delay Between Requests` (expects page-1 Serper response to contain `searchParameters.q` and `ll`).
- **Output:** To `Extract Businesses From Maps`.
- **Edge cases / failures:**
  - If Serper response doesn’t contain `ll` or `searchParameters.q`, expressions resolve to empty → bad requests.
  - POST array format must be supported by Serper; if API changes, this fails.
  - 429 throttling for 11-page burst.

#### Node: Extract Businesses From Maps
- **Type / role:** Code — parses Serper maps output and produces business items.
- **Configuration (logic):**
  - Combines:
    - All items from `Fetch Maps Results (Page 1)`
    - plus current input items (pages 2–12 results)
  - For each `place` in `item.json.places`:
    - Keeps only if `place.phoneNumber` exists
    - Extracts: `name (title)`, `phoneNumber`, `website`, `rating`, `type`
- **Output:** Each company as its own item → `Process Businesses in Batches`.
- **Edge cases:**
  - Businesses without phone numbers are discarded (intentional but may reduce lead volume).
  - `place.website` may be missing; downstream scraping uses it and may fail/skip.
  - Duplicates across pages are not removed until later.

---

### 2.3 Website Scraping & On-page Contact Extraction
**Overview:** Visits each business website, extracts social profile links and email addresses from HTML/response text, and then prepares a “smart” social search query.  
**Nodes involved:** `Process Businesses in Batches`, `Scrape Business Website`, `Extract Socials & Email`, `Build Social Search Query`

**Sticky note (applies to this block):**
- **Content:** “Scraping section: Visits each business website and looks for emails and social media links. Handles broken pages and redirects automatically.”

#### Node: Process Businesses in Batches
- **Type / role:** Split In Batches — controls scraping throughput.
- **Configuration:** `batchSize: 5`
- **Outputs:**
  - To `Scrape Business Website` (scrape path)
  - To `Check Email Exists` (email validation path is started early but ultimately relies on normalized data later; see note below)
- **Edge cases:** If batch size too high, can overload target websites or hit Serper/LLM limits later.

> **Important workflow design note:** This node fans out to both scraping and email validation logic. However, the `Check Email Exists` node references `Normalize Social Data`, which happens later after social resolution. In practice, this means execution order/availability of referenced items can be fragile if run outside the intended path. The workflow still works because n8n expression references can access other nodes’ items, but it requires consistent item alignment.

#### Node: Scrape Business Website
- **Type / role:** HTTP Request — fetches website HTML.
- **Configuration:**
  - URL: `{{ $json.website }}`
  - Headers: User-Agent + Accept + Accept-Language
  - `allowUnauthorizedCerts: true`
  - **onError:** `continueRegularOutput` (does not fail the workflow on HTTP errors)
- **Input:** Business items.
- **Output:** To `Extract Socials & Email`.
- **Failure types / edge cases:**
  - Website missing/invalid URL → request fails but continues; extraction will likely yield “-” and no email.
  - Some sites block bots; you may get 403, CAPTCHA HTML, or JS-heavy content.

#### Node: Extract Socials & Email
- **Type / role:** Code — parses HTML/response to find socials and emails.
- **Configuration (logic):**
  - Converts entire response JSON to a lowercase string (`html`)
  - Extracts first matching URL for each social platform via regex:
    - Instagram, Facebook, LinkedIn, Twitter/X, YouTube, TikTok
  - Extracts emails:
    - Handles obfuscation patterns like `[at]`, `(dot)`, ` at `
    - Filters “bad” placeholders like `example.com`, `test.com`, `noreply@`
    - Deduplicates and selects first email if present
  - Returns combined object using:
    - Base business fields from `Extract Businesses From Maps`.item.json
    - Plus `email` and social URLs
- **Outputs:**
  - To `Combine Website + Serper Results` (merge input 0)
  - To `Build Social Search Query`
- **Edge cases:**
  - Regex may pick a non-profile link (e.g., share links) but later social resolver tries to filter.
  - If website content is not in response body (binary, redirects, JS-rendered), extraction quality drops.

#### Node: Build Social Search Query
- **Type / role:** Code — creates a more robust social search query string.
- **Configuration (logic):**
  - Cleans/normalizes business name (punctuation removal, accent stripping, noise-word removal)
  - Builds variations (quoted/unquoted) and optionally city (but `city` is not populated anywhere upstream)
  - Adds social site constraints via `(site:instagram.com OR site:facebook.com ... )`
  - Outputs `smartQuery`
- **Output:** To `Search Social Profiles (Serper)`.
- **Edge cases:**
  - `smartQuery` is computed but **not used** by the next node (next node uses `q: {{ $json.name }}`), so improvements are currently unused unless you modify `Search Social Profiles` to use `smartQuery`.
  - Removing “noise words” may harm accuracy for some niches (e.g., “Coffee House” brand name).

---

### 2.4 Social Discovery (Serper Web Search) + Confidence Scoring
**Overview:** Searches the web for social profiles using Serper, parses candidate profile URLs, merges with scraped socials, and assigns a confidence score per platform.  
**Nodes involved:** `Search Social Profiles (Serper)`, `Parse Social Results`, `Combine Website + Serper Results`, `Resolve & Score Social Profiles`, `Normalize Social Data`

**Sticky note (applies to this block):**
- **Content:** “Social section: If social profiles are not found on the website, it runs a separate search to find Instagram, Facebook, LinkedIn, Twitter, TikTok and YouTube with a confidence score for each.”

#### Node: Search Social Profiles (Serper)
- **Type / role:** HTTP Request — Serper Search endpoint.
- **Configuration:**
  - POST `https://google.serper.dev/search`
  - JSON body:
    - `q: {{ $json.name }}`
    - `num: 10`, `gl: "us"`
    - `sites: [instagram.com, facebook.com, ...]`
  - Auth: Serper header auth.
- **Input:** From `Build Social Search Query`.
- **Output:** To `Parse Social Results`.
- **Edge cases:**
  - This node ignores `smartQuery`; results can be worse for ambiguous names.
  - Serper may return non-company pages (posts, videos, share links).

#### Node: Parse Social Results
- **Type / role:** Code — extracts first matching URLs from Serper response.
- **Configuration (logic):**
  - Stringifies Serper JSON response
  - Regex extracts first match for each platform into:
    - `serperInstagram`, `serperFacebook`, `serperLinkedin`, `serperTwitter`, `serperYoutube`, `serperTiktok`
  - Tries to exclude certain paths (e.g., IG `/reel|p/`, FB `share|sharer|videos`, etc.)
- **Output:** To `Combine Website + Serper Results` (merge input 1).
- **Edge cases:**
  - “First match wins” can pick wrong business if name collision.
  - Regex for LinkedIn is limited to `/company/` pages (misses `/in/` personal pages, which might be acceptable).

#### Node: Combine Website + Serper Results
- **Type / role:** Merge — combines scraped and Serper-parsed items by position.
- **Configuration:** `mode: combine`, `combineBy: position`
- **Inputs:**
  - Input 0: from `Extract Socials & Email`
  - Input 1: from `Parse Social Results`
- **Output:** To `Resolve & Score Social Profiles`.
- **Edge cases:**
  - Position-based combine requires both branches to emit items in the same order and count. If one branch errors/skips items, data can misalign (wrong socials assigned to wrong business).

#### Node: Resolve & Score Social Profiles
- **Type / role:** Code — reconciles website-found socials with Serper-discovered socials and assigns confidence.
- **Configuration (logic):**
  - `clean(url)` filters out undesirable patterns (IG posts, YouTube video links, Twitter statuses, etc.)
  - `merge(primary, fallback)`:
    - If only website URL → HIGH
    - If only Serper URL → MEDIUM
    - If both and same username → HIGH
    - If both but different → combine as `"primary | fallback"` with MEDIUM
    - If neither → `"-"` with LOW
  - Outputs per platform: URL + `XConfidence`
- **Output:** To `Normalize Social Data`.
- **Edge cases:**
  - Username extraction by `url.split("/").pop()` can fail on URLs ending in query strings or trailing slashes (partly handled by `clean`).
  - Combined `"a | b"` URLs can be inconvenient downstream; you may prefer selecting the best single URL.

#### Node: Normalize Social Data
- **Type / role:** Code — standardizes empty values and shapes final lead object pre-validation.
- **Configuration (logic):**
  - `safe()` converts null/empty → `"-"`
  - Keeps `email` as empty string if absent (important for validation step)
  - Ensures confidence defaults to LOW
- **Output:** To `Process Businesses in Batches` (this connection exists in JSON; it creates a loop-like structure that can be confusing—validate in UI).
- **Edge cases:**
  - Because it wraps output as `{ json: {...} }`, downstream nodes must reference `.json.*` fields (which they do).

---

### 2.5 Email Validation + Lead Normalization
**Overview:** If an email exists, validates it through an external API; otherwise marks as not found. Produces a unified lead record regardless of validation path, then formats phone and cleans fields.  
**Nodes involved:** `Check Email Exists`, `Validate Email Address`, `Prepare Valid Leads`, `Prepare Leads Without Email`, `Merge Lead Results`, `Clean Lead Data`

**Sticky note (applies to this block):**
- **Content:** “Email validation section: Checks every email found and removes ones that are invalid or likely to bounce before saving to your sheet.”

#### Node: Check Email Exists
- **Type / role:** IF — routes depending on whether email is non-empty.
- **Configuration:**
  - Condition: `{{ $('Normalize Social Data').item.json.email }}` **is not empty**
  - Case-insensitive, strict validation.
- **Outputs:**
  - True → `Validate Email Address`
  - False → `Prepare Leads Without Email`
- **Edge cases:**
  - This IF reads from `Normalize Social Data` via expression; if item alignment differs, it may check the wrong item’s email.
  - Email may be present but placeholder; validation API handles some but not all cases.

#### Node: Validate Email Address
- **Type / role:** HTTP Request — external email verification.
- **Configuration:**
  - GET/Query call to `https://rapid-email-verifier.fly.dev/api/validate?email={{ $json.email }}`
- **Input:** From `Check Email Exists` (true branch) and expects `$json.email`.
- **Output:** To `Prepare Valid Leads`.
- **Failure types:** API downtime/timeouts; rate limiting; schema changes (`result` vs `status`).

#### Node: Prepare Valid Leads
- **Type / role:** Code — combines original lead + validation result into normalized output.
- **Configuration (logic):**
  - Reads original lead from `$('Check Email Exists').item.json`
  - Reads validator result from current `$json.result || $json.status`
  - If invalid/undeliverable → clears email and sets `emailStatus = NOT_FOUND_OR_INVALID`
  - If risky/catch-all/unknown → `emailStatus = RISKY`
  - Else: `emailStatus = VALID`
- **Output:** To `Merge Lead Results` (input 0).

#### Node: Prepare Leads Without Email
- **Type / role:** Code — standardizes lead when no email was found.
- **Configuration:** Sets `email: ""`, `emailStatus: "NOT_FOUND"` and passes other fields.
- **Output:** To `Merge Lead Results` (input 1).

#### Node: Merge Lead Results
- **Type / role:** Merge — reunifies the two email branches.
- **Configuration:** Default merge behavior (not explicitly set).
- **Input:** From both “with email” and “without email” branches.
- **Output:** To `Clean Lead Data`.
- **Edge cases:** Depending on merge mode defaults, may behave as “append” or “merge by position”. Verify in UI; incorrect mode can drop items.

#### Node: Clean Lead Data
- **Type / role:** Code — ensures safe placeholders and formats phone numbers for Sheets.
- **Configuration (logic):**
  - `safe()` converts undefined/null/empty → `"-"`
  - `formatPhone()`:
    - Strips non-digits, handles optional country code
    - Formats 10-digit US numbers as `'(XXX) XXX-XXXX` and prefixes with `'` to force text in Google Sheets
    - Intl formatting: `(+CC) grouped digits`, also prefixed with `'`
  - Outputs clean object with socials + confidences
- **Output:** To `Calculate Lead Score`.
- **Edge cases:**
  - Non-US 10-digit numbers may be incorrectly formatted as US.
  - Prefixing `'` is helpful for Sheets but may be undesirable if exporting elsewhere.

---

### 2.6 Deduplication, Lead Scoring, AI Enrichment & Message Generation
**Overview:** Scores each lead, removes duplicates, then calls an LLM to generate services + summary and a short outreach message, with wait buffers to reduce rate limiting/timeouts.  
**Nodes involved:** `Calculate Lead Score`, `Remove Duplicate Leads`, `Process Leads for AI Enrichment`, `AI Rate Limit Buffer`, `Analyze Business (AI)`, `Extract AI Insights`, `AI Processing Buffer`, `Process Outreach Message`, `Message Rate Limit Buffer`, `Create Outreach Message`, `Clean Outreach Message`

**Sticky notes (apply to this block):**
- **AI section note:** “Sends each business to an AI model which reads the data and writes a short summary, lists their services, and creates a personalized outreach message…”
- **Ollama setup note:** Contains endpoint examples and alternatives:
  - OpenAI-compatible: `https://api.openai.com/v1/chat/completions`, Groq, Together, Mistral, Anthropic, etc.
  - Reminder to update Authorization header if using hosted APIs.
- **Export section note (partly overlaps):** scoring and dedupe described there.

#### Node: Calculate Lead Score
- **Type / role:** Code — assigns a score out of 10.
- **Configuration (logic):**
  - Adds points for: website present, Instagram present, multiple socials, high rating, valid email
  - Adds “growth opportunity bonus” if no socials; also bonus if Instagram exists but website missing
  - Outputs `leadScore` as string like `"7/10"`
- **Output:** To `Remove Duplicate Leads`.
- **Edge cases:** Scores depend heavily on placeholder `"-"`; must keep consistent placeholder conventions.

#### Node: Remove Duplicate Leads
- **Type / role:** Remove Duplicates — dedupe by selected fields.
- **Configuration:** Compare selected fields: `website, name`
- **Output:** To `Process Leads for AI Enrichment`.
- **Edge cases:** If website is `"-"` for many, name-only duplicates might not be removed; consider including phone or place ID if available.

#### Node: Process Leads for AI Enrichment
- **Type / role:** Split In Batches — controls AI throughput.
- **Configuration:** Default options.
- **Outputs:**
  - To `Extract AI Insights` (index 0) — note: this is unusual because `Extract AI Insights` expects an AI response; verify path correctness.
  - To `AI Rate Limit Buffer` (index 0) → AI call
- **Edge cases:** Miswiring risk: `Extract AI Insights` is logically supposed to run **after** `Analyze Business (AI)`. In the provided connections, `Analyze Business (AI)` outputs back into `Process Leads for AI Enrichment`, which creates a loop and can confuse execution order. Confirm in n8n canvas that the intended path is:
  - Leads → Wait → Analyze AI → Extract Insights → Wait → Message generation.

#### Node: AI Rate Limit Buffer
- **Type / role:** Wait — buffers calls to AI.
- **Configuration:** `amount: 2` seconds.
- **Output:** To `Analyze Business (AI)`.

#### Node: Analyze Business (AI)
- **Type / role:** HTTP Request — calls an LLM endpoint (Ollama example) to generate structured JSON.
- **Configuration:**
  - POST `YOUR_OLLAMA_URL/api/generate`
  - Timeout: 60s
  - Body includes prompt requesting **ONLY valid JSON**:
    - `{"services":["",""],"summary":""}`
  - Headers: `Content-Type: application/json`
- **Output:** Connected to `Process Leads for AI Enrichment` (loop). Intended next step is `Extract AI Insights`.
- **Failure types / edge cases:**
  - Endpoint unreachable (common on n8n Cloud without public Ollama)
  - Model returns non-JSON; handled by `Extract AI Insights` with fallback
  - Long response times

#### Node: Extract AI Insights
- **Type / role:** Code — parses model output into fields.
- **Configuration (logic):**
  - Reads `$json.response`
  - Extracts first `{ ... }` block via regex
  - Attempts `JSON.parse`; if fails, uses fallback:
    - `services`: parsed services joined or defaults to business type
    - `summary`: parsed summary or a generic sentence
- **Output:** To `AI Processing Buffer`.
- **Edge cases:** If AI response field name differs (e.g., OpenAI uses `choices[0].message.content`), this parser must be adjusted.

#### Node: AI Processing Buffer
- **Type / role:** Wait — spacing before message generation batching.
- **Configuration:** `amount: 2` seconds.
- **Output:** To `Process Outreach Message`.

#### Node: Process Outreach Message
- **Type / role:** Split In Batches — controls message generation throughput.
- **Configuration:** Default options.
- **Outputs:**
  - To `Clean Outreach Message`
  - To `Message Rate Limit Buffer`
- **Edge cases:** Similar miswiring risk: `Clean Outreach Message` should occur after `Create Outreach Message`. In the provided connections, `Create Outreach Message` outputs back into `Process Outreach Message` (loop). Verify intended order.

#### Node: Message Rate Limit Buffer
- **Type / role:** Wait — buffers LLM message calls.
- **Configuration:** `amount: 2` seconds.
- **Output:** To `Create Outreach Message`.

#### Node: Create Outreach Message
- **Type / role:** HTTP Request — calls LLM to draft outreach text.
- **Configuration:**
  - POST `YOUR_OLLAMA_URL/api/generate`
  - Prompt uses expressions referencing `Clean Lead Data` fields for personalization.
  - Requires “Return ONLY the message text.”
- **Output:** Back to `Process Outreach Message` (loop). Intended next step is `Clean Outreach Message`.
- **Edge cases:** Same as above; if you switch to OpenAI Chat Completions, output shape differs.

#### Node: Clean Outreach Message
- **Type / role:** Code — sanitizes LLM output.
- **Configuration (logic):**
  - Reads `$json.response`
  - Removes “Here is …” preambles
  - Strips surrounding quotes
  - Flattens newlines
  - Outputs `message`
- **Output:** To `Save Leads to Google Sheets`.
- **Edge cases:** If model returns empty response, message becomes empty string; consider default message fallback.

---

### 2.7 Export to Google Sheets
**Overview:** Writes lead data + AI outputs + confidence fields into Google Sheets via append-or-update.  
**Nodes involved:** `Save Leads to Google Sheets`

**Sticky note (applies to this block):**
- **Content:** “Export section: Scores each lead from 0 to 10…, removes duplicates, and saves everything clean to Google Sheets ready for outreach.”

#### Node: Save Leads to Google Sheets
- **Type / role:** Google Sheets — append or update lead rows.
- **Configuration (interpreted):**
  - Operation: `appendOrUpdate`
  - Document ID: `YOUR_GOOGLE_SHEET_ID`
  - Sheet: configured via ID mode (also `YOUR_GOOGLE_SHEET_ID` in JSON placeholder)
  - Matching column: `Name`
  - Columns mapped from multiple nodes via expressions:
    - `Clean Lead Data` (name/type/email/phone/website/socials/confidences/etc.)
    - `Calculate Lead Score` (LeadScore)
    - `Extract AI Insights` (Services, Summary)
    - `Clean Outreach Message` (Draft message)
- **Credentials:** Google Sheets OAuth2 in n8n.
- **Edge cases / failures:**
  - Using `Name` as the only matching key may overwrite different businesses with same name (common).
  - Sheet column headers contain line breaks (e.g., `"Email\n Status"`, `"Instagram\n Confidence"`). The sheet must match exactly.
  - Permissions issues (403) if spreadsheet not shared with the OAuth account.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note1 | Sticky Note | Overall workflow description / requirements |  |  | ## AI Lead Generation & Social Discovery Automation / What you need / How to run (as written) |
| Start Lead Generation | Manual Trigger | Entry point (manual run) |  | Split Search Queries | ## AI Lead Generation & Social Discovery Automation / What you need / How to run (as written) |
| Split Search Queries | Split Out | Split `queries[]` into items | Start Lead Generation | Process Each Query |  |
| Process Each Query | Split In Batches | Iterate queries in controlled batches | Split Search Queries; Rate Limit Protection | Fetch Maps Results (Page 1); Rate Limit Protection |  |
| Rate Limit Protection | Wait | Throttle between query batches | Process Each Query | Process Each Query |  |
| Fetch Maps Results (Page 1) | HTTP Request | Serper Maps page 1 fetch | Process Each Query | Delay Between Requests | ## Maps section: Searches Google Maps using your queries and pulls business name, website, phone number, rating and category for each result. |
| Delay Between Requests | Wait | Delay before multi-page fetch | Fetch Maps Results (Page 1) | Fetch Maps Results (Pages 2–12) | ## Maps section: Searches Google Maps using your queries and pulls business name, website, phone number, rating and category for each result. |
| Fetch Maps Results (Pages 2–12) | HTTP Request | Serper Maps pages 2–12 bulk fetch | Delay Between Requests | Extract Businesses From Maps | ## Maps section: Searches Google Maps using your queries and pulls business name, website, phone number, rating and category for each result. |
| Extract Businesses From Maps | Code | Parse Serper Maps results into business items | Fetch Maps Results (Pages 2–12) + reference Fetch Maps Results (Page 1) | Process Businesses in Batches | ## Maps section: Searches Google Maps using your queries and pulls business name, website, phone number, rating and category for each result. |
| Process Businesses in Batches | Split In Batches | Batch businesses for scraping/processing | Extract Businesses From Maps; Normalize Social Data | Scrape Business Website; Check Email Exists | ## Scraping section: Visits each business website and looks for emails and social media links. Handles broken pages and redirects automatically. |
| Scrape Business Website | HTTP Request | Fetch website HTML | Process Businesses in Batches | Extract Socials & Email | ## Scraping section: Visits each business website and looks for emails and social media links. Handles broken pages and redirects automatically. |
| Extract Socials & Email | Code | Extract socials + email from HTML | Scrape Business Website + reference Extract Businesses From Maps | Combine Website + Serper Results; Build Social Search Query | ## Scraping section: Visits each business website and looks for emails and social media links. Handles broken pages and redirects automatically. |
| Build Social Search Query | Code | Build robust social search query (smartQuery) | Extract Socials & Email | Search Social Profiles (Serper) | ## Social section: If social profiles are not found on the website, it runs a separate search… |
| Search Social Profiles (Serper) | HTTP Request | Serper web search for social profiles | Build Social Search Query | Parse Social Results | ## Social section: If social profiles are not found on the website, it runs a separate search… |
| Parse Social Results | Code | Extract candidate social URLs from Serper response | Search Social Profiles (Serper) | Combine Website + Serper Results | ## Social section: If social profiles are not found on the website, it runs a separate search… |
| Combine Website + Serper Results | Merge | Combine scraped + Serper social results | Extract Socials & Email; Parse Social Results | Resolve & Score Social Profiles | ## Social section: If social profiles are not found on the website, it runs a separate search… |
| Resolve & Score Social Profiles | Code | Reconcile social URLs and confidence scoring | Combine Website + Serper Results | Normalize Social Data | ## Social section: If social profiles are not found on the website, it runs a separate search… |
| Normalize Social Data | Code | Normalize fields (use “-” placeholders) | Resolve & Score Social Profiles | Process Businesses in Batches |  |
| Check Email Exists | IF | Route: validate email or mark missing | Process Businesses in Batches (uses expression to Normalize Social Data) | Validate Email Address; Prepare Leads Without Email | ## Email validation section: Checks every email found and removes ones that are invalid or likely to bounce before saving to your sheet. |
| Validate Email Address | HTTP Request | Validate email via rapid-email-verifier API | Check Email Exists | Prepare Valid Leads | ## Email validation section: Checks every email found and removes ones that are invalid or likely to bounce before saving to your sheet. |
| Prepare Valid Leads | Code | Apply validation results + set emailStatus | Validate Email Address + reference Check Email Exists | Merge Lead Results | ## Email validation section: Checks every email found and removes ones that are invalid or likely to bounce before saving to your sheet. |
| Prepare Leads Without Email | Code | Standardize leads with no email | Check Email Exists (false branch) | Merge Lead Results | ## Email validation section: Checks every email found and removes ones that are invalid or likely to bounce before saving to your sheet. |
| Merge Lead Results | Merge | Unify “with email” and “no email” branches | Prepare Valid Leads; Prepare Leads Without Email | Clean Lead Data |  |
| Clean Lead Data | Code | Final cleanup + phone formatting | Merge Lead Results | Calculate Lead Score |  |
| Calculate Lead Score | Code | Compute score out of 10 | Clean Lead Data | Remove Duplicate Leads | ## Export section: Scores each lead from 0 to 10 based on their digital presence, removes duplicates, and saves everything clean to Google Sheets ready for outreach. |
| Remove Duplicate Leads | Remove Duplicates | Deduplicate by website + name | Calculate Lead Score | Process Leads for AI Enrichment | ## Export section: Scores each lead from 0 to 10 based on their digital presence, removes duplicates, and saves everything clean to Google Sheets ready for outreach. |
| Process Leads for AI Enrichment | Split In Batches | Batch leads before AI calls | Remove Duplicate Leads; Analyze Business (AI) | Extract AI Insights; AI Rate Limit Buffer | ## AI section: Sends each business to an AI model… |
| AI Rate Limit Buffer | Wait | Throttle AI analysis calls | Process Leads for AI Enrichment | Analyze Business (AI) | ## Ollama Setup (endpoints + alternative providers) |
| Analyze Business (AI) | HTTP Request | LLM call to generate services + summary JSON | AI Rate Limit Buffer | Process Leads for AI Enrichment | ## Ollama Setup (endpoints + alternative providers) |
| Extract AI Insights | Code | Parse LLM JSON or fallback | Process Leads for AI Enrichment | AI Processing Buffer | ## AI section: Sends each business to an AI model… |
| AI Processing Buffer | Wait | Buffer between AI tasks | Extract AI Insights | Process Outreach Message |  |
| Process Outreach Message | Split In Batches | Batch outreach message generation | AI Processing Buffer; Create Outreach Message | Clean Outreach Message; Message Rate Limit Buffer |  |
| Message Rate Limit Buffer | Wait | Throttle outreach message calls | Process Outreach Message | Create Outreach Message |  |
| Create Outreach Message | HTTP Request | LLM call to draft message | Message Rate Limit Buffer | Process Outreach Message | ## Ollama Setup (endpoints + alternative providers) |
| Clean Outreach Message | Code | Clean/sanitize message text | Process Outreach Message | Save Leads to Google Sheets |  |
| Save Leads to Google Sheets | Google Sheets | Append/update lead row | Clean Outreach Message |  | ## Export section: Scores each lead from 0 to 10 based on their digital presence, removes duplicates, and saves everything clean to Google Sheets ready for outreach. |
| Sticky Note | Sticky Note | Section description |  |  | ## Maps section: (as written) |
| Sticky Note2 | Sticky Note | Section description |  |  | ## Scraping section: (as written) |
| Sticky Note3 | Sticky Note | Section description |  |  | ## Social section: (as written) |
| Sticky Note4 | Sticky Note | Section description |  |  | ## Email validation section: (as written) |
| Sticky Note5 | Sticky Note | Section description |  |  | ## AI section: (as written) |
| Sticky Note6 | Sticky Note | Section description |  |  | ## Export section: (as written) |
| Sticky Note7 | Sticky Note | Provider setup notes |  |  | ## Ollama Setup + Not using Ollama? (endpoints + links as written) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**
   - Add **Manual Trigger** node named **Start Lead Generation**.
   - When running, provide input JSON like:
     - `{"queries":["dentists in Pune","gyms in Delhi"]}`

2. **Split queries array**
   - Add **Split Out** node named **Split Search Queries**.
   - Field to split out: `queries`
   - Destination field name: `query`
   - Connect: `Start Lead Generation → Split Search Queries`.

3. **Batch queries**
   - Add **Split In Batches** named **Process Each Query**.
   - Connect: `Split Search Queries → Process Each Query`.

4. **Add query throttle loop**
   - Add **Wait** node named **Rate Limit Protection** (3 seconds).
   - Connect: `Process Each Query → Rate Limit Protection → Process Each Query`.

5. **Serper Maps page 1**
   - Add **HTTP Request** named **Fetch Maps Results (Page 1)**.
   - Method: GET
   - URL: `https://google.serper.dev/maps?q={{ $json.query }}`
   - Authentication: **Header Auth** credential (e.g., header `X-API-KEY: <SERPER_KEY>`).
   - Connect: `Process Each Query → Fetch Maps Results (Page 1)`.

6. **Delay before multi-page**
   - Add **Wait** named **Delay Between Requests** (set a small value like 1–2 seconds).
   - Connect: `Fetch Maps Results (Page 1) → Delay Between Requests`.

7. **Serper Maps pages 2–12**
   - Add **HTTP Request** named **Fetch Maps Results (Pages 2–12)**.
   - Method: POST
   - URL: `https://google.serper.dev/maps`
   - Body: JSON (array of objects) with `q`, `page`, `ll` based on page-1 response:
     - `q: {{ $json.searchParameters.q }}`
     - `ll: {{ $json.ll }}`
   - Auth: same Serper header auth.
   - Connect: `Delay Between Requests → Fetch Maps Results (Pages 2–12)`.

8. **Extract businesses from Maps**
   - Add **Code** node **Extract Businesses From Maps**.
   - Implement logic to merge page-1 + pages-2–12 and output items with at least:
     - `name`, `phone`, `website`, `rating`, `type`
   - Connect: `Fetch Maps Results (Pages 2–12) → Extract Businesses From Maps`.

9. **Batch businesses**
   - Add **Split In Batches** node **Process Businesses in Batches**, batch size 5.
   - Connect: `Extract Businesses From Maps → Process Businesses in Batches`.

10. **Scrape website**
    - Add **HTTP Request** node **Scrape Business Website**:
      - URL: `{{ $json.website }}`
      - Add common browser headers
      - Allow unauthorized certs: enabled
      - On error: “Continue”
    - Connect: `Process Businesses in Batches → Scrape Business Website`.

11. **Extract socials + email**
    - Add **Code** node **Extract Socials & Email**:
      - Parse HTML/response text for social URLs and emails
      - Output combined data alongside business basics
    - Connect: `Scrape Business Website → Extract Socials & Email`.

12. **Build social search query (optional improvement)**
    - Add **Code** node **Build Social Search Query** that outputs `smartQuery`.
    - Connect: `Extract Socials & Email → Build Social Search Query`.

13. **Serper web search for socials**
    - Add **HTTP Request** node **Search Social Profiles (Serper)**:
      - POST `https://google.serper.dev/search`
      - Body includes `sites` list; ideally set `q` to `{{ $json.smartQuery }}` (recommended) rather than name only.
      - Auth: Serper header auth.
    - Connect: `Build Social Search Query → Search Social Profiles (Serper)`.

14. **Parse Serper social results**
    - Add **Code** node **Parse Social Results** to extract candidate social URLs into fields like `serperInstagram`, etc.
    - Connect: `Search Social Profiles (Serper) → Parse Social Results`.

15. **Merge website + Serper social results**
    - Add **Merge** node **Combine Website + Serper Results**:
      - Mode: Combine
      - Combine by position
    - Connect:
      - `Extract Socials & Email → Combine Website + Serper Results (Input 0)`
      - `Parse Social Results → Combine Website + Serper Results (Input 1)`

16. **Resolve and score socials**
    - Add **Code** node **Resolve & Score Social Profiles**:
      - Clean URLs (drop post/video/share links)
      - Merge website vs Serper per platform and output confidence
    - Connect: `Combine Website + Serper Results → Resolve & Score Social Profiles`.

17. **Normalize social data**
    - Add **Code** node **Normalize Social Data**:
      - Convert blanks to `"-"` (except email should remain empty string if missing)
    - Connect: `Resolve & Score Social Profiles → Normalize Social Data`.

18. **Email existence check**
    - Add **IF** node **Check Email Exists**:
      - Condition: email is not empty
    - Feed it with the normalized lead item (recommended: connect from `Normalize Social Data → Check Email Exists` to avoid cross-node item referencing issues).

19. **Validate email (true branch)**
    - Add **HTTP Request** node **Validate Email Address**:
      - GET with query param `email={{ $json.email }}`
      - URL: `https://rapid-email-verifier.fly.dev/api/validate`
    - Add **Code** node **Prepare Valid Leads** to map validator output into `emailStatus` and possibly clear invalid emails.

20. **No-email path (false branch)**
    - Add **Code** node **Prepare Leads Without Email**:
      - Set `emailStatus: NOT_FOUND`.

21. **Merge validation branches**
    - Add **Merge** node **Merge Lead Results** (append mode preferred).
    - Connect both branches into it.

22. **Clean lead data + format phone**
    - Add **Code** node **Clean Lead Data** for safe placeholders and phone formatting.

23. **Score and dedupe**
    - Add **Code** node **Calculate Lead Score**.
    - Add **Remove Duplicates** node **Remove Duplicate Leads**:
      - Compare selected fields: `website, name`.

24. **AI enrichment: services + summary**
    - Add **Split In Batches** **Process Leads for AI Enrichment** (batch size as desired).
    - Add **Wait** **AI Rate Limit Buffer** (2 seconds).
    - Add **HTTP Request** **Analyze Business (AI)**:
      - For Ollama: `http://localhost:11434/api/generate` (or your environment’s URL)
      - Send prompt that returns JSON only
      - Timeout ~60s
    - Add **Code** **Extract AI Insights** to parse JSON from LLM response (and implement fallback).
    - Ensure the connections are linear:
      - `Remove Duplicate Leads → Process Leads for AI Enrichment → AI Rate Limit Buffer → Analyze Business (AI) → Extract AI Insights`

25. **AI outreach message**
    - Add **Wait** **AI Processing Buffer** (2 seconds) after insights.
    - Add **Split In Batches** **Process Outreach Message**.
    - Add **Wait** **Message Rate Limit Buffer** (2 seconds).
    - Add **HTTP Request** **Create Outreach Message** to call the LLM with the outreach prompt.
    - Add **Code** **Clean Outreach Message** to sanitize the returned text.
    - Linear connections recommended:
      - `Extract AI Insights → AI Processing Buffer → Process Outreach Message → Message Rate Limit Buffer → Create Outreach Message → Clean Outreach Message`

26. **Google Sheets export**
    - Add **Google Sheets** node **Save Leads to Google Sheets**:
      - Credentials: Google Sheets OAuth2
      - Operation: Append or Update
      - Spreadsheet Document ID: your sheet
      - Sheet tab: select the correct sheet/tab
      - Matching columns: ideally use a unique key (Website or Phone) rather than Name only
      - Map columns to:
        - Clean lead fields (name, website, email, socials, confidences)
        - Lead score
        - AI services/summary
        - Draft message
    - Connect: `Clean Outreach Message → Save Leads to Google Sheets`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Lead Generation & Social Discovery Automation… What you need: Serper API key; Google Sheets OAuth; Email validation API; Ollama or any LLM endpoint… How to run…” | From workflow sticky note (overall instructions). |
| Ollama endpoint examples: Mac Docker `http://docker.for.mac.host.internal:11434/api/generate`; Windows/Linux Docker `http://host.docker.internal:11434/api/generate`; Local `http://localhost:11434/api/generate`; Remote server `http://YOUR_SERVER_IP:11434/api/generate`; n8n Cloud requires public URL/tunnel | From “Ollama Setup” sticky note. |
| Alternative AI providers/endpoints listed: OpenAI, Groq, Together AI, Mistral, Anthropic; reminder to update Authorization header | From “Ollama Setup” sticky note. |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.