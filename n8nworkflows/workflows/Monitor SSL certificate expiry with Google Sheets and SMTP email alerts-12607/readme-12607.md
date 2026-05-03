Monitor SSL certificate expiry with Google Sheets and SMTP email alerts

https://n8nworkflows.xyz/workflows/monitor-ssl-certificate-expiry-with-google-sheets-and-smtp-email-alerts-12607


# Monitor SSL certificate expiry with Google Sheets and SMTP email alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Monitor SSL certificate expiry with Google Sheets and SMTP email alerts  
**Purpose:** Periodically scan a list of domains, retrieve each SSL certificate’s expiry/issuer via local `openssl`, write results back to Google Sheets, and email an HTML alert when any certificate is expired, near expiry, or cannot be checked.

**Target use cases:**
- DevOps/SRE teams tracking many TLS endpoints without third-party SSL monitoring services
- Website owners needing a lightweight, self-hosted certificate expiry monitor
- Environments where API-based certificate checkers are restricted or rate-limited

### Logical Blocks
**1.1 Scheduling & Entry**
- Runs on a fixed cadence (every 3 days at 10:00).

**1.2 Domain Input & Normalization**
- Reads domain rows from Google Sheets.
- Extracts valid domains and sets an alert threshold (days).

**1.3 SSL Inspection (OpenSSL)**
- Runs an `openssl s_client` command per domain with a timeout.
- Outputs raw cert metadata (dates, issuer).

**1.4 Parsing, Classification & Alert Set**
- Parses `notAfter` date, computes days remaining, extracts issuer.
- Builds two datasets: full results and “alertDomains” subset.
- Produces summary counts and a `hasAlerts` boolean.

**1.5 Output: Persist & Notify**
- Writes per-domain results back into Google Sheets (append-or-update by Domain).
- If alerts exist, builds a styled HTML email and sends via SMTP; otherwise ends.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Entry

**Overview:** Starts workflow executions on a schedule so SSL checks run automatically without manual triggering.

**Nodes Involved:**
- Schedule Trigger (Every 3 Days at 10AM)

#### Node: Schedule Trigger (Every 3 Days at 10AM)
- **Type / role:** `Schedule Trigger` — workflow entrypoint.
- **Configuration choices:**
  - Interval: every **3 days**
  - Time: **10:00** (instance timezone)
- **Inputs / outputs:**
  - **Input:** none (trigger)
  - **Output:** one trigger item to `Read Domain List from Google Sheets`
- **Version notes:** typeVersion **1.2** (standard schedule trigger behavior).
- **Edge cases / failures:**
  - Timezone mismatches (instance settings vs expected local time)
  - Workflow inactive (`active: false` currently) prevents executions

---

### 2.2 Domain Input & Normalization

**Overview:** Fetches the domain list from Google Sheets and converts it into a clean array plus an alert threshold, then expands into one item per domain for downstream processing.

**Nodes Involved:**
- Read Domain List from Google Sheets
- Prepare Domain List and Set Threshold
- Split Into Individual Domains

#### Node: Read Domain List from Google Sheets
- **Type / role:** `Google Sheets` — reads rows from a sheet containing domains.
- **Configuration choices (interpreted):**
  - Document: placeholder `YOUR-GOOGLE-SHEET-ID`
  - Sheet tab: `gid=0` (first/default tab)
  - Operation is not explicitly shown in parameters; in n8n, this node typically defaults to **Read/Get Many** behavior depending on UI selection. The workflow assumes it outputs multiple row items containing a `Domain` field.
- **Key fields expected in output:**
  - `Domain` column (capital D) used by later Code nodes.
- **Connections:**
  - **Input:** from Schedule Trigger
  - **Output:** to `Prepare Domain List and Set Threshold`
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Version notes:** typeVersion **4.5**
- **Edge cases / failures:**
  - Wrong documentId/sheetId → “not found” / permission errors
  - OAuth token expiry / missing scopes
  - Column header mismatch (must be exactly `Domain` for this workflow)
  - Empty sheet results in empty domain list and later steps produce no checks

