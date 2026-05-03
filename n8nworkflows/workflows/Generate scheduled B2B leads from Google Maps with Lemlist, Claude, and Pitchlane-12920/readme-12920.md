Generate scheduled B2B leads from Google Maps with Lemlist, Claude, and Pitchlane

https://n8nworkflows.xyz/workflows/generate-scheduled-b2b-leads-from-google-maps-with-lemlist--claude--and-pitchlane-12920


# Generate scheduled B2B leads from Google Maps with Lemlist, Claude, and Pitchlane

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate scheduled B2B leads from Google Maps with Lemlist, Claude, and Pitchlane  
**Purpose:** Automatically generate B2B leads from Google Maps on a weekday schedule, extract owner/contact details from German “Impressum” pages using AI, enrich leads (email/LinkedIn), generate personalized German compliments, create personalized videos via Pitchlane, then push the final lead + video URL into an outbound tool (the workflow’s final node actually targets **Instantly**, despite the node name mentioning Lemlist).

### 1.1 Scheduled Execution & Search Configuration
Runs Monday–Friday at 9:00, sets query/location/max results, then calls Serper.dev Google Maps endpoint.

### 1.2 Company Discovery (Google Maps → Lead Records)
Splits returned places into individual items and maps relevant fields into a normalized lead schema.

### 1.3 Website Fetch + Impressum Cleaning
Filters out leads without a website, fetches `/<website>/impressum`, converts HTML to text, and extracts the likely Impressum portion.

### 1.4 Owner/Contact Extraction (AI)
Uses OpenAI (JSON schema output) to extract owner name/role and general email from Impressum text, then parses/merges AI output and filters to leads where an owner was found.

### 1.5 Email Lookup + Lemlist Enrichment
Uses Findymail (custom code) to find a direct email by first/last name and domain, falls back to Impressum email, then enriches the person via Lemlist (including LinkedIn enrichment).

### 1.6 Messaging + Marketing Analysis + Google Doc
Generates a short personalized compliment in German with Claude, merges it back into lead data, extracts compliment, generates a marketing analysis via Gemini, and creates a Google Doc.

### 1.7 Video Creation + Callback Handling + Outbound Upload
Sends recipient + variables to Pitchlane to generate a personalized video, then waits for Pitchlane webhook callback, extracts video URL and recipient data, and posts the lead to Instantly with video/thumbnail/compliment.

**Entry points (multiple):**
- **Schedule trigger** (main generation path)
- **Webhook for video completion** (asynchronous callback path from Pitchlane)

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Execution & Search Setup
**Overview:** Triggers the workflow on weekdays, defines search parameters, and prepares downstream nodes to query Google Maps results through Serper.dev.  
**Nodes involved:**  
- Schedule trigger (Monday-Friday 9 AM)  
- Configure search parameters  

#### Node: Schedule trigger (Monday-Friday 9 AM)
- **Type / role:** Schedule Trigger; starts workflow on a cron schedule.
- **Config:** Cron expression `0 9 * * 1-5` (Mon–Fri at 09:00).
- **Outputs:** To **Configure search parameters**.
- **Edge cases:** Instance timezone affects “9 AM”; if you need local time, ensure n8n timezone is set appropriately.

#### Node: Configure search parameters
- **Type / role:** Set; defines variables for query execution.
- **Config choices:** Sets:
  - `searchQuery` (e.g., “Marketing Agentur”)
  - `location` (e.g., “Berlin, Germany”)
  - `maxResults` (e.g., 10)
- **Outputs:** To **Search Google Maps**.
- **Edge cases:** Non-numeric `maxResults` causes Serper request body issues; overly broad queries can return irrelevant places.

---

### Block 2 — Company Discovery (Google Maps → Lead Records)
**Overview:** Calls Serper.dev to scrape Google Maps, splits results into individual leads, and normalizes lead fields.  
**Nodes involved:**  
- Search Google Maps  
- Process search results  
- Extract lead data  

