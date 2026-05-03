Scrape verified Instagram leads with Apify and Airtable

https://n8nworkflows.xyz/workflows/scrape-verified-instagram-leads-with-apify-and-airtable-11428


# Scrape verified Instagram leads with Apify and Airtable

## 1. Workflow Overview

**Title:** Scrape verified Instagram leads with Apify and Airtable

**Purpose:**  
This workflow collects Instagram lead prospects by running an Apify Google Search Scraper query (targeting pages that publicly expose emails), then filters and verifies extracted emails, and finally stores qualified leads in Airtable for outreach.

**Primary use cases:**
- Build lead lists from Instagram profiles/pages indexed by Google for a given niche keyword
- Automatically validate emails before saving them
- Maintain a deduplicated Airtable table (upsert by Username)

### 1.1 Input Reception
A form collects a single “Enter Query” value (keyword/niche).

### 1.2 Scraping via Apify (Google Search Scraper)
The workflow calls an Apify actor endpoint to run a Google query like:  
`site:instagram.com + <query> + @gmail.com`

### 1.3 Result Processing
The workflow waits (buffer step), then splits the Apify results array into individual lead items.

### 1.4 Email Filtering + Verification
It keeps only items whose extracted email matches common consumer domains, then calls an email validation API. Only “VALID” + mailbox exists + MX records leads pass.

### 1.5 Persistence to Airtable
Qualified leads are upserted into Airtable with mapped fields (Username, URL, Followers, Contact Details, Email Verifier).

---

## 2. Block-by-Block Analysis

### Block A — Input + Scraping
**Overview:** Collects a keyword from a form and submits a POST request to Apify’s Google Search Scraper actor to retrieve dataset items synchronously.  
**Nodes involved:** `Query Form`, `Apify Scraping Data`

#### Node: Query Form
- **Type / role:** `Form Trigger` — workflow entry point; collects user input.
- **Configuration (interpreted):**
  - Form title: **Enter Data**
  - One field: **Enter Query**
- **Key variables/expressions:**
  - Output JSON includes a field accessible as: `{{$json['Enter Query']}}`
- **Connections:**
  - **Output →** `Apify Scraping Data`
- **Potential failures / edge cases:**
  - Empty query submitted → downstream query becomes weak or invalid (no guardrails present).
  - If field label changes, expressions referencing `Enter Query` must be updated.
- **Version notes:** Form Trigger v2.3.

#### Node: Apify Scraping Data
- **Type / role:** `HTTP Request` — calls Apify actor “run-sync-get-dataset-items”.
- **Configuration (interpreted):**
  - Method: **POST**
  - URL: `https://api.apify.com/v2/acts/apify~google-search-scraper/run-sync-get-dataset-items?token=YOUR_TOKEN_HERE`
  - Body: JSON payload (sent as JSON)
  - Query construction:
    - `queries`: `"site:instagram.com + {{ $json['Enter Query'] }} + @gmail.com"`
  - Scraper parameters include: `maxPagesPerQuery: 10`, `resultsPerPage: 100`, etc.
- **Key expressions/variables:**
  - `{{ $json['Enter Query'] }}` from `Query Form`
- **Connections:**
  - **Input ←** `Query Form`
  - **Output →** `Processing Data`
- **Potential failures / edge cases:**
  - Invalid/missing Apify token → 401/403.
  - Wrong actor endpoint URL (must be the “run-sync-get-dataset-items” endpoint).
  - Apify/Google rate limits or actor failures → non-200 response, empty dataset.
  - Returned schema may differ depending on actor settings; later nodes assume `organicResults` exists.
- **Version notes:** HTTP Request v4.2.

---

### Block B — Data Processing + Split
**Overview:** Introduces a wait step (acts as a buffer) and then splits the returned array of results into one item per lead candidate.  
**Nodes involved:** `Processing Data`, `Split Out`

#### Node: Processing Data
- **Type / role:** `Wait` — used here as a pacing/buffer node between scraping and processing.
- **Configuration (interpreted):**
  - No explicit wait parameters set (defaults apply for this node configuration).
