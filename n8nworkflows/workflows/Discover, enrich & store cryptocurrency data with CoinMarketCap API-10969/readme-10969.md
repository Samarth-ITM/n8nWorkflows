Discover, enrich & store cryptocurrency data with CoinMarketCap API

https://n8nworkflows.xyz/workflows/discover--enrich---store-cryptocurrency-data-with-coinmarketcap-api-10969


# Discover, enrich & store cryptocurrency data with CoinMarketCap API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Discover, enrich & store cryptocurrency data with CoinMarketCap API  
**Workflow name (n8n):** Get Tokens From CMC Using Their Free API  
**Purpose:** Periodically discover cryptocurrency tokens via CoinMarketCap (CMC) listings, enrich each token with official metadata (website + social links), normalize/clean the website URL, and store clean results (example: NocoDB). The workflow uses batching + delays to reduce rate-limit issues.

### 1.1 Scheduling & Randomized Discovery
- Runs every 3 days at 10AM (Africa/Lagos).
- Randomizes the CMC listing “page” (start/limit/sort/sort_dir) to sample different parts of the listings.

### 1.2 Fetch Listings & Normalize Token Rows
- Calls `v1/cryptocurrency/listings/latest`.
- Splits the `data` array into one item per token.
- Keeps only a minimal set of token fields needed for enrichment.

### 1.3 Batch Processing & Enrichment
- Processes tokens in batches of 10.
- For each token, calls `v2/cryptocurrency/info?id=...` to obtain official URLs.

### 1.4 Website Cleaning, Output Formatting, and Looping
- Normalizes the website URL (array/string, ensures https).
- Waits 1 minute between cycles to reduce API pressure.
- Creates a standardized output structure and loops back to the batch node to continue.

### 1.5 Storage (Example: NocoDB)
- Prepares final “token records” (skipping items without valid websites).
- Inserts rows into NocoDB.
- Uses an additional batching node for the storage step.

---

## 2. Block-by-Block Analysis

### Block A — Scheduling & Random Page Generation
**Overview:** Triggers periodically and generates randomized query parameters (limit/start/sort/sort_dir) for diversified token discovery.

**Nodes involved:**
- Every 3 Days At 10AM
- Generate Random Page

#### Node: Every 3 Days At 10AM
- **Type / role:** Schedule Trigger; entry point.
- **Config:** Runs every 3 days at 10:00 (workflow timezone: `Africa/Lagos`).
- **Outputs to:** Generate Random Page
- **Failure modes / edge cases:**
  - Workflow inactive → no runs.
  - Timezone misalignment if instance timezone differs from desired market time.

#### Node: Generate Random Page
- **Type / role:** Code node; builds random listings URL parameters.
- **Config choices:**
  - Chooses random `limit` from `[50,100,200]`
  - Chooses random `start` from `1..5000`
  - Chooses random `sort` from multiple fields (market cap, volume, percent changes, etc.)
  - Chooses random `sort_dir` from `asc|desc`
  - Computes `page = ceil(start/limit)`
- **Key outputs:**
  - `fullURL` (composed, informational)
  - `params: { start, limit, sort, sort_dir, page }`
- **Outputs to:** Get 100 Tokens From CMC
- **Failure modes / edge cases:**
  - `start` beyond free-plan accessible range can yield empty results depending on API plan.
  - Random sorts may produce unstable results across runs (expected).

---

### Block B — Fetch Listings (CMC) & Split into Items
**Overview:** Fetches latest CMC listings and converts the returned `data` array into per-token items.

**Nodes involved:**
- Get 100 Tokens From CMC
- Split Out

#### Node: Get 100 Tokens From CMC
- **Type / role:** HTTP Request; calls CMC listings endpoint.
- **Endpoint:** `https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest`
- **Auth pattern:** Uses header `X-CMC_PRO_API_KEY` (value is empty in JSON; must be filled).
- **Parameter passing (implemented as headers in this workflow):**
  - `limit = {{$json.params.limit}}`
  - `sort = {{$json.params.sort}}`
  - `sort_dir = {{$json.params.sort_dir}}`
  - `start = {{$json.params.start}}`
  - Note: these are typically **query parameters**, not headers. CMC may ignore them if sent as headers.
