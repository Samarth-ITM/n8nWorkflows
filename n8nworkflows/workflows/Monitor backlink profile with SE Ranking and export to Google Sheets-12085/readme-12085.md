Monitor backlink profile with SE Ranking and export to Google Sheets

https://n8nworkflows.xyz/workflows/monitor-backlink-profile-with-se-ranking-and-export-to-google-sheets-12085


# Monitor backlink profile with SE Ranking and export to Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor a domain’s backlink profile using the SE Ranking API and export a structured, multi-section report into Google Sheets.

**Typical use cases:** SEO teams tracking link acquisition/loss, website owners monitoring referring domains and anchor texts, marketers reviewing authority trends and backlink history.

### 1.1 Trigger & Initialization
Starts manually (can be replaced by a schedule).

### 1.2 Backlink Data Collection (SE Ranking)
Pulls multiple datasets from SE Ranking:
- Summary snapshot (totals, follow/nofollow, ranks)
- New backlinks (last 30 days)
- Lost backlinks (last 30 days)
- Referring domains
- Anchor texts
- Authority metrics
- Daily new/lost counts (last 30 days)
- Cumulative backlink history (last 60 days)
- Pages with backlinks

### 1.3 Consolidation, Formatting & Export
Combines the two final streams, formats everything into a single “sheet-ready” table (with section headers and blank separators), then appends rows to a Google Sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Entry

**Overview:** Provides a manual entry point to run the workflow on demand.

**Nodes involved:**
- Manual Trigger

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — starts the workflow manually from the UI.
- **Configuration:** No parameters.
- **Inputs / outputs:**
  - **Output →** `Get backlinks summary`
- **Version notes:** TypeVersion `1` (standard).
- **Edge cases / failures:** None (only user-initiated execution).

---

### Block 2 — Data Collection (SE Ranking)

**Overview:** Calls the SE Ranking community node multiple times to collect backlink-related datasets for a single target domain (`example.com`) and defined date ranges.

**Nodes involved:**
- Get backlinks summary
- Get new backlinks
- Get lost backlinks
- Get referring domains
- Get anchor texts
- Get authority metrics
- Get daily backlinks count
- Get cumulative history
- Get pages with backlinks

**Shared dependency:** All SE Ranking nodes use **SE Ranking API credentials** (`seRankingApi`).

#### Node: Get backlinks summary
- **Type / role:** `@seranking/n8n-nodes-seranking.seRanking` — fetches overall backlink summary.
- **Key configuration:**
  - **Resource:** `backlinks`
  - **Target:** `example.com`
  - **Operation:** not explicitly set (defaults to the node’s “summary”-style operation for backlinks in this community node).
- **Outputs / connections:**
  - **Output →** `Get new backlinks`
  - **Output →** `Get lost backlinks`
- **Key data used later:** `json.summary` (expected array).
- **Failure types:**
  - Auth/token invalid or expired
  - Target domain unsupported/incorrect
  - API quota/rate limiting
  - Schema differences (if the community node version returns different keys than expected by the Code node)

#### Node: Get new backlinks
- **Type / role:** SE Ranking node — fetches new/lost backlinks history for a period (used here as “new backlinks” dataset).
- **Key configuration:**
  - **Resource:** `backlinks`
  - **Operation:** `getHistory`
  - **Target:** `example.com`
  - **dateFrom:** `{{ $now.minus(30, 'days').toFormat('yyyy-MM-dd') }}`
  - **dateTo:** `{{ $now.toFormat('yyyy-MM-dd') }}`
  - **Additional fields:**
    - `mode: as_root` (root domain mode)
    - `limit: 100`
- **Input / output:**
  - **Input ←** `Get backlinks summary`
  - **Output →** `Get referring domains`
- **Key data used later:** `json.new_lost_backlinks` (expected array)
- **Edge cases:**
  - Date formatting/local timezone mismatches (rare but possible if SE Ranking expects UTC boundaries)
  - Large result sets truncated by `limit: 100`

#### Node: Get lost backlinks
- **Type / role:** SE Ranking node — also `getHistory` for the same period (intended to represent “lost backlinks”).
- **Key configuration:** identical to “Get new backlinks” (same operation and dates).
  - **Important note:** Because it uses the same `getHistory` operation, the differentiation between “new” and “lost” is assumed to be inside the returned dataset (`new_lost_backlinks`). The formatting code treats this node as “lost”.
