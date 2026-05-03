Extract and qualify local business leads and draft cold emails with OpenAI, Apify and Hunter

https://n8nworkflows.xyz/workflows/extract-and-qualify-local-business-leads-and-draft-cold-emails-with-openai--apify-and-hunter-12855


# Extract and qualify local business leads and draft cold emails with OpenAI, Apify and Hunter

## 1. Workflow Overview

**Workflow name:** AI-Powered Local Business Lead Scraping, Qualification and Outreach System  
**Stated title:** Extract and qualify local business leads and draft cold emails with OpenAI, Apify and Hunter

**Purpose / use case**  
This workflow pulls local business leads from an Apify dataset (SERP-like results), visits each business website, uses OpenAI to (1) qualify the lead and (2) extract/normalize contact details, verifies the email with Hunter, then writes qualified leads and AI-written cold email drafts into a Google Sheets “CRM”.

### 1.1 Lead intake (Apify → controlled sample)
- Pull 1+ lead items from an Apify Dataset.
- Limit to a small number for safe testing.
- Iterate item-by-item.

### 1.2 Website scraping (HTTP → HTML → text)
- Fetch each lead’s homepage URL.
- Extract the visible page body text for downstream AI parsing.

### 1.3 AI qualification + email deliverability validation
- OpenAI classifies “quality tier” and key signals.
- Hunter verifies the email deliverability.
- Filter keeps only qualified + deliverable leads.

### 1.4 Enrichment + AI normalization + cold email drafting
- Tavily searches based on the discovered email (optional enrichment/validation).
- OpenAI “Normalize Agent” extracts structured contact details from scraped text.
- OpenAI “Cold Email Writer” generates a personalized cold email draft (JSON payload).

### 1.5 CRM writes (Google Sheets) + flow control
- Write normalized lead data to the CRM sheet (append-or-update).
- Write cold email draft to the same CRM sheet (append-or-update).
- Merge branches to continue looping through items.

---

## 2. Block-by-Block Analysis

### Block A — Lead Source (Apify) + controlled execution
**Overview:** Pulls lead items from a prebuilt Apify dataset and intentionally limits throughput for safe validation before scaling.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Get dataset items (Apify)
- Limit

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger; starts the workflow on demand.
- **Config:** No parameters.
- **Outputs to:** `Get dataset items`.
- **Failure modes:** None (manual start only).

#### Node: Get dataset items
- **Type / role:** Apify node (`Datasets`); fetches items from a specific dataset.
- **Config choices:**
  - Resource: **Datasets**
  - Dataset ID: **DYsNRwZgOhOhmXttk**
  - Limit: **1** (node-level)
  - Auth: **Apify OAuth2**
- **Outputs to:** `Limit`
- **Assumptions / required input structure:** Downstream expects each item to contain **`organicResults.url`** (used as the website URL).
- **Failure modes / edge cases:**
  - Invalid dataset ID / permissions → auth/404 errors.
  - Dataset items not shaped as expected (missing `organicResults.url`) → later HTTP Request fails.
  - Rate limits depending on Apify plan.
- **Credentials:** `apifyOAuth2Api`

#### Node: Limit
- **Type / role:** Limit; caps number of processed items for testing.
- **Config choices:** `maxItems: 3`
- **Outputs to:** `Loop Over Items`
- **Edge cases:** If upstream returns fewer than 3 items, it just processes what exists.

**Sticky note coverage (applies to nodes in this block):**  
- “## Lead Source (Apify)… The Limit node is intentional — don’t remove it until you trust the pipeline.”

---

### Block B — Iteration + website scraping
**Overview:** Loops through each lead, fetches the homepage, extracts the body text for AI consumption. HTTP failures are tolerated.

**Nodes involved:**
- Loop Over Items (Split in Batches)
- Scrape Home (HTTP Request)
- Scrape Lead Website (HTML extract)

#### Node: Loop Over Items
- **Type / role:** Split In Batches; controls looping over lead items.
- **Config choices:** Default options (batch size not explicitly set in JSON).
- **Connections:**
  - **Input:** from `Limit`
  - **Output (index 1)** → `Scrape Home` (this is the “current batch item” path in this workflow)
  - **Output (index 0)** is unused (empty), which is unusual but still workable depending on how the node is configured/behaves.
  - **Second input:** receives from `Merge` to continue the loop.
- **Edge cases / failure modes:**
  - Miswired outputs can cause “no items” flowing to scraping. Here, the workflow is wired to output index **1**, so verify in n8n UI that the intended output is used.
  - If you change SplitInBatches settings, connection indexes may need review.

