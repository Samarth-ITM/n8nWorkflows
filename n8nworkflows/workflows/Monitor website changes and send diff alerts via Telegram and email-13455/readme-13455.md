Monitor website changes and send diff alerts via Telegram and email

https://n8nworkflows.xyz/workflows/monitor-website-changes-and-send-diff-alerts-via-telegram-and-email-13455


# Monitor website changes and send diff alerts via Telegram and email

## 1. Workflow Overview

**Workflow name:** Monitor website changes with diff detection and get instant Telegram and email alerts  
**Purpose:** Periodically fetch one or more webpages, extract readable text + metadata, compare the current snapshot with the previously stored snapshot (Google Sheets), generate a line-level diff + severity score, and notify via **Telegram** and **email** only when changes are detected.  
**Primary use cases:** competitor monitoring, pricing/policy changes, job postings, public content tracking, operational monitoring of key web pages.

### 1.1 Trigger & URL Configuration
A schedule trigger starts executions at a fixed interval; a Code node outputs a list of URLs (each as an item) to monitor.

### 1.2 Fetch & Extract
Each URL is fetched as HTML; a Code node cleans the HTML into readable text, extracts title/description, optionally focuses on a keyword-based “selector”, and computes a content hash.

### 1.3 Load Baseline & Diff
The workflow reads prior snapshots from Google Sheets and compares them to the current snapshot. It short-circuits when hashes match, handles “first run” (baseline creation), otherwise computes added/removed lines and severity.

### 1.4 Save & Alert
If content changed, it saves the new snapshot to Google Sheets and sends Telegram + email alerts. If not changed, it still updates the snapshot storage (depending on mapping) and logs a status message.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & URL Configuration

**Overview:** Starts on a timer and emits a configurable list of URLs as separate items, enabling per-URL processing downstream.  
**Nodes involved:** `⏰ Every 4 Hours`, `📋 URL List`

#### Node: ⏰ Every 4 Hours
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — workflow entry point.
- **Configuration (interpreted):** Runs every **4 hours** (interval-based schedule).
- **Inputs/Outputs:** No input; outputs one trigger item into `📋 URL List`.
- **Version notes:** TypeVersion `1.2`.
- **Edge cases / failures:**
  - If n8n instance is paused, inactive workflow (`active: false`), or schedules disabled, it won’t run.
  - Time drift / missed runs can happen if instance is down.

#### Node: 📋 URL List
- **Type / role:** Code (`n8n-nodes-base.code`) — emits URL items for monitoring.
- **Configuration (interpreted):**
  - Hardcoded array `urlsToMonitor` with objects `{ name, url, selector }`.
  - Returns `urlsToMonitor.map(item => ({ json: item }))`, creating one item per URL.
- **Key variables/expressions:**
  - `selector` is optional; used later as a **keyword focus**, not a true CSS selector.
- **Inputs/Outputs:**
  - Input: trigger item from `⏰ Every 4 Hours`.
  - Output: items to `🌐 Fetch Page`.
- **Version notes:** TypeVersion `2`.
- **Edge cases / failures:**
  - Invalid URL strings cause HTTP failures downstream.
  - Large URL lists increase runtime and may hit execution limits.
  - Duplicate `name` values matter later because Sheets upsert matches on `Name`.

**Sticky note(s) covering this block:**
- “## ⏰ Trigger + URL Config …”
- “## ⚠️ Add Your URLs …”

---

### Block 2 — Fetch & Extract

**Overview:** Fetches the webpage HTML and converts it into a stable, comparable text snapshot with metadata and a hash for quick change detection.  
**Nodes involved:** `🌐 Fetch Page`, `📄 Extract Content`

#### Node: 🌐 Fetch Page
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — downloads page content.
- **Configuration (interpreted):**
  - URL: `={{ $json.url }}` (from `📋 URL List` item).
  - Response format: **text**.
  - Timeout: **15,000 ms**.
  - Redirects: up to **5**.
  - Sends custom headers: User-Agent, Accept, Accept-Language (to reduce bot-blocking / vary response).
  - **onError:** `continueRegularOutput` (execution continues even if request errors).
- **Inputs/Outputs:**
  - Input: each URL item from `📋 URL List`.
  - Output: raw response to `📄 Extract Content`.