#### Node: Search Google Maps
- **Type / role:** HTTP Request; calls Serper.dev Google Maps endpoint.
- **Config choices:**
  - POST `https://google.serper.dev/maps`
  - JSON body builds query from `searchQuery` + `location`, and uses `maxResults` as `num`
  - Auth via **HTTP Header Auth** credential (Serper typically requires `X-API-KEY` header).
- **Expressions:**
  - `q`: `{{ $json.searchQuery }} {{ $json.location }}`
  - `num`: `{{ $json.maxResults }}`
- **Outputs:** To **Process search results**.
- **Failure types:** Invalid API key/headers (401/403), request quota exceeded (429), or schema changes (missing `places`).
- **Version notes:** Node typeVersion 4.2.

#### Node: Process search results
- **Type / role:** Split Out; emits one item per place.
- **Config:** Splits field `places` from Serper response.
- **Outputs:** To **Extract lead data**.
- **Edge cases:** If Serper returns no `places`, node yields 0 items (workflow continues with no data unless handled).

#### Node: Extract lead data
- **Type / role:** Set; maps Google Maps place fields into lead fields.
- **Config mappings (expressions):**
  - `companyName = $json.title`
  - `address = $json.address`
  - `phone = $json.phone`
  - `website = $json.website`
  - `rating = $json.rating`
  - `reviewCount = $json.reviews`
  - `leadSource = "Google Maps"`
- **Outputs:** To **Check if website exists**.
- **Edge cases:** Some places have no website/phone; downstream nodes must handle null/empty fields.

---

### Block 3 — Website Fetch + Impressum Cleaning
**Overview:** Filters out leads without websites, fetches the Impressum page, strips HTML, and isolates likely Impressum content.  
**Nodes involved:**  
- Check if website exists  
- Fetch company website  
- Clean impressum data  

#### Node: Check if website exists
- **Type / role:** Code; filters items to only those with `website`.
- **Config:** JS filter: keeps items where `website` is not null/undefined/empty.
- **Outputs:** To **Fetch company website**.
- **Edge cases:** Websites with whitespace-only strings pass unless trimmed; consider normalizing.

#### Node: Fetch company website
- **Type / role:** HTTP Request; fetches `{{website}}/impressum`.
- **Config choices:**
  - URL: `={{ $json.website }}/impressum`
  - **Method is POST** (unusual for page fetch)
  - Response format: text
  - `continueOnFail: true` (workflow continues even if request fails)
- **Outputs:** To **Clean impressum data**.
- **Likely issue:** Most websites require **GET**, not POST. POST may return 405/403 or different content.
- **Edge cases:**  
  - Websites that already end with `/` → double slash (usually OK but not guaranteed)  
  - Non-German sites may not have `/impressum`  
  - Redirects/Cloudflare/consent walls may block scraping  
  - With `continueOnFail`, downstream may see empty HTML and produce low-quality extraction.

#### Node: Clean impressum data
- **Type / role:** Code; converts HTML to readable text and extracts probable Impressum section.
- **Key logic:**
  - Pulls HTML from `item.json.data || item.json.body || item.json.html || ''`
  - Removes scripts/styles/nav/footer/head, converts block tags to line breaks, decodes common entities (incl. German umlauts)
  - Attempts to isolate Impressum using regex patterns:
    - `/Impressum...(?=Datenschutz|Privacy|Cookie|$)/i`
    - `/Angaben gemäß.../i`
    - `/Verantwortlich.../i`
  - Truncates to 4000 chars
  - Outputs: `impressumText`, `impressumLength`, `cleanupSuccess`
  - Preserves `pairedItem` for item pairing integrity
- **Outputs:** To **Extract impressum data with AI**.
- **Edge cases:**  
  - Very JS-heavy sites: HTML response may not contain Impressum text  
  - Over-aggressive stripping could remove relevant content  
  - Regex may capture wrong section if page structure differs.

---

### Block 4 — Owner/Contact Extraction (AI) + Validation
**Overview:** Uses OpenAI structured output to extract owner/contact details from Impressum text, then parses/merges the AI output and keeps only leads with an owner name.  
**Nodes involved:**  
- Extract impressum data with AI  
- Parse owner information  
- Extract owner name  
- Verify owner found  

