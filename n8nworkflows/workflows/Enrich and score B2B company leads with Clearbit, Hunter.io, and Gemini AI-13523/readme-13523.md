Enrich and score B2B company leads with Clearbit, Hunter.io, and Gemini AI

https://n8nworkflows.xyz/workflows/enrich-and-score-b2b-company-leads-with-clearbit--hunter-io--and-gemini-ai-13523


# Enrich and score B2B company leads with Clearbit, Hunter.io, and Gemini AI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Enrich and score B2B company leads with Clearbit, Hunter.io, and Gemini AI

**Purpose:**  
This workflow collects B2B lead inputs via an n8n Form, enriches the company using Clearbit, finds contacts with Hunter.io, optionally pulls reputation signals (Google Maps) and website intelligence, then uses Google Gemini (via the LangChain nodes) to score the lead. Results are saved to Google Sheets, and “hot” leads trigger a Slack alert.

**Target use cases:**
- Inbound lead qualification (website form → enrichment → scoring)
- Sales/SDR lead prioritization with automated enrichment and AI scoring
- Building a lightweight lead-enrichment pipeline without a CRM dependency

### Logical blocks
1. **1.1 Lead intake (Form Trigger)**
2. **1.2 Company enrichment (Clearbit) + normalization**
3. **1.3 Contact discovery (Hunter.io) + merge**
4. **1.4 Reputation signals (Google Maps) + merge**
5. **1.5 Website fetching + website intelligence extraction**
6. **1.6 AI scoring (Gemini via LangChain)**
7. **1.7 Final shaping, persistence (Google Sheets), and alerting (Slack)**

---

## 2. Block-by-Block Analysis

### 2.1 Lead intake (Form Trigger)

**Overview:**  
Receives a new lead submission and starts the pipeline. The captured fields are expected to include company-identifying information (e.g., domain or company name).

**Nodes involved:**
- Lead Input Form

#### Node: Lead Input Form
- **Type / role:** `Form Trigger` (n8n) — entry point, produces one item per submission.
- **Configuration (interpreted):** Parameters are empty in the JSON, so the form fields are not visible here. Typically configured with fields such as `company`, `domain`, `name`, `email`, etc.
- **Key variables/expressions:** Not shown (depends on form field keys).
- **Connections:**  
  - **Output →** Clearbit Enrichment
- **Version notes:** `typeVersion 2.1`
- **Potential failures / edge cases:**
  - Missing required fields (domain/company) causing downstream enrichment calls to fail.
  - Unexpected field names (if code nodes expect specific keys).
  - Spam or malformed URLs/domains.

---

### 2.2 Company enrichment (Clearbit) + normalization

**Overview:**  
Calls Clearbit to enrich company data, then parses/normalizes the response into a consistent internal structure for later steps.

**Nodes involved:**
- Clearbit Enrichment
- Parse Company Data

#### Node: Clearbit Enrichment
- **Type / role:** `HTTP Request` — calls Clearbit Enrichment API.
- **Configuration (interpreted):** Not provided in JSON; typically:
  - Method: `GET`
  - URL: Clearbit Company/Enrichment endpoint (often `https://company.clearbit.com/v2/companies/find?domain=...` or similar)
  - Auth: Clearbit API key (usually via Header Authorization)
  - Response: JSON
- **Key expressions/variables (expected):**
  - Domain pulled from the form submission, e.g. `{{$json.domain}}`
- **Connections:**  
  - **Input ←** Lead Input Form  
  - **Output →** Parse Company Data
- **Version notes:** `typeVersion 4.2`
- **Potential failures / edge cases:**
  - 401/403 if Clearbit key missing/invalid.
  - 404/no match if domain is unknown.
  - Rate limiting (429).
  - If input is company name (not domain), endpoint choice matters.

#### Node: Parse Company Data
- **Type / role:** `Code` — transforms Clearbit response into a normalized company object.
- **Configuration (interpreted):** Code not included; typically:
  - Extracts fields like `companyName`, `domain`, `industry`, `employeeCount`, `location`, `description`, `tags`, `funding`, `tech`, etc.
  - Produces a simplified JSON structure for downstream nodes.
- **Key expressions/variables:** Depends on Clearbit response keys.
- **Connections:**  
  - **Input ←** Clearbit Enrichment  
  - **Output →** Hunter.io Contacts