- **Version notes:** TypeVersion `4.2`.
- **Edge cases / failures:**
  - 403/429 bot protection; headers may help but not guarantee access.
  - Timeouts on slow sites.
  - With `continueRegularOutput`, downstream nodes may receive error-shaped data instead of HTML; extraction logic tries to handle this but may produce noisy snapshots.
  - Dynamic pages (client-rendered) may return minimal HTML; diffs may be misleading.

#### Node: 📄 Extract Content
- **Type / role:** Code — parses HTML/text into a normalized “cleanText” snapshot + metadata.
- **Configuration (interpreted):**
  - Reads URL metadata from `$('📋 URL List').item.json`.
  - Attempts to obtain HTML from `$json` in multiple shapes:
    - if `$json` is string → uses it
    - else uses `$json.data` or `$json.body` or stringified object
  - Extracts:
    - `<title>` via regex
    - meta description via regex (two attribute order variants)
  - Cleans HTML:
    - removes `script/style/noscript/svg`
    - replaces `<nav>` and `<footer>` with placeholders
    - converts various block tags to newlines
    - strips remaining tags
    - decodes a limited set of HTML entities
    - normalizes whitespace/newlines
  - “Selector” behavior:
    - If `selector` provided, it searches for lines containing the keyword and captures up to **50 subsequent lines** per match.
    - If matches found, `focusedContent` becomes only those relevant lines.
  - Hash:
    - Custom 32-bit rolling hash over first **10,000 chars**, then converted to base36.
  - Storage limits:
    - `cleanText: focusedContent.substring(0, 50000)`
- **Key variables/expressions:**
  - `const urlData = $('📋 URL List').item.json;`
  - `selector` keyword logic (not true CSS selection).
  - `contentHash` computed locally.
- **Inputs/Outputs:**
  - Input: response from `🌐 Fetch Page`.
  - Output: structured item to `📋 Load Previous Snapshot` with fields:
    - `name,url,selector,pageTitle,metaDescription,cleanText,contentHash,contentLength,fetchedAt,httpStatus`
- **Version notes:** TypeVersion `2`.
- **Edge cases / failures:**
  - If HTTP node returns an error object, `httpStatus` fallback may incorrectly become 200 (since it defaults to 200 when status is missing).
  - Regex HTML parsing can fail on unusual markup; title/description may be empty.
  - Entity decoding is partial; some pages may compare with encoded variants.
  - Keyword focusing can miss changes outside the captured window, or “flip-flop” if keyword appears/disappears.

**Sticky note(s) covering this block:**
- “## 🌐 Fetch + Extract …”

---

### Block 3 — Load Baseline & Diff

**Overview:** Loads prior stored snapshots from Google Sheets, determines whether this is the first run, then compares previous and current content (hash-first, then line-based) and assigns severity.  
**Nodes involved:** `📋 Load Previous Snapshot`, `🔍 Diff Engine`, `🔄 Changed?`

#### Node: 📋 Load Previous Snapshot
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads stored snapshot rows.
- **Configuration (interpreted):**
  - Document: `YOUR_GOOGLE_SHEET_ID`
  - Sheet tab: `website` (gid `299160514`)
  - Uses **Filters UI** with many “lookupColumn/lookupValue” pairs (Name, url, selector, etc.).
  - `alwaysOutputData: true` ensures output even if nothing is found.
- **Inputs/Outputs:**
  - Input: current extracted item from `📄 Extract Content`.
  - Output: rows to `🔍 Diff Engine`.
- **Version notes:** TypeVersion `4.6`.
- **Edge cases / failures:**
  - Missing/invalid OAuth credentials or spreadsheet permissions.
  - Column names in the note (“Name, url, selector, pageTitle…”) do not perfectly match columns referenced later in the diff code (see below).
  - The configured filter columns include variants like `page title`, `clean text`, etc. If the sheet does not have these exact headers, filtering may return empty results.

