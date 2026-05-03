Fetch Threads analytics for your latest posts and email an HTML report

https://n8nworkflows.xyz/workflows/fetch-threads-analytics-for-your-latest-posts-and-email-an-html-report-13509


# Fetch Threads analytics for your latest posts and email an HTML report

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Fetch Threads analytics for your latest posts and email an HTML report  
**Workflow name (in JSON):** Fetch Threads posts and send an HTML analytics report via email

**Purpose:**  
Fetch up to 100 latest Threads posts using the Threads Graph API, retrieve per-post engagement insights (views, likes, replies, reposts, quotes), assemble an HTML report (totals + table), and send it via Gmail as an inline HTML email.

**Primary use cases:**
- Periodic performance monitoring of Threads content
- Email-based reporting for creators/marketers
- Lightweight analytics without external BI tooling

### 1.1 Triggers (Entry Points)
Two entry points start the same pipeline:
- Manual run for ad-hoc reporting
- Scheduled run every 2 days (default at 23:00)

### 1.2 Token Retrieval & Preparation
Reads a long-lived Threads access token from an n8n Data Table row filtered by `Platform = "Threads"` and maps it to a `Threads_Token` field for downstream usage.

### 1.3 Fetch Posts
Calls `GET /v1.0/me/threads` to retrieve posts (id, text, timestamp, permalink), then splits the returned `data[]` array into one item per post.

### 1.4 Fetch Insights Per Post
For each post item, calls `GET /v1.0/{post_id}/insights` with metrics (views, likes, replies, reposts, quotes).

### 1.5 Merge + Build HTML + Email
Merges each post’s base data with its insight metrics, builds a styled HTML report with totals and a newest-first table, then emails it via Gmail.

---

## 2. Block-by-Block Analysis

### Block A — Triggering & Run Initiation
**Overview:** Starts the workflow either manually or on a fixed schedule, feeding into token retrieval.  
**Nodes involved:**  
- Manual Trigger  
- Schedule Trigger (Every 2 Days)

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual entry point for testing or on-demand runs.
- **Configuration:** No parameters.
- **Connections:**  
  - **Output →** Get Threads Token
- **Edge cases / failures:** None (other than user not executing).
- **Version notes:** typeVersion `1`.

#### Node: Schedule Trigger (Every 2 Days)
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — automatic entry point.
- **Configuration choices:**
  - Runs every **2 days** at **23:00** (`triggerAtHour: 23`).
- **Connections:**  
  - **Output →** Get Threads Token
- **Edge cases / failures:**
  - Timezone depends on n8n instance settings; may not match user expectation.
  - If the instance is down at trigger time, executions may be missed depending on n8n scheduling behavior.
- **Version notes:** typeVersion `1.2`.

---

### Block B — Token Retrieval & Normalization
**Overview:** Fetches a Threads token from an n8n Data Table and normalizes it into a predictable field name used by HTTP requests.  
**Nodes involved:**  
- Get Threads Token  
- Set Token

#### Node: Get Threads Token
- **Type / role:** `n8n-nodes-base.dataTable` — reads stored token from n8n Data Tables.
- **Configuration choices (interpreted):**
  - **Operation:** `get`
  - **Data Table:** `YOUR_DATA_TABLE_ID` (must be replaced with a real table)
  - **Filter:** `Platform` equals `"Threads"`
  - Expects the returned row to include a `Token` field.
- **Connections:**  
  - **Input ←** Manual Trigger OR Schedule Trigger  
  - **Output →** Set Token
- **Key fields/variables:**
  - Uses Data Table column `Platform` for filtering.
  - Uses output field `$json.Token`.
- **Edge cases / failures:**
  - No matching row → downstream expressions may fail (missing token).
  - Multiple matches → behavior depends on Data Table node result; could return multiple items and unintentionally trigger multiple fetches.
  - Permissions/access to Data Table.
- **Version notes:** typeVersion `1`.

#### Node: Set Token
- **Type / role:** `n8n-nodes-base.set` — creates a stable token field for later nodes.
- **Configuration choices:**
  - Sets `Threads_Token = $json.Token`
  - Keeps only as configured by Set node behavior (here it’s simple assignment; other fields may pass through depending on defaults).
- **Connections:**  
  - **Input ←** Get Threads Token  
  - **Output →** Fetch Threads Posts
