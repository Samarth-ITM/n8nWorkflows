Monitor Docker host health via SSH with GPT-4o-mini and alerts to Discord

https://n8nworkflows.xyz/workflows/monitor-docker-host-health-via-ssh-with-gpt-4o-mini-and-alerts-to-discord-13565


# Monitor Docker host health via SSH with GPT-4o-mini and alerts to Discord

## 1. Workflow Overview

**Workflow name:** *Monitor Docker host health via SSH with AI analysis and alerts to Discord*  
**Purpose:** Automated health monitoring for a Docker host (“homelab” style). It gathers OS + Docker metrics over SSH, normalizes them into structured data, optionally compares against recent history stored in Google Sheets, has GPT‑4o‑mini produce actionable findings as strict JSON, and delivers a multi-embed dashboard to Discord. In parallel, it runs a lightweight check every 5 minutes to send **immediate Discord alerts** on critical thresholds. A one-time manual path creates and formats the Google Sheet used for history/trends.

### 1.1 Daily Digest Path (07:00)
- Trigger daily
- Load user config (webhook, thresholds, Sheet ID)
- SSH collect system metrics + Docker metrics
- Parse/normalize metrics + compute health score
- Read historical metrics from Google Sheets
- Build AI prompt with current + historical context
- LLM returns structured JSON findings
- Build 4-embed Discord payload and send
- Store today’s metrics row back to Google Sheets

### 1.2 Critical Alerts Path (every 5 minutes)
- Trigger every 5 minutes
- Load alert thresholds (separate config node)
- Run quick SSH command
- Compare against critical thresholds
- If anything critical is found, format and send alert to Discord

### 1.3 First-Time Setup Path (manual)
- Manual trigger
- Create a new Google Spreadsheet with a `metrics` sheet
- Apply formatting (headers, frozen row, column widths, conditional formatting) via Sheets API batchUpdate

---

## 2. Block-by-Block Analysis

### Block A — Documentation / On-canvas Notes
**Overview:** Sticky notes explain what each area does, required credentials, and setup steps. They do not affect execution.  
**Nodes involved:**  
- Sticky Note - Overview  
- Sticky Note - Configuration  
- Sticky Note - Data Collection  
- Sticky Note - AI Analysis  
- Sticky Note - Delivery  
- Sticky Note - Critical Alerts  
- Sticky Note - API Cost Warning  
- Sticky Note - First-Time Setup  

**Node details (all Sticky Note nodes)**
- **Type/role:** `n8n-nodes-base.stickyNote` — visual documentation only.
- **Configuration choices:** Width/height/color/content define layout and text.
- **Connections:** None (no inputs/outputs).
- **Failure modes:** None (non-executing).

---

### Block B — Daily Digest: Trigger + Central Configuration
**Overview:** Starts the daily run and sets all user-editable settings in one place (Discord webhook, thresholds, history days, Sheet ID).  
**Nodes involved:**  
- Run daily health check  
- ⚙️ Configure monitoring settings

#### Node: Run daily health check
- **Type/role:** `scheduleTrigger` — daily entry point.
- **Config:** Cron `0 7 * * *` (7:00 AM). Workflow timezone is set to **America/Chicago** at workflow level.
- **Outputs:** To **⚙️ Configure monitoring settings**.
- **Edge cases:** Cron fires according to workflow timezone; ensure n8n instance timezone doesn’t conflict with workflow timezone expectations.

#### Node: ⚙️ Configure monitoring settings
- **Type/role:** `set` — centralized configuration for the daily digest path.
- **Config (assigned fields):**
  - `discord_webhook_url` (placeholder)
  - Warning/critical thresholds: `disk_warning_pct`, `disk_critical_pct`, `memory_warning_pct`, `memory_critical_pct`, `inode_critical_pct`
  - `container_restart_threshold` (present but not directly used by the critical path code)
  - `history_days` (used conceptually; prompt builder hard-slices last 7 items)
  - `google_sheet_id` (placeholder)
- **Outputs:** To **Collect system metrics**.
- **Edge cases:**
  - If `google_sheet_id` remains `YOUR_GOOGLE_SHEET_ID`, Sheets nodes will fail (handled by “continue” on read; store will fail without valid ID).
  - Discord webhook placeholder will cause HTTP 404/401-like errors from Discord.

---

### Block C — Daily Digest: SSH Data Collection
**Overview:** Runs two SSH commands: one for system metrics and one for Docker metrics. Both are designed to return a single `stdout` blob that is later parsed.  
**Nodes involved:**  
- Collect system metrics  
- Collect Docker container metrics