#### Node: Prepare Domain List and Set Threshold
- **Type / role:** `Code` — filters domains and sets the alert threshold.
- **Configuration choices:**
  - Hardcoded `ALERT_THRESHOLD_DAYS = 20`
  - Iterates all input rows, pulls `items[i].json.Domain`
  - Ignores blanks and ignores literal string `'Domain'` (attempt to skip header-like value)
- **Outputs:**
  - A single item: `{ domains: [ ... ], ALERT_THRESHOLD_DAYS: 20 }`
- **Connections:**
  - **Input:** from Google Sheets read
  - **Output:** to `Split Into Individual Domains`
- **Version notes:** typeVersion **2** (modern Code node runtime).
- **Edge cases / failures:**
  - If sheet uses `domain` or `URL` instead of `Domain`, domains array will be empty
  - The `'Domain'` exclusion is brittle; if a real domain equals `"Domain"` it will be skipped (unlikely)
  - Threshold is duplicated later (see parsing block) and changing it here does not change parsing threshold

#### Node: Split Into Individual Domains
- **Type / role:** `Code` — fan-out to one execution item per domain.
- **Configuration choices:**
  - Reads `domains` and `ALERT_THRESHOLD_DAYS` from the first item
  - Emits items like `{ domain: "<domain>", threshold: 20 }`
- **Connections:**
  - **Input:** from Prepare Domain List and Set Threshold
  - **Output:** to `Check SSL Certificate via OpenSSL`
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**
  - If `domains` is undefined/empty → outputs empty array; downstream nodes won’t run
  - Threshold is passed forward, but downstream parsing node ignores it (uses its own constant)

---

### 2.3 SSL Inspection (OpenSSL)

**Overview:** For each domain item, runs an OpenSSL command to retrieve certificate dates and issuer information, with a 10-second timeout.

**Nodes Involved:**
- Check SSL Certificate via OpenSSL

#### Node: Check SSL Certificate via OpenSSL
- **Type / role:** `Execute Command` — executes shell command on the n8n host.
- **Configuration choices:**
  - Command (expression-based):
    - `echo | timeout 10 openssl s_client -servername {{ $json.domain }} -connect {{ $json.domain }}:443 2>&1 | openssl x509 -noout -dates -issuer 2>&1`
  - Uses SNI (`-servername`) for correct certificate on multi-tenant hosts.
  - Pipes output into `openssl x509` to extract `-dates` and `-issuer`.
  - `timeout 10` prevents hanging connections.
- **Inputs / outputs:**
  - **Input:** items `{domain, threshold}`
  - **Output:** items containing `stdout` (used later), one per domain
- **Connections:**
  - **Input:** from Split Into Individual Domains
  - **Output:** to Parse SSL Results and Identify Expiring Certs
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - Requires `openssl` and `timeout` binaries installed on the n8n runtime host/container
  - DNS failures, blocked outbound 443, firewall restrictions
  - Some servers require specific TLS versions/ciphers; `s_client` may fail
  - Internationalized domains (IDN) may need punycode conversion (not handled)
  - If the command fails, `stdout` may be empty and parsing will mark `ERROR`

**Security note:** This node executes shell commands; only trusted users should be able to edit this workflow.

---

### 2.4 Parsing, Classification & Alert Set

**Overview:** Interprets the OpenSSL outputs: extracts expiry date and issuer, computes days remaining, assigns status, and builds an alert list and summary metadata.

**Nodes Involved:**
- Parse SSL Results and Identify Expiring Certs