#### Node: Extract impressum data with AI
- **Type / role:** OpenAI (LangChain) node; extracts structured fields from Impressum.
- **Config choices:**
  - Model: `gpt-4.1-nano`
  - Enforces **JSON Schema** output (strict fields, `additionalProperties: false`)
  - Prompt is in German, instructing to return *only* the JSON object.
- **Key expressions:**
  - References `{{ $json.impressumText }}`
  - Also references node names that likely do not exist in this workflow:
    - `{{ $('📂 Split Places').item.json.website }}`
    - `{{ $('📂 Split Places').item.json.title }}`
- **Outputs:** To **Parse owner information**.
- **Failure types:** Auth/config errors, schema mismatch (model outputs invalid JSON), token limits if `impressumText` is large (mitigated by 4000 char cap).
- **Critical integration issue:** The node-name references (`📂 Split Places`) do not match actual nodes (here it is “Process search results”). This can cause expression evaluation failures or wrong data.

#### Node: Parse owner information
- **Type / role:** Code; attempts to extract JSON from multiple possible AI response shapes and merge back into item.
- **Config:** Looks for AI response in:
  - `message.content`, `text`, `content`, `candidates[0]...`
  - Extracts first `{...}` block and parses JSON
  - Adds defaults if parsing fails
  - Merges `...item.json` with `...ownerData`
  - Preserves `pairedItem`
- **Outputs:** To **Extract owner name**.
- **Edge cases:** If OpenAI node already returns structured JSON fields directly (not as a string), this parser may not find it, leaving defaults.

#### Node: Extract owner name
- **Type / role:** Code; another parser/normalizer that tries multiple AI formats and removes `output` field.
- **Config:** Tries several paths (`output[0].content[0].text`, `content[0].text`, `message.content`) and JSON-parses `{...}`.
- **Outputs:** To **Verify owner found**.
- **Risk:** Redundant with “Parse owner information”; may override/lose fields depending on how upstream outputs are shaped.

#### Node: Verify owner found
- **Type / role:** Code; filters to items with a valid `inhaber_name`.
- **Config:** Accepts if `inhaber_name` is a non-empty string and not `'null'`.
- **Outputs:** To **Find email addresses**.
- **Edge cases:** If none found, returns a single info item `{ info: 'Keine Leads mit Inhaber gefunden' }` which will break downstream assumptions (e.g., email lookup expects name fields).

---

### Block 5 — Email Lookup + Lemlist Enrichment
**Overview:** Finds a direct email using Findymail (with fallback to general email from Impressum) and enriches the lead via Lemlist (person enrichment + LinkedIn).  
**Nodes involved:**  
- Find email addresses  
- Enrich lead with Lemlist  

#### Node: Find email addresses
- **Type / role:** Code; calls Findymail API and falls back to `generelle_email`.
- **Config choices:**
  - Uses `FINDYMAIL_API_KEY = 'API_KEY_HERE'` (must be replaced; better stored in n8n credentials/env vars)
  - Extracts domain from `website` or from `generelle_email`
  - Calls `POST https://app.findymail.com/api/search/name`
  - Adds:
    - `email`, `emailConfidence`, `emailSource`, `emailError`, `domain`
- **Outputs:** To **Enrich lead with Lemlist**.
- **Failure types:** Network errors, invalid API key (401), rate limits, missing `fetch` support depending on n8n version/runtime (modern n8n supports `fetch` in Code node).
- **Edge cases:** If the previous “no owner found” info item reaches here, domain parsing and API call will fail; you may need an IF node to stop.

#### Node: Enrich lead with Lemlist
- **Type / role:** Lemlist node; enriches a person record and optionally LinkedIn.
- **Config choices:**
  - Resource: `enrich`
  - Operation: `enrichPerson`
  - Inputs:
    - `firstName = inhaber_vorname`
    - `lastName = inhaber_nachname`
    - `companyName = firma_name`
  - `linkedinEnrichment: true`
  - `alwaysOutputData: true` (continues even if enrichment fails)