#### Node: Collect system metrics
- **Type/role:** `ssh` — remote command execution against the Docker host.
- **Authentication:** Private key (`sshPrivateKey` credential: “Projectbox SSH Key”).
- **Command behavior:** Emits `KEY=VALUE` lines plus section markers:
  - Hostname, boot time (`uptime -s`), `/proc/loadavg`
  - Memory (from `free -m`)
  - Root disk percent and inode percent (`df`, `df -i`)
  - Swap usage
  - CPU temperature from `/sys/class/thermal/...` (may be missing)
  - CPU utilization computed via `/proc/stat` delta across 1 second
  - `---FS---` followed by `df -P` for all non-tmpfs/devtmpfs/squashfs filesystems
  - `---NET---` parsed from `/proc/net/dev` excluding lo/docker/bridge/veth
  - `---TOP5CPU---` from `ps aux --sort=-%cpu` (top 5)
  - Zombies, failed systemd services count, established connections
- **onError:** `continueRegularOutput` (workflow continues even if SSH fails).
- **Outputs:** To **Collect Docker container metrics**.
- **Edge cases / failures:**
  - Host lacking `systemctl` (non-systemd) → failed services may error; command suppresses some errors with `2>/dev/null`.
  - CPU temp file missing → returns `N/A` (handled later).
  - Permission issues for some `/proc` or `ss` commands (usually fine as non-root; but Docker commands are in the other node).
  - SSH connectivity/auth failures; node continues but downstream parsing may see empty stdout.

#### Node: Collect Docker container metrics
- **Type/role:** `ssh` — Docker-focused remote command execution.
- **Authentication:** Same SSH private key credential.
- **Command behavior:** Emits:
  - `RUNNING=...`, `TOTAL=...`
  - `docker ps -a --format 'name|status|image'`
  - `---STATS---` then `docker stats --no-stream --format '...'`
  - `---HEALTH---` then `docker inspect` for running containers (restart count + health status if defined)
  - `---DISKUSAGE---` then `docker system df --format ...`
  - `DANGLING=...` and `UNUSED_VOLS=...`
- **onError:** `continueRegularOutput`.
- **Outputs:** To **Parse and normalize all metrics**.
- **Edge cases / failures:**
  - Docker not installed, daemon down, or user not in `docker` group → many commands fail; output may be partial/empty.
  - `docker inspect ... $(docker ps -q)` returns nothing if no running containers; stderr suppressed.

---

### Block D — Daily Digest: Parsing, Normalization, Health Score
**Overview:** Converts the two SSH `stdout` blobs into structured JSON metrics, computes derived values (CPU%), aggregates network totals, parses container stats/health, and calculates a 0–100 health score.  
**Nodes involved:**  
- Parse and normalize all metrics

#### Node: Parse and normalize all metrics
- **Type/role:** `code` — transforms unstructured command output into structured data.
- **Key inputs:**
  - `$('Collect system metrics').first().json.stdout`
  - `$('Collect Docker container metrics').first().json.stdout`
- **Key logic:**
  - Parses simple `KEY=VALUE` pairs from system output.
  - CPU% computed from `CPU_DELTA` (two `/proc/stat` snapshots).
  - Extracts memory%, disk%, inode%, swap%.
  - Converts CPU temp to °C if available.
  - Parses filesystem list from the `---FS---` section.
  - Sums network RX/TX bytes and errors across interfaces from `---NET---`.
  - Parses top 5 CPU processes from `---TOP5CPU---`.
  - Docker parsing splits into sections: containers list, stats map, health map, disk usage.
  - Computes:
    - `containers_down` = count of containers whose status does not start with `Up`
    - `total_restarts` sum from inspect restart counts
    - `health_score` (100 minus penalties for disk/mem/cpu/load/inodes/swap/restarts/down/zombies/failed services, plus minor image reclaimable penalty)
- **Outputs:** Single item with a large `json` object containing:
  - Timestamp, hostname, uptime days, CPU load/percent, memory totals and %, disk %, inode %, swap %, temps, filesystems array, network totals, top processes, zombies/services/connections, docker container summary array, docker disk summary, and computed health score.
- **Edge cases / failures:**
  - Empty SSH outputs → many fields default to `0`, `'N/A'`, or empty arrays; score may remain high (100) even though data is missing.
  - Filesystem parsing assumes `df -P` columns and uses `cols[5]` for mountpoint; odd whitespace or exotic `df` formats could misparse.
  - Docker stats/inspect output differences across versions could impact parsing (format strings help stabilize this).

---

### Block E — Daily Digest: Load History + Build AI Prompt
**Overview:** Loads up to the last 7 days of metrics from Google Sheets, then builds a context-rich prompt instructing the LLM to output **only valid JSON** with actionable findings and commands.  
**Nodes involved:**  
- Read metrics history (last 7 days)  
- Build analysis prompt with history

