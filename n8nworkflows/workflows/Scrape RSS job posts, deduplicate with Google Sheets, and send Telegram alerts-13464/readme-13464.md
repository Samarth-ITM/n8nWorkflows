Scrape RSS job posts, deduplicate with Google Sheets, and send Telegram alerts

https://n8nworkflows.xyz/workflows/scrape-rss-job-posts--deduplicate-with-google-sheets--and-send-telegram-alerts-13464


# Scrape RSS job posts, deduplicate with Google Sheets, and send Telegram alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor a job source (JSON API/RSS-like endpoint), extract and normalize job postings, filter them by role/location keywords, deduplicate against a Google Sheet, log new jobs, and send instant Telegram alerts.

**Target use cases:**
- Continuous job hunting for remote roles with minimal noise
- Multi-board monitoring (extendable by adding more fetch nodes)
- Maintaining a historical list of seen jobs to avoid duplicate notifications

### 1.1 Scheduling & Data Retrieval
Runs on a schedule and fetches job data from a configured endpoint.

### 1.2 Parsing & Normalization
Converts raw response items into a consistent “job” object model and drops invalid entries.

### 1.3 Keyword Filtering
Keeps only jobs matching target roles and target locations, while excluding unwanted seniority/leadership terms.

### 1.4 Deduplication & Persistence
Checks whether a job was already seen (via Google Sheets) and keeps only truly new jobs; logs new jobs to Sheets.

### 1.5 Alerting & No-Result Handling
Sends formatted Telegram messages for each new job; otherwise emits a summary “no new jobs” output.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Data Retrieval

**Overview:** Triggers every 6 hours and fetches job postings from a remote endpoint (currently RemoteOK API).  
**Nodes involved:** `⏰ Every 6 Hours`, `🌐 Fetch Job Posts`

#### Node: ⏰ Every 6 Hours
- **Type / role:** Schedule Trigger — entry point that starts the workflow periodically.
- **Configuration (interpreted):**
  - Interval-based schedule: every **6 hours**.
- **Inputs / outputs:**
  - **Input:** none (trigger)
  - **Output:** to `🌐 Fetch Job Posts`
- **Edge cases / failures:**
  - Workflow inactive (`active: false`) means it won’t run until activated.
  - Timezone considerations depend on n8n instance settings (important if you later switch to cron rules).

#### Node: 🌐 Fetch Job Posts
- **Type / role:** HTTP Request — retrieves job data.
- **Configuration (interpreted):**
  - URL: `https://remoteok.com/api` (placeholder for your chosen board)
  - Response format: JSON
- **Inputs / outputs:**
  - **Input:** from `⏰ Every 6 Hours`
  - **Output:** to `🔧 Parse & Extract Jobs`
- **Edge cases / failures:**
  - Non-200 responses, rate limits, network/DNS failures.
  - Schema mismatch: downstream parsing expects per-item job-like properties; if the API returns a nested list, you may need to adjust parsing.
  - Some RSS feeds return XML; this node is set for JSON, so RSS/XML sources would require different handling (e.g., set response to text + XML parsing).

**Sticky notes applying:**
- “Sticky Note - Data Source”: Schedule trigger + HTTP request to fetch job posts from RSS/API
- “Sticky Note - Warning URL”: Replace with your job board URL

---

### Block 2 — Parsing & Normalization

**Overview:** Normalizes heterogeneous job fields into a standard structure and ensures minimal required fields are present.  
**Nodes involved:** `🔧 Parse & Extract Jobs`

#### Node: 🔧 Parse & Extract Jobs
- **Type / role:** Code node (JavaScript) — transforms input items into normalized job objects.
- **Configuration (interpreted):**
  - Reads **all incoming items** (`$input.all()`).
  - Skips non-job entries (e.g., RemoteOK metadata item) by checking for `position/title/job_title`.
  - Builds a normalized object:
    - `title`, `company`, `location`, `url`, `description` (truncated to 500 chars), `tags`, `salary`, `postedDate`, `source`, `scrapedAt`
    - `uniqueId`: derived from URL/apply link or `position-company`, lowercased and trimmed.
  - Only emits jobs where:
    - title is not `Unknown Title`
    - and `url` exists
  - If nothing parsed, emits a single item: `{ noJobs: true, message: ... }`