- **Input / output:**
  - **Input ←** `Get backlinks summary`
  - **Output →** `Get anchor texts`
- **Key data used later:** `json.new_lost_backlinks`
- **Edge cases:**
  - If the API returns combined new+lost rows without filtering, your “lost” section may include new entries unless the node/operation supports a filter (not configured here).

#### Node: Get referring domains
- **Type / role:** SE Ranking node — fetches referring domains list.
- **Key configuration:**
  - **Resource:** `backlinks`
  - **Operation:** `getRefDomains`
  - **Target:** `example.com`
  - **Additional fields:** `mode: as_root`, `limit: 100`
- **Input / output:**
  - **Input ←** `Get new backlinks`
  - **Output →** `Get authority metrics`
- **Key data used later:** `json.refdomains` (expected array)
- **Edge cases:** ranking/sorting defaults may not match expectations unless `order_by` is added (mentioned in sticky note, not implemented).

#### Node: Get anchor texts
- **Type / role:** SE Ranking node — fetches anchor text distribution.
- **Key configuration:**
  - **Resource:** `backlinks`
  - **Operation:** `getAnchors`
  - **Target:** `example.com`
  - **Additional fields:** `mode: as_root`, `limit: 50`
- **Input / output:**
  - **Input ←** `Get lost backlinks`
  - **Output →** `Get daily backlinks count`
- **Key data used later:** `json.anchors` (expected array)
- **Edge cases:** empty anchors may be represented as empty string; the code renders it as `[empty]`.

#### Node: Get authority metrics
- **Type / role:** SE Ranking node — fetches authority-related metrics.
- **Key configuration:**
  - **Resource:** `backlinks`
  - **Operation:** `getAuthority`
  - **Target:** `example.com`
- **Input / output:**
  - **Input ←** `Get referring domains`
  - **Output →** `Get cumulative history`
- **Key data used later:** `json.pages` (expected array of objects containing `url`, `inlink_rank`, `domain_inlink_rank`)
- **Edge cases:** if the API changes the key name (e.g., `page`/`items`), the formatting code will produce an empty section.

#### Node: Get daily backlinks count
- **Type / role:** SE Ranking node — fetches day-by-day counts of new/lost backlinks.
- **Key configuration:**
  - **Resource:** `backlinks`
  - **Operation:** `getHistoryCount`
  - **Target:** `example.com`
  - **dateFrom:** last 30 days, **dateTo:** today (same expressions as above)
  - **Additional fields:** `mode: as_root`
- **Input / output:**
  - **Input ←** `Get anchor texts`
  - **Output →** `Get pages with backlinks`
- **Key data used later:** `json.new_lost_backlinks_count` (expected array with `date`, `new`, `lost`)
- **Edge cases:** missing `new`/`lost` fields; code defaults to `0`.

#### Node: Get cumulative history
- **Type / role:** SE Ranking node — fetches cumulative backlink totals over time.
- **Key configuration:**
  - **Resource:** `backlinks`
  - **Operation:** `getCumulativeHistory`
  - **Target:** `example.com`
  - **dateFrom:** `{{ $now.minus(60, 'days').toFormat('yyyy-MM-dd') }}`
  - **dateTo:** `{{ $now.toFormat('yyyy-MM-dd') }}`
  - **Additional fields:** `mode: as_root`
- **Input / output:**
  - **Input ←** `Get authority metrics`
  - **Output →** `Merge` (input 0)
- **Key data used later:** `json.backlinks_count` (expected array with `date`, `backlinks`)
- **Edge cases:** the returned period may be shorter if SE Ranking has less historical data for the domain.

#### Node: Get pages with backlinks
- **Type / role:** SE Ranking node — fetches pages on the target that have backlinks.
- **Key configuration:**
  - **Resource:** `backlinks`
  - **Operation:** `getPages`
  - **Target:** `example.com`
  - **Additional fields:** `mode: as_root`, `limit: 100`
- **Input / output:**
  - **Input ←** `Get daily backlinks count`
  - **Output →** `Merge` (input 1)
- **Important note:** This dataset is collected and merged, **but it is not actually used** by the `Format for Sheet` code. As written, it has no effect on the exported rows.
- **Edge cases:** none beyond usual API/auth/limit issues.

