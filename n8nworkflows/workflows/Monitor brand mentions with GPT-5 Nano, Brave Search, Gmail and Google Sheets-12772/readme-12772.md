Monitor brand mentions with GPT-5 Nano, Brave Search, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-brand-mentions-with-gpt-5-nano--brave-search--gmail-and-google-sheets-12772


# Monitor brand mentions with GPT-5 Nano, Brave Search, Gmail and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor daily brand mentions on the web using **Brave Search** (past 24h), filter out unwanted domains, verify relevance using **GPT‑5 Nano**, deduplicate against a **Google Sheets** history log, then **email** new relevant mentions via **Gmail** and append them to the sheet.

**Target use cases:**
- Daily PR/brand monitoring for an artist/band (or any brand) with disambiguation needs (same-name noise).
- Lightweight monitoring without crawling: relies on Brave Search snippets + AI verification.

### 1.1 Daily Trigger & Brand Configuration
- Runs once per day, sets the monitored **keyword** and **description**, then generates multiple search queries (“dorky”).

### 1.2 Load Historical Mentions (Google Sheets)
- Reads existing logged URLs from a Google Sheet so the workflow can detect what’s new.

### 1.3 Search Loop (per dork) + Domain Filtering
- Iterates dork queries sequentially, searches Brave (past day), splits results, extracts normalized fields, filters out banned domains.

### 1.4 Cleanup + AI Relevance Verification
- Ensures URL exists, removes duplicates across dorks, sends each candidate to GPT‑5 Nano to classify relevance (strict), merges AI output with original metadata.

### 1.5 New Mention Detection, Email + Append to Sheet
- Matches verified results against historical sheet URLs, sends an HTML email if new items exist, then appends sent rows to the sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Daily Trigger & Brand Setup
**Overview:** Starts the workflow daily at a fixed hour, sets the brand keyword/description, and builds a list of search queries plus filtering rules.

**Nodes involved:**
- **Every Day** (Schedule Trigger)
- **Set keyword and description** (Set)
- **Set Config** (Code)

#### Node: Every Day
- **Type/role:** `Schedule Trigger` — entry point, runs daily.
- **Config:** Triggers at **06:00** (local instance time).
- **Outputs:** Two parallel branches:
  - To **Set keyword and description**
  - To **Check & Log to Sheet** (loads historical sheet data)
- **Edge cases:**
  - Timezone mismatch (instance timezone vs expected).
  - If execution overlaps (long runs), schedule concurrency depends on n8n settings.

#### Node: Set keyword and description
- **Type/role:** `Set` — static configuration values.
- **Config:** Sets:
  - `keyword`: `"Goofy Cow"`
  - `description`: `"Goofy Cow is a czech music band. "`
- **Output:** To **Set Config**
- **Edge cases:** If keyword is empty/typo, all searches and AI verification degrade.

#### Node: Set Config
- **Type/role:** `Code` — builds search query variations (“dorky”) and banned domain list.
- **Config choices (interpreted):**
  - Reads `keyword` and `description` from incoming item.
  - Creates an array `dorky` of multiple query strings (music/concert/festival/album/band plus OR variant removing spaces).
  - Defines `bannedDomains` list (Spotify/Apple Music/Bandcamp/Social platforms, etc.).
  - Sets `timeRange: 'pd'` (past day) for Brave freshness.
  - **Returns one item per dork**: each item contains `currentDork`, `keyword`, `description`, `bannedDomains`, `timeRange`.
- **Output:** To **Split Dorky**
- **Edge cases/failures:**
  - If keyword contains quotes/special chars, query string may become awkward (Brave usually tolerates, but could reduce results).
  - Large dork arrays increase runtime/Brave quota usage.

---

### Block 2 — Load Historical Mentions (Google Sheets)
**Overview:** Loads prior logged results from Google Sheets so new URLs can be detected later via merge.

**Nodes involved:**
- **Check & Log to Sheet** (Google Sheets)
- **Merge** (Merge) *(also used later for newness detection)*