- **Inputs / outputs:**
  - **Input:** from `🌐 Fetch Job Posts`
  - **Output:** to `🎯 Keyword Filter`
- **Key variables/fields created:**
  - `job.uniqueId` is intended as a stable dedupe key; in practice, downstream dedupe uses sheet URL columns (see Block 4).
- **Edge cases / failures:**
  - If the HTTP response is an array but n8n produces a single item containing the array, the loop won’t iterate over jobs as intended. You may need to “Split Out Items” or adjust parsing to iterate over `item.json` if it’s an array.
  - Truncating description to 500 chars may cut important info; safe but may affect filtering accuracy.
  - If `url` is missing for a source, the job will be dropped entirely.
  - Data types vary (`tags` can be array or string); later nodes handle arrays defensively.

**Sticky notes applying:**
- “Sticky Note - Parse Filter”: Extract job fields, then filter by keywords, location, and role

---

### Block 3 — Keyword Filtering & Gating

**Overview:** Filters normalized jobs using configurable keyword arrays, then routes either to dedupe or to a “no new jobs” summary path.  
**Nodes involved:** `🎯 Keyword Filter`, `✅ Has Valid Jobs?`

#### Node: 🎯 Keyword Filter
- **Type / role:** Code node (JavaScript) — applies inclusion/exclusion filters.
- **Configuration (interpreted):**
  - **Customizable arrays** at top:
    - `targetRoles` (strings searched in title and tags)
    - `targetLocations` (strings searched in location)
    - `excludeTerms` (strings searched in title)
  - For each job item:
    - Builds lowercase versions of title/location/description/tags.
    - `roleMatch`: any `targetRoles` found in title or tags.
    - `locationMatch`: any `targetLocations` found in location.
    - `isExcluded`: any `excludeTerms` found in title.
    - Keeps job only if `(roleMatch && locationMatch && !isExcluded)`
  - Adds metadata to passing jobs:
    - `matchedRole`, `matchedLocation`, `filterPassedAt`
  - If no matches, emits a single item: `{ noMatches: true, message: ..., checkedAt: ... }`
- **Inputs / outputs:**
  - **Input:** from `🔧 Parse & Extract Jobs`
  - **Output:** to `✅ Has Valid Jobs?`
- **Edge cases / failures:**
  - Location filtering only checks `job.location`; if your source puts “remote” in description or tags instead, it will be missed unless you modify logic.
  - Exclusion uses naive substring matches; e.g., “vp ” includes a trailing space and may miss “VP,” or match odd cases depending on punctuation/case handling.
  - If `tags` is an array of objects, the node tries `t.name` / `t.label` fallbacks; unusual tag shapes may still stringify poorly.

#### Node: ✅ Has Valid Jobs?
- **Type / role:** IF node — gates execution to dedupe vs “no results” branch.
- **Configuration (interpreted):**
  - Condition uses **OR**:
    1) `$json.url` is not empty  
    2) `$json.noMatches` is not true
  - **True output** → `📋 Read Seen Jobs`  
  - **False output** → `💤 No New Jobs`
- **Inputs / outputs:**
  - **Input:** from `🎯 Keyword Filter`
  - **Output (true):** to `📋 Read Seen Jobs`
  - **Output (false):** to `💤 No New Jobs`
- **Important behavior note (logic risk):**
  - Because it’s **OR**, the “true” branch can still be taken when `noMatches` exists (since “notTrue(noMatches)” may evaluate true when undefined/falsey), and/or when some items are summary flags.
  - If the `🎯 Keyword Filter` outputs the single `{ noMatches: true }` item, condition (2) becomes false; (1) is also false (no url), so it should go to false branch. That’s correct.
  - However, if any non-job items slip through without `noMatches` but also without `url`, they may still pass depending on evaluation; safest would typically be **AND**: “has url” AND “noMatches is not true”.