- **Key expressions:**
  - `Threads_Token` value: `{{ $json.Token }}`
- **Edge cases / failures:**
  - If `Token` is missing/empty, the Authorization header becomes invalid, causing 401s.
- **Version notes:** typeVersion `3.4`.

---

### Block C — Fetch Posts & Split into Items
**Overview:** Pulls the latest posts from Threads and converts the returned array into individual items for per-post processing.  
**Nodes involved:**  
- Fetch Threads Posts  
- Split Out

#### Node: Fetch Threads Posts
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Threads Graph API to fetch posts.
- **Configuration choices:**
  - **Method:** (not explicitly set; defaults to GET in n8n HTTP Request)
  - **URL:** `https://graph.threads.net/v1.0/me/threads`
  - **Query params:**
    - `fields = id,text,timestamp,permalink`
    - `locale = zh_TW`
    - `limit = 100`
  - **Headers:**
    - `Authorization: Bearer {{ $json.Threads_Token }}`
- **Connections:**  
  - **Input ←** Set Token  
  - **Output →** Split Out
- **Key expressions:**
  - Authorization header uses the token produced by Set Token.
- **Edge cases / failures:**
  - 401/403 if token expired/invalid or missing scopes.
  - Rate limiting: Threads API daily request limits apply.
  - Response shape changes (e.g., missing `data`) would break Split Out.
  - Locale parameter may be unnecessary; if unsupported it may be ignored.
- **Version notes:** typeVersion `4.2`.

#### Node: Split Out
- **Type / role:** `n8n-nodes-base.splitOut` — splits an array field into one item per element.
- **Configuration choices:**
  - Splits field `data` (expects `response.data` array from Threads API).
- **Connections:**  
  - **Input ←** Fetch Threads Posts  
  - **Output →** Fetch Post Insights
- **Edge cases / failures:**
  - If `data` is missing/null, yields 0 items or errors depending on node behavior/settings.
- **Version notes:** typeVersion `1`.

---

### Block D — Fetch Insights Per Post
**Overview:** For each post item, fetches insights metrics from Threads API.  
**Nodes involved:**  
- Fetch Post Insights

#### Node: Fetch Post Insights
- **Type / role:** `n8n-nodes-base.httpRequest` — per-item insights lookup.
- **Configuration choices:**
  - **URL (expression):** `https://graph.threads.net/v1.0/{{$json["id"]}}/insights`
    - Uses the post `id` from each Split Out item.
  - **Query params:**
    - `metric = views,likes,replies,reposts,quotes`
    - `locale = zh_TW`
  - **Headers:**
    - `Authorization: Bearer {{ $('Set Token').item.json.Threads_Token }}`
      - Notably references the token from the **Set Token** node directly.
- **Connections:**  
  - **Input ←** Split Out  
  - **Output →** Merge Posts and Insights
- **Key expressions / variables:**
  - Post id: `$json["id"]`
  - Token: `$('Set Token').item.json.Threads_Token`
- **Edge cases / failures:**
  - Token lookup via `$('Set Token').item` assumes that item context is accessible; if execution/data mapping changes, could break. A safer pattern is passing `Threads_Token` along with each item.
  - Rate limits: up to ~100 calls per run if 100 posts. The sticky note warns of API daily limits.
  - Partial failures: if some insight calls fail, merge logic may misalign posts/insights (see next block).
- **Version notes:** typeVersion `4.2`.

---

### Block E — Merge, Report Generation, Email Delivery
**Overview:** Combines post metadata with insight metrics, renders an HTML report with totals and a table, and emails it via Gmail.  
**Nodes involved:**  
- Merge Posts and Insights  
- Build HTML Report  
- Email Analytics Report

#### Node: Merge Posts and Insights
- **Type / role:** `n8n-nodes-base.code` — joins two streams: posts from Split Out and insights from current input.
- **Configuration choices (logic):**
  - Reads all post items via `$items('Split Out')`.
  - Iterates over current `items` (insights results) and pairs them by array index with posts from Split Out.
  - Converts insights `data[]` array into a metrics object `m` keyed by `name` (likes, replies, etc.).
  - Builds unified item schema:
    - `id, text, timestamp, permalink, likes, replies, reposts, quotes, views`
  - Timestamp fallback order: `post.timestamp || post.created_time || post.created_at || ''`
  - Views fallback: `m.views ?? m.impressions ?? 0`