#### Node: Check & Log to Sheet
- **Type/role:** `Google Sheets` — expected to **read** existing sheet rows (node name implies logging, but it is used as “history input”).
- **Config:** `documentId` and `sheetName` are placeholders (template); operation is not explicitly shown in parameters, so it relies on default or was omitted in template export.
- **Credentials:** Google Sheets OAuth2 (`TEMPLATE` placeholder).
- **Output:** To **Merge** (input 1)
- **Edge cases/failures:**
  - Missing/invalid OAuth credentials.
  - Document/sheet not selected.
  - If the sheet doesn’t contain a URL column matching the merge key (see Merge node), newness detection will fail.

#### Node: Merge
- **Type/role:** `Merge` — combines “historical sheet rows” with “current verified results” to identify new URLs.
- **Config choices:**
  - Mode: **combine**, advanced enabled.
  - `joinMode: keepNonMatches`
  - Merge by fields: **Sheet field `URL`** ↔ **Result field `url`**
  - `outputDataFrom: input2` (i.e., outputs the “current results” side when merging)
- **Inputs:**
  - **Input 1:** from **Check & Log to Sheet** (historical)
  - **Input 2:** from **AnyRelevantResults?** (verified relevant results)
- **Output:** To **NewURL?**
- **Edge cases/failures:**
  - If the sheet column is not exactly named `URL` (case sensitive), no matches occur → everything may look “new”.
  - If sheet contains duplicates, merge logic can behave unexpectedly (multiple matches).

---

### Block 3 — Search Loop (Split Dorky → Brave → Extract → Filter banned domains)
**Overview:** Iterates each dork query sequentially, runs Brave Search, extracts normalized fields per result, filters out banned domains, then loops until all dorks are processed.

**Nodes involved:**
- **Split Dorky** (Split In Batches)
- **5s** (Wait)
- **Loop Setup** (Set)
- **Brave Search** (Brave Search community node)
- **HasResult?** (IF)
- **Split Results** (Split Out)
- **Extract Data** (Set)
- **FilterBannedDomains** (Filter)
- **Continue Loop** (NoOp)

#### Node: Split Dorky
- **Type/role:** `Split In Batches` — controls iteration over dork items.
- **Config:** Default batch behavior (batch size not explicitly set in JSON; n8n default is typically 1).
- **Outputs:**
  - Main output 1 → **URL_Is_NOT_empty** (starts downstream pipeline)
  - Main output 2 → **5s** (used to pace requests, then returns to loop)
- **Edge cases:**
  - Misconfiguration of batch size can overload Brave API or reduce throughput.
  - If it outputs nothing, downstream sees no items.

#### Node: 5s
- **Type/role:** `Wait` — throttling between dork executions.
- **Config:** No explicit duration shown; node name suggests 5 seconds but parameters are empty (so it may be default/manual resume depending on n8n version). In many n8n setups, Wait must be configured (time or webhook resume).
- **Output:** To **Loop Setup**
- **Edge cases/failures:**
  - If not configured for time-based wait, execution may pause indefinitely.
  - If configured, ensure it is actually 5 seconds to avoid rate limit hits.

#### Node: Loop Setup
- **Type/role:** `Set` — carries loop item fields forward consistently.
- **Config:** Copies from current item:
  - `currentDork`, `keyword`, `bannedDomains`, `timeRange`
- **Output:** To **Brave Search**
- **Edge cases:** Expression references `$json.*` rely on upstream items containing those keys.

#### Node: Brave Search
- **Type/role:** `@brave/n8n-nodes-brave-search.braveSearch` — executes web search.
- **Config:**
  - `query`: `={{ $json.currentDork }}`
  - `count: 3` (top 3 results per dork)
  - `freshness: "pd"` (past day)
- **Credentials:** Brave Search API (`TEMPLATE` placeholder).
- **Output:** To **HasResult?**
- **Edge cases/failures:**
  - API key missing/invalid, quota exceeded.
  - Response shape changes (affects `$json.web.results` access later).