- **Outputs:** To **Generate message with Claude**.
- **Failure types:** Invalid Lemlist credentials, missing required fields, enrichment quota limits.

---

### Block 6 — Compliment Generation + Merge + Marketing Analysis + Google Doc
**Overview:** Generates a short German compliment using Claude, merges it into lead data, generates a marketing analysis via Gemini, and creates a Google Doc.  
**Nodes involved:**  
- Generate message with Claude  
- Process data with JavaScript  
- Generate compliment message  
- Create marketing analysis  
- Create Google Doc  

#### Node: Generate message with Claude
- **Type / role:** Anthropic (LangChain) node; creates the personalized compliment text.
- **Config choices:**
  - Model: `claude-opus-4-20250514`
  - Prompt: detailed RISEN framework; outputs German compliment max 2 sentences / 30 words; forbids specific words and punctuation.
- **Critical expression issue:** The prompt injects:
  - `{{ $('🗺️ Serper Google Maps').item.json.places[0].website }}`
  This node name does not exist in this workflow (actual is “Search Google Maps”). Also it references `places[0]`, which ignores per-lead items and can mismatch.
- **Outputs:** To **Process data with JavaScript**.
- **Failure types:** Auth, rate limits, prompt too large, model availability, expression resolution errors due to wrong node name.

#### Node: Process data with JavaScript
- **Type / role:** Code; merges Claude output with original lead items.
- **Config logic:**
  - Loads lead items via `$('Inhaber gefunden?').all()`
  - Reads current input as AI responses
  - Extracts compliment from `content[0].text` or `message.content` etc.
  - Produces merged items: `{...leadItem.json, compliment}`
- **Critical issue:** References a node name **“Inhaber gefunden?”** which does not exist (closest is “Verify owner found”). This will fail at runtime.
- **Outputs:** To **Generate compliment message**.

#### Node: Generate compliment message
- **Type / role:** Set; sets `compliment = $json.compliment` while keeping all other fields.
- **Outputs:** To **Create marketing analysis**.
- **Edge cases:** If `compliment` is missing, downstream video and Instantly payload will be empty.

#### Node: Create marketing analysis
- **Type / role:** Google Gemini (LangChain) node; creates a structured marketing analysis.
- **Config choices:**
  - Model: `models/gemini-3-pro-preview`
  - Uses lead fields like `firma_name`, `inhaber_name`, etc.
- **Expression issue:** Again references `{{ $('🗺️ Serper Google Maps').item.json.places[0].website }}` (non-existent node name; also not per-item).
- **Outputs:** To **Create Google Doc**.
- **Failure types:** Credential issues, model availability, token limits.

#### Node: Create Google Doc
- **Type / role:** Google Docs node; creates a document to store analysis (content insertion is not configured here).
- **Config choices:**
  - Title: “Marketing Analyse”
  - Drive: `myDrive`
  - FolderId: `Marketing reports` (as a literal value; in Google it’s usually an ID, not a name)
- **Outputs:** To **Create video with Pitchlane**.
- **Edge cases:** Folder identifier mismatch (ID vs name), permission errors. Also: no step writes the Gemini analysis text into the doc; it only creates an empty doc unless default content behavior exists (it usually doesn’t).

---

### Block 7 — Video Creation (Pitchlane) + Callback + Outbound Upload
**Overview:** Requests a personalized Pitchlane video, then separately handles Pitchlane’s callback webhook to send the final lead with video link to Instantly.  
**Nodes involved:**  
- Create video with Pitchlane  
- Set video status to pending  
- Webhook for video completion  
- Extract video data  
- 📤 Lemlist: Add Lead1  
- Set status to completed  

#### Node: Create video with Pitchlane
- **Type / role:** HTTP Request; creates a Pitchlane video in a campaign.
- **Config choices:**
  - POST to `https://api.pitchlane.com/api/public/v1/campaigns/.../videos`
  - Body includes:
    - `template_id` placeholder
    - `recipient.first_name/last_name/email/company`
    - `variables.compliment/company_name/first_name`
    - `callback_url` placeholder to n8n webhook
  - Auth via HTTP Header Auth credential.