- **Connections:**  
  - **Input ←** Fetch Post Insights  
  - **Output →** Build HTML Report
- **Edge cases / failures:**
  - **Index-based join risk:** if any insights request fails or returns fewer items, posts and insights can become misaligned (wrong metrics attached to wrong post).
  - If insights API returns unexpected shape (missing `data`, missing `values[0].value`), metrics default to 0 (safe), but misalignment still possible.
- **Version notes:** typeVersion `2`.

#### Node: Build HTML Report
- **Type / role:** `n8n-nodes-base.code` — creates final HTML string for email.
- **Configuration choices (logic):**
  - If `items.length === 0`, outputs `{ hasItems:false, html:'<p>No posts found.</p>' }`
  - Escapes `<` and `>` in post text to reduce HTML injection risk.
  - Converts timestamps to local string via `new Date(ts).toLocaleString()`.
  - Computes totals across all items: posts, likes, replies, reposts, quotes, views.
  - Sorts rows by `timestamp` descending (newest first).
  - Truncates text preview to 200 chars and adds ellipsis.
  - Generates HTML with:
    - Title: “Threads Analytics Report (N posts)”
    - Totals line
    - Table with columns: Time, Content, 👍, 💬, 🔁, 🗨️, 👁️, Link
- **Connections:**  
  - **Input ←** Merge Posts and Insights  
  - **Output →** Email Analytics Report
- **Edge cases / failures:**
  - Invalid timestamp strings may result in “Invalid Date” or sorting anomalies.
  - Only `<` and `>` are escaped; other characters (e.g., `&`) are not escaped, which can still affect HTML rendering in rare cases.
  - Includes emoji in HTML content; should render in most clients, but not guaranteed.
- **Version notes:** typeVersion `2`.

#### Node: Email Analytics Report
- **Type / role:** `n8n-nodes-base.gmail` — sends the generated HTML report via Gmail.
- **Configuration choices:**
  - **To:** `YOUR_EMAIL_ADDRESS` (must be replaced)
  - **Subject:** `Threads Analytics Report (Latest 100 Posts)`
  - **Message body (expression):** `{{ $json.html }}`
  - Sends HTML inline (as provided).
- **Connections:**  
  - **Input ←** Build HTML Report  
  - **Output:** none (terminal node)
- **Credential requirements:**
  - Gmail OAuth2 connection in n8n (Google account authorized for sending).
- **Edge cases / failures:**
  - Gmail auth expired/revoked.
  - Sending limits/quota (Google account restrictions).
  - If `$json.html` is missing (prior node error), email may send blank or fail.