#### Node: Read metrics history (last 7 days)
- **Type/role:** `googleSheets` — reads rows from an existing spreadsheet.
- **Config:**
  - Document ID from expression: `={{ $('⚙️ Configure monitoring settings').first().json.google_sheet_id }}`
  - Sheet name: `metrics`
  - `alwaysOutputData: true`
  - `onError: continueRegularOutput`
- **Outputs:** Rows as items (each item is a row).
- **Edge cases / failures:**
  - Invalid/missing Sheet ID, missing sheet name, auth failure → continues but returns no/empty items; prompt builder falls back to “No historical data available yet (first run).”
  - If the sheet exists but headers don’t match expected names, downstream still works (it checks `item.json.date` as a signal).

#### Node: Build analysis prompt with history
- **Type/role:** `code` — constructs the LLM prompt.
- **Key inputs:**
  - Current metrics from **Parse and normalize all metrics**
  - Config from **⚙️ Configure monitoring settings**
  - History items from **Read metrics history**
- **Prompt characteristics:**
  - Includes system metrics, filesystem usage breakdown, network totals/errors, top processes, docker disk usage, container status summaries.
  - Adds historical lines for up to 7 most recent rows (hard-coded `slice(-7)`).
  - Specifies exact JSON schema:
    ```json
    {
      "status": "healthy|warning|critical",
      "headline": "...",
      "findings": [{"severity":"high|medium|low","title":"...","detail":"...","command":"... or null"}],
      "trend_summary": "... or null",
      "top_recommendation": "..."
    }
    ```
- **Outputs:** `{ prompt: "<string>" }`
- **Edge cases:**
  - If history sheet rows are large, prompt size can grow; may hit model context limits in extreme cases (usually safe with 7 rows + summarized tables).
  - Container list could become long; prompt includes a line per container.

---

### Block F — Daily Digest: LLM Execution (LangChain) + Model
**Overview:** Sends the prompt to GPT‑4o‑mini via the LangChain chain node with a strict “JSON only” system instruction and actionable constraints.  
**Nodes involved:**  
- Generate daily health digest  
- OpenAI GPT-4o-mini

#### Node: Generate daily health digest
- **Type/role:** `@n8n/n8n-nodes-langchain.chainLlm` — orchestrates prompt + model.
- **Config:**
  - `text` input is `={{ $json.prompt }}` from the prompt builder.
  - “Define” prompt type with a strong instruction message:
    - Must be actionable; no restating visible numbers
    - Must focus on changes/trends and imminent failures
    - High/medium findings must include CLI command
    - If no issues, findings must be empty array
    - Output must be valid JSON only
- **Connections:**
  - Main input from **Build analysis prompt with history**
  - AI language model input from **OpenAI GPT-4o-mini** via `ai_languageModel`
  - Main output to **Format digest for Discord**
- **Edge cases:**
  - Model can still occasionally return non-JSON; downstream includes a JSON parse fallback.

#### Node: OpenAI GPT-4o-mini
- **Type/role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI chat model connector.
- **Config:** Model `gpt-4o-mini`, temperature `0.3`.
- **Credentials:** OpenAI API credential (“OpenAi account”).
- **Outputs:** Used by **Generate daily health digest** as its language model.
- **Edge cases:**
  - Missing/invalid API key, insufficient quota, rate limits → chain fails; downstream won’t get usable output unless n8n continues (not configured here).
  - Model name availability depends on OpenAI account and n8n node version.

---

### Block G — Daily Digest: Discord Dashboard + Storage
**Overview:** Parses the AI output JSON, creates a 4-embed Discord message payload (status, findings, docker/trends, footer), posts to Discord webhook, then stores a normalized metrics row into Google Sheets for trend tracking.  
**Nodes involved:**  
- Format digest for Discord  
- Send daily digest to Discord  
- Prepare metrics row for storage  
- Store today's metrics in history

#### Node: Format digest for Discord
- **Type/role:** `code` — transforms AI output + metrics into Discord webhook payload.
- **Key inputs:**
  - AI chain output: expects `$json.text` or `$json.output` to contain the model response text.
  - Current metrics from **Parse and normalize all metrics**
  - Prompt for token estimate fallback from **Build analysis prompt with history**
- **Key logic:**
  - Attempts JSON.parse of the AI response; strips code fences if present.
  - On parse failure: creates a warning payload with “AI Parse Error” finding.
  - Computes an estimated API cost:
    - If `tokenUsageEstimate` present: uses it
    - Else estimates tokens from character length and uses:
      - $0.15 / 1M input tokens
      - $0.60 / 1M output tokens
  - Sets embed color by health score (>=80 green, >=50 yellow, else red).
  - Embed 1: headline + 6 inline metric fields (+ up to 3 extra filesystems >=75% not `/`)
  - Embed 2: findings (up to 6) with severity icons and optional command in code block
  - Embed 3: Docker disk, network I/O, trends, top recommendation
  - Embed 4: footer with hostname, CST time, duration since metrics timestamp, and API cost