#### Node: HasResult?
- **Type/role:** `IF` — checks if Brave returned any results.
- **Condition:** `$json.web.results.length > 0`
- **True output:** to **Split Results**
- **False output:** to **Continue Loop**
- **Edge cases:** If `web.results` is missing, expression fails unless n8n tolerates undefined; may require safe checks.

#### Node: Split Results
- **Type/role:** `Split Out` — turns array `web.results` into one item per result.
- **Config:** `fieldToSplitOut: "web.results"`
- **Output:** To **Extract Data**
- **Edge cases:** If the field is not an array, node errors.

#### Node: Extract Data
- **Type/role:** `Set` — normalizes Brave result fields for later steps.
- **Config (key mappings):**
  - `url` ← `$json.url`
  - `title` ← `$json.title`
  - `source` ← `$json.meta_url.hostname`
  - `published` ← `$json.page_age`
  - `snippet` ← `$json.description`
  - `domain` ← `$json.profile.long_name`
  - `bannedDomains` ← `$('Loop Setup').item.json.bannedDomains` (imports loop-level list)
- **Output:** To **FilterBannedDomains**
- **Edge cases:**
  - Some Brave results may lack `profile.long_name` or `meta_url.hostname`.
  - If those keys are undefined, later filtering may behave incorrectly.

#### Node: FilterBannedDomains
- **Type/role:** `Filter` — excludes results whose domain matches banned domains list.
- **Condition:** `={{ !$json.bannedDomains.some(d => $json.domain.includes(d)) }}`
- **alwaysOutputData:** true (even if filtered out, it can pass empty/no-match behavior depending on n8n filter semantics).
- **Output:** To **Continue Loop**
- **Edge cases:**
  - Uses `$json.domain.includes(d)`; but `domain` is set from `profile.long_name`, which may not be the actual hostname (could be a display name). This can cause false negatives/positives.
  - If `domain` is undefined, `.includes` throws; consider guarding with `($json.domain || '')`.

#### Node: Continue Loop
- **Type/role:** `NoOp` — loop connector.
- **Output:** back to **Split Dorky** to fetch next batch.
- **Edge cases:** None, but it defines control flow.

---

### Block 4 — Data Cleanup + AI Relevance Verification
**Overview:** Consolidates results from all dorks, removes empty URLs and duplicates, then uses GPT‑5 Nano to determine whether each result truly refers to the intended brand.

**Nodes involved:**
- **URL_Is_NOT_empty** (Filter)
- **Remove Duplicates** (Remove Duplicates)
- **RelevancyVerification** (Split In Batches)
- **Message a model** (OpenAI via LangChain node)
- **MergeOutput&Input** (Code)
- **FilterRelevantOnly** (Filter)

#### Node: URL_Is_NOT_empty
- **Type/role:** `Filter` — rejects items without a URL.
- **Condition:** `$json.url` is not empty.
- **Output:** To **Remove Duplicates**
- **Edge cases:** If URL is present but malformed, it still passes.

#### Node: Remove Duplicates
- **Type/role:** `Remove Duplicates` — deduplicates across multiple dorks.
- **Config:** Compare selected fields → `url`
- **Output:** To **RelevancyVerification**
- **Edge cases:** URL variants (tracking params, http/https) will not dedupe unless normalized.

#### Node: RelevancyVerification
- **Type/role:** `Split In Batches` — iterates results for AI verification.
- **Config:** Default batch behavior (not explicitly set).
- **Outputs:**
  - Output 1 → **FilterRelevantOnly** (passes items forward in parallel branch)
  - Output 2 → **Message a model** (sends items to AI)
- **Important note:** This wiring is unusual: typically, you’d call the model first and then filter based on model output. Here, **FilterRelevantOnly** receives items before the model result exists, while **Message a model** runs separately. The intended logic is likely “batch loop where model output is merged back,” but the current graph risks ordering/data mismatch.

#### Node: Message a model
- **Type/role:** `@n8n/n8n-nodes-langchain.openAi` — calls OpenAI model for classification.
- **Model:** `gpt-5-nano`
- **Output format:** JSON object (textFormat: `json_object`, verbosity low)
- **Prompts:**
  - **System:** strict brand mention verification; return only JSON `{isRelevant, confidence, reason}`.
  - **User content:** includes Title/Snippet/URL and brand Keyword/Description.