#### Node: 🔍 Diff Engine
- **Type / role:** Code — compares current vs. previous snapshots and builds diff summary.
- **Configuration (interpreted):**
  - Pulls current data from `$('📄 Extract Content').item.json`.
  - Pulls all sheet rows from `$('📋 Load Previous Snapshot').all()`.
  - Determines “previous snapshot” by matching `row.json.URL || row.json.url` to `currentData.url`.
  - Reads prior fields using multiple possible header variants:
    - previous content: `row.json.Content || row.json.content`
    - previous hash: `row.json['Content Hash'] || row.json.contentHash`
    - last checked: `row.json['Last Checked'] || row.json.lastChecked`
  - Hash short-circuit:
    - If previous hash equals current hash → `hasChanged: false`
  - First run:
    - If no matching row found → `isFirstRun: true`, `hasChanged: false`, baseline message.
  - Line-level diff:
    - Splits previous and current into trimmed non-empty lines
    - Uses sets to compute:
      - `addedLines` (in current not in previous)
      - `removedLines` (in previous not in current)
    - Calculates `changePercentage = round((added+removed)/max(lines)) * 100`
    - Builds `diffSummary` including up to 20 added + 20 removed lines (each truncated to 200 chars)
  - Severity rules:
    - >50% = `critical`
    - >20% = `high`
    - >5% = `medium`
    - else `low`
- **Inputs/Outputs:**
  - Input: data context from both Extract + Sheets nodes.
  - Output: one item to `🔄 Changed?` containing diff fields:
    - `hasChanged,isFirstRun,changePercentage,severity,diffSummary,addedCount,removedCount,lastChecked,comparedAt,...`
- **Version notes:** TypeVersion `2`.
- **Edge cases / failures (important):**
  - **Column name mismatch risk:** The extractor produces `cleanText/contentHash/fetchedAt`, but the diff engine looks for previous `Content`, `Content Hash`, `Last Checked`, etc. Unless your Google Sheet uses those exact headers (or you’ve previously stored with those names), `previousContent` may be blank and the diff may become meaningless.
  - **Matching logic risk:** It matches only by URL, ignoring selector/name. If you monitor same URL with different selectors, rows can collide.
  - **Set-based diff:** Reordering lines won’t be detected as “changed” lines; duplicates are ignored, and context is lost.
  - If previousContent is empty (because of headers mismatch), most lines appear “added”, causing inflated changePercentage.

#### Node: 🔄 Changed?
- **Type / role:** IF (`n8n-nodes-base.if`) — routes only changed pages to alerts.
- **Configuration (interpreted):**
  - Condition: `$json.hasChanged` is boolean **true**.
- **Inputs/Outputs:**
  - Input: result from `🔍 Diff Engine`.
  - Output (true): to `💾 Save Snapshot`
  - Output (false): to `💾 Save (No Change)`
- **Version notes:** TypeVersion `2.2`.
- **Edge cases / failures:**
  - If `hasChanged` is missing or not boolean (string `"true"`), strict validation may fail; here it expects boolean true.

**Sticky note(s) covering this block:**
- “## 🔍 Compare + Diff …”
- “## ⚠️ First Run = Baseline …”

---

### Block 4 — Save & Alert

**Overview:** Persists the current snapshot to Google Sheets and sends notifications when changes are detected; otherwise saves/logs a no-change status.  
**Nodes involved:** `💾 Save Snapshot`, `📲 Telegram Alert`, `📧 Email Alert`, `💾 Save (No Change)`, `📝 Log Result`

#### Node: 💾 Save Snapshot
- **Type / role:** Google Sheets — upsert snapshot when change detected.
- **Configuration (interpreted):**
  - Operation: **appendOrUpdate**
  - Matching column(s): `Name` (upsert key)
  - Mapping mode: auto-map input data; convert fields to string enabled.
  - Document: `YOUR_GOOGLE_SHEET_ID`, Sheet: `website`.
- **Inputs/Outputs:**
  - Input: “changed” items from `🔄 Changed?` true branch.
  - Output: triggers both `📲 Telegram Alert` and `📧 Email Alert`.
- **Version notes:** TypeVersion `4.6`.
- **Edge cases / failures:**
  - **Upsert key risk:** Using `Name` as the only matching column means two different URLs with the same name will overwrite each other.
  - Because the “columns.value” explicitly only sets `Name`, correctness relies heavily on auto-mapping to populate the rest; if headers differ, fields may not be written where expected.