#### Node: Parse SSL Results and Identify Expiring Certs
- **Type / role:** `Code` — parse stdout, compute daysLeft, decide alerting.
- **Configuration choices (important behaviors):**
  - **Hardcoded** `ALERT_THRESHOLD_DAYS = 20` (does **not** use the `threshold` value passed from previous nodes).
  - Reads original domain items via: `$('Split Into Individual Domains').all()`
  - Reads SSL outputs from `$input.all()` (expects same ordering and count as original items)
  - For each domain:
    - If `stdout` does not contain `notAfter` → status `ERROR`, error message “Could not retrieve certificate”, and **added to alerts**
    - Else:
      - Parse with regex `/notAfter=(.+)/`
      - Compute `daysLeft = floor((expiryDate - now) / days)`
      - Status:
        - `EXPIRED` if `daysLeft <= 0` → **added to alerts**
        - Otherwise `OK`; if `daysLeft <= ALERT_THRESHOLD_DAYS` → **added to alerts**
      - Issuer parsing:
        - Prefer regex for Org field: `/issuer=.*?O\s*=\s*([^,\n\/]+)/`
        - Else fallback `/issuer=(.+)/` truncated to 50 chars
    - `lastChecked` is `YYYY-MM-DD` (UTC ISO date split)
  - Outputs a single summary object:
    - `results` (array of all domains)
    - `alertDomains` (subset)
    - `hasAlerts` boolean
    - `totalChecked`, `alertCount`, `threshold`
- **Connections:**
  - **Input:** from Execute Command (per-domain outputs)
  - **Output:** to:
    - `Has Expiring Certificates`
    - `Split Results for Sheet Update`
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**
  - **Ordering dependency:** assumes the i-th OpenSSL result corresponds to i-th domain from `Split Into Individual Domains`. Parallelism or partial failures could break mapping.
  - Date parsing: `new Date(notAfterMatch[1])` depends on runtime parsing of OpenSSL date format; typically works, but locale/timezone quirks can occur.
  - Status “WARNING” is referenced later in email builder but never set here; near-expiry certs remain `OK` yet still alert.
  - Threshold duplication: changing threshold in the earlier node does not affect this node unless edited here too.

---

### 2.5 Output: Persist & Notify

**Overview:** Writes per-domain results back to the same Google Sheet and sends an HTML email if any domain is in the alert set.

**Nodes Involved:**
- Split Results for Sheet Update
- Update Google Sheet with Results
- Has Expiring Certificates
- Build HTML Email Alert
- Send Alert Email via SMTP
- No Alerts Needed (All Certs OK)

#### Node: Split Results for Sheet Update
- **Type / role:** `Code` — expands the `results` array into individual items for Google Sheets upsert.
- **Configuration choices:**
  - Reads `results` from `$input.first().json.results`
  - Outputs one item per domain result with fields: `domain, expiryDate, daysLeft, status, issuer, lastChecked, error`
- **Connections:**
  - **Input:** from Parse SSL Results...
  - **Output:** to Update Google Sheet with Results
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**
  - If Parse node outputs malformed/empty `results`, the sheet update will do nothing or error

#### Node: Update Google Sheet with Results
- **Type / role:** `Google Sheets` — append or update rows keyed by Domain.
- **Configuration choices (interpreted):**
  - Operation: **Append or Update**
  - Match column: **Domain**
  - Writes columns:
    - Domain ← `$json.domain`
    - Issuer ← `$json.issuer`
    - Status ← `$json.status`
    - Days Left ← `$json.daysLeft`
    - Expiry Date ← `$json.expiryDate`
    - Last Checked ← `$json.lastChecked`
  - Document: placeholder `YOUR-GOOGLE-SHEET-ID`
  - Sheet tab: `gid=0`
- **Connections:**
  - **Input:** from Split Results for Sheet Update (one item per domain)
  - **Output:** none (end of that branch)
- **Credentials:** Google Sheets OAuth2
- **Version notes:** typeVersion **4.5**
- **Edge cases / failures:**
  - If the sheet does not contain those exact column headers, mapping/matching fails
  - Duplicate domains in sheet can cause unexpected matching behavior
  - Rate limits/quota on Google Sheets API (still possible, even though SSL checks have no API limits)