- **Connections:**
  - **Input ←** `Apify Scraping Data`
  - **Output →** `Split Out`
- **Potential failures / edge cases:**
  - If Apify returns very large payloads, execution memory/time can still be a constraint; this node does not reduce payload.
- **Version notes:** Wait v1.1.

#### Node: Split Out
- **Type / role:** `Split Out` — converts an array field into multiple items.
- **Configuration (interpreted):**
  - Field to split: `organicResults`
- **Connections:**
  - **Input ←** `Processing Data`
  - **Output →** `Passing Emails`
- **Potential failures / edge cases:**
  - If `organicResults` is missing/not an array → node errors or yields zero items.
  - If Apify returns nested structures differently, mapping downstream fields (`url`, `displayedUrl`, `followersAmount`, etc.) may break.
- **Version notes:** Split Out v1.

---

### Block C — Email Domain Filtering
**Overview:** Filters lead items to keep only those whose extracted email matches a set of common domains (gmail/hotmail/aol/yahoo/outlook).  
**Nodes involved:** `Passing Emails`

#### Node: Passing Emails
- **Type / role:** `If` — conditional filter (OR conditions).
- **Configuration (interpreted):**
  - OR conditions: email contains one of:
    - `@gmail.com`, `@hotmail.com`, `@aol.com`, `@yahoo.com`, `@outlook.com`
  - Left value used in all conditions: `{{ $json.emphasizedKeywords[0] }}`
- **Connections:**
  - **Input ←** `Split Out`
  - **True output →** `Email Verifier`
  - (False output not connected; items that fail are dropped from the pipeline.)
- **Potential failures / edge cases:**
  - If `emphasizedKeywords` is missing/empty, `emphasizedKeywords[0]` evaluates to `undefined` and conditions may error or always fail (depending on n8n expression behavior).
  - This filter only allows specific domains; valid business emails (custom domains) are excluded.
  - The sticky note claims “contains Gmail address”, but the node also allows other domains.
- **Version notes:** If v2.2.

---

### Block D — Email Verification + Qualification
**Overview:** Calls an email verification API, then keeps only emails with “VALID” status plus MX and mailbox checks.  
**Nodes involved:** `Email Verifier`, `If1`

#### Node: Email Verifier
- **Type / role:** `HTTP Request` — validates an email via an external verification service.
- **Configuration (interpreted):**
  - Method: default (GET implied by “sendQuery: true”; node config primarily uses query parameters)
  - URL: `https://rapid-email-verifier.fly.dev/api/validate`
  - Query parameter: `email={{ $json.emphasizedKeywords[0] }}`
  - Header: `accept: application/json`
- **Connections:**
  - **Input ←** `Passing Emails` (true path)
  - **Output →** `If1`
- **Potential failures / edge cases:**
  - API downtime / 5xx errors / rate limiting.
  - If `emphasizedKeywords[0]` is not a valid email, API may return error/invalid status.
  - Response shape must include `status` and `validations.mx_records` + `validations.mailbox_exists` for the next If node.
- **Version notes:** HTTP Request v4.2.

#### Node: If1
- **Type / role:** `If` — qualifies verified emails.
- **Configuration (interpreted):** ALL conditions must be true:
  1. `{{ $json.validations.mx_records }}` is true
  2. `{{ $json.status }}` equals `"VALID"`
  3. `{{ $json.validations.mailbox_exists }}` is true
- **Connections:**
  - **Input ←** `Email Verifier`
  - **True output →** `Airtable DB`
  - (False output not connected; non-qualified leads are dropped.)
- **Potential failures / edge cases:**
  - If `validations` object is missing, strict validation may cause evaluation issues.
  - Some verifiers return “UNKNOWN”/“RISKY” statuses; these are excluded.
- **Version notes:** If v2.2.

---

### Block E — Store Qualified Leads in Airtable
**Overview:** Upserts qualified leads into Airtable, mapping scraped profile fields plus verification status.  
**Nodes involved:** `Airtable DB`