- **Expressions:** Uses `inhaber_vorname`, `inhaber_nachname`, and `email` (from Findymail step) and `companyName`.
- **Outputs:** To **Set video status to pending**.
- **Failure types:** Missing template ID, invalid campaign ID, auth errors, callback URL not publicly reachable.
- **Edge case:** If `email` is null, Pitchlane may reject or generate unusable callback.

#### Node: Set video status to pending
- **Type / role:** Set; stores Pitchlane video id and status.
- **Config mappings:**
  - `pitchlaneVideoId = $json.video_id || $json.id`
  - `videoStatus = "pending"`
- **Outputs:** No further connection (acts as an informational end for the “request” path).
- **Edge cases:** If Pitchlane response uses a different id field, this may be empty.

#### Node: Webhook for video completion
- **Type / role:** Webhook trigger; receives Pitchlane callback when video is ready.
- **Config:** Path `video-ready-callback` (webhookId shown, but path is what matters).
- **Outputs:** To **Extract video data**.
- **Failure types:** Not reachable externally, wrong callback URL registered at Pitchlane, signature validation not implemented (anyone could post unless you add verification).

#### Node: Extract video data
- **Type / role:** Set; maps callback payload to fields used for outbound upload.
- **Config mappings (from `$json.body...`):**
  - `videoUrl`, `videoId`, `recipientEmail`, `recipientFirstName`, `recipientLastName`, `companyName`, `compliment`, `thumbnailUrl`
- **Outputs:** To **📤 Lemlist: Add Lead1**.
- **Edge cases:** Callback body schema differences; missing fields cause broken upload payload.

#### Node: 📤 Lemlist: Add Lead1
- **Type / role:** HTTP Request; adds lead to **Instantly** campaign (despite the name).
- **Config choices:**
  - POST `https://api.instantly.ai/api/v1/lead/add`
  - Uses placeholders `YOUR_INSTANTLY_API_KEY` and `YOUR_CAMPAIGN_ID`
  - Sends `custom_variables`: `video_url`, `thumbnail_url`, `compliment`
  - Auth mode set to HTTP Header Auth credential, but body also includes `api_key` (duplicative/inconsistent).
- **Outputs:** To **Set status to completed**.
- **Failure types:** Invalid API key/campaign id, workspace restrictions, payload validation errors.
- **Integration note:** Rename node or change endpoint if you actually intend Lemlist upload.

#### Node: Set status to completed
- **Type / role:** Set; marks process completed and adds timestamp.
- **Config:**
  - `status = "completed"`
  - `processedAt = $now.toISO()`