#### Node: Has Expiring Certificates
- **Type / role:** `IF` — branch based on whether alerts exist.
- **Configuration choices:**
  - Condition: `$('Parse SSL Results and Identify Expiring Certs').item.json.hasAlerts == true`
  - Strict boolean comparison
- **Connections:**
  - **Input:** from Parse SSL Results...
  - **True output:** to Build HTML Email Alert
  - **False output:** to No Alerts Needed (All Certs OK)
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**
  - If Parse node produces no item or missing `hasAlerts`, expression evaluation can fail

#### Node: Build HTML Email Alert
- **Type / role:** `Code` — constructs a styled HTML email and a dynamic subject.
- **Configuration choices:**
  - Reads `alertDomains`, `threshold`, totals from Parse node output via:
    - `$('Parse SSL Results and Identify Expiring Certs').item.json`
  - Builds an HTML table with:
    - Domain, Status badge, Days Left, Expiry Date/Error
  - Subject example pattern:
    - `SSL Alert: <count> certificate(s) expiring soon - <timestamp>`
  - Returns `{ subject, emailHtml, emailBody }` (emailBody duplicates HTML)
- **Connections:**
  - **Input:** from IF (true branch)
  - **Output:** to Send Alert Email via SMTP
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**
  - Status rendering: sets “WARNING” styling, but upstream never sets `cert.status = 'WARNING'`; near-expiry items will display “OK” unless you adjust parsing logic.
  - `cert.daysLeft` comparisons when empty (ERROR items) could behave unexpectedly, though code uses `cert.error ? 'N/A' : cert.daysLeft` for display.
  - Large alert sets could create very long emails; some SMTP providers limit message size.

#### Node: Send Alert Email via SMTP
- **Type / role:** `Email Send` — sends email with HTML + text.
- **Configuration choices:**
  - Subject: from `$json.subject`
  - HTML: `$json.emailHtml`
  - Text: `$json.emailHtml` (so the “text” part is not plain text; may reduce deliverability)
  - Format: `both`
  - Recipient/sender fields are not shown in provided parameters; these must be configured in the node UI (To/From).
- **Connections:**
  - **Input:** from Build HTML Email Alert
  - **Output:** none
- **Credentials:** SMTP (`SMTP account`)
- **Version notes:** typeVersion **2.1**
- **Edge cases / failures:**
  - Missing `To`/`From` configuration will prevent sending
  - SMTP auth failures, blocked ports (25/587/465), TLS negotiation issues
  - HTML-only “text part” can trigger spam filters; consider generating a real plaintext alternative

#### Node: No Alerts Needed (All Certs OK)
- **Type / role:** `NoOp` — terminates the “no alerts” branch.
- **Connections:**
  - **Input:** from IF (false branch)
  - **Output:** none