#### Node: Scrape Home
- **Type / role:** HTTP Request; fetches the website homepage.
- **Config choices:**
  - URL: `={{ $json.organicResults.url }}`
  - Redirects: up to **5**
  - `onError: continueErrorOutput` (important: failures won’t stop the workflow)
- **Outputs to:** `Scrape Lead Website`
- **Edge cases / failure modes:**
  - Missing/invalid URL → request error.
  - 403/429/Cloudflare blocking → request error output; downstream may get empty/partial data.
  - Non-HTML content (PDF, image) → HTML extraction may produce poor/empty text.

#### Node: Scrape Lead Website
- **Type / role:** HTML node; extracts HTML content from the fetched page.
- **Operation:** Extract HTML content
- **Extraction:** CSS selector `body` → key `content`
- **Data property name:** `data` (but extraction key is `content`; confirm resulting structure in execution data)
- **Outputs to:** `Lead Qualification Agent`
- **Edge cases / failure modes:**
  - If upstream request failed and returned no HTML, this node may output empty `content`.
  - Some pages are SPA/minified; “body” may be large/noisy.

**Sticky note coverage:**  
- “## Website Scraping… Some sites will fail or block scraping… Failures won’t stop the workflow…”

---

### Block C — AI qualification + email verification + filtering
**Overview:** Uses OpenAI to classify the lead (quality tier and signals), verifies the extracted email via Hunter, then filters out unqualified or undeliverable leads.

**Nodes involved:**
- Lead Qualification Agent (OpenAI)
- Hunter (Email Verifier)
- Filter Qualified Leads

#### Node: Lead Qualification Agent
- **Type / role:** OpenAI (LangChain) chat/completion; produces a JSON object describing lead quality.
- **Model:** `gpt-5-mini`
- **Output format:** JSON object (`textFormat: json_object`)
- **Prompt intent (interpreted):**
  - Determine three boolean categories:
    - has_email_and_site
    - has_lead_ops_need
    - has_social_presence
  - Produce `quality_tier` integer 0–3 based on those signals
  - Requires an additional field `"email (if exists)"` (but the prompt/schema is inconsistent—see edge cases)
- **Input content:** `={{ $('Scrape Lead Website').item.json.content }}`
- **Outputs to:** `Hunter`
- **Edge cases / failure modes:**
  - **Schema/prompt inconsistency:** The prompt includes malformed/duplicated JSON examples (`"has social: false,` missing quote; repeated sections). This can cause invalid JSON output.
  - The required `"email (if exists)"` is referenced later; if the model doesn’t include it exactly, downstream expressions will fail or become empty.
  - If scraped content is empty, model may default to tier 0.

#### Node: Hunter
- **Type / role:** Hunter API; verifies an email address deliverability.
- **Operation:** Email Verifier
- **Email expression:**
  - `={{ $json.output[1].content[0].text["email (if exists)"] }}`
  - This assumes the OpenAI node outputs data under `output[1].content[0].text` and includes the key `"email (if exists)"`.
- **Outputs to:** `Filter Qualified Leads`
- **Failure modes / edge cases:**
  - Missing email → Hunter may error or return invalid status.
  - Hunter API quota/rate limits.
  - If the OpenAI output path changes (different node version/setting), expression breaks.

#### Node: Filter Qualified Leads
- **Type / role:** Filter; gates leads based on quality tier and Hunter status.
- **Conditions (AND):**
  1) `quality_tier != 0` using:
     - `={{ $('Lead Qualification Agent').item.json.output[1].content[0].text.quality_tier }}`
  2) `status == "valid"` using:
     - `={{ $json.status }}` (Hunter response)
- **Outputs to:** `Search` (Tavily)
- **Edge cases / failure modes:**
  - If `quality_tier` is missing/non-numeric → strict validation may fail; lead may be dropped or error depending on node behavior.
  - Hunter statuses can be `invalid`, `accept_all`, `unknown`, etc. This filter only allows exactly `valid`, which is strict.

**Sticky note coverage:**  
- “## Qualification & Validation… adjust the filter — not the AI prompt.”

---

### Block D — Enrichment (Tavily) + AI normalization + cold email drafting
**Overview:** Optionally enriches/validates using Tavily search, then OpenAI extracts normalized contact data and writes a cold email draft.

**Nodes involved:**
- Search (Tavily)
- Lead Normalize Agent (OpenAI)
- Cold Email Writer AI Agent1 (OpenAI)

