Enrich and score Japanese B2B leads with gBizINFO, web scraping, and Gemini AI

https://n8nworkflows.xyz/workflows/enrich-and-score-japanese-b2b-leads-with-gbizinfo--web-scraping--and-gemini-ai-13522


# Enrich and score Japanese B2B leads with gBizINFO, web scraping, and Gemini AI

## 1. Workflow Overview

**Purpose:** Collect Japanese B2B lead details via an n8n Form, enrich the lead with Japanese government/company registries (Corporate Number API + gBizINFO), add reputation signals (Google Maps), scrape the company website for additional context, then use **Google Gemini** to **score the lead**. Finally, it saves results to **Google Sheets** and sends a **Slack alert** for “hot” leads.

**Primary use cases:**
- Sales ops / growth teams qualifying Japanese B2B prospects
- Automating enrichment + lightweight due diligence + prioritization
- Creating a structured lead database with consistent scoring

### Logical Blocks
1.1 **Lead intake** (Form submission entry point)  
1.2 **Government identifier + base corporate data** (Corporate Number API + parsing)  
1.3 **gBizINFO enrichment** (company profile enrichment + merge)  
1.4 **Reputation enrichment** (Google Maps + merge)  
1.5 **Website fetch + extraction** (scrape website and distill signals)  
1.6 **AI scoring (Gemini)** (LLM chain produces lead score/summary)  
1.7 **Persist + route hot leads** (Sheets storage + filter + Slack alert)

---

## 2. Block-by-Block Analysis

### 1.1 Lead intake
**Overview:** Accepts a lead submission from an n8n Form trigger and passes the submitted fields into the enrichment pipeline.  
**Nodes involved:** `Lead Input Form`

#### Node: Lead Input Form
- **Type / role:** `Form Trigger` (n8n-nodes-base.formTrigger) — workflow entry point.
- **Configuration (interpreted):** Uses n8n’s built-in form endpoint; exact form fields are not specified in the provided JSON (empty parameters), so they must be defined in the UI (e.g., company name, address, website, contact, notes).
- **Key variables/expressions:** Not visible in JSON; output is typically `{{$json}}` containing form fields.
- **Connections:**  
  - **Out:** `Corporate Number API`
- **Edge cases / failures:**
  - Missing required fields (if not enforced in form settings).
  - Inconsistent Japanese company naming (kana/kanji variants) causing downstream lookup failures.
- **Version notes:** TypeVersion `2.1` (Form Trigger).

---

### 1.2 Government identifier + base corporate data
**Overview:** Looks up the company via a corporate number service, then transforms/normalizes the response into a consistent internal structure.  
**Nodes involved:** `Corporate Number API`, `Parse Corporate Data`

#### Node: Corporate Number API
- **Type / role:** `HTTP Request` — calls a corporate registry lookup service (likely Japan Corporate Number Publication Site API or a proxy).
- **Configuration:** Not provided (empty parameters). In practice this must define:
  - Method (GET/POST), URL, query parameters (e.g., company name/address), headers, auth (API key if required).
- **Connections:**  
  - **In:** `Lead Input Form`  
  - **Out:** `Parse Corporate Data`
- **Edge cases / failures:**
  - 401/403 (missing/invalid API key), 429 rate limiting, timeouts.
  - No match / multiple matches; response format variations.

#### Node: Parse Corporate Data
- **Type / role:** `Code` — extracts corporate number and normalized fields from the API response.
- **Configuration:** Not provided (empty). Typically:
  - Select best match (exact vs fuzzy)
  - Map fields like corporate number, official name, address, status, etc.
- **Connections:**  
  - **In:** `Corporate Number API`  
  - **Out:** `gBizINFO Enrichment`
- **Edge cases / failures:**
  - Code errors when expected keys are absent.
  - Character encoding issues (JP text) if assumptions are made.
- **Version notes:** Code node TypeVersion `2`.

---