- **Batching option:** batching enabled with `batchSize: 3` (n8n HTTP batching feature).
- **Inputs from:** Generate Random Page
- **Outputs to:** Split Out
- **Failure modes / edge cases:**
  - Missing/invalid API key → 401/403.
  - Rate limiting (429) on free tier.
  - If CMC ignores “params in headers”, you may always get default listing output.
  - Network timeouts; mitigated by `retryOnFail` and `waitBetweenTries`.

#### Node: Split Out
- **Type / role:** Split Out; converts an array field into multiple items.
- **Config:**
  - `fieldToSplitOut: data`
  - `destinationFieldName: data`
  - `include: allOtherFields` (retains other top-level fields alongside each split item)
- **Inputs from:** Get 100 Tokens From CMC
- **Outputs to:** CLEAN CMC FIELDS
- **Failure modes / edge cases:**
  - If response doesn’t contain `data` array → node fails.
  - If `data` is empty → no downstream items.

---

### Block C — Normalize Minimal Token Fields & Batch Loop Control
**Overview:** Keeps only required token identifiers (id/name/symbol/slug) and then processes items in batches.

**Nodes involved:**
- CLEAN CMC FIELDS
- CMC SPLIT IN BATCHES
- Clean up Output
- Wait 1 Min

#### Node: CLEAN CMC FIELDS
- **Type / role:** Set; normalizes key fields to a standard structure.
- **Config choices (fields created):**
  - `SOURCE = "COINMARKET CAP"`
  - `TOKEN = {{$json.data.name}}`
  - `TICKER = {{$json.data.symbol}}`
  - `SLUG = {{$json.data.slug}}`
  - `data.id = {{$json.data.id}}` (stores the ID under a nested key name literally called `data.id`)
- **Inputs from:** Split Out
- **Outputs to:** CMC SPLIT IN BATCHES
- **Failure modes / edge cases:**
  - If `data.name/symbol/slug/id` missing → expressions evaluate to `null` and can break later nodes that require `data.id`.

#### Node: CMC SPLIT IN BATCHES
- **Type / role:** Split In Batches; controls batch processing + looping.
- **Config:** `batchSize: 10`
- **Inputs from:** CLEAN CMC FIELDS and later **loop-back** from Clean up Output.
- **Outputs to (two parallel branches):**
  1. `DATA` (data cleanup/validation branch)
  2. `Get Token Details From CMC` (enrichment branch)
- **Failure modes / edge cases:**
  - If loop-back is miswired, workflow can stop after one batch or loop incorrectly.
  - If downstream nodes return no items (e.g., filter-like code returning `null`), batch completion behavior can be confusing during debugging.

#### Node: Wait 1 Min
- **Type / role:** Wait; throttling between batch cycles.
- **Config:** 1 minute wait.
- **Inputs from:** CLEAN WEBSITE
- **Outputs to:** Clean up Output
- **Failure modes / edge cases:**
  - Long workflows accumulate execution time; ensure n8n execution time limits allow it (especially on hosted plans).

#### Node: Clean up Output
- **Type / role:** Set; creates run metadata and re-maps cleaned token fields for consistent output and loop-back.
- **Config choices (fields created):**
  - `TIMESTAMP = {{ new Date().toISOString() }}`
  - `BATCH NUMBER = {{ $runIndex + 1 }}`
  - `DELAY SECONDS = {{ $runIndex + 1 }}` (name suggests delay but equals runIndex+1, not actual seconds)
  - `TOKENS PROCESSED = {{ $items("Get Token Details From CMC").length }}`
  - `SOURCE = "COINMARKETCAP"`
  - `WEBSITE = {{ $('CLEAN WEBSITE').item.json.WEBSITE }}`
  - `TOKEN = {{ $('CLEAN WEBSITE').item.json.TOKEN }}`
  - `TICKER = {{ $('CLEAN WEBSITE').item.json.TICKER }}`
- **Inputs from:** Wait 1 Min
- **Outputs to:** CMC SPLIT IN BATCHES (loop continuation)
- **Failure modes / edge cases:**
  - Expressions like `$('CLEAN WEBSITE').item.json...` depend on item pairing. If item indexes diverge, you can get the wrong website/token or “item not found”.
  - `$items("Get Token Details From CMC").length` refers to all items from that node in the execution context; may not match “current batch” expectation.

---

### Block D — Enrich Each Token with CMC “Info” Endpoint
**Overview:** Uses token `id` to request official metadata (website/social) from CMC.