- **Outputs:** End.
- **Edge cases:** None significant.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule trigger (Monday-Friday 9 AM) | Schedule Trigger | Time-based workflow start | — | Configure search parameters | ## Scheduled lead generation with video outreach *(see below)* / ## Stage 1: Company discovery |
| Configure search parameters | Set | Define searchQuery/location/maxResults | Schedule trigger (Monday-Friday 9 AM) | Search Google Maps | ## Scheduled lead generation with video outreach / ## Stage 1: Company discovery |
| Search Google Maps | HTTP Request | Query Serper.dev Google Maps scraping API | Configure search parameters | Process search results | ## Scheduled lead generation with video outreach / ## Stage 1: Company discovery |
| Process search results | Split Out | Split `places` array into per-place items | Search Google Maps | Extract lead data | ## Stage 1: Company discovery |
| Extract lead data | Set | Normalize place fields into lead fields | Process search results | Check if website exists | ## Stage 1: Company discovery |
| Check if website exists | Code | Filter leads missing website | Extract lead data | Fetch company website | ## Stage 2: Website and owner extraction |
| Fetch company website | HTTP Request | Retrieve `/impressum` page (HTML) | Check if website exists | Clean impressum data | ## Stage 2: Website and owner extraction |
| Clean impressum data | Code | Strip HTML → text, isolate Impressum section | Fetch company website | Extract impressum data with AI | ## Stage 2: Website and owner extraction |
| Extract impressum data with AI | OpenAI (LangChain) | Extract owner/contact fields as JSON schema | Clean impressum data | Parse owner information | ## Stage 2: Website and owner extraction |
| Parse owner information | Code | Parse/merge AI JSON from varied response shapes | Extract impressum data with AI | Extract owner name | ## Stage 2: Website and owner extraction |
| Extract owner name | Code | Normalize extracted owner fields (redundant parser) | Parse owner information | Verify owner found | ## Stage 2: Website and owner extraction |
| Verify owner found | Code | Keep only leads with `inhaber_name` | Extract owner name | Find email addresses | ## Stage 2: Website and owner extraction |
| Find email addresses | Code | Findymail lookup + fallback to general email | Verify owner found | Enrich lead with Lemlist | ## Stage 3: Lead enrichment and messaging |
| Enrich lead with Lemlist | Lemlist | Person enrichment + LinkedIn enrichment | Find email addresses | Generate message with Claude | ## Stage 3: Lead enrichment and messaging |
| Generate message with Claude | Anthropic (LangChain) | Generate short German compliment | Enrich lead with Lemlist | Process data with JavaScript | ## Stage 3: Lead enrichment and messaging |
| Process data with JavaScript | Code | Merge Claude compliment back into lead items | Generate message with Claude | Generate compliment message | ## Stage 3: Lead enrichment and messaging |
| Generate compliment message | Set | Ensure `compliment` field exists | Process data with JavaScript | Create marketing analysis | ## Stage 3: Lead enrichment and messaging |
| Create marketing analysis | Google Gemini (LangChain) | Generate structured marketing analysis text | Generate compliment message | Create Google Doc | ## Stage 3: Lead enrichment and messaging |
| Create Google Doc | Google Docs | Create a Google Doc record | Create marketing analysis | Create video with Pitchlane | ## Stage 3: Lead enrichment and messaging |
| Create video with Pitchlane | HTTP Request | Create personalized video via Pitchlane | Create Google Doc | Set video status to pending | ## Stage 4: Video creation and upload |
| Set video status to pending | Set | Track Pitchlane video id/status | Create video with Pitchlane | — | ## Stage 4: Video creation and upload |
| Webhook for video completion | Webhook | Receive Pitchlane callback when video ready | — | Extract video data | ## Stage 4: Video creation and upload |
| Extract video data | Set | Map callback payload to lead fields | Webhook for video completion | 📤 Lemlist: Add Lead1 | ## Stage 4: Video creation and upload |
| 📤 Lemlist: Add Lead1 | HTTP Request | Add lead to Instantly with video variables | Extract video data | Set status to completed | ## Stage 4: Video creation and upload |
| Set status to completed | Set | Final status + timestamp | 📤 Lemlist: Add Lead1 | — | ## Stage 4: Video creation and upload |

**Sticky note content (referenced above):**
- “## Scheduled lead generation with video outreach … Setup steps …”
- “## Stage 1: Company discovery …”
- “## Stage 2: Website and owner extraction …”
- “## Stage 3: Lead enrichment and messaging …”
- “## Stage 4: Video creation and upload …”

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add **Schedule Trigger** node  
   2) Set cron to `0 9 * * 1-5` (Mon–Fri 09:00)

2. **Add Search Parameter Setup**
   1) Add **Set** node “Configure search parameters”  
   2) Add fields:
      - `searchQuery` (String) e.g. `Marketing Agentur`
      - `location` (String) e.g. `Berlin, Germany`
      - `maxResults` (Number) e.g. `10`
   3) Connect: **Schedule Trigger → Configure search parameters**

3. **Serper Google Maps Request**
   1) Add **HTTP Request** node “Search Google Maps”  
   2) Method: **POST**  
   3) URL: `https://google.serper.dev/maps`  
   4) Body: JSON with:
      - `q`: `{{$json.searchQuery}} {{$json.location}}`
      - `num`: `{{$json.maxResults}}`
   5) Authentication: **Header Auth** credential (set Serper API key header as required by Serper)
   6) Connect: **Configure search parameters → Search Google Maps**