**Sticky notes applying:**
- “Sticky Note - Warning Keywords”: Edit your keywords
- “Sticky Note - Parse Filter”: Extract job fields, then filter…

---

### Block 4 — Deduplication & Logging

**Overview:** Reads previously seen jobs from Google Sheets, filters out already-seen URLs, then appends new ones to Sheets.  
**Nodes involved:** `📋 Read Seen Jobs`, `🧹 Deduplicate`, `🆕 New Jobs Only?`, `📊 Log New Jobs`

#### Node: 📋 Read Seen Jobs
- **Type / role:** Google Sheets node — looks up existing rows to help dedupe.
- **Configuration (interpreted):**
  - Spreadsheet: placeholder `YOUR_GOOGLE_SHEET_ID` (“Job Tracker”), sheet `jobs`
  - Uses a **lookup filter** on column `title` with value `{{$json.title}}`
  - `alwaysOutputData: true` ensures output even if no rows match
- **Inputs / outputs:**
  - **Input:** from `✅ Has Valid Jobs?` (true branch)
  - **Output:** to `🧹 Deduplicate`
- **Edge cases / failures:**
  - Credentials missing/expired (OAuth), permission issues, wrong spreadsheet ID/sheet name.
  - Lookup by **title** is weak for dedupe (titles repeat across companies); also it reads only matching titles, not the full “seen set”.
  - Column naming mismatch: dedupe code checks `item.json.URL` (uppercase) but your sheet schema later uses `Url` (capital U + lowercase rl). This mismatch can break dedupe unless you align headers.

#### Node: 🧹 Deduplicate
- **Type / role:** Code node (JavaScript) — keeps only jobs not found in previously seen sheet data.
- **Configuration (interpreted):**
  - Reads current jobs from `$('✅ Has Valid Jobs?').all()`
  - Reads seen jobs from `$('📋 Read Seen Jobs').all()`
  - Builds `seenUrls` set using (in order): `item.json.URL` OR `item.json.url` OR `item.json.uniqueId`
  - Compares against each current job’s `job.url` or `job.uniqueId`
  - Marks new jobs with:
    - `isNew: true`, `deduplicatedAt`
  - If none new, emits a single `{ noNewJobs: true, message: ... }` item
- **Inputs / outputs:**
  - **Input:** from `📋 Read Seen Jobs`
  - **Output:** to `🆕 New Jobs Only?`
- **Edge cases / failures:**
  - **Key issue:** If the sheet stores the URL under header `Url` (as configured in `📊 Log New Jobs`), this code won’t see it (`URL`/`url` only), so duplicates may slip through. Fix by checking `item.json.Url` as well.
  - Since `📋 Read Seen Jobs` is filtered by title, `seenUrls` may not contain URLs for other titles; duplicates with different titles or formatting changes can bypass dedupe.
  - Reliance on `$('Node Name')` references requires node names to remain unchanged.

#### Node: 🆕 New Jobs Only?
- **Type / role:** IF node — routes new jobs to logging/alerts, others to no-new-jobs path.
- **Configuration (interpreted):**
  - Condition: `$json.isNew` is true
  - **True output** → `📊 Log New Jobs`
  - **False output** → `💤 No New Jobs`
- **Inputs / outputs:**
  - **Input:** from `🧹 Deduplicate`
  - **Output (true):** to `📊 Log New Jobs`
  - **Output (false):** to `💤 No New Jobs`
- **Edge cases / failures:**
  - When dedupe outputs the summary `{ noNewJobs: true }`, `isNew` is undefined → goes false branch (expected).

#### Node: 📊 Log New Jobs
- **Type / role:** Google Sheets node — appends new jobs to a tracking sheet.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Spreadsheet: `YOUR_GOOGLE_SHEET_ID`, sheet `jobs`
  - Uses auto-mapping with explicit values provided for some columns:
    - `Salary` ← `{{$json.salary}}`
    - `Location` ← `{{$json.location}}`
    - `Matched Role` ← `{{$json.matchedRole}}`
  - Schema lists columns including: `Title ` (note trailing space), `Company name`, `Location`, `Url`, `Description`, `Posted date`, `Salary`, `Matched Role`, `Scraped date`