**Nodes involved:**
- Get Token Details From CMC
- EXTRACT WEBSITE
- CLEAN WEBSITE

#### Node: Get Token Details From CMC
- **Type / role:** HTTP Request; calls CMC info endpoint per token.
- **Endpoint:** `https://pro-api.coinmarketcap.com/v2/cryptocurrency/info?id={{ $json.data.id }}`
- **Auth pattern:** header `X-CMC_PRO_API_KEY` (empty in JSON; must be filled).
- **Inputs from:** CMC SPLIT IN BATCHES
- **Outputs to:** EXTRACT WEBSITE
- **Failure modes / edge cases:**
  - `{{$json.data.id}}` may be missing because earlier node stored `data.id` as a *flat key name*, not nested `data.id`. If the incoming item doesn’t actually have a nested object `data: { id }`, this URL becomes invalid.
  - Rate limits: multiple per-token calls can trigger 429; retries + wait help but may still fail.

#### Node: EXTRACT WEBSITE
- **Type / role:** Set; extracts token name/symbol/website and some social arrays from the CMC info response.
- **Key expressions:**
  - Token name/symbol pulled via:  
    `{{$json.data[Object.keys($json.data)[0]].name ...}}`  
    (because response data is keyed by token ID)
  - `WEBSITE` set as an **array** type but value expression returns a string:  
    `{{$json.data[Object.keys($json.data)[0]].urls.website[0].trim()}}`
  - `SOURCE = "COINMARKETCAP"`
  - Also sets:
    - `data["1"].urls.twitter = {{$json.data["1"].urls.twitter}}`
    - `data["1"].urls.facebook = {{$json.data["1"].urls.facebook}}`
    (hard-coded `"1"` key is likely incorrect unless the token ID is literally 1)
- **Inputs from:** Get Token Details From CMC
- **Outputs to:** CLEAN WEBSITE
- **Failure modes / edge cases:**
  - If `urls.website` is empty/missing → expression error unless ignored; `ignoreConversionErrors: true` reduces some failures but empty path can still yield `undefined`.
  - Hard-coded `data["1"]...` will fail or produce irrelevant data for most tokens.

#### Node: CLEAN WEBSITE
- **Type / role:** Set; normalizes website format and ensures HTTPS.
- **Logic (expression):**
  - Reads `$json["WEBSITE"]`
  - If it looks like a JSON array string (`"["...`) tries `JSON.parse`
  - If array → uses first element
  - Rewrites `http://` to `https://`
  - Returns empty string if missing
- **Inputs from:** EXTRACT WEBSITE
- **Outputs to:** Wait 1 Min
- **Failure modes / edge cases:**
  - If `WEBSITE` is a non-string object → `.replace()` can fail (mitigated partially by earlier normalization).
  - Some project websites intentionally don’t support HTTPS; forcing https can yield a “valid-looking” URL that fails later usage.

---

### Block E — Clean, Validate, and Assemble Final Token Records (skip invalid)
**Overview:** Produces a compact record `{token, ticker, source, website, timestamp}` and skips items without a usable website.

**Nodes involved:**
- DATA

#### Node: DATA
- **Type / role:** Code (Run Once for Each Item); canonical “record builder” + validator.
- **Key logic:**
  - `pickWebsite()` accepts array/string, parses stringified arrays, adds protocol if missing, forces https, strips stray quotes/brackets.
  - Builds output:
    - `token: j.TOKEN || j.name || null`
    - `ticker: (j.TICKER || j.symbol || '').toUpperCase()`
    - `source: j.SOURCE || 'COINGECKO'` (default says COINGECKO even though workflow is CMC)
    - `website: pickWebsite(j.WEBSITE || j.links?.homepage || j.homepage || null)`
    - `timestamp: new Date().toISOString()`
  - **Filter behavior:** `if (!out.website) return null;` → item is dropped.
- **Inputs from:** CMC SPLIT IN BATCHES
- **Outputs to:** CMC Token DATA
- **Failure modes / edge cases:**
  - If upstream never provides `WEBSITE`, nearly all items are dropped.
  - Default `source` mismatch (COINGECKO) can create inconsistent data if `SOURCE` missing upstream.

---

### Block F — Storage Preparation & Write to NocoDB (Example)
**Overview:** Collects cleaned items, optionally batches them for DB insertion, and creates rows in NocoDB.

**Nodes involved:**
- CMC Token DATA
- ADD TO CMC
- ADD TO CMC Sheet
- No Operation, do nothing3