#### Node: Airtable DB
- **Type / role:** `Airtable` — database persistence via upsert.
- **Configuration (interpreted):**
  - Operation: **Upsert**
  - Base: **IG Leads**
  - Table: **Leads**
  - Matching column(s): **Username** (used to decide update vs create)
  - Mapped fields:
    - **URL** = `{{ $('Split Out').item.json.url }}`
    - **Username** = `{{ $('Split Out').item.json.displayedUrl }}`
    - **Followers** = `{{ $('Split Out').item.json.followersAmount }}`
    - **Email Verifier** = `{{ $json.status }}`
    - **Contact Details** = `{{ $json.email }}`
- **Important expression behavior:**
  - This node references **two data contexts**:
    - From `Split Out` via `$('Split Out').item.json...`
    - From the verifier response via `$json.status` and `$json.email`
  - **Risk:** The verifier node query uses `emphasizedKeywords[0]` but the Airtable mapping uses `$json.email`. If the verification API does not return an `email` field, **Contact Details may be blank**. You may need `{{ $('Split Out').item.json.emphasizedKeywords[0] }}` instead.
- **Connections:**
  - **Input ←** `If1` (true path only)
  - **Output →** none
- **Credentials:**
  - Uses an Airtable Personal Access Token credential (PAT).
- **Potential failures / edge cases:**
  - Credential missing/expired → auth error.
  - Base/table/field names must exist and match expected schema.
  - Upsert requires the matching column (“Username”) to be consistent; if `displayedUrl` changes format, duplicates can occur.
- **Version notes:** Airtable node v2.1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Query Form | Form Trigger | Collects keyword query from user | — | Apify Scraping Data | ## Input + Scraping / Enter a keyword query. Apify Google Search Scraper runs a search for Instagram pages containing public emails. |
| Apify Scraping Data | HTTP Request | Runs Apify Google Search Scraper and returns dataset items | Query Form | Processing Data | ## Input + Scraping / Enter a keyword query. Apify Google Search Scraper runs a search for Instagram pages containing public emails. |
| Processing Data | Wait | Buffer/pacing step before processing results | Apify Scraping Data | Split Out | ## Data Processing + Email Filtering / Waits for Apify results and splits the data into individual lead items and Filterss results that contain Gmail address. |
| Split Out | Split Out | Splits `organicResults` array into items | Processing Data | Passing Emails | ## Data Processing + Email Filtering / Waits for Apify results and splits the data into individual lead items and Filterss results that contain Gmail address. |
| Passing Emails | If | Filters results by email domain | Split Out | Email Verifier | ## Data Processing + Email Filtering / Waits for Apify results and splits the data into individual lead items and Filterss results that contain Gmail address. |
| Email Verifier | HTTP Request | Validates extracted email via verification API | Passing Emails | If1 | ## Email Verify + Store / Verifies emails using an email verification API. If valid, the lead is stored into Airtable. |
| If1 | If | Accepts only “VALID” with MX + mailbox checks | Email Verifier | Airtable DB | ## Email Verify + Store / Verifies emails using an email verification API. If valid, the lead is stored into Airtable. |
| Airtable DB | Airtable | Upserts qualified leads into Airtable | If1 | — | ## Email Verify + Store / Verifies emails using an email verification API. If valid, the lead is stored into Airtable. |
| Sticky Note1 | Sticky Note | Documentation / setup notes | — | — | ## How it works … (contains Apify + Airtable setup links and clone table link) |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Email Verify + Store / Verifies emails using an email verification API. If valid, the lead is stored into Airtable. |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## Input + Scraping / Enter a keyword query. Apify Google Search Scraper runs a search for Instagram pages containing public emails. |
| Sticky Note | Sticky Note | Documentation | — | — | ## Data Processing + Email Filtering / Waits for Apify results and splits the data into individual lead items and Filterss results that contain Gmail address. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Scrape verified Instagram leads with Apify and Airtable*.

2. **Add “Form Trigger” node** (`Query Form`)
   - Title: **Enter Data**
   - Add one form field with label: **Enter Query**
   - This is your entry point.