- **Output:** To **MergeOutput&Input**
- **Credentials:** OpenAI API (`TEMPLATE` placeholder)
- **Edge cases/failures:**
  - Model may return invalid JSON despite instruction; parser must handle.
  - Rate limits/timeouts on OpenAI.
  - Prompt relies on `$('Set Config').item.json.*` being available (it will be, but only if execution context has that item).

#### Node: MergeOutput&Input
- **Type/role:** `Code` — merges AI classification with original extracted data.
- **Config choices:**
  - Attempts to parse AI result from nested structure: `aiData.output[0].content[0].text`
  - On failure: falls back to `{isRelevant:false, confidence:'low', reason:'Failed to parse AI response'}`
  - Pulls original data from `$('Extract Data').item.json` and outputs combined object:
    - url, title, source, domain, published, snippet
    - isRelevant, verificationConfidence, verificationReason
- **Output:** To **RelevancyVerification** (feeds back into batch loop)
- **Edge cases/failures:**
  - **High risk of item mismatch:** `$('Extract Data').item.json` references a specific item, but in loops/batches you can accidentally read the wrong item if multiple items are in-flight.
  - AI node output structure may differ; parsing path may be wrong depending on n8n node version and OpenAI response structure.

#### Node: FilterRelevantOnly
- **Type/role:** `Filter` — keeps only relevant mentions.
- **Condition:** `{{ $json.isRelevant }}` is true.
- **alwaysOutputData:** true
- **Output:** To **AnyRelevantResults?**
- **Edge cases:** If `isRelevant` is missing (because model merge didn’t happen), everything may be filtered out or error depending on strict validation.

---

### Block 5 — New Mention Detection, Email, Append to Sheet
**Overview:** Ensures there are any relevant results, compares them to historical URLs, sends an email for new URLs, and logs what was sent.

**Nodes involved:**
- **AnyRelevantResults?** (IF)
- **No Relevant Results** (NoOp)
- **NewURL?** (IF)
- **No New Results** (NoOp)
- **Format Email** (Code)
- **Send Email** (Gmail)
- **InsertSentRows** (Google Sheets)
- **Merge** (already described in Block 2)

#### Node: AnyRelevantResults?
- **Type/role:** `IF` — checks whether upstream data is non-empty.
- **Condition:** `Object.keys($json).length > 0`
- **True output:** To **Merge** (input 2)
- **False output:** To **No Relevant Results**
- **Edge cases:** This is a weak “non-empty” test; better to check item count. With `alwaysOutputData`, you may still receive placeholder/empty items.

#### Node: No Relevant Results
- **Type/role:** `NoOp` — end branch when nothing relevant.

#### Node: NewURL?
- **Type/role:** `IF` — determines if merge produced a “new URL” record.
- **Condition:** `Object.keys($json).length > 0`
- **True output:** To **Format Email**
- **False output:** To **No New Results**
- **Edge cases:** Again, non-empty object check may not reliably represent “newness.” Correctness depends heavily on Merge output behavior.

#### Node: No New Results
- **Type/role:** `NoOp` — end branch when nothing new.

#### Node: Format Email
- **Type/role:** `Code` — builds HTML email body and subject.
- **Config:**
  - Collects all input items (`$input.all()`).
  - If none, returns `[]` (prevents sending).
  - Constructs HTML with per-item blocks including title/source/domain/published/url/snippet.
  - Subject uses keyword from `$('Set Config').item.json.keyword`.
  - Output JSON: `{ subject, body, count }`
- **Output:** To **Send Email**
- **Edge cases:**
  - Uses special characters and inline HTML; email clients may render differently.
  - References `Set Config` item; ensure available in execution context.
  - Language mix: heading includes “nová zmínka…”, while body is English.

#### Node: Send Email
- **Type/role:** `Gmail` — sends email notification.
- **Config:**
  - To: `user@example.com` (placeholder)
  - Subject: `={{$json.subject}}`
  - Message body: `={{$json.body}}`