- **Inputs / outputs:**
  - **Input:** from `🆕 New Jobs Only?` (true branch)
  - **Output:** to `📲 Telegram Alert`
- **Edge cases / failures:**
  - Header mismatches are likely:
    - `Title ` has a trailing space (different from “Title”).
    - URL column is `Url` (not `URL`), which affects dedupe if not updated.
  - If `mappingMode` auto-map doesn’t map fields like `title/company/url/description/postedDate/scrapedAt` to the intended sheet headers, rows may be incomplete unless you explicitly map each column.

**Sticky notes applying:**
- “Sticky Note - Deduplicate”: Check against previously seen jobs, save new ones to Google Sheets

---

### Block 5 — Alerting & No-Result Handling

**Overview:** Sends a Telegram message for each new job; otherwise produces a structured “no new jobs” item.  
**Nodes involved:** `📲 Telegram Alert`, `💤 No New Jobs`

#### Node: 📲 Telegram Alert
- **Type / role:** Telegram node — sends a message to a chat.
- **Configuration (interpreted):**
  - Chat ID: `YOUR_TELEGRAM_CHAT_ID` (placeholder)
  - Parse mode: Markdown
  - Disable link previews: true
  - Message template includes: title, company, location, salary, tags (first 5), apply link, posted date, matched role/location
- **Inputs / outputs:**
  - **Input:** from `📊 Log New Jobs`
  - **Output:** none (end of main “new jobs” path)
- **Edge cases / failures:**
  - Invalid bot token/credentials, chat ID mismatch, bot not allowed in the chat.
  - Markdown formatting can break if job title/company contains special characters (`*`, `_`, `[`, `]`, `(`, `)`); consider escaping to avoid Telegram API errors or malformed messages.
  - If `url` is empty (should be prevented upstream), the markdown link could be invalid.

#### Node: 💤 No New Jobs
- **Type / role:** Code node (JavaScript) — emits a summary/status object when no actionable jobs exist.
- **Configuration (interpreted):**
  - Uses `$json.message` if present; otherwise defaults.
  - Outputs:
    - `status: 'no_new_jobs'`
    - `message`, `checkedAt`, `nextCheck: 'In 6 hours'`
- **Inputs / outputs:**
  - **Input:** from `✅ Has Valid Jobs?` (false branch) and `🆕 New Jobs Only?` (false branch)
  - **Output:** none (end)
- **Edge cases / failures:**
  - `nextCheck` is hardcoded to 6 hours; if schedule interval changes, this becomes inaccurate.