- **Version notes:** typeVersion `2.1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual entry point | — | Get Threads Token | ## How it works  This workflow pulls up to 100 of your latest Threads posts along with their engagement metrics — views, likes, replies, reposts and quotes — directly from the Threads Graph API.  For each post, a separate insights API call retrieves engagement data. The results are merged into a unified dataset, then a Code node builds a clean HTML table sorted by publish time (newest first), with a totals summary at the top. The report is delivered as an inline HTML email via Gmail.  Supports both manual execution and automatic scheduling (every 2 days by default).  ## Setup steps  1. Get a long-lived Threads access token from Meta for Developers → Threads API → Access Tokens. Enter it in the **Get Threads Token** node (Data Table, Platform = "Threads"). 2. Open **Email Analytics Report**, connect your Google account, and update the recipient email address. 3. (Optional) Adjust the interval in **Schedule Trigger** — default is every 2 days at 11 pm.  _Note: The Threads API allows up to 500 requests/day. Fetching 100 posts uses approximately 100 insight API calls._ |
| Schedule Trigger (Every 2 Days) | Schedule Trigger | Scheduled entry point | — | Get Threads Token | ## Triggers  Run manually on demand or on a schedule. Default: every 2 days at 11 pm. Adjust the interval in the Schedule Trigger node. |
| Get Threads Token | Data Table | Retrieve Threads token from storage | Manual Trigger; Schedule Trigger (Every 2 Days) | Set Token | ## Token setup  Reads the Threads long-lived access token from an n8n Data Table and passes it to the downstream fetch nodes. |
| Set Token | Set | Normalize token field | Get Threads Token | Fetch Threads Posts | ## Token setup  Reads the Threads long-lived access token from an n8n Data Table and passes it to the downstream fetch nodes. |
| Fetch Threads Posts | HTTP Request | Fetch latest Threads posts | Set Token | Split Out | ## Fetch posts & insights  Calls the Threads API to retrieve up to 100 posts (id, text, timestamp, permalink), splits them into individual items, then fetches engagement metrics for each post. |
| Split Out | Split Out | Convert posts array into items | Fetch Threads Posts | Fetch Post Insights | ## Fetch posts & insights  Calls the Threads API to retrieve up to 100 posts (id, text, timestamp, permalink), splits them into individual items, then fetches engagement metrics for each post. |
| Fetch Post Insights | HTTP Request | Fetch metrics per post | Split Out | Merge Posts and Insights | ## Fetch posts & insights  Calls the Threads API to retrieve up to 100 posts (id, text, timestamp, permalink), splits them into individual items, then fetches engagement metrics for each post. |
| Merge Posts and Insights | Code | Combine post + metrics datasets | Fetch Post Insights | Build HTML Report | ## Build & send report  Merges post data with insights, builds a styled HTML table, and sends it as an inline email to the address configured in the Email Analytics Report node. |
| Build HTML Report | Code | Render totals + HTML table | Merge Posts and Insights | Email Analytics Report | ## Build & send report  Merges post data with insights, builds a styled HTML table, and sends it as an inline email to the address configured in the Email Analytics Report node. |
| Email Analytics Report | Gmail | Send HTML email | Build HTML Report | — | ## Build & send report  Merges post data with insights, builds a styled HTML table, and sends it as an inline email to the address configured in the Email Analytics Report node. |
| Sticky Note - Main | Sticky Note | Documentation | — | — | ## How it works  This workflow pulls up to 100 of your latest Threads posts along with their engagement metrics — views, likes, replies, reposts and quotes — directly from the Threads Graph API.  For each post, a separate insights API call retrieves engagement data. The results are merged into a unified dataset, then a Code node builds a clean HTML table sorted by publish time (newest first), with a totals summary at the top. The report is delivered as an inline HTML email via Gmail.  Supports both manual execution and automatic scheduling (every 2 days by default).  ## Setup steps  1. Get a long-lived Threads access token from Meta for Developers → Threads API → Access Tokens. Enter it in the **Get Threads Token** node (Data Table, Platform = "Threads"). 2. Open **Email Analytics Report**, connect your Google account, and update the recipient email address. 3. (Optional) Adjust the interval in **Schedule Trigger** — default is every 2 days at 11 pm.  _Note: The Threads API allows up to 500 requests/day. Fetching 100 posts uses approximately 100 insight API calls._ |
| Sticky Note - Triggers | Sticky Note | Documentation | — | — | ## Triggers  Run manually on demand or on a schedule. Default: every 2 days at 11 pm. Adjust the interval in the Schedule Trigger node. |
| Sticky Note - Token setup | Sticky Note | Documentation | — | — | ## Token setup  Reads the Threads long-lived access token from an n8n Data Table and passes it to the downstream fetch nodes. |
| Sticky Note - Fetch | Sticky Note | Documentation | — | — | ## Fetch posts & insights  Calls the Threads API to retrieve up to 100 posts (id, text, timestamp, permalink), splits them into individual items, then fetches engagement metrics for each post. |
| Sticky Note - Build and send | Sticky Note | Documentation | — | — | ## Build & send report  Merges post data with insights, builds a styled HTML table, and sends it as an inline email to the address configured in the Email Analytics Report node. |
| Sticky Note - More Templates | Sticky Note | Documentation / promotion link | — | — | ## More Threads automation  Want to auto-publish Threads posts from a Notion CMS on a schedule?  Check out the **Content Automation Bundle** — includes Threads Publisher, LinkedIn Publisher, Facebook Publisher, AI Content Rewriter and more.  👉 https://jasonchuang0818.gumroad.com/l/n8n-content-automation-bundle |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   “Fetch Threads posts and send an HTML analytics report via email”.

2. **Add triggers (two entry points):**
   1) Add **Manual Trigger** node.  
   2) Add **Schedule Trigger** node:
      - Interval: **Every 2 days**
      - Trigger time: **23:00** (11 pm)
      - Ensure timezone matches your reporting expectation (instance timezone).

3. **Create/store your Threads token source:**
   - Create (or choose) an **n8n Data Table** (in n8n’s Data Tables feature).
   - Add at least one row with columns such as:
     - `Platform` = `Threads`
     - `Token` = `<YOUR_LONG_LIVED_THREADS_ACCESS_TOKEN>`

4. **Add “Get Threads Token” node (Data Table):**
   - Node type: **Data Table**
   - Operation: **Get**
   - Table: select your table (replace `YOUR_DATA_TABLE_ID`)
   - Filter/conditions:
     - Key: `Platform`
     - Value: `Threads`
   - Connect:
     - **Manual Trigger → Get Threads Token**
     - **Schedule Trigger → Get Threads Token**

5. **Add “Set Token” node (Set):**
   - Add a field:
     - Name: `Threads_Token`
     - Type: String
     - Value expression: `{{ $json.Token }}`
   - Connect:
     - **Get Threads Token → Set Token**

6. **Add “Fetch Threads Posts” node (HTTP Request):**
   - Method: **GET**
   - URL: `https://graph.threads.net/v1.0/me/threads`
   - Enable **Send Query Parameters**:
     - `fields` = `id,text,timestamp,permalink`
     - `locale` = `zh_TW` (optional; keep to match workflow)
     - `limit` = `100`
   - Enable **Send Headers**:
     - `Authorization` = `Bearer {{ $json.Threads_Token }}`
   - Connect:
     - **Set Token → Fetch Threads Posts**