- **Credentials:** Gmail OAuth2 (`TEMPLATE` placeholder)
- **Output:** To **InsertSentRows**
- **Edge cases/failures:**
  - OAuth scopes/refresh failures.
  - Gmail sending limits.
  - If message is HTML, ensure node is configured to send HTML (Gmail node typically sends as raw; confirm “HTML mode” behavior in your n8n version).

#### Node: InsertSentRows
- **Type/role:** `Google Sheets` — appends newly sent mentions to tracking sheet.
- **Config:** Operation `append`; documentId/sheetName placeholders.
- **Credentials:** Google Sheets OAuth2 (`TEMPLATE`)
- **Input:** from **Send Email**
- **Edge cases:**
  - Requires columns mapping; template omits explicit column mapping—must configure to match your sheet structure (URL/title/source/etc.).
  - If append fails, you may resend the same mentions next run.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Day | Schedule Trigger | Daily entry point (06:00) | — | Set keyword and description; Check & Log to Sheet | # How it works / Trigger fires once per day… Configuration Requirements… |
| Set keyword and description | Set | Define keyword + description | Every Day | Set Config | ## 1. Setup your brand : … Configuration Fields… |
| Set Config | Code | Build dork queries + banned domains list | Set keyword and description | Split Dorky | ## 1. Setup your brand : … Configuration Fields… |
| Check & Log to Sheet | Google Sheets | Load historical mentions from sheet | Every Day | Merge | ## 2. Load File with previous results … |
| Split Dorky | Split In Batches | Iterate each dork query sequentially | Set Config; Continue Loop | URL_Is_NOT_empty; 5s | ##  3. Split Dorky Section … |
| 5s | Wait | Throttle between dorks | Split Dorky | Loop Setup | ##  3. Split Dorky Section … |
| Loop Setup | Set | Carry loop variables (currentDork, bannedDomains, etc.) | 5s | Brave Search | ##  3. Split Dorky Section … |
| Brave Search | Brave Search | Search web (past day) for each dork | Loop Setup | HasResult? | ##  3. Split Dorky Section … |
| HasResult? | IF | Check Brave returned results | Brave Search | Split Results (true); Continue Loop (false) | ##  3. Split Dorky Section … |
| Split Results | Split Out | Split `web.results` into items | HasResult? | Extract Data | ##  3. Split Dorky Section … |
| Extract Data | Set | Normalize result fields | Split Results | FilterBannedDomains | ##  3. Split Dorky Section … |
| FilterBannedDomains | Filter | Exclude unwanted domains | Extract Data | Continue Loop | ##  3. Split Dorky Section … |
| Continue Loop | NoOp | Loop connector back to Split Dorky | HasResult? (false); FilterBannedDomains | Split Dorky | ##  3. Split Dorky Section … |
| URL_Is_NOT_empty | Filter | Remove items without URL | Split Dorky | Remove Duplicates | ## 4. Verification … |
| Remove Duplicates | Remove Duplicates | Deduplicate by URL | URL_Is_NOT_empty | RelevancyVerification | ## 4. Verification … |
| RelevancyVerification | Split In Batches | Batch/loop AI verification | Remove Duplicates; MergeOutput&Input | FilterRelevantOnly; Message a model | ## 4. Verification … |
| Message a model | OpenAI (LangChain) | GPT‑5 Nano relevance classification | RelevancyVerification | MergeOutput&Input | ## Prompt Note … |
| MergeOutput&Input | Code | Merge AI JSON with original fields | Message a model | RelevancyVerification | ## 4. Verification … |
| FilterRelevantOnly | Filter | Keep only relevant mentions | RelevancyVerification | AnyRelevantResults? | ## 4. Verification … |
| AnyRelevantResults? | IF | Gate: proceed only if relevant results exist | FilterRelevantOnly | Merge (true); No Relevant Results (false) | ## 5. Merge & Send … |
| No Relevant Results | NoOp | End branch: nothing relevant | AnyRelevantResults? (false) | — | ## 5. Merge & Send … |
| Merge | Merge | Compare results vs sheet history (URL matching) | Check & Log to Sheet; AnyRelevantResults? | NewURL? | ## 5. Merge & Send … |
| NewURL? | IF | Gate: proceed only if new items exist | Merge | Format Email (true); No New Results (false) | ## 5. Merge & Send … |
| No New Results | NoOp | End branch: nothing new | NewURL? (false) | — | ## 5. Merge & Send … |
| Format Email | Code | Build HTML email for all new mentions | NewURL? (true) | Send Email | ## 5. Merge & Send … |
| Send Email | Gmail | Send notification email | Format Email | InsertSentRows | ## 5. Merge & Send … |
| InsertSentRows | Google Sheets | Append sent mentions to sheet | Send Email | — | ## 5. Merge & Send … |
| Sticky Note | Sticky Note | Comment block: how it works + requirements | — | — | # How it works … Configuration Requirements… |
| Sticky Note1 | Sticky Note | Comment block: setup brand | — | — | ## 1. Setup your brand : … |
| Sticky Note2 | Sticky Note | Comment block: verification | — | — | ## 4. Verification … |
| Sticky Note3 | Sticky Note | Comment block: load previous results | — | — | ## 2. Load File with previous results … |
| Sticky Note4 | Sticky Note | Comment block: split dorky section | — | — | ##  3. Split Dorky Section … |
| Sticky Note5 | Sticky Note | Comment block: prompt note | — | — | ## Prompt Note … |
| Sticky Note6 | Sticky Note | Comment block: merge & send | — | — | ## 5. Merge & Send … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1. Add **Schedule Trigger** node named **Every Day**.
   2. Set rule to run **daily at 06:00**.