- **Version notes:** `typeVersion 2`
- **Potential failures / edge cases:**
  - Code throws if Clearbit returns null/empty fields.
  - Schema drift (Clearbit changes response structure).
  - Multiple items: code must handle arrays vs single objects consistently.

---

### 2.3 Contact discovery (Hunter.io) + merge

**Overview:**  
Fetches contacts/emails associated with the enriched company domain using Hunter.io, then merges the contact results into the working lead record.

**Nodes involved:**
- Hunter.io Contacts
- Merge Contact Data

#### Node: Hunter.io Contacts
- **Type / role:** `HTTP Request` — calls Hunter.io domain search / email-finder style endpoint.
- **Configuration (interpreted):** Not provided; typically:
  - Method: `GET`
  - URL: `https://api.hunter.io/v2/domain-search?domain=...&api_key=...` (or similar)
  - Response: JSON
- **Key expressions/variables (expected):**
  - `domain` from Parse Company Data output
- **Connections:**  
  - **Input ←** Parse Company Data  
  - **Output →** Merge Contact Data
- **Version notes:** `typeVersion 4.2`
- **Potential failures / edge cases:**
  - 401 if API key invalid.
  - 429 rate limiting.
  - Empty contact list for small/private companies.
  - Compliance: ensure outreach respects applicable email/marketing laws.

#### Node: Merge Contact Data
- **Type / role:** `Code` — merges Hunter contact list into the main lead JSON.
- **Configuration (interpreted):** Code not provided; typically:
  - Selects “best” contacts (by seniority/department/confidence score)
  - Adds `contacts[]` and possibly `bestContact`
- **Connections:**  
  - **Input ←** Hunter.io Contacts  
  - **Output →** Google Maps Reputation
- **Version notes:** `typeVersion 2`
- **Potential failures / edge cases:**
  - Handling when Hunter returns no `data.emails`.
  - Deduplication and sorting logic errors.
  - Large arrays causing downstream prompt/token bloat for AI scoring.

---

### 2.4 Reputation signals (Google Maps) + merge

**Overview:**  
Queries a Google Maps–style reputation source (likely Places API) and merges rating/review signals into the lead profile.

**Nodes involved:**
- Google Maps Reputation
- Merge Reputation Data

#### Node: Google Maps Reputation
- **Type / role:** `HTTP Request` — calls a Google Maps/Places API endpoint.
- **Configuration (interpreted):** Not provided; commonly:
  - Place search by company name + location OR website domain
  - Fetch `rating`, `user_ratings_total`, place details
- **Connections:**  
  - **Input ←** Merge Contact Data  
  - **Output →** Merge Reputation Data
- **Version notes:** `typeVersion 4.2`
- **Potential failures / edge cases:**
  - API key restrictions/billing errors.
  - Ambiguous place matches (wrong company).
  - No results for B2B companies without a public listing.

#### Node: Merge Reputation Data
- **Type / role:** `Code` — merges reputation signals into the lead JSON.
- **Configuration (interpreted):** Not provided; typically adds:
  - `maps.rating`, `maps.reviewCount`, `maps.placeId`, `maps.url`
- **Connections:**  
  - **Input ←** Google Maps Reputation  
  - **Output →** Fetch Company Website
- **Version notes:** `typeVersion 2`
- **Potential failures / edge cases:**
  - No place found → code must default safely.
  - Multiple candidates → selection criteria required.

---

### 2.5 Website fetching + website intelligence extraction

**Overview:**  
Fetches the company website HTML (non-fatal on errors) and extracts useful signals (copy, headings, offerings, tech hints) for AI scoring.

**Nodes involved:**
- Fetch Company Website
- Extract Website Intel

#### Node: Fetch Company Website
- **Type / role:** `HTTP Request` — retrieves website content.
- **Configuration (interpreted):**
  - `onError: continueRegularOutput` is explicitly set: failures will not stop the workflow.
  - The URL is not shown; expected to be built from the domain (e.g., `https://{{$json.domain}}`).
- **Connections:**  
  - **Input ←** Merge Reputation Data  
  - **Output →** Extract Website Intel
- **Version notes:** `typeVersion 4.2`
- **Potential failures / edge cases:**
  - Timeouts, TLS errors, redirects, bot protections (403), large payloads.
  - Non-HTML responses (PDF, JS-heavy site).
  - Because errors continue, downstream code must handle missing/empty body safely.