#### Node: 📲 Telegram Alert
- **Type / role:** Telegram (`n8n-nodes-base.telegram`) — sends formatted Markdown alert.
- **Configuration (interpreted):**
  - Chat ID: placeholder `YOUR_TELEGRAM_CHAT_ID`
  - Message text: generated via an inline IIFE; includes:
    - severity emoji mapping (critical/high/medium/low)
    - name, URL, change %, added/removed counts
    - diff summary truncated to 2000 chars inside Markdown code block
    - comparedAt + lastChecked
  - Parse mode: `Markdown`, web preview disabled.
- **Inputs/Outputs:**
  - Input: output item from `💾 Save Snapshot`.
  - Output: none used further.
- **Version notes:** TypeVersion `1.2`.
- **Edge cases / failures:**
  - Telegram Markdown is fragile: underscores/brackets in diff lines can break formatting; code block helps but header lines still may break.
  - Message length limits apply; truncation helps but still can fail if expanded.
  - Bot token/chat permissions issues.

#### Node: 📧 Email Alert
- **Type / role:** Email Send (`n8n-nodes-base.emailSend`) — sends HTML email.
- **Configuration (interpreted):**
  - Subject: emoji + “Website Changed: {name} ({changePercentage}%)”
  - HTML template includes:
    - severity-colored header (critical/high/medium/low)
    - URL, pageTitle, added/removed counts, diff summary (truncated to 3000)
    - CTA button to open the page
  - From/To: both set to `user@example.com` placeholders.
- **Inputs/Outputs:**
  - Input: output item from `💾 Save Snapshot`.
  - Output: none used further.
- **Version notes:** TypeVersion `2.1`.
- **Edge cases / failures:**
  - Requires SMTP/Gmail credentials; misconfiguration causes send errors.
  - Some email providers strip styles; the email remains readable but may lose formatting.
  - If `diffSummary` is missing, `.substring()` can throw; here it assumes string exists—however Diff Engine always sets `diffSummary`, so it’s usually safe.

#### Node: 💾 Save (No Change)
- **Type / role:** Google Sheets — upsert snapshot on no-change path (and on first run).
- **Configuration (interpreted):**
  - Operation: **appendOrUpdate**
  - Matching column(s): `Name`
  - Mapping mode: auto-map input data
  - `convertFieldsToString: false` (different from Save Snapshot node)
- **Inputs/Outputs:**
  - Input: `🔄 Changed?` false branch.
  - Output: to `📝 Log Result`.
- **Version notes:** TypeVersion `4.6`.
- **Edge cases / failures:**
  - “columns.value” is empty; relies entirely on auto-mapping.
  - Same upsert-key collision risk as above.

#### Node: 📝 Log Result
- **Type / role:** Code — produces a compact log/status item.
- **Configuration (interpreted):**
  - `status` becomes one of: `baseline_saved` / `change_detected` / `no_change`
  - Builds a human-readable `message`.
- **Inputs/Outputs:**
  - Input: from `💾 Save (No Change)`.
  - Output: not connected further (used for inspection/debugging).
- **Version notes:** TypeVersion `2`.
- **Edge cases / failures:**
  - If upstream did not provide expected fields (e.g., `isFirstRun`), status logic still works but may be less accurate.