- **Outputs:** `{ embeds: [ ...4 embeds... ] }`
- **Edge cases:**
  - If metrics timestamp parsing fails, duration may be odd.
  - Discord embed limits: description capped to 4096 chars; code truncates findings to 2000 and embed3 to 1500, but very large fields could still risk limits if modified.
  - Uses `America/Chicago` for display time; independent of server locale.

#### Node: Send daily digest to Discord
- **Type/role:** `httpRequest` — posts the dashboard to Discord webhook.
- **Config:**
  - POST to `discord_webhook_url` from config node.
  - Sends JSON body as a string: `jsonBody = {{ JSON.stringify($json) }}`
  - Timeout 10s, retries enabled: `maxTries: 3`, wait 2000ms.
- **Input:** From **Format digest for Discord**.
- **Output:** Not used further (fan-out happens before this node).
- **Edge cases / failures:**
  - Invalid webhook URL → Discord returns 404.
  - Rate limits → Discord may return 429; retries may or may not respect Discord retry-after headers (node uses fixed delay).

#### Node: Prepare metrics row for storage
- **Type/role:** `code` — maps the full metrics object into a compact row matching the Google Sheet headers created by the setup path.
- **Key logic:**
  - Converts timestamp to `date` (YYYY-MM-DD).
  - Picks 16 columns: `hostname`, cpu%, load, memory%, disk%, inode%, swap%, health_score, containers running/total, net rx/tx bytes, docker_reclaimable_gb, zombie_count, connections.
  - Parses `docker_disk.images_reclaimable` into GB if it contains `GB` or `MB` (other formats remain 0).
- **Outputs:** One item containing the row object.
- **Edge cases:**
  - If reclaimable is like `"12.3GB (45%)"` parsing via `parseFloat()` still yields 12.3 (OK).
  - If reclaimable uses `kB` or no unit, will store 0.

#### Node: Store today's metrics in history
- **Type/role:** `googleSheets` — appends the row to the `metrics` sheet.
- **Config:**
  - Operation: `append`
  - Document ID from config node expression
  - Mapping: `autoMapInputData`
- **Credentials:** Google Sheets OAuth2.
- **Input:** From **Prepare metrics row for storage**.
- **Edge cases / failures:**
  - Missing sheet / wrong headers won’t necessarily fail (append adds values), but trend reading later may be inconsistent if column names don’t match.
  - Auth errors, Sheet permissions, API quotas.

---

### Block H — Critical Alerts (5-minute checks)
**Overview:** Runs a lightweight SSH check every 5 minutes and alerts Discord immediately when any critical threshold is exceeded or containers are down/restarting excessively.  
**Nodes involved:**  
- Check for critical issues  
- ⚙️ Alert settings  
- Quick system health check  
- Check against critical thresholds  
- Any critical issues found?  
- Format critical alert message  
- Send critical alert

#### Node: Check for critical issues
- **Type/role:** `scheduleTrigger` — frequent entry point.
- **Config:** Cron `*/5 * * * *` (every 5 minutes).
- **Outputs:** To **⚙️ Alert settings**.

#### Node: ⚙️ Alert settings
- **Type/role:** `set` — separate configuration for alerting.
- **Config (assigned fields):**
  - `discord_webhook_url` (placeholder)
  - `disk_critical_pct`, `memory_critical_pct`, `inode_critical_pct`
  - `load_multiplier` (default 1.5, compared to cores)
- **Outputs:** To **Quick system health check**.
- **Edge cases:** Placeholder webhook URL must be replaced to deliver alerts.

#### Node: Quick system health check
- **Type/role:** `ssh` — minimal metrics collection.
- **Command behavior:**
  - Root disk %, memory %, 1m load, cores, inode %
  - Lists “DOWN=container” for exited/restarting containers
  - Lists “RESTART=/name|count” for running containers
- **onError:** `continueRegularOutput`.
- **Outputs:** To **Check against critical thresholds**.
- **Edge cases:**
  - Docker permissions/daemon down will affect container checks.
  - If docker commands fail, you might miss container-down signals while still alerting on disk/mem/load.