### 1.3 gBizINFO enrichment
**Overview:** Uses the corporate number (or normalized company identity) to pull richer attributes from **gBizINFO**, then merges them into the lead record.  
**Nodes involved:** `gBizINFO Enrichment`, `Merge Government Data`

#### Node: gBizINFO Enrichment
- **Type / role:** `HTTP Request` — calls gBizINFO API for corporate enrichment.
- **Configuration:** Not provided. Typically includes:
  - Endpoint for corporate profile
  - Query: corporate number (`houjin-bangou`) or similar identifier
  - Auth: token/API key if required by the chosen access method
- **Connections:**  
  - **In:** `Parse Corporate Data`  
  - **Out:** `Merge Government Data`
- **Edge cases / failures:**
  - API auth errors, rate limits, temporary outages.
  - Partial/empty enrichment for small or newly registered entities.

#### Node: Merge Government Data
- **Type / role:** `Code` — merges Corporate Number API parsed data + gBizINFO response into one consolidated JSON object.
- **Configuration:** Not provided; should:
  - Prefer official fields from gBizINFO when present
  - Preserve original form input for traceability
- **Connections:**  
  - **In:** `gBizINFO Enrichment`  
  - **Out:** `Google Maps Reputation`
- **Edge cases / failures:**
  - Conflicting addresses/names; require deterministic precedence rules.
  - Null checks to prevent runtime errors.

---

### 1.4 Reputation enrichment (Google Maps)
**Overview:** Retrieves reputation signals (e.g., rating, review count) from Google Maps/Places, then merges them into the lead record.  
**Nodes involved:** `Google Maps Reputation`, `Merge Reputation Data`

#### Node: Google Maps Reputation
- **Type / role:** `HTTP Request` — likely calls Google Places API (Text Search / Find Place / Place Details).
- **Configuration:** Not provided. Typically:
  - Search query built from company name + address
  - API key in query or header
- **Connections:**  
  - **In:** `Merge Government Data`  
  - **Out:** `Merge Reputation Data`
- **Edge cases / failures:**
  - Wrong place match (common with chain names).
  - No results for B2B offices not listed on Maps.
  - Quota limits and billing restrictions.

#### Node: Merge Reputation Data
- **Type / role:** `Code` — merges Google Maps results into consolidated lead object.
- **Configuration:** Not provided; should:
  - Extract rating, total reviews, place URL, category, etc.
  - Handle “no match” gracefully (set nulls/defaults)
- **Connections:**  
  - **In:** `Google Maps Reputation`  
  - **Out:** `Fetch Company Website`
- **Edge cases / failures:**
  - Unexpected Google API response shape.
  - Multiple candidates; choose best by address proximity or name similarity.

---

### 1.5 Website fetch + extraction
**Overview:** Downloads the company website (if any) and extracts relevant signals (products, industry, positioning, contact presence) to help AI scoring.  
**Nodes involved:** `Fetch Company Website`, `Extract Website Intel`

#### Node: Fetch Company Website
- **Type / role:** `HTTP Request` — fetches the website HTML.
- **Configuration (interpreted):**
  - `onError: continueRegularOutput` means failures won’t stop the workflow; output continues (likely with error info / empty body).
  - Actual URL construction is not shown; it likely uses an enriched “website” field.
- **Connections:**  
  - **In:** `Merge Reputation Data`  
  - **Out:** `Extract Website Intel`
- **Edge cases / failures:**
  - DNS/timeout/SSL failures, redirects, 403/503, bot protection.
  - Non-HTML content, very large pages.
- **Version notes:** HTTP Request TypeVersion `4.2`.

#### Node: Extract Website Intel
- **Type / role:** `Code` — parses HTML/text and distills structured website intelligence.
- **Configuration:** Not provided; often includes:
  - Strip HTML, extract title/meta description/H1
  - Detect keywords (industry/services), contact info, languages, ICP fit signals