#### Node: Search
- **Type / role:** Tavily search; enrichment/validation step based on email.
- **Query expression:**
  - `={{ $('Lead Qualification Agent').item.json.output[1].content[0].text["email (if exists)"] }}`
- **Outputs to:** `Lead Normalize Agent` and `Cold Email Writer AI Agent1` (fan-out)
- **Edge cases / failure modes:**
  - If email missing → query is empty/low-signal.
  - Tavily API quota/rate limiting.
  - Results shape (`$json.results`) is assumed downstream in AI prompts.

#### Node: Lead Normalize Agent
- **Type / role:** OpenAI JSON extraction; converts scraped text into a strict, single-object contact record.
- **Model:** `gpt-5-mini`
- **Output format:** JSON object
- **Prompt intent (interpreted):**
  - Extract: company name, email, phone (raw + E.164 if confident), socials, messengers, contact page URL, address, person name/title/contact, contexts, notes, confidence score, and a one-line icebreaker.
  - Must match a strict JSON Schema (additionalProperties=false, empty strings instead of null).
- **Input content combines:**
  - `$('Scrape Lead Website').item.json.content`
  - `{{ $json.results }}` from Tavily
- **Outputs to:** `Write Qualified Leads to CRM`
- **Edge cases / failure modes:**
  - Very long page text may exceed token limits; consider truncation or adding an HTML-to-text cleaner.
  - The “Icebreaker” instruction includes `{him/her}` but also says use “them” if unclear; ensure model complies.
  - If Tavily returns no `results`, expression concatenation still works but may add `undefined` in some cases (depends on runtime coercion).

#### Node: Cold Email Writer AI Agent1
- **Type / role:** OpenAI JSON generation; produces structured cold email parts.
- **Model:** `gpt-5-mini`
- **Output format:** JSON object with keys:
  - `email_address`, `subjectLine`, `icebreaker`, `elevatorPitch`, `callToAction`, `ps`
- **Input content combines:**
  - `$('Scrape Lead Website').item.json.content`
  - `{{ $json.results }}` from Tavily
- **Outputs to:** `Write Cold Email to CRM`
- **Edge cases / failure modes:**
  - Prompt says “based on LinkedIn profile” but no LinkedIn is provided—output quality may be generic.
  - If it fails to output valid JSON, Sheets write will fail.

**Sticky note coverage:**  
- “## Cold Email Drafting… Do not auto-send unless you add your own approval layer.”

---

### Block E — CRM writes (Google Sheets) + merge + looping
**Overview:** Writes structured lead data and cold email drafts into Google Sheets using “append or update” keyed by Email, then merges the two branches and returns control to the loop.

**Nodes involved:**
- Write Qualified Leads to CRM (Google Sheets)
- Write Cold Email to CRM (Google Sheets)
- Merge

#### Node: Write Qualified Leads to CRM
- **Type / role:** Google Sheets; upsert lead record into CRM.
- **Operation:** Append or Update
- **Spreadsheet:** Document “CRM” (ID `1ROLrrdi9qCrbt80dpn27UrnXydCRH_oGKbW61_G_liw`), Sheet `Sheet1` (gid=0)
- **Matching column:** `Email` (prevents duplicates on reruns)
- **Mapped fields:** Many columns filled from:
  - `={{ $json.output[1].content[0].text.<field> }}`
  - Example: `Source URL`, `Company Legal Name`, `Phone Raw/E164`, contexts, etc.
- **Outputs to:** `Merge` (input 0)
- **Failure modes / edge cases:**
  - If the extracted `Email` is empty, upsert matching becomes unreliable (may append duplicates or fail).
  - Column names must exactly match the sheet headers.
  - OAuth scopes/permissions issues on the Google account.
  - If the OpenAI node output path differs, all mappings break.

#### Node: Write Cold Email to CRM
- **Type / role:** Google Sheets; upserts the cold email draft into the same CRM row.
- **Operation:** Append or Update
- **Matching column:** `Email`
- **Mapped fields:**
  - `Email` = `={{ $json.output[1].content[0].text.email_address }}`
  - `Cold Email` = `={{ $json.output[1].content[0].text }}`
- **Important mismatch:** The Cold Email Writer returns a JSON object with separate parts, but this mapping attempts to write `$json.output[1].content[0].text` (likely an object) into a single cell. Depending on n8n coercion, it may become `[object Object]` or stringified JSON.
- **Outputs to:** `Merge` (input 1)
- **Failure modes / edge cases:**
  - Email mismatch between normalized lead email vs cold-email `email_address` can cause two separate rows or failure to update the intended one.
  - Object-to-string conversion issue for `Cold Email`.