2. **Add brand configuration**
   1. Add a **Set** node named **Set keyword and description**.
   2. Add fields:
      - `keyword` (string) e.g., `Goofy Cow`
      - `description` (string) e.g., `Goofy Cow is a czech music band.`
   3. Connect **Every Day → Set keyword and description**.

3. **Generate dorks + banned domains**
   1. Add a **Code** node named **Set Config**.
   2. Paste logic that:
      - reads `keyword`, `description`
      - builds `dorky[]` (query variations)
      - defines `bannedDomains[]`
      - returns `dorky.map(...)` producing items with `currentDork`, `keyword`, `description`, `bannedDomains`, `timeRange:'pd'`
   3. Connect **Set keyword and description → Set Config**.

4. **Load historical URLs from Google Sheets**
   1. Add **Google Sheets** node named **Check & Log to Sheet**.
   2. Configure Google Sheets OAuth2 credentials.
   3. Select **Document** and **Sheet** that contain your historical mentions.
   4. Configure operation to **Read/Get Many rows** (so it outputs rows including a column named `URL`).
   5. Connect **Every Day → Check & Log to Sheet**.

5. **Create dork loop**
   1. Add **Split In Batches** node named **Split Dorky** (batch size 1 recommended).
   2. Connect **Set Config → Split Dorky**.

6. **Throttle (optional but recommended)**
   1. Add **Wait** node named **5s**.
   2. Configure it to wait **5 seconds** (time-based wait).
   3. Connect **Split Dorky (second output) → 5s**.

7. **Loop setup + Brave Search**
   1. Add **Set** node **Loop Setup** that maps:
      - `currentDork` = `{{$json.currentDork}}`
      - `keyword` = `{{$json.keyword}}`
      - `bannedDomains` = `{{$json.bannedDomains}}`
      - `timeRange` = `{{$json.timeRange}}`
   2. Connect **5s → Loop Setup**.
   3. Add **Brave Search** node named **Brave Search**.
      - Credential: Brave Search API key
      - Query: `{{$json.currentDork}}`
      - Count: `3`
      - Freshness: `pd`
   4. Connect **Loop Setup → Brave Search**.

8. **Handle no-result / split results**
   1. Add **IF** node **HasResult?** with condition:
      - `{{$json.web.results.length}} > 0`
   2. Connect **Brave Search → HasResult?**
   3. Add **Split Out** node **Split Results** with field `web.results`.
   4. Connect **HasResult? (true) → Split Results**
   5. Add **NoOp** node **Continue Loop**.
   6. Connect **HasResult? (false) → Continue Loop**