#### Node: CMC Token DATA
- **Type / role:** Code; aggregates items from `DATA` node.
- **Behavior:**
  - Reads all items from node `"DATA"` via `$items("DATA").map(item => item.json)`
  - Returns each row with an added `index`
- **Inputs from:** DATA
- **Outputs to:** ADD TO CMC
- **Failure modes / edge cases:**
  - If `DATA` returns no items (all filtered), output is empty → storage step does nothing.
  - This node is “global-ish”: it re-reads `$items("DATA")`, not just current item context.

#### Node: ADD TO CMC
- **Type / role:** Split In Batches; controls storage batching / iteration.
- **Config:** default options (batch size not explicitly set; n8n defaults apply).
- **Inputs from:** CMC Token DATA and loop-back from ADD TO CMC Sheet.
- **Outputs to:**
  1. No Operation, do nothing3
  2. ADD TO CMC Sheet
- **Failure modes / edge cases:**
  - If default batch size is 1, inserts one row at a time (slower but safer).
  - If loop-back is misconfigured, could create an infinite loop or stop early.

#### Node: ADD TO CMC Sheet
- **Type / role:** NocoDB node; creates a record.
- **Credentials:** `nocoDbApiToken` (must be configured).
- **Config choices:**
  - Operation: `create`
  - `projectId` and `table` are empty in JSON → must be set.
  - Field mapping:
    - `FIRST_NAME = {{$json.token}}` (odd naming; likely table schema mismatch)
    - `WEBSITE = {{ $('ADD TO CMC').item.json.website }}`
    - `TICKER = {{$json.ticker}}`
    - `SOURCE = "COINMARKETCAP"`
    - `TOKEN = {{$json.token}}`
    - `DATE ADDED = {{ new Date().toISOString().split('T')[0] }}`
  - `executeOnce: true` is enabled (important): node runs once for the first item only.
- **Inputs from:** ADD TO CMC
- **Outputs to:** ADD TO CMC (loop-back)
- **Failure modes / edge cases:**
  - If `executeOnce` remains enabled, only one record is inserted per execution (likely unintended).
  - Empty `projectId` / `table` → node fails.
  - Schema mismatch (`FIRST_NAME`) → create fails or stores in wrong column.
  - `$('ADD TO CMC').item.json.website` can break if item pairing differs; simpler is `{{$json.website}}`.