#### Node: Extract Website Intel
- **Type / role:** `Code` — parses website response into structured “intel”.
- **Configuration (interpreted):** Code not shown; likely:
  - Strips HTML → text
  - Extracts meta title/description, H1/H2, keywords, product/service cues
  - Produces a concise summary field for the AI prompt
- **Connections:**  
  - **Input ←** Fetch Company Website  
  - **Output →** AI Lead Scorer
- **Version notes:** `typeVersion 2`
- **Potential failures / edge cases:**
  - Handling empty response if fetch failed.
  - Excessive text size (should truncate to avoid large LLM prompts).

---

### 2.6 AI scoring (Gemini via LangChain)

**Overview:**  
Runs an LLM chain to score and possibly classify the lead (ICP fit, intent, priority), using Google Gemini as the chat model.

**Nodes involved:**
- AI Lead Scorer
- Google Gemini Chat Model

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini` (LangChain) — provides the LLM backend for the chain.
- **Configuration (interpreted):** Not shown; typically:
  - Model selection (e.g., Gemini 1.5)
  - Credentials: Google AI / Gemini API key
  - Optional temperature/max output tokens
- **Connections:**  
  - **Output (ai_languageModel) →** AI Lead Scorer (language model input)
- **Version notes:** `typeVersion 1`
- **Potential failures / edge cases:**
  - Auth errors, quota limits.
  - Safety filters or blocked content (rare in B2B lead text but possible).
  - Large prompts causing token limit issues.

#### Node: AI Lead Scorer
- **Type / role:** `chainLlm` (LangChain) — runs a prompt/chain to compute lead score and rationale.
- **Configuration (interpreted):** Not shown; typically:
  - Prompt includes merged enrichment/contact/reputation/website intel
  - Output likely JSON-like with fields such as `score`, `tier`, `reasons`, `recommendedNextStep`
- **Connections:**  
  - **Input ←** Extract Website Intel  
  - **Model input ←** Google Gemini Chat Model (ai_languageModel)
  - **Output →** Prepare Final Output
- **Version notes:** `typeVersion 1.4`
- **Potential failures / edge cases:**
  - Non-JSON responses if you expect machine-readable output (should enforce structured output).
  - Hallucinated facts if the prompt does not restrict to provided data.
  - Inconsistent scoring scale unless clearly specified.

---

### 2.7 Final shaping, persistence, and alerting

**Overview:**  
Consolidates all fields into a final record, saves it to Google Sheets, and triggers a Slack alert for leads passing a “hot lead” condition.

**Nodes involved:**
- Prepare Final Output
- Save to Google Sheets
- Hot Lead Filter
- Slack Hot Lead Alert

#### Node: Prepare Final Output
- **Type / role:** `Code` — final data model assembly for storage and filtering.
- **Configuration (interpreted):** Not shown; typically:
  - Flattens nested structures into sheet-friendly columns
  - Ensures presence of fields like `company`, `domain`, `leadScore`, `hotLead`, `topContacts`, `rating`, etc.
- **Connections:**  
  - **Input ←** AI Lead Scorer  
  - **Outputs →** Save to Google Sheets AND Hot Lead Filter (fan-out)
- **Version notes:** `typeVersion 2`
- **Potential failures / edge cases:**
  - Missing AI fields if LLM output parsing fails.
  - Type issues (numbers vs strings) impacting filter logic and Sheets insertion.

#### Node: Save to Google Sheets
- **Type / role:** `Google Sheets` — writes the final lead record to a spreadsheet.
- **Configuration (interpreted):** Not shown; typically:
  - Operation: Append row
  - Spreadsheet ID + Sheet name/tab
  - Column mapping from `Prepare Final Output`
  - Credentials: Google OAuth2 or Service Account
- **Connections:**  
  - **Input ←** Prepare Final Output
- **Version notes:** `typeVersion 4.5`
- **Potential failures / edge cases:**
  - Permission/credential errors.
  - Column mismatch (sheet columns changed).
  - API quota limits.

#### Node: Hot Lead Filter
- **Type / role:** `Filter` — checks whether the lead qualifies as “hot”.
- **Configuration (interpreted):** Not provided; likely conditions like:
  - `leadScore >= X`
  - or `tier == "A"`
- **Connections:**  
  - **Input ←** Prepare Final Output  
  - **Output →** Slack Hot Lead Alert (only if filter passes)
- **Version notes:** `typeVersion 2`
- **Potential failures / edge cases:**
  - If `leadScore` is missing or non-numeric, filter may behave unexpectedly.
  - Ensure consistent scoring scale (e.g., 0–100).

#### Node: Slack Hot Lead Alert
- **Type / role:** `Slack` — posts a message to Slack when lead is hot.
- **Configuration (interpreted):** Not shown; typically:
  - Resource: Message
  - Channel: e.g., `#sales-leads`
  - Message includes company, score, key contacts, and link to the sheet row
  - Slack credentials (OAuth)