#### Node: Merge
- **Type / role:** Merge; joins the two branches after both sheet writes.
- **Mode:** Choose Branch
- **Output:** Returns to `Loop Over Items` to continue processing next lead.
- **Edge cases:**
  - “Choose Branch” generally forwards data from a selected branch; if one branch fails or doesn’t run, you may loop with partial data. Consider “Wait for both” style merge if you need synchronization.

**Sticky note coverage:**  
- “## CRM Write… append-or-update logic… If you’re debugging, always inspect the sheet first.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get dataset items | # AI-Powered Local Business Lead Scraping, Qualification & Outreach System… Required Setup… How to run… |
| Get dataset items | Apify (Datasets) | Pull leads from Apify dataset | When clicking ‘Execute workflow’ | Limit | ## Lead Source (Apify)… The Limit node is intentional… |
| Limit | Limit | Cap number of processed leads | Get dataset items | Loop Over Items | ## Lead Source (Apify)… The Limit node is intentional… |
| Loop Over Items | Split In Batches | Iterate over lead items | Limit, Merge | Scrape Home | ## Lead Source (Apify)… The Limit node is intentional… |
| Scrape Home | HTTP Request | Fetch homepage HTML | Loop Over Items | Scrape Lead Website | ## Website Scraping… failures expected… |
| Scrape Lead Website | HTML | Extract body text from HTML | Scrape Home | Lead Qualification Agent | ## Website Scraping… failures expected… |
| Lead Qualification Agent | OpenAI (LangChain) | Classify lead + output quality tier | Scrape Lead Website | Hunter | ## Qualification & Validation… adjust the filter… |
| Hunter | Hunter | Verify email deliverability | Lead Qualification Agent | Filter Qualified Leads | ## Qualification & Validation… adjust the filter… |
| Filter Qualified Leads | Filter | Keep only quality_tier != 0 and status=valid | Hunter | Search | ## Qualification & Validation… adjust the filter… |
| Search | Tavily | Optional enrichment/validation lookup | Filter Qualified Leads | Lead Normalize Agent; Cold Email Writer AI Agent1 | ## Qualification & Validation… adjust the filter… |
| Lead Normalize Agent | OpenAI (LangChain) | Extract + normalize contact details into strict schema | Search | Write Qualified Leads to CRM | ## CRM Write… system of record… |
| Write Qualified Leads to CRM | Google Sheets | Upsert qualified lead record | Lead Normalize Agent | Merge | ## CRM Write… system of record… |
| Cold Email Writer AI Agent1 | OpenAI (LangChain) | Generate structured cold email parts | Search | Write Cold Email to CRM | ## Cold Email Drafting… drafts only… |
| Write Cold Email to CRM | Google Sheets | Upsert cold email into CRM | Cold Email Writer AI Agent1 | Merge | ## Cold Email Drafting… drafts only… |
| Merge | Merge | Rejoin branches and continue loop | Write Qualified Leads to CRM; Write Cold Email to CRM | Loop Over Items | (no sticky note directly over it in JSON) |
| Sticky Note | Sticky Note | Comment block | — | — | # AI-Powered Local Business Lead Scraping, Qualification & Outreach System… Required Setup… |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Lead Source (Apify)… |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Website Scraping… |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Qualification & Validation… |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## CRM Write… |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## Cold Email Drafting… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named similar to: *AI-Powered Local Business Lead Scraping, Qualification and Outreach System*.

2) **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3) **Add Apify: Get dataset items**
   - Node: **Apify**
   - Resource: **Datasets**
   - Operation: **Get dataset items**
   - Dataset ID: set to your dataset (replace `DYsNRwZgOhOhmXttk`)
   - Limit: `1` (or more later)
   - Credentials: **Apify OAuth2** connected to your Apify account
   - Connect: Manual Trigger → Apify

4) **Add Limit**
   - Node: **Limit**
   - Max items: `3` (keep small initially)
   - Connect: Apify → Limit

5) **Add Split In Batches**
   - Node: **Split In Batches**
   - Name: `Loop Over Items`
   - Leave default batch settings initially
   - Connect: Limit → Split In Batches

6) **Add HTTP Request (homepage fetch)**
   - Node: **HTTP Request**
   - Name: `Scrape Home`
   - URL: `={{ $json.organicResults.url }}`
   - Redirects: enable, maxRedirects = `5`
   - Error handling: set **On Error** = *Continue (output error)*
   - Connect: Split In Batches → HTTP Request  
     (ensure you connect the output that emits the current item)