---

### Block 3 — Merge, Format & Export

**Overview:** Combines the two terminal branches, then uses a Code node to build a single list of rows with section headers and blank lines, and appends those rows into Google Sheets.

**Nodes involved:**
- Merge
- Format for Sheet
- Export to Google Sheets

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines two inputs into one output.
- **Configuration:**
  - **Mode:** `combine`
  - This mode pairs/combines items from both inputs; in this workflow it functions mainly as a synchronization point before formatting.
- **Inputs / outputs:**
  - **Input 0 ←** `Get cumulative history`
  - **Input 1 ←** `Get pages with backlinks`
  - **Output →** `Format for Sheet`
- **Edge cases:**
  - If one input returns 0 items and the other returns items, `combine` behavior can yield unexpected pairing/emptiness. Since the Code node references other nodes directly (by name), the Merge output content is not actually used—Merge is effectively just a “join point”.

#### Node: Format for Sheet
- **Type / role:** `n8n-nodes-base.code` — transforms multiple node outputs into a sheet-ready row list.
- **Configuration choices (interpreted):**
  - Creates an `output` array of items like `{ json: { A: ..., B: ..., ... } }` where columns are named `A`, `B`, `C`, etc.
  - Inserts:
    - Section header row (column A: `=== SECTION ===`)
    - A header row for each section (column names)
    - Data rows for each entry
    - Five blank spacer rows between sections
  - Pulls data using n8n’s node data accessors, e.g.:
    - `$('Get backlinks summary').first().json.summary`
    - `$('Get authority metrics').first().json.pages`
    - `$('Get new backlinks').first().json.new_lost_backlinks`
    - `$('Get lost backlinks').first().json.new_lost_backlinks`
    - `$('Get referring domains').first().json.refdomains`
    - `$('Get anchor texts').first().json.anchors`
    - `$('Get daily backlinks count').first().json.new_lost_backlinks_count`
    - `$('Get cumulative history').first().json.backlinks_count`
- **Inputs / outputs:**
  - **Input ←** `Merge` (but not used in code logic)
  - **Output →** `Export to Google Sheets`
- **Version notes:** TypeVersion `2` (Code node).
- **Edge cases / failure types:**
  - If any referenced node returns **no items**, `.first()` can be `undefined` in some n8n versions/configs; however the code assumes `.first().json` exists. This can cause runtime errors. (Mitigation: optional chaining like `$('Node').first()?.json?...`.)
  - If SE Ranking node returns different property names than expected, sections will be empty.
  - Sections “NEW BACKLINKS” and “LOST BACKLINKS” both rely on `new_lost_backlinks`; correctness depends on SE Ranking API semantics for each call (no filter configured).

#### Node: Export to Google Sheets
- **Type / role:** `n8n-nodes-base.googleSheets` — appends formatted rows into a spreadsheet.
- **Configuration:**
  - **Operation:** `append`
  - **Document ID:** selected from list (not filled in JSON)
  - **Sheet name:** selected from list (not filled in JSON)
  - Appends items where each item has keys `A`, `B`, `C`, etc.
- **Credentials:** Google Sheets OAuth2 (`googleSheetsOAuth2Api`)
- **Version notes:** TypeVersion `4.7`
- **Edge cases / failure types:**
  - OAuth not connected or insufficient permissions to the spreadsheet
  - Sheet/tab name mismatch or missing
  - Appending large volumes may hit API quotas
  - Column mapping expectations: this setup assumes the node will map object keys `A`, `B`, … into columns (common pattern). If your sheet expects named headers instead, you’ll need to adjust.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual start | — | Get backlinks summary |  |