- **Connections:**  
  - **Input ←** Hot Lead Filter
- **Version notes:** `typeVersion 2.2`
- **Potential failures / edge cases:**
  - Slack auth revoked / channel not found.
  - Message formatting errors if expected fields are undefined.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Canvas annotation | — | — |  |
| Sticky Note1 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note2 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note3 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note4 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note5 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note6 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note7 | stickyNote | Canvas annotation | — | — |  |
| Lead Input Form | formTrigger | Entry point (lead submission intake) | — | Clearbit Enrichment |  |
| Clearbit Enrichment | httpRequest | Company enrichment via Clearbit API | Lead Input Form | Parse Company Data |  |
| Parse Company Data | code | Normalize/enforce internal company schema | Clearbit Enrichment | Hunter.io Contacts |  |
| Hunter.io Contacts | httpRequest | Retrieve contacts/emails for domain | Parse Company Data | Merge Contact Data |  |
| Merge Contact Data | code | Merge/select contacts into lead record | Hunter.io Contacts | Google Maps Reputation |  |
| Google Maps Reputation | httpRequest | Fetch public reputation signals | Merge Contact Data | Merge Reputation Data |  |
| Merge Reputation Data | code | Merge reputation fields into lead record | Google Maps Reputation | Fetch Company Website |  |
| Fetch Company Website | httpRequest | Download website HTML/content (non-fatal errors) | Merge Reputation Data | Extract Website Intel |  |
| Extract Website Intel | code | Extract structured intel from website content | Fetch Company Website | AI Lead Scorer |  |
| AI Lead Scorer | chainLlm | LLM chain to score/classify lead | Extract Website Intel; (Model) Google Gemini Chat Model | Prepare Final Output |  |
| Google Gemini Chat Model | lmChatGoogleGemini | Gemini chat model provider for LLM chain | — | AI Lead Scorer (ai_languageModel) |  |
| Prepare Final Output | code | Build final record for storage + filtering | AI Lead Scorer | Save to Google Sheets; Hot Lead Filter |  |
| Save to Google Sheets | googleSheets | Persist lead record to Sheets | Prepare Final Output | — |  |
| Hot Lead Filter | filter | Gate for “hot lead” criteria | Prepare Final Output | Slack Hot Lead Alert |  |
| Slack Hot Lead Alert | slack | Notify Slack for hot leads | Hot Lead Filter | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add “Lead Input Form”**
   - Node type: **Form Trigger**
   - Define fields (recommended minimum):
     - `domain` (text, required) OR `companyName` + `website`
     - optional: `contactName`, `contactEmail`, `notes`
   - Save the form trigger and note the test URL for submissions.

3. **Add “Clearbit Enrichment”**
   - Node type: **HTTP Request**
   - Method: `GET`
   - Use a Clearbit enrichment endpoint appropriate to your plan (commonly company lookup by domain).
   - Auth:
     - Add header like `Authorization: Bearer <CLEARBIT_API_KEY>` (or Clearbit’s required scheme).
   - Query parameter: domain from the form, e.g. `{{$json.domain}}`
   - Connect: **Lead Input Form → Clearbit Enrichment**

4. **Add “Parse Company Data”**
   - Node type: **Code**
   - Implement logic to:
     - Read Clearbit response
     - Create normalized fields: `companyName`, `domain`, `industry`, `employeeCount`, `location`, `description`, etc.
     - Ensure defaults when missing (empty string/null)
   - Connect: **Clearbit Enrichment → Parse Company Data**

5. **Add “Hunter.io Contacts”**
   - Node type: **HTTP Request**
   - Method: `GET`
   - Endpoint: Hunter.io domain search (or chosen contact endpoint).
   - Auth: Hunter API key (query param or header, depending on endpoint).
   - Use `domain` from parsed company data.
   - Connect: **Parse Company Data → Hunter.io Contacts**