#### Node: Check against critical thresholds
- **Type/role:** `code` — parses the quick SSH stdout and builds an alerts array.
- **Key inputs:** `$json.stdout` from quick SSH; config from **⚙️ Alert settings**.
- **Key logic:**
  - Collects:
    - `disk_pct`, `mem_pct`, `inode_pct`, `load`, `cores`
    - `downContainers[]` from `DOWN=`
    - `highRestarts[]` where restart count >= 3 (hard-coded in code; **not** using `container_restart_threshold`)
  - Threshold comparisons:
    - Disk >= `disk_critical_pct`
    - Mem >= `memory_critical_pct`
    - Inode >= `inode_critical_pct`
    - Load > `cores * load_multiplier`
    - Any down container → critical
    - Any restart loop → warning-level message (still triggers overall alert)
  - Outputs:
    - `has_critical` boolean (true if any alerts)
    - `alert_message` (joined with blank lines)
    - summary numeric fields for embed
- **Outputs:** To **Any critical issues found?**
- **Edge cases:**
  - If stdout is empty, parsed values become 0; has_critical false (silent failure).
  - Restart threshold is fixed at 3 inside code; if you want it configurable, bind it to config.

#### Node: Any critical issues found?
- **Type/role:** `if` — branching.
- **Condition:** `$json.has_critical === true`
- **True output:** To **Format critical alert message**
- **False output:** No action.
- **Edge cases:** If `has_critical` is missing (e.g., code node changed), the IF may never pass.

#### Node: Format critical alert message
- **Type/role:** `code` — builds a Discord webhook payload for alerts.
- **Payload:**
  - `content`: “🚨 HOMELAB CRITICAL ALERT”
  - One embed with title, description (up to 4000 chars), red color, Disk/Memory/CPU Load fields, timestamp
- **Outputs:** To **Send critical alert**
- **Edge cases:** Discord payload limits apply; description truncation helps.

#### Node: Send critical alert
- **Type/role:** `httpRequest` — posts alert to Discord webhook.
- **Config:**
  - POST to `discord_webhook_url` from **⚙️ Alert settings**
  - JSON body is `JSON.stringify($json)`
  - Timeout 10s, retries (3) with 2s delay
- **Edge cases:** Same Discord webhook failure/rate limit considerations as daily digest.

---

### Block I — First-Time Setup (Create & Format Google Sheet)
**Overview:** One-time manual run creates the Google Sheet used for storing history and applies dashboard-like formatting using the Sheets API batchUpdate endpoint.  
**Nodes involved:**  
- ▶️ Run first-time setup  
- Create health dashboard spreadsheet  
- Build dashboard layout  
- Format spreadsheet as dashboard

#### Node: ▶️ Run first-time setup
- **Type/role:** `manualTrigger` — manual entry point.
- **Outputs:** To **Create health dashboard spreadsheet**.

#### Node: Create health dashboard spreadsheet
- **Type/role:** `googleSheets` — creates a new spreadsheet.
- **Config:**
  - Resource: `spreadsheet`
  - Title: “Homelab Health Dashboard”
  - Creates a sheet titled `metrics`
  - `onError: continueRegularOutput`
- **Outputs:** Spreadsheet metadata including `spreadsheetId`, `spreadsheetUrl`, and sheet properties.
- **Edge cases:** OAuth credential must have permission to create spreadsheets in the target Google account/drive.

#### Node: Build dashboard layout
- **Type/role:** `code` — constructs a Google Sheets `batchUpdate` request body.
- **Key logic:**
  - Detects the `metrics` sheetId.
  - Writes a header row of 16 columns matching what “Prepare metrics row for storage” outputs:
    - `date`, `hostname`, `cpu_percent`, `cpu_load`, `memory_percent`, `disk_percent`, `inode_percent`, `swap_percent`, `health_score`, `containers_running`, `containers_total`, `net_rx_bytes`, `net_tx_bytes`, `docker_reclaimable_gb`, `zombie_count`, `connections`
  - Applies:
    - Header formatting (dark background, bold light text, bottom border)
    - Freeze first row
    - Tab color
    - Conditional formatting gradients for health score, CPU%, memory%, disk%, inode%, swap%
    - Column widths
- **Outputs:** Object containing:
  - `spreadsheetId`, `spreadsheetUrl`
  - `batchUpdateBody` (`{ requests: [...] }`)
- **Edge cases:** If Google Sheets API changes fields, batchUpdate may reject invalid request parts.