9. **Extract fields and filter banned domains**
   1. Add **Set** node **Extract Data** mapping:
      - url/title/snippet/etc. from Brave result
      - `bannedDomains` from `Loop Setup` (`$('Loop Setup').item.json.bannedDomains`)
   2. Connect **Split Results → Extract Data**
   3. Add **Filter** node **FilterBannedDomains** with condition:
      - `{{ !$json.bannedDomains.some(d => $json.domain.includes(d)) }}`
   4. Connect **Extract Data → FilterBannedDomains**
   5. Connect **FilterBannedDomains → Continue Loop**
   6. Connect **Continue Loop → Split Dorky** (to iterate).

10. **URL present + dedupe**
   1. Connect **Split Dorky (first output) → URL_Is_NOT_empty** (Filter: url not empty).
   2. Add **Remove Duplicates** node comparing field `url`.
   3. Connect **URL_Is_NOT_empty → Remove Duplicates**.

11. **AI verification**
   1. Add **Split In Batches** node **RelevancyVerification** (batch size 1 recommended).
   2. Connect **Remove Duplicates → RelevancyVerification**.
   3. Add **OpenAI (LangChain)** node **Message a model**:
      - Credentials: OpenAI API
      - Model: `gpt-5-nano`
      - Response format: JSON object
      - System prompt: strict relevance JSON schema
      - User prompt: include title/snippet/url + brand keyword/description
   4. Connect **RelevancyVerification → Message a model**.
   5. Add **Code** node **MergeOutput&Input** to parse model output and merge with the original extracted data.
   6. Connect **Message a model → MergeOutput&Input**.
   7. Connect **MergeOutput&Input → RelevancyVerification** (to continue batching).

   > Important: as written in the template, ensure that **FilterRelevantOnly** runs **after** `MergeOutput&Input` output (i.e., after `isRelevant` exists). If you keep the template wiring, validate that the execution order produces `isRelevant` before filtering.

12. **Filter relevant only**
   1. Add **Filter** node **FilterRelevantOnly** condition: `{{$json.isRelevant}}` is true.
   2. Connect the post-merge stream (from **RelevancyVerification** output that contains merged items) to **FilterRelevantOnly**.

13. **Check any relevant results**
   1. Add **IF** node **AnyRelevantResults?** (non-empty check).
   2. Connect **FilterRelevantOnly → AnyRelevantResults?**
   3. Add **NoOp** node **No Relevant Results** for the false branch.

14. **Detect new URLs vs sheet history**
   1. Add **Merge** node configured:
      - Mode: combine, join keep non-matches
      - Match: sheet field `URL` with result field `url`
      - Output data from input2
   2. Connect:
      - **Check & Log to Sheet → Merge (input1)**
      - **AnyRelevantResults? (true) → Merge (input2)**

15. **Send email + append**
   1. Add **IF** node **NewURL?** (non-empty check) after merge.
   2. True branch:
      - **Code** node **Format Email** that builds HTML from `$input.all()`, returns `{subject, body, count}`
      - **Gmail** node **Send Email** (configure OAuth2, recipient, subject/body expressions)
      - **Google Sheets** node **InsertSentRows** with operation **Append** to the same tracking sheet (map columns to url/title/source/etc.)
   3. False branch: **NoOp** node **No New Results**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Trigger fires once per day; Brave searches past 24h; banned domain filtering; GPT‑5 Nano verification; cross-match with Google Sheet; email + append. | From sticky note “How it works” |
| Configuration requirements: keyword, description (optional), dorky array, banned domains list. | From sticky note “How it works” |
| Prompt design: system prompt defines strict verification; user prompt provides title/snippet/url + brand context. | From sticky note “Prompt Note” |
| The workflow uses template placeholder credentials and blank sheet selectors; these must be set before activation. | Based on node configuration placeholders (`TEMPLATE`, empty documentId/sheetName, `user@example.com`) |