| Get backlinks summary | @seranking/n8n-nodes-seranking.seRanking | Fetch backlink summary snapshot | Manual Trigger | Get new backlinks; Get lost backlinks | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Get new backlinks | @seranking/n8n-nodes-seranking.seRanking | Fetch backlink history (intended “new”) for last 30 days | Get backlinks summary | Get referring domains | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Get lost backlinks | @seranking/n8n-nodes-seranking.seRanking | Fetch backlink history (intended “lost”) for last 30 days | Get backlinks summary | Get anchor texts | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Get referring domains | @seranking/n8n-nodes-seranking.seRanking | Fetch referring domains list | Get new backlinks | Get authority metrics | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Get anchor texts | @seranking/n8n-nodes-seranking.seRanking | Fetch anchor text distribution | Get lost backlinks | Get daily backlinks count | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Get authority metrics | @seranking/n8n-nodes-seranking.seRanking | Fetch authority metrics | Get referring domains | Get cumulative history | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Get daily backlinks count | @seranking/n8n-nodes-seranking.seRanking | Fetch daily new/lost backlink counts (30 days) | Get anchor texts | Get pages with backlinks | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Get cumulative history | @seranking/n8n-nodes-seranking.seRanking | Fetch cumulative backlink totals (60 days) | Get authority metrics | Merge | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Get pages with backlinks | @seranking/n8n-nodes-seranking.seRanking | Fetch target pages that have backlinks | Get daily backlinks count | Merge | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Merge | n8n-nodes-base.merge | Synchronize/combine final branches | Get cumulative history; Get pages with backlinks | Format for Sheet | ### 📤 Formatting & Export<br>Combines all the data, organizes it into clear sections with headers, and saves to Google Sheets. |
| Format for Sheet | n8n-nodes-base.code | Build a single tabular export with section headers | Merge | Export to Google Sheets | ### 📤 Formatting & Export<br>Combines all the data, organizes it into clear sections with headers, and saves to Google Sheets. |
| Export to Google Sheets | n8n-nodes-base.googleSheets | Append rows to Google Sheets | Format for Sheet | — | ### 📤 Formatting & Export<br>Combines all the data, organizes it into clear sections with headers, and saves to Google Sheets. |
| Workflow Overview | n8n-nodes-base.stickyNote | Documentation / instructions | — | — | ## Monitor backlink profile with SE Ranking<br><br>## Who is this for<br>- SEO pros tracking client link building progress<br>- Website owners watching their backlink growth<br>- Digital marketers analyzing domain authority trends<br><br>## What this workflow does<br>Check your backlink profile to see which sites are linking to you, track new links you've gained, spot links you've lost, and export everything to Google Sheets.<br><br>## What you'll get<br>- Total backlinks and referring domains snapshot<br>- New backlinks you gained in the last 30 days<br>- Lost backlinks from the last 30 days (with reasons why)<br>- Your top referring domains ranked by authority<br>- Anchor text analysis showing how sites link to you<br>- Domain authority scores and trust metrics<br>- Daily trends showing link gains and losses<br>- Historical growth data over the last 60 days<br><br>## How it works<br>1. Pulls your overall backlink profile stats<br>2. Checks for new backlinks from the last month<br>3. Identifies which backlinks you lost recently<br>4. Lists your top referring domains by authority<br>5. Analyzes the anchor text people use to link to you<br>6. Gets your domain authority and trust scores<br>7. Shows daily gains and losses over 30 days<br>8. Grabs historical data to see growth over 60 days<br>9. Organizes everything into clear sections in a spreadsheet<br><br>## Requirements<br>- Self-hosted n8n instance<br>- SE Ranking community node installed<br>- SE Ranking API token ([Get one here](https://online.seranking.com/admin.api.dashboard.html))<br>- Google Sheets account (optional)<br><br>## Setup<br>1. Install the [SE Ranking community node](https://www.npmjs.com/package/@seranking/n8n-nodes-seranking)<br>2. Add your SE Ranking API credentials<br>3. Replace `example.com` with your domain in all SE Ranking nodes<br>4. Connect Google Sheets if you want to export data (optional)<br><br>## Customization<br>- Change the time periods (currently 30 or 60 days) by editing `dateFrom` and `dateTo`<br>- Get more or fewer results by adjusting the `limit` settings<br>- Sort by different metrics by changing `order_by` (like first_seen or backlinks)<br>- Add a Schedule Trigger to run this automatically every week |
| Data Collection | n8n-nodes-base.stickyNote | Comment block label | — | — | ### 📊 Data Collection<br>Grabs backlink data from SE Ranking: summary stats, new/lost links, referring domains, anchor texts, authority metrics, and historical trends. |
| Formatting & Export | n8n-nodes-base.stickyNote | Comment block label | — | — | ### 📤 Formatting & Export<br>Combines all the data, organizes it into clear sections with headers, and saves to Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Install prerequisites**
   1. Run n8n in an environment that supports community nodes (often self-hosted).
   2. Install SE Ranking community node: https://www.npmjs.com/package/@seranking/n8n-nodes-seranking
   3. Obtain an SE Ranking API token: https://online.seranking.com/admin.api.dashboard.html