#### Node: Format spreadsheet as dashboard
- **Type/role:** `httpRequest` — calls Google Sheets API directly for batchUpdate (formatting).
- **Authentication:** Uses predefined credential type `googleSheetsOAuth2Api`.
- **URL:** `https://sheets.googleapis.com/v4/spreadsheets/{spreadsheetId}:batchUpdate`
- **Body:** `JSON.stringify($json.batchUpdateBody)`
- **Retries:** `maxTries: 2`
- **Edge cases:**
  - Requires Sheets API access on the OAuth credential; if scopes are too limited, request fails.
  - If the spreadsheetId is missing due to earlier failure, URL becomes invalid.

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | stickyNote | On-canvas overview & setup info | — | — | # Homelab Health Dashboard; includes setup links and steps (SSH/OpenAI/Sheets/Discord) |
| Sticky Note - Configuration | stickyNote | Notes for config area | — | — | ## ⚙️ Configuration; “All user settings are in one place.” |
| Sticky Note - Data Collection | stickyNote | Notes for SSH collection | — | — | ## 📊 Data Collection; “Two SSH commands collect 30+ metrics…” |
| Sticky Note - AI Analysis | stickyNote | Notes for history+LLM | — | — | ## 🤖 AI Analysis; “Loads 7 days… returns structured JSON…” |
| Sticky Note - Delivery | stickyNote | Notes for Discord+storage | — | — | ## 📬 Delivery & Storage; “4-embed Discord dashboard… Stores 16 metric columns…” |
| Sticky Note - Critical Alerts | stickyNote | Notes for 5-min alerts | — | — | ## 🚨 Critical Alerts; “Runs every 5 minutes… Fires immediately if…” |
| Sticky Note - API Cost Warning | stickyNote | Notes about OpenAI key | — | — | ⚠️ Requires OpenAI API key; links: https://platform.openai.com/api-keys and https://nxsi.io/guides/openai-api-setup |
| Sticky Note - First-Time Setup | stickyNote | Notes about creating the sheet | — | — | ## 🚀 First-Time Setup; “Test workflow… auto-create formatted sheet…” |
| Run daily health check | scheduleTrigger | Daily entry point | — | ⚙️ Configure monitoring settings | (covered by Overview note) |
| ⚙️ Configure monitoring settings | set | Central daily config | Run daily health check | Collect system metrics | ## ⚙️ Configuration… |
| Collect system metrics | ssh | Collect OS/system metrics via SSH | ⚙️ Configure monitoring settings | Collect Docker container metrics | ## 📊 Data Collection… |
| Collect Docker container metrics | ssh | Collect Docker metrics via SSH | Collect system metrics | Parse and normalize all metrics | ## 📊 Data Collection… |
| Parse and normalize all metrics | code | Parse outputs + compute health score | Collect Docker container metrics | Read metrics history (last 7 days) | (context: data collection → analysis) |
| Read metrics history (last 7 days) | googleSheets | Load historical rows | Parse and normalize all metrics | Build analysis prompt with history | ## 🤖 AI Analysis… |
| Build analysis prompt with history | code | Build LLM prompt | Read metrics history (last 7 days) | Generate daily health digest | ## 🤖 AI Analysis… |
| Generate daily health digest | chainLlm (LangChain) | Run LLM with prompt + system rules | Build analysis prompt with history | Format digest for Discord | ## 🤖 AI Analysis… |
| OpenAI GPT-4o-mini | lmChatOpenAi | LLM model provider | — (AI port) | Generate daily health digest (AI port) | ⚠️ Requires OpenAI API key… |
| Format digest for Discord | code | Build 4-embed Discord payload | Generate daily health digest | Send daily digest to Discord; Prepare metrics row for storage | ## 📬 Delivery & Storage… |
| Send daily digest to Discord | httpRequest | Post dashboard to Discord webhook | Format digest for Discord | — | ## 📬 Delivery & Storage… |
| Prepare metrics row for storage | code | Map metrics → 16 columns row | Format digest for Discord | Store today's metrics in history | ## 📬 Delivery & Storage… |
| Store today's metrics in history | googleSheets | Append row to history sheet | Prepare metrics row for storage | — | ## 📬 Delivery & Storage… |
| Check for critical issues | scheduleTrigger | 5-minute entry point | — | ⚙️ Alert settings | ## 🚨 Critical Alerts… |
| ⚙️ Alert settings | set | Central alert config | Check for critical issues | Quick system health check | ## 🚨 Critical Alerts… |
| Quick system health check | ssh | Lightweight SSH check | ⚙️ Alert settings | Check against critical thresholds | ## 🚨 Critical Alerts… |
| Check against critical thresholds | code | Compare metrics to thresholds | Quick system health check | Any critical issues found? | ## 🚨 Critical Alerts… |
| Any critical issues found? | if | Branch on `has_critical` | Check against critical thresholds | Format critical alert message (true) | ## 🚨 Critical Alerts… |
| Format critical alert message | code | Build Discord alert payload | Any critical issues found? (true) | Send critical alert | ## 🚨 Critical Alerts… |
| Send critical alert | httpRequest | Post alert to Discord webhook | Format critical alert message | — | ## 🚨 Critical Alerts… |
| ▶️ Run first-time setup | manualTrigger | Manual entry point (one-time) | — | Create health dashboard spreadsheet | ## 🚀 First-Time Setup… |
| Create health dashboard spreadsheet | googleSheets | Create spreadsheet + metrics sheet | ▶️ Run first-time setup | Build dashboard layout | ## 🚀 First-Time Setup… |
| Build dashboard layout | code | Build Sheets batchUpdate requests | Create health dashboard spreadsheet | Format spreadsheet as dashboard | ## 🚀 First-Time Setup… |
| Format spreadsheet as dashboard | httpRequest | Call Sheets API batchUpdate | Build dashboard layout | — | ## 🚀 First-Time Setup… |