7. **Add “Split Out” node (Split Out):**
   - Field to split out: `data`
   - Connect:
     - **Fetch Threads Posts → Split Out**

8. **Add “Fetch Post Insights” node (HTTP Request):**
   - Method: **GET**
   - URL (expression): `https://graph.threads.net/v1.0/{{ $json.id }}/insights`
   - Query parameters:
     - `metric` = `views,likes,replies,reposts,quotes`
     - `locale` = `zh_TW`
   - Headers:
     - `Authorization` = `Bearer {{ $('Set Token').item.json.Threads_Token }}`
       - (Optional improvement: pass `Threads_Token` forward so you can use `{{ $json.Threads_Token }}` here.)
   - Connect:
     - **Split Out → Fetch Post Insights**

9. **Add “Merge Posts and Insights” node (Code):**
   - Paste logic equivalent to:
     - Load posts with `$items('Split Out')`
     - Map insight results to metrics per item
     - Return unified items with: id/text/timestamp/permalink + metrics
   - Connect:
     - **Fetch Post Insights → Merge Posts and Insights**

10. **Add “Build HTML Report” node (Code):**
    - Implement:
      - Empty case → “No posts found.”
      - Totals aggregation
      - Sort by timestamp desc
      - Generate HTML string with totals + table + permalink links
    - Connect:
      - **Merge Posts and Insights → Build HTML Report**

11. **Add “Email Analytics Report” node (Gmail):**
    - Set up **Gmail OAuth2 credentials** (connect your Google account).
    - To: set your recipient email (replace `YOUR_EMAIL_ADDRESS`)
    - Subject: `Threads Analytics Report (Latest 100 Posts)`
    - Message/body (expression): `{{ $json.html }}`
    - Connect:
      - **Build HTML Report → Email Analytics Report**

12. **Validate execution:**
    - Run via **Manual Trigger** first.
    - Confirm:
      - Posts are returned (`data[]` exists)
      - Insights calls succeed (no 401/rate-limit)
      - Email arrives with a populated HTML table

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Threads API request limit note: “Threads API allows up to 500 requests/day. Fetching 100 posts uses approximately 100 insight API calls.” | From Sticky Note - Main |
| “More Threads automation… Content Automation Bundle…” | https://jasonchuang0818.gumroad.com/l/n8n-content-automation-bundle |
| Setup guidance: obtain long-lived Threads access token from Meta for Developers → Threads API → Access Tokens. | From Sticky Note - Main |
| Default schedule: every 2 days at 11 pm; adjust in Schedule Trigger node. | From Sticky Note - Main / Triggers |