#### Node: No Operation, do nothing3
- **Type / role:** NoOp; terminator branch placeholder.
- **Inputs from:** ADD TO CMC
- **Outputs:** none
- **Failure modes / edge cases:** none (used to end a branch cleanly).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every 3 Days At 10AM | Schedule Trigger | Periodic workflow entry | — | Generate Random Page | ## 📊 COINMARKET CAP TOKEN DISCOVERY (FREE API) … Setup steps… |
| Generate Random Page | Code | Randomize listing query params | Every 3 Days At 10AM | Get 100 Tokens From CMC | ## STEP 1: Fetch Tokens Selects random pages… |
| Get 100 Tokens From CMC | HTTP Request | Fetch token listings | Generate Random Page | Split Out | ## STEP 1: Fetch Tokens Selects random pages… |
| Split Out | Split Out | Split listings array into items | Get 100 Tokens From CMC | CLEAN CMC FIELDS | ## STEP 2: Normalize Data Breaks API responses… |
| CLEAN CMC FIELDS | Set | Keep minimal token identifiers | Split Out | CMC SPLIT IN BATCHES | ## STEP 2: Normalize Data Breaks API responses… |
| CMC SPLIT IN BATCHES | Split In Batches | Batch controller + loop | CLEAN CMC FIELDS, Clean up Output | DATA; Get Token Details From CMC | ## STEP 3: Enrich Tokens Fetches additional details… |
| Get Token Details From CMC | HTTP Request | Fetch token info (website/social) | CMC SPLIT IN BATCHES | EXTRACT WEBSITE | ## STEP 3: Enrich Tokens Fetches additional details… |
| EXTRACT WEBSITE | Set | Extract website/name/symbol from info payload | Get Token Details From CMC | CLEAN WEBSITE | ## STEP 4: Clean Websites Normalizes website URLs… |
| CLEAN WEBSITE | Set | Normalize website, force https | EXTRACT WEBSITE | Wait 1 Min | ## STEP 4: Clean Websites Normalizes website URLs… |
| Wait 1 Min | Wait | Throttle between cycles | CLEAN WEBSITE | Clean up Output | ## STEP 5: Store Results Saves cleaned token data… |
| Clean up Output | Set | Format output + loop-back | Wait 1 Min | CMC SPLIT IN BATCHES | ## STEP 6: Prepare Clean Output This step formats… |
| DATA | Code | Build final record; skip invalid websites | CMC SPLIT IN BATCHES | CMC Token DATA | ## STEP 6: Prepare Clean Output This step formats… |
| CMC Token DATA | Code | Collect rows and add index | DATA | ADD TO CMC | ## STEP 7: Save Token Data This step stores… |
| ADD TO CMC | Split In Batches | Storage batching loop | CMC Token DATA; ADD TO CMC Sheet | No Operation, do nothing3; ADD TO CMC Sheet | ## STEP 7: Save Token Data This step stores… |
| ADD TO CMC Sheet | NocoDB | Insert record into NocoDB | ADD TO CMC | ADD TO CMC | ## STEP 5: Store Results Saves cleaned token data… |
| No Operation, do nothing3 | NoOp | End branch placeholder | ADD TO CMC | — | ## STEP 7: Save Token Data This step stores… |
| Sticky Note | Sticky Note | Documentation | — | — | ## 📊 COINMARKET CAP TOKEN DISCOVERY (FREE API) … Setup steps… |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## STEP 1: Fetch Tokens … |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## STEP 2: Normalize Data … |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## STEP 3: Enrich Tokens … |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## STEP 4: Clean Websites … |
| Sticky Note5 | Sticky Note | Documentation | — | — | ## STEP 5: Store Results … |
| Sticky Note6 | Sticky Note | Documentation | — | — | ## STEP 6: Prepare Clean Output … |
| Sticky Note7 | Sticky Note | Documentation | — | — | ## STEP 7: Save Token Data … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: “Get Tokens From CMC Using Their Free API”
- Set timezone to `Africa/Lagos` (Workflow Settings).

2) **Add Trigger**
- Node: **Schedule Trigger**
- Name: “Every 3 Days At 10AM”
- Configure: interval every **3 days**, trigger at **10:00**.

3) **Add random query generator**
- Node: **Code**
- Name: “Generate Random Page”
- Paste the provided logic conceptually:
  - Create `params = { start, limit, sort, sort_dir, page }`
  - Output one item with `json.params = params` (and optional `fullURL`).

4) **Fetch listings from CoinMarketCap**
- Node: **HTTP Request**
- Name: “Get 100 Tokens From CMC”
- Method: GET
- URL: `https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest`
- Authentication: none (CMC uses API key header)
- Headers:
  - `X-CMC_PRO_API_KEY`: **your CMC API key**
- **Important correction (recommended):** Put `start`, `limit`, `sort`, `sort_dir` as **Query Parameters** (not headers):
  - `start = {{$json.params.start}}`
  - `limit = {{$json.params.limit}}`
  - `sort = {{$json.params.sort}}`
  - `sort_dir = {{$json.params.sort_dir}}`
- (Optional) enable retry on fail / backoff.
- Connect: Schedule Trigger → Generate Random Page → Get 100 Tokens From CMC

5) **Split listing results into individual tokens**
- Node: **Split Out**
- Name: “Split Out”
- Field to split: `data`
- Destination field name: `data`
- Include: “All other fields”
- Connect: Get 100 Tokens From CMC → Split Out

6) **Normalize minimal fields**
- Node: **Set**
- Name: “CLEAN CMC FIELDS”
- Add fields:
  - `SOURCE` (string): `COINMARKET CAP`
  - `TOKEN` (string): `{{$json.data.name}}`
  - `TICKER` (string): `{{$json.data.symbol}}`
  - `SLUG` (string): `{{$json.data.slug}}`
  - **Recommended fix:** create a real nested field `data` (object) with `id`, or simply create `id`:
    - `id` (number): `{{$json.data.id}}`
- Connect: Split Out → CLEAN CMC FIELDS

7) **Batch controller**
- Node: **Split In Batches**
- Name: “CMC SPLIT IN BATCHES”
- Batch size: 10
- Connect: CLEAN CMC FIELDS → CMC SPLIT IN BATCHES