---

## 4. Reproducing the Workflow from Scratch (Manual Build Steps)

1. **Create a new workflow**
   - Set workflow **Timezone** to `America/Chicago` (Workflow settings).

2. **Add Sticky Notes (optional but recommended)**
   - Create sticky notes matching the sections: Overview, Configuration, Data Collection, AI Analysis, Delivery, Critical Alerts, API key warning, First-Time Setup.
   - Paste the provided note contents (including the links).

### Daily Digest path

3. **Add trigger: “Run daily health check”**
   - Node: **Schedule Trigger**
   - Mode: Cron expression `0 7 * * *`.

4. **Add config node: “⚙️ Configure monitoring settings”**
   - Node: **Set**
   - Add fields:
     - `discord_webhook_url` (string)
     - `disk_warning_pct` (number, e.g., 80)
     - `disk_critical_pct` (number, e.g., 90)
     - `memory_warning_pct` (number, e.g., 85)
     - `memory_critical_pct` (number, e.g., 95)
     - `inode_critical_pct` (number, e.g., 90)
     - `container_restart_threshold` (number, e.g., 3)
     - `history_days` (number, e.g., 7)
     - `google_sheet_id` (string)
   - Connect: **Run daily health check → ⚙️ Configure monitoring settings**

5. **Add SSH node: “Collect system metrics”**
   - Node: **SSH**
   - Auth: **Private Key** (create SSH credential with host/user/key; in the provided workflow the credential name is “Projectbox SSH Key”).
   - Command: use the same compound command that prints `KEY=VALUE` and sections (`---FS---`, `---NET---`, `---TOP5CPU---`).
   - Set **On Error** to “Continue (regular output)” if you want same behavior.
   - Connect: **⚙️ Configure monitoring settings → Collect system metrics**

6. **Add SSH node: “Collect Docker container metrics”**
   - Node: **SSH** (same credential)
   - Command: use the provided docker command that prints `RUNNING=`, `TOTAL=`, container list, `---STATS---`, `---HEALTH---`, `---DISKUSAGE---`, `DANGLING=`, `UNUSED_VOLS=`.
   - On Error: continue.
   - Connect: **Collect system metrics → Collect Docker container metrics**

7. **Add Code node: “Parse and normalize all metrics”**
   - Node: **Code**
   - Paste the provided parsing JS (it references the two SSH nodes by name).
   - Connect: **Collect Docker container metrics → Parse and normalize all metrics**

8. **Add Google Sheets node: “Read metrics history (last 7 days)”**
   - Node: **Google Sheets**
   - Credential: Google Sheets OAuth2 (connect your Google account).
   - Document ID: expression `{{ $('⚙️ Configure monitoring settings').first().json.google_sheet_id }}`
   - Sheet name: `metrics`
   - Enable “Always output data” and set “On Error” to continue (to allow first run without history).
   - Connect: **Parse and normalize all metrics → Read metrics history (last 7 days)**

9. **Add Code node: “Build analysis prompt with history”**
   - Node: **Code**
   - Paste the prompt-building JS.
   - Connect: **Read metrics history (last 7 days) → Build analysis prompt with history**

10. **Add LangChain node: “Generate daily health digest”**
   - Node: **Chain LLM**
   - Set input text to `{{ $json.prompt }}`
   - Add the instruction message (the “senior DevOps engineer” rules) in the node’s defined prompt/messages.
   - Connect: **Build analysis prompt with history → Generate daily health digest**

11. **Add OpenAI model node: “OpenAI GPT-4o-mini”**
   - Node: **OpenAI Chat Model** (LangChain OpenAI)
   - Credential: OpenAI API key in n8n Credentials.
   - Model: `gpt-4o-mini`
   - Temperature: `0.3`
   - Connect the model to the chain node via the **AI language model** connection.

12. **Add Code node: “Format digest for Discord”**
   - Node: **Code**
   - Mode: “Run once for each item”
   - Paste the Discord embed builder code (parses JSON, builds 4 embeds, estimates cost).
   - Connect: **Generate daily health digest → Format digest for Discord**