- **Connections:**  
  - **In:** `Fetch Company Website`  
  - **Out:** `AI Lead Scorer`
- **Edge cases / failures:**
  - Missing/empty HTML because fetch failed (must check before parsing).
  - Encoding issues (Shift-JIS/EUC-JP) if not handled by HTTP node settings.

---

### 1.6 AI scoring (Gemini)
**Overview:** Uses an LLM chain to score and summarize the lead using all consolidated enrichment signals. Gemini is attached as the chat model provider.  
**Nodes involved:** `AI Lead Scorer`, `Google Gemini Chat Model`

#### Node: Google Gemini Chat Model
- **Type / role:** `LangChain Chat Model (Google Gemini)` — provides the LLM for the chain.
- **Configuration:** Not provided; typically:
  - Model selection (e.g., Gemini 1.5)
  - Credentials: Google AI Studio / Vertex AI (depending on node mode)
- **Connections:**  
  - **Out (ai_languageModel):** `AI Lead Scorer`
- **Edge cases / failures:**
  - Auth/permission errors, safety blocks, quota limits.
  - Token/context overflows if you pass full HTML instead of extracted text.

#### Node: AI Lead Scorer
- **Type / role:** `LangChain Chain LLM` — prompts the model and returns structured scoring.
- **Configuration:** Not provided; typically:
  - Prompt template referencing enriched fields
  - Output parser (JSON score, rationale, tags, recommended next step)
- **Connections:**  
  - **In:** `Extract Website Intel`  
  - **In (ai_languageModel):** `Google Gemini Chat Model`  
  - **Out:** `Prepare Final Output`
- **Edge cases / failures:**
  - Non-JSON/invalid structure responses if you expect strict output.
  - Hallucinated facts if prompt doesn’t constrain to provided data.

---

### 1.7 Persist + route hot leads
**Overview:** Builds the final normalized output record, appends it to Google Sheets, and sends a Slack alert if the lead passes a “hot” threshold.  
**Nodes involved:** `Prepare Final Output`, `Save to Google Sheets`, `Hot Lead Filter`, `Slack Hot Lead Alert`

#### Node: Prepare Final Output
- **Type / role:** `Code` — consolidates all fields (form + enrichment + reputation + website intel + AI score) into final schema.
- **Configuration:** Not provided; typically:
  - Create flat columns for Sheets (score, company name, corp number, address, website, rating, review_count, summary, etc.)
  - Add timestamps, source metadata
- **Connections:**  
  - **In:** `AI Lead Scorer`  
  - **Out:** `Save to Google Sheets`, `Hot Lead Filter`
- **Edge cases / failures:**
  - Missing AI fields; ensure defaults.
  - Data type mismatches (numbers vs strings) for Sheets columns.

#### Node: Save to Google Sheets
- **Type / role:** `Google Sheets` — persists the final record (append/update).
- **Configuration:** Not provided; typically requires:
  - OAuth2 credentials for Google
  - Spreadsheet ID, sheet/tab name, operation (Append)
  - Column mapping matching `Prepare Final Output`
- **Connections:**  
  - **In:** `Prepare Final Output`  
  - **Out:** (none shown)
- **Edge cases / failures:**
  - Auth expired, missing permissions, wrong spreadsheet ID.
  - Column mismatch causing partial writes or errors.
- **Version notes:** TypeVersion `4.5`.

#### Node: Hot Lead Filter
- **Type / role:** `Filter` — checks if the lead qualifies as “hot” (likely based on AI score or rules).
- **Configuration:** Not provided; typically:
  - Condition like `score >= X` and/or `fit = true`
- **Connections:**  
  - **In:** `Prepare Final Output`  
  - **Out:** `Slack Hot Lead Alert`
- **Edge cases / failures:**
  - Filter condition references fields that are absent (expression errors).
- **Version notes:** TypeVersion `2`.