**Sticky note(s) covering this block:**
- “## 📣 Alert + Save …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Main Overview | Sticky Note | Documentation / overview |  |  | ## Monitor website changes with diff detection and get instant Telegram and email alerts… (includes setup/customization text) |
| Sticky Note - Trigger | Sticky Note | Documentation for trigger/config |  |  | ## ⏰ Trigger + URL Config<br>Schedule trigger fires at set interval, URL List defines pages to monitor |
| Sticky Note - Fetch | Sticky Note | Documentation for fetch/extract |  |  | ## 🌐 Fetch + Extract<br>Download page HTML and extract clean text content with metadata |
| Sticky Note - Diff | Sticky Note | Documentation for compare/diff |  |  | ## 🔍 Compare + Diff<br>Load previous snapshot from Sheets, compare content, calculate diff and severity |
| Sticky Note - Alerts | Sticky Note | Documentation for save/alerts |  |  | ## 📣 Alert + Save<br>Save new snapshot to Sheets, send Telegram and email alerts for detected changes |
| Sticky Note - Warning URLs | Sticky Note | Warning / setup hint |  |  | ## ⚠️ Add Your URLs<br>Edit the "📋 URL List" node to add the websites you want to monitor… |
| Sticky Note - Warning First Run | Sticky Note | Warning / behavior note |  |  | ## ⚠️ First Run = Baseline<br>The first run saves snapshots only — no alerts will fire… |
| ⏰ Every 4 Hours | Schedule Trigger | Starts workflow on interval |  | 📋 URL List | ## ⏰ Trigger + URL Config<br>Schedule trigger fires at set interval, URL List defines pages to monitor |
| 📋 URL List | Code | Defines URLs to monitor (fan-out) | ⏰ Every 4 Hours | 🌐 Fetch Page | ## ⏰ Trigger + URL Config<br>Schedule trigger fires at set interval, URL List defines pages to monitor<br>## ⚠️ Add Your URLs<br>Edit the "📋 URL List" node… |
| 🌐 Fetch Page | HTTP Request | Downloads HTML/text for each URL | 📋 URL List | 📄 Extract Content | ## 🌐 Fetch + Extract<br>Download page HTML and extract clean text content with metadata |
| 📄 Extract Content | Code | Cleans HTML, extracts metadata, computes hash | 🌐 Fetch Page | 📋 Load Previous Snapshot | ## 🌐 Fetch + Extract<br>Download page HTML and extract clean text content with metadata |
| 📋 Load Previous Snapshot | Google Sheets | Reads prior snapshots (baseline) | 📄 Extract Content | 🔍 Diff Engine | ## 🔍 Compare + Diff<br>Load previous snapshot from Sheets, compare content, calculate diff and severity |
| 🔍 Diff Engine | Code | Hash + line diff, severity scoring | 📋 Load Previous Snapshot | 🔄 Changed? | ## 🔍 Compare + Diff<br>Load previous snapshot from Sheets…<br>## ⚠️ First Run = Baseline… |
| 🔄 Changed? | IF | Routes changed vs unchanged | 🔍 Diff Engine | 💾 Save Snapshot (true), 💾 Save (No Change) (false) | ## 🔍 Compare + Diff… |
| 💾 Save Snapshot | Google Sheets | Upsert snapshot when changed | 🔄 Changed? (true) | 📲 Telegram Alert, 📧 Email Alert | ## 📣 Alert + Save… |
| 📲 Telegram Alert | Telegram | Sends Telegram change alert | 💾 Save Snapshot |  | ## 📣 Alert + Save… |
| 📧 Email Alert | Email Send | Sends HTML email change alert | 💾 Save Snapshot |  | ## 📣 Alert + Save… |
| 💾 Save (No Change) | Google Sheets | Upsert snapshot when not changed / baseline | 🔄 Changed? (false) | 📝 Log Result | ## 📣 Alert + Save… |
| 📝 Log Result | Code | Produces log/status output | 💾 Save (No Change) |  | ## 📣 Alert + Save… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Schedule Trigger** node  
   - Name: `⏰ Every 4 Hours`  
   - Configure: interval → every **4 hours**.
3. **Add Code** node  
   - Name: `📋 URL List`  
   - Paste code that returns items like `{ name, url, selector }` (keyword).  
   - Ensure it returns `return urlsToMonitor.map(item => ({ json: item }));`
4. **Connect** `⏰ Every 4 Hours` → `📋 URL List`.
5. **Add HTTP Request** node  
   - Name: `🌐 Fetch Page`  
   - URL: `{{ $json.url }}`  
   - Response: **text**  
   - Timeout: **15000 ms**  
   - Redirects: max **5**  
   - Headers:
     - `User-Agent`: a modern browser UA string
     - `Accept`: `text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`
     - `Accept-Language`: `en-US,en;q=0.5`
   - Error handling: set node option to **Continue on Fail** (equivalent to `onError: continueRegularOutput`).
6. **Connect** `📋 URL List` → `🌐 Fetch Page`.
7. **Add Code** node  
   - Name: `📄 Extract Content`  
   - Implement:
     - extraction of title/meta description (regex)
     - HTML stripping and whitespace normalization
     - optional keyword focusing using `selector`
     - compute `contentHash`
     - output fields: `name,url,selector,pageTitle,metaDescription,cleanText,contentHash,contentLength,fetchedAt,httpStatus`