13. **Add HTTP Request node: “Send daily digest to Discord”**
   - Node: **HTTP Request**
   - Method: POST
   - URL: `{{ $('⚙️ Configure monitoring settings').first().json.discord_webhook_url }}`
   - Body type: JSON (send body)
   - Body value: `{{ JSON.stringify($json) }}`
   - Timeout: 10000ms; retries enabled (3 tries, 2000ms wait).
   - Connect: **Format digest for Discord → Send daily digest to Discord**

14. **Add Code node: “Prepare metrics row for storage”**
   - Node: **Code**
   - Paste the mapping JS (produces 16 columns).
   - Connect: **Format digest for Discord → Prepare metrics row for storage** (second output branch)

15. **Add Google Sheets node: “Store today's metrics in history”**
   - Node: **Google Sheets**
   - Operation: Append
   - Document ID: `{{ $('⚙️ Configure monitoring settings').first().json.google_sheet_id }}`
   - Sheet name: `metrics`
   - Mapping: Auto-map input data
   - Connect: **Prepare metrics row for storage → Store today's metrics in history**

### Critical alerts path

16. **Add trigger: “Check for critical issues”**
   - Node: **Schedule Trigger**
   - Cron: `*/5 * * * *`

17. **Add config node: “⚙️ Alert settings”**
   - Node: **Set**
   - Fields:
     - `discord_webhook_url`
     - `disk_critical_pct` (90)
     - `memory_critical_pct` (95)
     - `inode_critical_pct` (90)
     - `load_multiplier` (1.5)
   - Connect: **Check for critical issues → ⚙️ Alert settings**

18. **Add SSH node: “Quick system health check”**
   - Node: **SSH** (same credential)
   - Command: quick disk/mem/load/cores/inodes + docker down/restart lines.
   - On Error: continue.
   - Connect: **⚙️ Alert settings → Quick system health check**

19. **Add Code node: “Check against critical thresholds”**
   - Node: **Code** (run once for each item)
   - Paste the threshold comparison JS (produces `has_critical`, `alerts`, `alert_message`).
   - Connect: **Quick system health check → Check against critical thresholds**

20. **Add IF node: “Any critical issues found?”**
   - Node: **IF**
   - Condition: Boolean equals, left `{{ $json.has_critical }}` right `true`
   - Connect: **Check against critical thresholds → Any critical issues found?**

21. **Add Code node: “Format critical alert message”**
   - Node: **Code**
   - Paste the alert payload JS.
   - Connect: **Any critical issues found? (true) → Format critical alert message**

22. **Add HTTP Request node: “Send critical alert”**
   - Node: **HTTP Request**
   - POST to `{{ $('⚙️ Alert settings').first().json.discord_webhook_url }}`
   - JSON body: `{{ JSON.stringify($json) }}`
   - Timeout 10s; retries enabled.
   - Connect: **Format critical alert message → Send critical alert**

### First-time setup path (Google Sheet creation)

23. **Add manual trigger: “▶️ Run first-time setup”**
   - Node: **Manual Trigger**

24. **Add Google Sheets node: “Create health dashboard spreadsheet”**
   - Node: **Google Sheets**
   - Resource: Spreadsheet → Create
   - Title: “Homelab Health Dashboard”
   - Create a sheet named `metrics`
   - Connect: **▶️ Run first-time setup → Create health dashboard spreadsheet**

25. **Add Code node: “Build dashboard layout”**
   - Node: **Code**
   - Paste the batchUpdate request builder JS (headers, formatting, conditional formatting, widths).
   - Connect: **Create health dashboard spreadsheet → Build dashboard layout**

26. **Add HTTP Request node: “Format spreadsheet as dashboard”**
   - Node: **HTTP Request**
   - Method: POST
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $json.spreadsheetId }}:batchUpdate`
   - Auth: Predefined credential type → `googleSheetsOAuth2Api`
   - JSON body: `{{ JSON.stringify($json.batchUpdateBody) }}`
   - Connect: **Build dashboard layout → Format spreadsheet as dashboard**
   - Run this setup path once, then copy the resulting `spreadsheetId` into `google_sheet_id` in **⚙️ Configure monitoring settings**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SSH key setup guide | https://nxsi.io/guides/ssh-key-setup |
| OpenAI API setup guide (n8n + cost management) | https://nxsi.io/guides/openai-api-setup |
| Create OpenAI API key | https://platform.openai.com/api-keys |
| n8n Google Sheets OAuth credential documentation | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/ |
| Discord webhook setup guide | https://nxsi.io/guides/discord-webhook |
| The workflow relies on strict JSON-only LLM output; formatting node includes a fallback when parsing fails. | Applies to the daily digest AI analysis path |
| Daily and critical alert webhooks are configured separately (two Set nodes). | Allows routing digest and alerts to different channels if desired |