4. **Split Places**
   1) Add **Split Out** node “Process search results”  
   2) Field to split out: `places`  
   3) Connect: **Search Google Maps → Process search results**

5. **Normalize Lead Fields**
   1) Add **Set** node “Extract lead data”  
   2) Map fields:
      - `companyName = {{$json.title}}`
      - `address = {{$json.address}}`
      - `phone = {{$json.phone}}`
      - `website = {{$json.website}}`
      - `rating = {{$json.rating}}`
      - `reviewCount = {{$json.reviews}}`
      - `leadSource = Google Maps`
   3) Connect: **Process search results → Extract lead data**

6. **Filter Leads Without Website**
   1) Add **Code** node “Check if website exists”  
   2) JS: filter items where `item.json.website` is non-empty  
   3) Connect: **Extract lead data → Check if website exists**

7. **Fetch Impressum Page**
   1) Add **HTTP Request** node “Fetch company website”  
   2) URL: `{{$json.website}}/impressum`  
   3) Response: **Text**
   4) Turn on **Continue On Fail**
   5) **Recommended fix:** use **GET** (instead of POST) for web page retrieval
   6) Connect: **Check if website exists → Fetch company website**

8. **Clean & Extract Impressum Text**
   1) Add **Code** node “Clean impressum data”  
   2) Paste logic to strip HTML and set:
      - `impressumText`, `impressumLength`, `cleanupSuccess`
   3) Connect: **Fetch company website → Clean impressum data**

9. **AI Extraction (OpenAI)**
   1) Add **OpenAI (LangChain)** node “Extract impressum data with AI”  
   2) Select model `gpt-4.1-nano`  
   3) Enable **structured JSON schema output** with required fields (owner name, email, address, etc.)  
   4) Prompt should use `{{$json.impressumText}}`  
   5) **Important:** remove or correct any references to non-existent nodes like `$('📂 Split Places')` and use current item fields (`$json.website`, `$json.companyName`, etc.)
   6) Configure OpenAI credentials  
   7) Connect: **Clean impressum data → Extract impressum data with AI**

10. **Parse/Normalize AI Output**
   1) Add **Code** node “Parse owner information” (parse JSON from model output; merge into item)  
   2) Connect: **Extract impressum data with AI → Parse owner information**

11. **Owner Extraction Normalization (optional/redundant)**
   1) Add **Code** node “Extract owner name”  
   2) Connect: **Parse owner information → Extract owner name**

12. **Filter Only Leads With Owner**
   1) Add **Code** node “Verify owner found” filtering by `inhaber_name`  
   2) Connect: **Extract owner name → Verify owner found**
   3) **Recommended:** if output is the “info” item, stop the workflow (use an IF node).

13. **Findymail Email Lookup**
   1) Add **Code** node “Find email addresses”  
   2) Store API key securely (recommended: env var or n8n credential), not inline  
   3) Call Findymail endpoint and implement fallback to `generelle_email`  
   4) Connect: **Verify owner found → Find email addresses**

14. **Lemlist Enrichment**
   1) Add **Lemlist** node “Enrich lead with Lemlist”  
   2) Operation: Enrich person; enable LinkedIn enrichment  
   3) Map first/last/company names from extracted owner fields  
   4) Configure Lemlist credentials  
   5) Connect: **Find email addresses → Enrich lead with Lemlist**

15. **Claude Compliment Generation**
   1) Add **Anthropic (LangChain)** node “Generate message with Claude”  
   2) Select model `claude-opus-4-20250514`  
   3) Paste the compliment prompt  
   4) **Critical fix:** replace `$('🗺️ Serper Google Maps').item.json.places[0].website` with a per-item value like `{{$json.website}}` (or your cleaned site info)  
   5) Configure Anthropic credentials  
   6) Connect: **Enrich lead with Lemlist → Generate message with Claude**