**Sticky notes applying:**
- “Sticky Note - Alerts”: Send formatted job notifications to Telegram with apply links

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Main Overview | Sticky Note | Documentation / overview |  |  | ## Scrape job posts from RSS feeds and get instant Telegram alerts… (contains setup steps and customization) |
| Sticky Note - Data Source | Sticky Note | Documentation for source block |  |  | ## 📥 Data Source; Schedule trigger + HTTP request to fetch job posts from RSS/API |
| Sticky Note - Parse Filter | Sticky Note | Documentation for parsing/filtering |  |  | ## 🔍 Parse & Filter; Extract job fields, then filter by keywords, location, and role |
| Sticky Note - Deduplicate | Sticky Note | Documentation for dedupe/logging |  |  | ## 🧹 Deduplicate & Log; Check against previously seen jobs, save new ones to Google Sheets |
| Sticky Note - Alerts | Sticky Note | Documentation for alerts |  |  | ## 📣 Alerts; Send formatted job notifications to Telegram with apply links |
| Sticky Note - Warning URL | Sticky Note | Warning / configuration note |  |  | ## ⚠️ Replace with your job board URL; Update the HTTP Request URL… |
| Sticky Note - Warning Keywords | Sticky Note | Warning / configuration note |  |  | ## ⚠️ Edit your keywords; Update `targetRoles`, `targetLocations`, and `excludeTerms`… |
| ⏰ Every 6 Hours | Schedule Trigger | Periodic trigger |  | 🌐 Fetch Job Posts | ## 📥 Data Source; Schedule trigger + HTTP request to fetch job posts from RSS/API |
| 🌐 Fetch Job Posts | HTTP Request | Fetch job posts JSON | ⏰ Every 6 Hours | 🔧 Parse & Extract Jobs | ## 📥 Data Source… / ## ⚠️ Replace with your job board URL… |
| 🔧 Parse & Extract Jobs | Code | Normalize raw job objects | 🌐 Fetch Job Posts | 🎯 Keyword Filter | ## 🔍 Parse & Filter… |
| 🎯 Keyword Filter | Code | Keyword/location filtering | 🔧 Parse & Extract Jobs | ✅ Has Valid Jobs? | ## 🔍 Parse & Filter… / ## ⚠️ Edit your keywords… |
| ✅ Has Valid Jobs? | IF | Gate to dedupe vs no-results | 🎯 Keyword Filter | 📋 Read Seen Jobs; 💤 No New Jobs |  |
| 📋 Read Seen Jobs | Google Sheets | Lookup previously seen entries | ✅ Has Valid Jobs? | 🧹 Deduplicate | ## 🧹 Deduplicate & Log… |
| 🧹 Deduplicate | Code | Remove already-seen jobs | 📋 Read Seen Jobs | 🆕 New Jobs Only? | ## 🧹 Deduplicate & Log… |
| 🆕 New Jobs Only? | IF | Route only new jobs | 🧹 Deduplicate | 📊 Log New Jobs; 💤 No New Jobs |  |
| 📊 Log New Jobs | Google Sheets | Append new jobs to sheet | 🆕 New Jobs Only? | 📲 Telegram Alert | ## 🧹 Deduplicate & Log… |
| 📲 Telegram Alert | Telegram | Send job alert message | 📊 Log New Jobs |  | ## 📣 Alerts… |
| 💤 No New Jobs | Code | Output summary status | ✅ Has Valid Jobs?; 🆕 New Jobs Only? |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Scrape job posts from RSS feeds and get instant Telegram alerts with deduplication* (or your preferred name).
   - Keep it inactive until testing is complete.

2. **Add the trigger**
   - Add node: **Schedule Trigger**
   - Name: `⏰ Every 6 Hours`
   - Set interval: **Every 6 hours** (adjust to your needs).

3. **Fetch jobs from a source**
   - Add node: **HTTP Request**
   - Name: `🌐 Fetch Job Posts`
   - Method: GET
   - URL: your job source endpoint (e.g., `https://remoteok.com/api`)
   - Response: **JSON**
   - Connect: `⏰ Every 6 Hours` → `🌐 Fetch Job Posts`

4. **Parse/normalize the response**
   - Add node: **Code**
   - Name: `🔧 Parse & Extract Jobs`
   - Paste the parsing JS logic (adapt field mappings if your source differs).
   - Connect: `🌐 Fetch Job Posts` → `🔧 Parse & Extract Jobs`

5. **Filter by role/location/exclusions**
   - Add node: **Code**
   - Name: `🎯 Keyword Filter`
   - Paste the keyword filter JS.
   - Edit arrays:
     - `targetRoles`
     - `targetLocations`
     - `excludeTerms`
   - Connect: `🔧 Parse & Extract Jobs` → `🎯 Keyword Filter`

6. **Gate: decide whether there are jobs to process**
   - Add node: **IF**
   - Name: `✅ Has Valid Jobs?`
   - Conditions (as implemented):
     - OR:
       - String: `{{$json.url}}` is not empty
       - Boolean: `{{$json.noMatches}}` is not true
   - Connect: `🎯 Keyword Filter` → `✅ Has Valid Jobs?`

7. **Prepare Google Sheet**
   - Create a Google Spreadsheet (example name “Job Tracker”).
   - Add a sheet named `jobs`.
   - Create columns (headers) aligned to what you will map, e.g.:
     - `Title`, `Company name`, `Location`, `Url`, `Description`, `Posted date`, `Salary`, `Matched Role`, `Scraped date`
   - Important: choose a consistent header for URL (recommend `Url` or `URL`) and use it everywhere.