6. **Add “Merge Contact Data”**
   - Node type: **Code**
   - Merge Hunter results into the working JSON:
     - `contacts`: array of selected contacts (truncate to a manageable number)
     - `bestContact`: optional single best match
   - Connect: **Hunter.io Contacts → Merge Contact Data**

7. **Add “Google Maps Reputation”**
   - Node type: **HTTP Request**
   - Use Google Places/Maps API:
     - Either text search (`companyName + location`) then place details
     - Or a single endpoint if your design supports it
   - Credentials: Google API key with Places enabled + billing as required
   - Connect: **Merge Contact Data → Google Maps Reputation**

8. **Add “Merge Reputation Data”**
   - Node type: **Code**
   - Extract and store `rating`, `reviewCount`, `placeId`, `mapsUrl` (as available).
   - Connect: **Google Maps Reputation → Merge Reputation Data**

9. **Add “Fetch Company Website”**
   - Node type: **HTTP Request**
   - URL built from domain/website field (e.g., `https://{{$json.domain}}`)
   - Set **Error Handling**: *Continue on Fail* (matches `onError: continueRegularOutput`)
   - Consider setting a reasonable timeout and limiting response size if your n8n version supports it.
   - Connect: **Merge Reputation Data → Fetch Company Website**

10. **Add “Extract Website Intel”**
    - Node type: **Code**
    - Parse `Fetch Company Website` response:
      - Strip HTML to text
      - Extract title/meta
      - Create `websiteSummary` and maybe `keywords/services`
      - Truncate long text (important for LLM usage)
    - Connect: **Fetch Company Website → Extract Website Intel**

11. **Add “Google Gemini Chat Model”**
    - Node type: **Google Gemini Chat Model** (LangChain)
    - Configure credentials:
      - Gemini/Google AI API key (as required by your n8n node)
    - Select model and optionally set temperature/max tokens.

12. **Add “AI Lead Scorer”**
    - Node type: **LLM Chain** (`chainLlm`)
    - Configure prompt to use only provided fields, and to output structured data (recommended JSON), e.g.:
      - score (0–100)
      - tier (A/B/C)
      - reasons (array)
      - recommendedNextStep
    - Connect:
      - **Extract Website Intel → AI Lead Scorer**
      - **Google Gemini Chat Model (ai_languageModel) → AI Lead Scorer**

13. **Add “Prepare Final Output”**
    - Node type: **Code**
    - Convert the enriched/scored object into a final flattened record for Sheets:
      - Include identifiers (companyName, domain)
      - Include score/tier and key reasons
      - Include top contact fields (name/title/email if present)
      - Include maps rating/review count
    - Connect: **AI Lead Scorer → Prepare Final Output**

14. **Add “Save to Google Sheets”**
    - Node type: **Google Sheets**
    - Credentials: Google OAuth2 (or service account)
    - Operation: Append row
    - Select Spreadsheet + Sheet
    - Map columns to fields created in “Prepare Final Output”
    - Connect: **Prepare Final Output → Save to Google Sheets**

15. **Add “Hot Lead Filter”**
    - Node type: **Filter**
    - Configure condition(s), e.g.:
      - `leadScore >= 80` OR `tier == "A"`
    - Connect: **Prepare Final Output → Hot Lead Filter**

16. **Add “Slack Hot Lead Alert”**
    - Node type: **Slack**
    - Credentials: Slack OAuth
    - Operation: Post message to channel
    - Compose message using fields from final output (company, score, best contact, key reasons).
    - Connect: **Hot Lead Filter → Slack Hot Lead Alert**

17. **Test end-to-end**
    - Submit the form with a known domain.
    - Verify:
      - Clearbit and Hunter responses
      - Website fetch behavior on failures
      - LLM output structure is consistent
      - Google Sheets row appended
      - Slack fires only when filter conditions are met

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist in the canvas but have empty content in the provided workflow JSON. | No additional annotations available from the workflow. |
| Website fetch is configured to continue on error. | This prevents hard-fail on unreachable websites; downstream code must handle missing/empty content safely. |
| Several nodes have empty parameters in the provided JSON. | Exact API endpoints, request headers, prompts, and code logic must be defined during rebuild. |

If you paste the missing node configurations (HTTP Request settings, Code node scripts, AI prompt), I can produce a fully exact node-by-node specification including field mappings and concrete expressions.