3. **Add “HTTP Request” node** (`Apify Scraping Data`)
   - Method: **POST**
   - URL: `https://api.apify.com/v2/acts/apify~google-search-scraper/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`
     - Replace `YOUR_APIFY_TOKEN` with your Apify API token.
   - Body content type: **JSON**
   - Enable “Send Body”
   - Body JSON (key parts):
     - `queries`: `site:instagram.com + {{ $json['Enter Query'] }} + @gmail.com`
     - `maxPagesPerQuery`: 10
     - `resultsPerPage`: 100
   - Connect: **Query Form → Apify Scraping Data**

4. **Add “Wait” node** (`Processing Data`)
   - Leave default settings (as in provided workflow).
   - Connect: **Apify Scraping Data → Processing Data**

5. **Add “Split Out” node** (`Split Out`)
   - Field to split out: `organicResults`
   - Connect: **Processing Data → Split Out**

6. **Add “If” node** (`Passing Emails`)
   - Condition group: **OR**
   - Add conditions (String → contains) against:
     - Left value: `{{ $json.emphasizedKeywords[0] }}`
     - Right values: `@gmail.com`, `@hotmail.com`, `@aol.com`, `@yahoo.com`, `@outlook.com`
   - Connect: **Split Out → Passing Emails** (from Split Out main output)

7. **Add “HTTP Request” node** (`Email Verifier`)
   - URL: `https://rapid-email-verifier.fly.dev/api/validate`
   - Add query parameter:
     - `email` = `{{ $json.emphasizedKeywords[0] }}`
   - Add header:
     - `accept: application/json`
   - Connect: **Passing Emails (true) → Email Verifier**

8. **Add “If” node** (`If1`) to qualify verified emails
   - Condition group: **AND**
   - Conditions:
     1. Boolean is true: `{{ $json.validations.mx_records }}`
     2. String equals: `{{ $json.status }}` equals `VALID`
     3. Boolean is true: `{{ $json.validations.mailbox_exists }}`
   - Connect: **Email Verifier → If1**

9. **Add “Airtable” node** (`Airtable DB`)
   - Credentials: create/connect **Airtable Personal Access Token**
     - Ensure PAT has access to the base.
   - Select **Base** and **Table** (create them first if needed).
   - Operation: **Upsert**
   - Matching column: **Username**
   - Create fields in Airtable table (at minimum):
     - Username (text)
     - Contact Details (text)
     - URL (text)
     - Followers (text/number)
     - Email Verifier (text)
   - Field mappings:
     - URL = `{{ $('Split Out').item.json.url }}`
     - Username = `{{ $('Split Out').item.json.displayedUrl }}`
     - Followers = `{{ $('Split Out').item.json.followersAmount }}`
     - Email Verifier = `{{ $json.status }}`
     - Contact Details = (recommended safer mapping) `{{ $('Split Out').item.json.emphasizedKeywords[0] }}`
       - Use this if the verifier response does not include `$json.email`.
   - Connect: **If1 (true) → Airtable DB**

10. **Run and test**
   - Execute workflow, submit the form query (e.g., `fitness coach london`)
   - Confirm Apify returns results with `organicResults`
   - Confirm Airtable rows are created/updated.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create an Apify account and use the Google Search Scraper actor. Copy the “Run actor synchronously and get dataset items” endpoint URL into the Apify Scraping Data node, and add your token. | Apify actor input: https://console.apify.com/actors/nFJndFXA5zjCTuudP/input |
| Airtable setup requires a base/table with fields: Username, Contact Details, URL, Followers, Email Verifier; connect Airtable credentials in the Airtable DB node. | Airtable credential guide: https://docs.n8n.io/integrations/builtin/credentials/airtable/ |
| Optional Airtable table shortcut: “Clone Table” | https://airtable.com/appOfLatszE9cHYw3/shrbctCMIY7tOdv15 |
| Workflow author contact reference | “Contact Me [@msiddhant](#)” |
| How it works summary: scrape Instagram leads via Apify, filter/verify emails, store into Airtable for outreach. | From the workflow sticky notes |