2. **Create credentials**
   1. In n8n, create **SE Ranking API** credentials for the community node (paste your API token).
   2. (Optional export) Create **Google Sheets OAuth2** credentials and connect the Google account that has access to the target spreadsheet.

3. **Create the trigger**
   1. Add node **Manual Trigger**.
   2. Leave defaults.

4. **Add SE Ranking nodes (use the same Target everywhere)**
   - Replace `example.com` with your real domain in all nodes.
   1. Add **Get backlinks summary** (SE Ranking node)
      - Resource: `backlinks`
      - Target: your domain
      - Connect: `Manual Trigger → Get backlinks summary`
   2. Add **Get new backlinks**
      - Resource: `backlinks`
      - Operation: `getHistory`
      - dateFrom: `{{ $now.minus(30, 'days').toFormat('yyyy-MM-dd') }}`
      - dateTo: `{{ $now.toFormat('yyyy-MM-dd') }}`
      - Additional: `mode=as_root`, `limit=100`
      - Connect: `Get backlinks summary → Get new backlinks`
   3. Add **Get lost backlinks**
      - Same settings as “Get new backlinks” (as in the provided workflow)
      - Connect: `Get backlinks summary → Get lost backlinks`
   4. Add **Get referring domains**
      - Operation: `getRefDomains`
      - Additional: `mode=as_root`, `limit=100`
      - Connect: `Get new backlinks → Get referring domains`
   5. Add **Get authority metrics**
      - Operation: `getAuthority`
      - Connect: `Get referring domains → Get authority metrics`
   6. Add **Get cumulative history**
      - Operation: `getCumulativeHistory`
      - dateFrom: `{{ $now.minus(60, 'days').toFormat('yyyy-MM-dd') }}`
      - dateTo: `{{ $now.toFormat('yyyy-MM-dd') }}`
      - Additional: `mode=as_root`
      - Connect: `Get authority metrics → Get cumulative history`
   7. Add **Get anchor texts**
      - Operation: `getAnchors`
      - Additional: `mode=as_root`, `limit=50`
      - Connect: `Get lost backlinks → Get anchor texts`
   8. Add **Get daily backlinks count**
      - Operation: `getHistoryCount`
      - dateFrom: last 30 days expression (above)
      - dateTo: today expression (above)
      - Additional: `mode=as_root`
      - Connect: `Get anchor texts → Get daily backlinks count`
   9. Add **Get pages with backlinks**
      - Operation: `getPages`
      - Additional: `mode=as_root`, `limit=100`
      - Connect: `Get daily backlinks count → Get pages with backlinks`
   10. Ensure every SE Ranking node uses the **SE Ranking credential** you created.

5. **Add Merge**
   1. Add node **Merge**
   2. Mode: `Combine`
   3. Connect:
      - `Get cumulative history → Merge (Input 1 / index 0)`
      - `Get pages with backlinks → Merge (Input 2 / index 1)`

6. **Add formatting Code node**
   1. Add node **Code** named **Format for Sheet**
   2. Paste the provided JavaScript (the workflow’s code) and keep it as-is.
   3. Connect: `Merge → Format for Sheet`

7. **Add Google Sheets export**
   1. Add node **Google Sheets** named **Export to Google Sheets**
   2. Operation: **Append**
   3. Pick:
      - **Document ID** (your spreadsheet)
      - **Sheet name** (tab)
   4. Connect: `Format for Sheet → Export to Google Sheets`
   5. Select your **Google Sheets OAuth2** credential.

8. **(Optional) Make it automatic**
   1. Replace Manual Trigger with **Schedule Trigger** (e.g., weekly).
   2. Keep downstream connections identical.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SE Ranking API token | https://online.seranking.com/admin.api.dashboard.html |
| SE Ranking community node package | https://www.npmjs.com/package/@seranking/n8n-nodes-seranking |
| Configuration reminders from workflow notes | Replace `example.com` everywhere; adjust `dateFrom/dateTo`; adjust `limit`; optionally set `order_by`; optionally add a Schedule Trigger. |