- **Version notes:** typeVersion **1**
- **Edge cases / failures:** none (safe terminal node)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | Sticky Note | Documentation / overview panel |  |  | ## 🔒 Monitor SSL certificate expiry… (includes setup steps and threshold note) |
| Sticky Note - Data Input | Sticky Note | Documentation for input block |  |  | ### 📥 Data Input — Fetch domains from Google Sheets and prepare for scanning |
| Sticky Note - SSL Check | Sticky Note | Documentation for SSL check block |  |  | ### 🔍 SSL Check & Processing — Verify certificates via OpenSSL and parse expiry results |
| Sticky Note - Output | Sticky Note | Documentation for output block |  |  | ### 📤 Output — Update sheet and send alerts |
| Schedule Trigger (Every 3 Days at 10AM) | Schedule Trigger | Entry point / scheduled run |  | Read Domain List from Google Sheets | ## 🔒 Monitor SSL certificate expiry… (includes setup steps and threshold note) |
| Read Domain List from Google Sheets | Google Sheets | Read domain inventory | Schedule Trigger (Every 3 Days at 10AM) | Prepare Domain List and Set Threshold | ### 📥 Data Input — Fetch domains from Google Sheets and prepare for scanning |
| Prepare Domain List and Set Threshold | Code | Filter domains, set threshold | Read Domain List from Google Sheets | Split Into Individual Domains | ### 📥 Data Input — Fetch domains from Google Sheets and prepare for scanning |
| Split Into Individual Domains | Code | Fan-out: one item per domain | Prepare Domain List and Set Threshold | Check SSL Certificate via OpenSSL | ### 🔍 SSL Check & Processing — Verify certificates via OpenSSL and parse expiry results |
| Check SSL Certificate via OpenSSL | Execute Command | Retrieve cert dates/issuer via OpenSSL | Split Into Individual Domains | Parse SSL Results and Identify Expiring Certs | ### 🔍 SSL Check & Processing — Verify certificates via OpenSSL and parse expiry results |
| Parse SSL Results and Identify Expiring Certs | Code | Parse cert output, compute daysLeft, build alerts | Check SSL Certificate via OpenSSL | Has Expiring Certificates; Split Results for Sheet Update | ### 🔍 SSL Check & Processing — Verify certificates via OpenSSL and parse expiry results |
| Split Results for Sheet Update | Code | Expand results array for upsert | Parse SSL Results and Identify Expiring Certs | Update Google Sheet with Results | ### 📤 Output — Update sheet and send alerts |
| Update Google Sheet with Results | Google Sheets | Persist results (append/update) | Split Results for Sheet Update |  | ### 📤 Output — Update sheet and send alerts |
| Has Expiring Certificates | IF | Branch: send email only if needed | Parse SSL Results and Identify Expiring Certs | Build HTML Email Alert; No Alerts Needed (All Certs OK) | ### 📤 Output — Update sheet and send alerts |
| Build HTML Email Alert | Code | Generate styled HTML + subject | Has Expiring Certificates (true) | Send Alert Email via SMTP | ### 📤 Output — Update sheet and send alerts |
| Send Alert Email via SMTP | Email Send | Deliver alert email | Build HTML Email Alert |  | ### 📤 Output — Update sheet and send alerts |
| No Alerts Needed (All Certs OK) | NoOp | End branch when no alerts | Has Expiring Certificates (false) |  | ### 📤 Output — Update sheet and send alerts |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **“Monitor SSL certificate expiry with Google Sheets and email alerts”** (or your preferred name).
   - Ensure the workflow will run on an environment that supports running shell commands.

2) **Add node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Set schedule:
     - Every **3 days**
     - At **10:00**
   - This is the entry node.

3) **Add node: Google Sheets (Read)**
   - Node type: **Google Sheets**
   - Credentials: create/select **Google Sheets OAuth2** credentials.
   - Configure:
     - **Document ID:** your spreadsheet ID
     - **Sheet:** the tab containing your data (e.g., `gid=0`)
     - Operation: configure to **read rows** (so each row becomes an item).
   - Your sheet should have headers (at least):
     - `Domain`, `Expiry Date`, `Days Left`, `Status`, `Issuer`, `Last Checked`
   - Connect: **Schedule Trigger → Google Sheets (Read)**

4) **Add node: Code (Prepare Domain List and Set Threshold)**
   - Node type: **Code**
   - Paste logic that:
     - sets `ALERT_THRESHOLD_DAYS` (default 20)
     - builds a `domains[]` array from the `Domain` column
     - returns one item with `{ domains, ALERT_THRESHOLD_DAYS }`
   - Connect: **Google Sheets (Read) → Prepare Domain List and Set Threshold**

5) **Add node: Code (Split Into Individual Domains)**
   - Node type: **Code**
   - Configure to output one item per domain:
     - `{ domain: domains[i], threshold: ALERT_THRESHOLD_DAYS }`
   - Connect: **Prepare Domain List and Set Threshold → Split Into Individual Domains**