7) **Add HTML extraction**
   - Node: **HTML**
   - Name: `Scrape Lead Website`
   - Operation: **Extract HTML Content**
   - CSS Selector: `body`
   - Key name: `content`
   - Connect: HTTP Request → HTML

8) **Add OpenAI node for qualification**
   - Node: **OpenAI (LangChain in n8n)**
   - Name: `Lead Qualification Agent`
   - Model: `gpt-5-mini` (or any available)
   - Response format: **JSON object**
   - Prompt: paste the qualification instructions (ensure it outputs consistent JSON keys, including the email key you will reference later)
   - Input: include `={{ $('Scrape Lead Website').item.json.content }}`
   - Credentials: **OpenAI API** configured
   - Connect: HTML → Qualification Agent

9) **Add Hunter Email Verifier**
   - Node: **Hunter**
   - Operation: **Email Verifier**
   - Email: map from the qualification output (or from normalized extraction if you prefer). As built:  
     `={{ $json.output[1].content[0].text["email (if exists)"] }}`
   - Credentials: **Hunter API key**
   - Connect: Qualification Agent → Hunter

10) **Add Filter**
   - Node: **Filter**
   - Conditions (AND):
     - `quality_tier` not equals `0` (from Qualification Agent)
     - Hunter `status` equals `"valid"`
   - Connect: Hunter → Filter

11) **Add Tavily Search (optional enrichment)**
   - Node: **Tavily**
   - Query: email from qualification output  
     `={{ $('Lead Qualification Agent').item.json.output[1].content[0].text["email (if exists)"] }}`
   - Credentials: **Tavily API**
   - Connect: Filter → Tavily

12) **Add OpenAI Lead Normalize Agent**
   - Node: **OpenAI (LangChain)**
   - Name: `Lead Normalize Agent`
   - Model: `gpt-5-mini`
   - Response format: **JSON object**
   - Prompt: paste the strict normalization/schema prompt
   - User/content input: combine website text + Tavily results, e.g.  
     `={{ $('Scrape Lead Website').item.json.content }}{{ $json.results }}`
   - Connect: Tavily → Lead Normalize Agent

13) **Add Google Sheets: Write Qualified Leads**
   - Node: **Google Sheets**
   - Operation: **Append or Update**
   - Document: select your “CRM” spreadsheet
   - Sheet: select the target sheet tab (e.g., Sheet1)
   - Matching columns: `Email`
   - Map columns from Normalize Agent JSON fields (Email, Phone, Address, contexts, etc.)
   - Credentials: **Google Sheets OAuth2**
   - Connect: Lead Normalize Agent → Google Sheets (Qualified write)

14) **Add OpenAI Cold Email Writer**
   - Node: **OpenAI (LangChain)**
   - Name: `Cold Email Writer AI Agent1`
   - Model: `gpt-5-mini`
   - Response format: **JSON object**
   - Prompt: paste the cold email JSON-shape prompt
   - Input: same combined content pattern as above
   - Connect: Tavily → Cold Email Writer (fan-out)

15) **Add Google Sheets: Write Cold Email**
   - Node: **Google Sheets**
   - Operation: **Append or Update**
   - Same Document + Sheet
   - Matching columns: `Email`
   - Map:
     - `Email` from `email_address`
     - `Cold Email` from the generated content (ideally build a single string from the JSON parts, or store JSON string)
   - Connect: Cold Email Writer → Google Sheets (Cold email write)

16) **Add Merge**
   - Node: **Merge**
   - Mode: **Choose Branch** (as in the original)
   - Connect:
     - Qualified Leads Sheets → Merge (Input 0)
     - Cold Email Sheets → Merge (Input 1)

17) **Close the loop**
   - Connect: Merge → Split In Batches (`Loop Over Items`) **second input** (to continue processing the next lead).

18) **Credentials checklist**
   - Apify OAuth2
   - OpenAI API
   - Hunter API
   - Google Sheets OAuth2
   - Tavily API (optional)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Required setup: Apify dataset, OpenAI model access (default gpt-5-mini), Google Sheets CRM destination, Hunter email verification, Tavily optional | From sticky note “AI-Powered Local Business Lead Scraping, Qualification & Outreach System” |
| Run strategy: start small (Limit is intentional), increase batch size after validating results | From sticky note “AI-Powered Local Business…” |
| Scraping failures are expected; workflow continues | From sticky note “Website Scraping” |
| CRM is the system of record; append-or-update prevents duplicates when Email matches | From sticky note “CRM Write” |
| Cold emails are drafts only; add an approval layer before any sending | From sticky note “Cold Email Drafting” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.