#### Node: Slack Hot Lead Alert
- **Type / role:** `Slack` — sends message to a channel/user when hot.
- **Configuration:** Not provided; requires:
  - Slack credentials (OAuth token/bot)
  - Channel + message template including key fields
- **Connections:**  
  - **In:** `Hot Lead Filter`
- **Edge cases / failures:**
  - Missing Slack scopes, channel not found, rate limiting.
- **Version notes:** TypeVersion `2.2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Visual annotation placeholder | — | — |  |
| Sticky Note1 | Sticky Note | Visual annotation placeholder | — | — |  |
| Sticky Note2 | Sticky Note | Visual annotation placeholder | — | — |  |
| Sticky Note3 | Sticky Note | Visual annotation placeholder | — | — |  |
| Sticky Note4 | Sticky Note | Visual annotation placeholder | — | — |  |
| Sticky Note5 | Sticky Note | Visual annotation placeholder | — | — |  |
| Sticky Note6 | Sticky Note | Visual annotation placeholder | — | — |  |
| Sticky Note7 | Sticky Note | Visual annotation placeholder | — | — |  |
| Lead Input Form | Form Trigger | Lead submission entry point | — | Corporate Number API |  |
| Corporate Number API | HTTP Request | Lookup corporate identifier/base data | Lead Input Form | Parse Corporate Data |  |
| Parse Corporate Data | Code | Normalize/parse registry response | Corporate Number API | gBizINFO Enrichment |  |
| gBizINFO Enrichment | HTTP Request | Enrich from gBizINFO | Parse Corporate Data | Merge Government Data |  |
| Merge Government Data | Code | Merge corporate registry + gBizINFO | gBizINFO Enrichment | Google Maps Reputation |  |
| Google Maps Reputation | HTTP Request | Fetch Maps/Places reputation signals | Merge Government Data | Merge Reputation Data |  |
| Merge Reputation Data | Code | Merge reputation signals | Google Maps Reputation | Fetch Company Website |  |
| Fetch Company Website | HTTP Request | Download website HTML (non-fatal errors) | Merge Reputation Data | Extract Website Intel |  |
| Extract Website Intel | Code | Extract structured info from website | Fetch Company Website | AI Lead Scorer |  |
| Google Gemini Chat Model | Gemini Chat Model (LangChain) | LLM provider for scoring | — | AI Lead Scorer (ai_languageModel) |  |
| AI Lead Scorer | LangChain Chain LLM | Prompt + score lead using Gemini | Extract Website Intel + Gemini model | Prepare Final Output |  |
| Prepare Final Output | Code | Build final record schema | AI Lead Scorer | Save to Google Sheets; Hot Lead Filter |  |
| Save to Google Sheets | Google Sheets | Store lead record | Prepare Final Output | — |  |
| Hot Lead Filter | Filter | Decide if lead is “hot” | Prepare Final Output | Slack Hot Lead Alert |  |
| Slack Hot Lead Alert | Slack | Notify sales team | Hot Lead Filter | — |  |

> Note: All sticky notes have empty content in the provided workflow JSON, so the “Sticky Note” column is blank.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add “Form Trigger” node** named **Lead Input Form**  
   - Define form fields (recommended): `company_name`, `address`, `website` (optional), `contact_name` (optional), `notes` (optional).  
   - Ensure required fields for downstream lookups (at least company name; ideally address).

3) **Add “HTTP Request” node** named **Corporate Number API** and connect:  
   `Lead Input Form → Corporate Number API`  
   - Configure URL to your corporate number lookup endpoint.  
   - Map query params using expressions from form fields (e.g., company name/address).  
   - Add auth (API key) if required.  
   - Set response format to JSON.

4) **Add “Code” node** named **Parse Corporate Data** and connect:  
   `Corporate Number API → Parse Corporate Data`  
   - Implement logic to:
     - Select best match from results
     - Output normalized fields: `corporate_number`, `official_name`, `address`, etc.
     - Carry forward original form input for traceability

5) **Add “HTTP Request” node** named **gBizINFO Enrichment** and connect:  
   `Parse Corporate Data → gBizINFO Enrichment`  
   - Configure gBizINFO endpoint and pass `corporate_number` (expression).  
   - Add necessary authentication headers/tokens.  
   - JSON response.

6) **Add “Code” node** named **Merge Government Data** and connect:  
   `gBizINFO Enrichment → Merge Government Data`  
   - Merge parsed corporate data + gBizINFO response into one object.

7) **Add “HTTP Request” node** named **Google Maps Reputation** and connect:  
   `Merge Government Data → Google Maps Reputation`  
   - Configure Google Places API call (Text Search/Find Place + optionally Place Details).  
   - Build query from `official_name` + `address`.  
   - Provide Google API key and enable billing as required.

8) **Add “Code” node** named **Merge Reputation Data** and connect:  
   `Google Maps Reputation → Merge Reputation Data`  
   - Extract and merge: `rating`, `user_ratings_total`, `place_id`, `maps_url` (or equivalents).  
   - Handle empty results safely.

9) **Add “HTTP Request” node** named **Fetch Company Website** and connect:  
   `Merge Reputation Data → Fetch Company Website`  
   - Use the best available website URL (from form or gBizINFO).  
   - Set **Error Handling** to **Continue (continueRegularOutput)** so the workflow doesn’t fail if the site can’t be fetched.  
   - Return raw HTML (string/body).

10) **Add “Code” node** named **Extract Website Intel** and connect:  
   `Fetch Company Website → Extract Website Intel`  
   - If body missing, output empty intel fields.  
   - Otherwise strip HTML and extract key snippets (title/description, services, keywords, contact presence).

11) **Add “Google Gemini Chat Model” node** named **Google Gemini Chat Model**  
   - Configure credentials (Google AI / Vertex AI depending on node mode).  
   - Select a Gemini chat model suitable for your plan/region.

12) **Add “AI Lead Scorer (Chain LLM)” node** named **AI Lead Scorer**  
   - Connect main data: `Extract Website Intel → AI Lead Scorer`  
   - Connect model: `Google Gemini Chat Model → AI Lead Scorer` via the **ai_languageModel** input.  
   - Configure prompt to produce structured output (recommended JSON) containing at least:
     - `score` (0–100 or 0–10)
     - `rationale`
     - `tags`
     - `recommended_next_step`

13) **Add “Code” node** named **Prepare Final Output** and connect:  
   `AI Lead Scorer → Prepare Final Output`  
   - Flatten data for storage + messaging:
     - lead identity, enrichment fields, reputation fields, website intel summary, AI score fields, timestamp.

14) **Add “Google Sheets” node** named **Save to Google Sheets** and connect:  
   `Prepare Final Output → Save to Google Sheets`  
   - Set Google OAuth2 credentials.  
   - Choose spreadsheet + sheet name.  
   - Operation: typically **Append**.  
   - Map columns to fields produced by `Prepare Final Output`.

15) **Add “Filter” node** named **Hot Lead Filter** and connect:  
   `Prepare Final Output → Hot Lead Filter`  
   - Configure condition(s), e.g.:
     - `score >= 80`, or `score >= X AND review_count >= Y`, etc.

16) **Add “Slack” node** named **Slack Hot Lead Alert** and connect:  
   `Hot Lead Filter → Slack Hot Lead Alert`  
   - Configure Slack credentials (bot token/OAuth).  
   - Pick channel.  
   - Message template including: company name, score, rationale, links (website/Maps), sheet row link if available.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Disclaimer (provided): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Context: compliance statement included in the request |
| Sticky notes exist but have empty content | Context: 8 Sticky Note nodes are present with blank `content` |

If you paste the missing node parameters (URLs, prompts, code contents), I can document the exact field mappings, expressions, and expected schemas end-to-end.