6) **Add node: Execute Command (Check SSL Certificate via OpenSSL)**
   - Node type: **Execute Command**
   - Command (as an expression using `$json.domain`):
     - `echo | timeout 10 openssl s_client -servername {{$json.domain}} -connect {{$json.domain}}:443 2>&1 | openssl x509 -noout -dates -issuer 2>&1`
   - Confirm the n8n host has:
     - `openssl`
     - `timeout` (often from GNU coreutils)
   - Connect: **Split Into Individual Domains → Execute Command**

7) **Add node: Code (Parse SSL Results and Identify Expiring Certs)**
   - Node type: **Code**
   - Parse each command `stdout` to extract:
     - `notAfter=...` → expiry date
     - compute `daysLeft`
     - parse `issuer=...`
     - assign status (`OK`, `EXPIRED`, `ERROR`) and build `alertDomains`
   - Output a single item containing:
     - `results[]`, `alertDomains[]`, `hasAlerts`, summary counts
   - Connect: **Execute Command → Parse SSL Results...**

   Note: If you want the threshold to be controlled from earlier nodes, reference `$json.threshold` or carry it forward; in the provided workflow it is hardcoded again.

8) **Add node: Code (Split Results for Sheet Update)**
   - Node type: **Code**
   - Convert `results[]` array to individual items.
   - Connect: **Parse SSL Results... → Split Results for Sheet Update**

9) **Add node: Google Sheets (Update Google Sheet with Results)**
   - Node type: **Google Sheets**
   - Credentials: same Google Sheets OAuth2.
   - Operation: **Append or Update**
   - Match column: **Domain**
   - Map fields:
     - Domain ← `{{$json.domain}}`
     - Issuer ← `{{$json.issuer}}`
     - Status ← `{{$json.status}}`
     - Days Left ← `{{$json.daysLeft}}`
     - Expiry Date ← `{{$json.expiryDate}}`
     - Last Checked ← `{{$json.lastChecked}}`
   - Connect: **Split Results for Sheet Update → Update Google Sheet with Results**

10) **Add node: IF (Has Expiring Certificates)**
   - Node type: **IF**
   - Condition: boolean equals
     - Left value: `{{$('Parse SSL Results and Identify Expiring Certs').item.json.hasAlerts}}`
     - Right value: `true`
   - Connect: **Parse SSL Results... → IF**

11) **Add node: Code (Build HTML Email Alert)**
   - Node type: **Code**
   - Build an HTML email from `alertDomains` and set `subject`.
   - Output: `{ subject, emailHtml }`
   - Connect: **IF (true) → Build HTML Email Alert**

12) **Add node: Email Send (Send Alert Email via SMTP)**
   - Node type: **Email Send**
   - Credentials: create/select **SMTP** credentials (host, port, user, password, TLS settings).
   - Configure:
     - **To:** your alert recipient(s)
     - **From:** a valid sender for your SMTP server
     - Subject: `{{$json.subject}}`
     - HTML: `{{$json.emailHtml}}`
     - Email format: **both** (optional; ideally provide real plaintext too)
   - Connect: **Build HTML Email Alert → Send Alert Email via SMTP**

13) **Add node: NoOp (No Alerts Needed)**
   - Node type: **NoOp**
   - Connect: **IF (false) → No Alerts Needed**

14) **Activate the workflow**
   - Toggle workflow to **Active** once credentials and sheet structure are confirmed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Uses direct OpenSSL commands to avoid third-party SSL check API costs and rate limits. | Sticky note “Overview” |
| Google Sheet must include columns: Domain, Expiry Date, Days Left, Status, Issuer, Last Checked. | Sticky note “Overview” |
| Alert threshold default is 20 days; it is configured in “Prepare Domain List and Set Threshold” and separately hardcoded again in “Parse SSL Results and Identify Expiring Certs”. Keep them consistent. | Sticky note “Overview” |
| SMTP node must be configured with valid sender/recipient addresses in addition to credentials. | Output block requirement inferred from Email Send node configuration |
| Workflow requires host-level binaries: `openssl` and `timeout`, plus outbound TCP/443 access to scanned domains. | Execute Command node dependency |