16. **Merge Compliment Back Into Lead**
   1) Add **Code** node “Process data with JavaScript”  
   2) **Critical fix:** replace `$('Inhaber gefunden?').all()` with the actual node name, e.g. `$('Verify owner found').all()` or refactor to rely purely on input pairing  
   3) Connect: **Generate message with Claude → Process data with JavaScript**

17. **Set Compliment Field**
   1) Add **Set** node “Generate compliment message” with `compliment = {{$json.compliment}}` and keep other fields  
   2) Connect: **Process data with JavaScript → Generate compliment message**

18. **Gemini Marketing Analysis**
   1) Add **Google Gemini (LangChain)** node “Create marketing analysis”  
   2) Choose model `models/gemini-3-pro-preview`  
   3) **Fix node references** in prompt to use per-item fields (`{{$json.website}}`, etc.)  
   4) Configure Google Gemini credentials  
   5) Connect: **Generate compliment message → Create marketing analysis**

19. **Create Google Doc**
   1) Add **Google Docs** node “Create Google Doc”  
   2) Set drive `myDrive` and folder **by folder ID** (recommended)  
   3) Configure Google Docs OAuth2 credentials  
   4) Connect: **Create marketing analysis → Create Google Doc**
   5) **Optional but recommended:** add a “Google Docs: Append/Insert text” step to actually write the analysis into the document.

20. **Create Pitchlane Video**
   1) Add **HTTP Request** node “Create video with Pitchlane”  
   2) POST Pitchlane endpoint with:
      - `template_id` (real ID)
      - recipient fields (first/last/email/company)
      - variables (compliment, etc.)
      - callback_url pointing to your n8n webhook public URL
   3) Configure Pitchlane header auth credential  
   4) Connect: **Create Google Doc → Create video with Pitchlane**

21. **Track “Pending” Status**
   1) Add **Set** node “Set video status to pending” capturing returned `video_id` and `pending` status  
   2) Connect: **Create video with Pitchlane → Set video status to pending**

22. **Webhook Callback for Video Ready**
   1) Add **Webhook** node “Webhook for video completion” with path `video-ready-callback`  
   2) Activate workflow and copy the **production webhook URL**  
   3) Put this URL into Pitchlane `callback_url`

23. **Extract Callback Data**
   1) Add **Set** node “Extract video data” mapping from `$json.body.*` to:
      - `videoUrl`, `thumbnailUrl`, `recipientEmail`, `recipientFirstName`, `recipientLastName`, `companyName`, `compliment`
   2) Connect: **Webhook for video completion → Extract video data**

24. **Push Lead to Outbound Tool (Instantly in this JSON)**
   1) Add **HTTP Request** node (rename to reflect Instantly)  
   2) POST `https://api.instantly.ai/api/v1/lead/add` with `api_key`, `campaign_id`, and lead payload including `custom_variables.video_url`  
   3) Store API key securely (credential/env var)  
   4) Connect: **Extract video data → outbound upload node**

25. **Finalize**
   1) Add **Set** node “Set status to completed” with:
      - `status = completed`
      - `processedAt = {{$now.toISO()}}`
   2) Connect: **Outbound upload → Set status to completed**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow sticky note: “Scheduled lead generation with video outreach” (describes stages, setup steps) | In-workflow documentation (yellow sticky note) |
| Stages 1–4 sticky notes summarize each phase (Company discovery → Owner extraction → Enrichment/messaging → Video/upload) | In-workflow documentation (stage sticky notes) |
| Multiple expression/node-name mismatches exist (e.g., `🗺️ Serper Google Maps`, `📂 Split Places`, `Inhaber gefunden?`) and must be corrected to actual node names or replaced with `$json` fields | Prevents runtime expression failures |
| “Fetch company website” uses POST for HTML retrieval; usually should be GET | Reduces 405/403 errors and improves extraction success |
| Node “📤 Lemlist: Add Lead1” actually posts to Instantly API | Rename or change endpoint to match intended tool |
| Webhook callback has no verification/signature check | Consider adding validation to prevent spoofed callbacks |