8) **Enrichment call (token info)**
- Node: **HTTP Request**
- Name: “Get Token Details From CMC”
- URL: `https://pro-api.coinmarketcap.com/v2/cryptocurrency/info`
- Query parameter: `id = {{$json.id}}` (or `{{$json.data.id}}` if you truly created nested `data.id`)
- Header: `X-CMC_PRO_API_KEY = your key`
- Connect: CMC SPLIT IN BATCHES → Get Token Details From CMC

9) **Extract website from the info payload**
- Node: **Set**
- Name: “EXTRACT WEBSITE”
- Extract:
  - `TOKEN = {{$json.data[Object.keys($json.data)[0]].name}}`
  - `TICKER = {{$json.data[Object.keys($json.data)[0]].symbol}}`
  - `WEBSITE = {{$json.data[Object.keys($json.data)[0]].urls.website[0]}}`
  - `SOURCE = COINMARKETCAP`
- (Optional) twitter/facebook:
  - Use the same dynamic key approach, not `data["1"]...`.
- Connect: Get Token Details From CMC → EXTRACT WEBSITE

10) **Clean website**
- Node: **Set**
- Name: “CLEAN WEBSITE”
- Set `WEBSITE` via expression that:
  - handles array/string/stringified array
  - ensures protocol and forces https
- Keep other fields.
- Connect: EXTRACT WEBSITE → CLEAN WEBSITE

11) **Throttle**
- Node: **Wait**
- Name: “Wait 1 Min”
- Amount: 1 minute
- Connect: CLEAN WEBSITE → Wait 1 Min

12) **Prepare clean output & loop**
- Node: **Set**
- Name: “Clean up Output”
- Create fields (or simplify to just pass-through the cleaned token):
  - `TIMESTAMP`, `SOURCE`, `WEBSITE`, `TOKEN`, `TICKER`, etc.
- Connect: Wait 1 Min → Clean up Output
- Connect loop-back: Clean up Output → CMC SPLIT IN BATCHES (so it continues next batch)

13) **Create canonical token record and filter invalid**
- Node: **Code** (Run once for each item)
- Name: “DATA”
- Implement:
  - `website = pickWebsite(...)`
  - If no website → `return null`
  - Return `{ token, ticker, source, website, timestamp }`
- Connect: CMC SPLIT IN BATCHES → DATA  
  (In the original workflow, CMC SPLIT IN BATCHES goes to DATA in parallel with enrichment; you may instead feed DATA from CLEAN WEBSITE to ensure it has a website.)

14) **Aggregate token records**
- Node: **Code**
- Name: “CMC Token DATA”
- Map `$items("DATA")` into indexed rows (or skip this and store directly).
- Connect: DATA → CMC Token DATA

15) **Storage batching**
- Node: **Split In Batches**
- Name: “ADD TO CMC”
- Default batch size or set one (e.g., 10/50).
- Connect: CMC Token DATA → ADD TO CMC

16) **NocoDB insert**
- Node: **NocoDB**
- Name: “ADD TO CMC Sheet”
- Credentials: configure **NocoDB API Token** credential.
- Set **Project** and **Table**.
- Operation: **Create**
- Map fields to your schema (recommended minimal):
  - TOKEN ← `{{$json.token}}`
  - TICKER ← `{{$json.ticker}}`
  - WEBSITE ← `{{$json.website}}`
  - SOURCE ← `{{$json.source}}`
  - DATE ADDED ← `{{ new Date().toISOString().split('T')[0] }}`
- **Disable** `Execute Once` unless you truly want only one inserted row.
- Connect: ADD TO CMC → ADD TO CMC Sheet
- Loop-back for batching: ADD TO CMC Sheet → ADD TO CMC

17) **Optional terminator**
- Node: **NoOp**
- Name: “No Operation, do nothing3”
- Connect: ADD TO CMC (first output) → NoOp

18) **Credentials & activation**
- Add your CMC API key to both HTTP Request nodes.
- Add NocoDB API token credential and configure project/table.
- Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow periodically fetches cryptocurrency listings from CoinMarketCap using the free API… runs in batches with delays to stay within API limits.” | Sticky note: COINMARKET CAP TOKEN DISCOVERY (FREE API) |
| Setup steps: (1) Create free CMC API key (2) Add key to HTTP nodes (3) Connect preferred storage (4) Adjust schedule (5) Activate | Sticky note: COINMARKET CAP TOKEN DISCOVERY (FREE API) |