8. **Connect** `🌐 Fetch Page` → `📄 Extract Content`.
9. **Prepare Google Sheets**
   - Create a Google Spreadsheet (e.g., “Website Monitor”) and a tab (e.g., `website`).
   - Add headers (you must keep them consistent with your nodes). Recommended to align with the extractor fields, e.g.:
     - `Name`, `url`, `selector`, `pageTitle`, `metaDescription`, `cleanText`, `contentHash`, `contentLength`, `fetchedAt`, `httpStatus`, plus optional `lastChecked`, `comparedAt`, etc.
   - In n8n, create/connect **Google Sheets OAuth2** credentials with edit access.
10. **Add Google Sheets** node (read)  
    - Name: `📋 Load Previous Snapshot`  
    - Configure to read rows from your document and `website` sheet.  
    - Ensure it returns rows even if none found (enable “Always Output Data” if available).
    - If using filters, configure them to locate the row for the current URL (ideally filter by `url` + `selector`).
11. **Connect** `📄 Extract Content` → `📋 Load Previous Snapshot`.
12. **Add Code** node  
    - Name: `🔍 Diff Engine`  
    - Logic:
      - Identify the matching previous row for the current URL (and ideally selector)
      - If none: set `isFirstRun: true`, `hasChanged: false`
      - If hashes match: `hasChanged: false`
      - Else compute added/removed lines, `changePercentage`, `diffSummary`, `severity`.
    - Important: Make sure the code reads the same column names that your save nodes write (e.g., use `cleanText/contentHash` consistently).
13. **Connect** `📋 Load Previous Snapshot` → `🔍 Diff Engine`.
14. **Add IF** node  
    - Name: `🔄 Changed?`  
    - Condition: `{{ $json.hasChanged }}` is **true** (boolean).
15. **Connect** `🔍 Diff Engine` → `🔄 Changed?`.
16. **Add Google Sheets** node (write, changed path)  
    - Name: `💾 Save Snapshot`  
    - Operation: **Append or Update**  
    - Matching columns: choose a stable key (recommended: `url` + `selector`; if you use `Name`, ensure uniqueness).  
    - Map fields from input to sheet columns (auto-map or manual map).
17. **Connect** `🔄 Changed?` (true) → `💾 Save Snapshot`.
18. **Add Telegram** node  
    - Name: `📲 Telegram Alert`  
    - Credentials: Telegram Bot Token  
    - Chat ID: your target chat (replace placeholder)  
    - Parse mode: Markdown; disable link previews  
    - Message: build from `$json` including diff summary truncation to avoid length errors.
19. **Add Email Send** node (optional)  
    - Name: `📧 Email Alert`  
    - Credentials: SMTP or Gmail (depending on your n8n setup)  
    - From/To: set real addresses  
    - Subject and HTML: use `$json` fields; truncate diff content.
20. **Connect** `💾 Save Snapshot` → `📲 Telegram Alert` and `💾 Save Snapshot` → `📧 Email Alert` (two parallel connections).
21. **Add Google Sheets** node (write, no-change path)  
    - Name: `💾 Save (No Change)`  
    - Operation: **Append or Update**  
    - Matching columns: same as “Save Snapshot”  
    - Map current fields so the stored snapshot stays up-to-date (at minimum update `cleanText`, `contentHash`, `fetchedAt`).
22. **Connect** `🔄 Changed?` (false) → `💾 Save (No Change)`.
23. **Add Code** node  
    - Name: `📝 Log Result`  
    - Output a status summary (`baseline_saved/no_change/change_detected`) for debugging.
24. **Connect** `💾 Save (No Change)` → `📝 Log Result`.
25. **Run once manually** to populate baselines. Alerts should start from the second run.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| First run saves snapshots only — no alerts will fire. Changes are detected from the second run onward. | From sticky note “## ⚠️ First Run = Baseline …” |
| Edit the `📋 URL List` node and add one item per URL with `name` and `url` fields; optionally add `selector`. | From sticky note “## ⚠️ Add Your URLs …” |
| Google Sheets should contain snapshot columns and requires OAuth credentials + spreadsheet ID configured in both Sheets nodes. | From main overview sticky note |
| Telegram requires a bot token and chat ID; email requires SMTP/Gmail credentials and valid from/to. | From main overview sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.