8. **Read previously seen jobs (Sheets)**
   - Add node: **Google Sheets**
   - Name: `📋 Read Seen Jobs`
   - Credentials: configure Google OAuth2 in n8n and authorize access.
   - Document: select your spreadsheet
   - Sheet: `jobs`
   - Configure lookup/filter (as in workflow):
     - Lookup column: `title`
     - Lookup value: `{{$json.title}}`
   - Enable “Always Output Data” (so downstream runs even if no match).
   - Connect (true branch): `✅ Has Valid Jobs?` → `📋 Read Seen Jobs`

9. **Deduplicate**
   - Add node: **Code**
   - Name: `🧹 Deduplicate`
   - Paste dedupe JS.
   - (Recommended adjustment) Ensure it checks your actual URL header name (e.g., `Url`) when building `seenUrls`.
   - Connect: `📋 Read Seen Jobs` → `🧹 Deduplicate`

10. **Gate: keep only new jobs**
    - Add node: **IF**
    - Name: `🆕 New Jobs Only?`
    - Condition:
      - Boolean `{{$json.isNew}}` is true
    - Connect: `🧹 Deduplicate` → `🆕 New Jobs Only?`

11. **Log new jobs to Sheets**
    - Add node: **Google Sheets**
    - Name: `📊 Log New Jobs`
    - Operation: **Append**
    - Document: your spreadsheet
    - Sheet: `jobs`
    - Map columns explicitly (recommended) e.g.:
      - Title ← `{{$json.title}}`
      - Company name ← `{{$json.company}}`
      - Location ← `{{$json.location}}`
      - Url ← `{{$json.url}}`
      - Description ← `{{$json.description}}`
      - Posted date ← `{{$json.postedDate}}`
      - Salary ← `{{$json.salary}}`
      - Matched Role ← `{{$json.matchedRole}}`
      - Scraped date ← `{{$json.scrapedAt}}`
    - Connect (true branch): `🆕 New Jobs Only?` → `📊 Log New Jobs`

12. **Send Telegram alert**
    - Add node: **Telegram**
    - Name: `📲 Telegram Alert`
    - Credentials: create a bot with @BotFather, store bot token in n8n Telegram credentials.
    - Set Chat ID to your target: `YOUR_TELEGRAM_CHAT_ID`
    - Message text: use the provided Markdown template.
    - Options:
      - Parse mode: Markdown
      - Disable web page preview: true
      - Append attribution: off (optional)
    - Connect: `📊 Log New Jobs` → `📲 Telegram Alert`

13. **No-new-jobs path**
    - Add node: **Code**
    - Name: `💤 No New Jobs`
    - Paste summary JS.
    - Connect false branches:
      - `✅ Has Valid Jobs?` (false) → `💤 No New Jobs`
      - `🆕 New Jobs Only?` (false) → `💤 No New Jobs`

14. **Test end-to-end**
    - Execute workflow manually.
    - Confirm:
      - Jobs are parsed into normalized items.
      - Filter keeps only intended roles/locations.
      - New jobs append to Sheets.
      - Telegram message arrives and formatting is correct.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace the HTTP Request URL with your target RSS feed or API (examples given: RemoteOK, Arbeitnow, “Any RSS feed from LinkedIn, Indeed, etc.”). | From “Sticky Note - Main Overview” and “Sticky Note - Warning URL” |
| Edit `targetRoles`, `targetLocations`, and `excludeTerms` arrays in the Keyword Filter node to match your search criteria. | From “Sticky Note - Main Overview” and “Sticky Note - Warning Keywords” |
| Google Sheets setup: set spreadsheet ID in both Sheets nodes; create columns: Title, Company name, Location, Url, Description, Posted date, Salary, Matched Role, Scraped date. | From “Sticky Note - Main Overview” |
| Telegram setup: create bot via @BotFather, get Chat ID, connect Telegram credential. | From “Sticky Note - Main Overview” |
| Customization ideas: multiple sources + merge, higher frequency, salary filtering, replace Telegram with Slack/Discord/email, add AI scoring/ranking. | From “Sticky Note - Main Overview” |