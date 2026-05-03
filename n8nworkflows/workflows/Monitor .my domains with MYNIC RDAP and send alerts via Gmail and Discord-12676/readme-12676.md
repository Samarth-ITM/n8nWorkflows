Monitor .my domains with MYNIC RDAP and send alerts via Gmail and Discord

https://n8nworkflows.xyz/workflows/monitor--my-domains-with-mynic-rdap-and-send-alerts-via-gmail-and-discord-12676


# Monitor .my domains with MYNIC RDAP and send alerts via Gmail and Discord

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor a list of `.my` domains stored in Google Sheets, check availability via the public **MYNIC RDAP** endpoint, and when a domain becomes available, send alerts via **Gmail** and **Discord**, then mark the domain as available in the sheet to avoid repeated alerts.

**Target use cases:**
- Domain “sniping” / monitoring for `.my` domains.
- Lightweight availability monitoring without paid API keys.
- Notification fan-out to email + Discord, with state tracked in Sheets.

### 1.1 Scheduling & Input Retrieval (Google Sheets)
Runs every 30 minutes, reads domains where `isAvailable = no`.

### 1.2 Per-domain Loop & RDAP Check
Iterates through each domain and queries the MYNIC RDAP endpoint.

### 1.3 Availability Decision
Checks RDAP response content to determine if the domain is available.

### 1.4 Notifications (Gmail + Discord)
If available: send an HTML email and post a Discord message with an embed link parsed from RDAP response.

### 1.5 State Update + Throttling
Updates the Google Sheet row to `isAvailable = yes`, waits 10 seconds, then continues the loop.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Fetching Target Domains
**Overview:** Triggers the workflow periodically and loads only domains that have not yet been marked available.  
**Nodes Involved:**  
- `Schedule: Every 30 Minutes`
- `Fetch Target Domains from Sheet`

#### Node: Schedule: Every 30 Minutes
- **Type / Role:** Schedule Trigger (entry point); runs workflow on interval.
- **Configuration:** Every **30 minutes**.
- **Inputs / Outputs:** No input. Output → `Fetch Target Domains from Sheet`.
- **Edge cases / failures:**
  - n8n instance downtime delays runs.
  - Overlapping executions if run time exceeds interval (depends on n8n execution settings).

#### Node: Fetch Target Domains from Sheet
- **Type / Role:** Google Sheets node; reads rows from a spreadsheet.
- **Configuration choices:**
  - Targets document **“Domain Target”** and sheet **“Sheet1”**.
  - Uses a filter: `isAvailable` equals `"no"` (so only unfulfilled targets are checked).
- **Key fields / variables:**
  - Expects columns at least: `Domain`, `isAvailable`.
- **Inputs / Outputs:** Input from schedule; output rows → `Loop Through Domains`.
- **Version-specific notes:** Node typeVersion `4.7` (Google Sheets v4+ behavior, mapping UI, etc.).
- **Edge cases / failures:**
  - OAuth credential expired/invalid → auth error.
  - Sheet schema mismatch (missing `Domain` or `isAvailable`) → empty results or runtime issues downstream.
  - Filter value casing differences (`No` vs `no`) causing no matches.

---

### Block 2 — Loop Through Domains & RDAP Query
**Overview:** Processes each sheet row one-by-one (batch loop) and performs RDAP lookup for the domain.  
**Nodes Involved:**  
- `Loop Through Domains`
- `RDAP: Check Domain Status`
- `Sticky Note1` (documentation note for RDAP endpoint)

#### Node: Loop Through Domains
- **Type / Role:** Split In Batches; iterates through the fetched domain rows.
- **Configuration choices:** Default options (batch size not explicitly set in JSON; n8n default applies).
- **Connections:**
  - Receives items from `Fetch Target Domains from Sheet`.
  - Output (loop/next batch) is wired via `Wait 10 Seconds` back into this node.
  - Main processing output → `RDAP: Check Domain Status` (connected on output index 1 in the workflow wiring).
- **Edge cases / failures:**
  - If no rows returned from Sheets, loop does nothing (expected).
  - Large lists: execution time may exceed schedule interval if throttled heavily.

#### Node: RDAP: Check Domain Status
- **Type / Role:** HTTP Request; calls RDAP endpoint to obtain domain status.
- **Configuration choices:**
  - URL: `https://rdap.mynic.my/rdap/domain/{{ $json.Domain }}`
  - Response format: JSON
  - **neverError: true** (important): HTTP errors won’t fail the node; it will return a response object instead.
- **Key expressions / variables:**
  - Uses current item field `{{$json.Domain}}`.
- **Inputs / Outputs:** Input from loop; output → `Domain Available?`.
- **Edge cases / failures:**
  - If RDAP returns non-JSON or unexpected structure, referencing `description[0]` later may error.
  - With `neverError: true`, 404/500 responses may pass through and break the IF expression or cause false negatives.
  - Rate-limits/network timeouts: may return partial/empty data.

#### Node: Sticky Note1 (RDAP reference)
- **Type / Role:** Sticky Note (documentation).
- **Content (important operational assumptions):**
  - Endpoint format and what “available” vs “registered” responses look like.
  - Notes: **No API Key required** (public API).

---

### Block 3 — Availability Decision
**Overview:** Determines if RDAP indicates the domain is available by checking `description[0]`.  
**Nodes Involved:**  
- `Domain Available?`

#### Node: Domain Available?
- **Type / Role:** IF node; routes execution based on condition.
- **Condition logic:**
  - Checks whether `{{ $json.description[0] }}` **contains** `"is available for registration"`.
- **Outputs / Connections:**
  - **True branch (index 0)** → `Gmail: Send Availability Alert`
  - **False branch (index 1)** → `Wait 10 Seconds`
- **Edge cases / failures:**
  - If `$json.description` is missing or not an array, expression may evaluate to `undefined` and can trigger evaluation issues.
  - RDAP wording changes (string mismatch) would cause missed alerts.
  - Non-availability responses might still contain `description` but with different indexing.

---

### Block 4 — Notifications (Gmail + Discord)
**Overview:** When a domain is available, the workflow sends a styled HTML email and posts a Discord message with an embed pointing to a registrar link extracted from RDAP response.  
**Nodes Involved:**  
- `Gmail: Send Availability Alert`
- `Discord: Notify Available Domain`

#### Node: Gmail: Send Availability Alert
- **Type / Role:** Gmail node; sends an email alert.
- **Configuration choices:**
  - To: `user@example.com` (placeholder)
  - Subject: `Sniper Report! {{ $('Loop Through Domains').item.json.Domain }} is Available`
  - Body: HTML template (retro “system access” style), includes timestamp `{{ $now.format('DD HH:mm:ss') }}` and branding link.
  - Attribution disabled (`appendAttribution: false`).
- **Key expressions / variables:**
  - Domain referenced via `$('Loop Through Domains').item.json.Domain` (pulls from the loop item context).
  - Timestamp via `$now.format(...)`.
- **Inputs / Outputs:** Input from IF true; output → `Discord: Notify Available Domain`.
- **Edge cases / failures:**
  - Gmail OAuth not authorized or token expired.
  - Sending limits / anti-spam policies.
  - The HTML body currently contains a **hard-coded domain string** (`berjayasaja.my`) in the template; if not replaced dynamically, email content may not match the actual domain (even though subject is dynamic).

#### Node: Discord: Notify Available Domain
- **Type / Role:** Discord node; posts a message to a specific channel.
- **Configuration choices:**
  - Resource: `message`
  - Content: `{{ $('Loop Through Domains').item.json.Domain }} is available to buy!`
  - Embed:
    - Title is extracted from RDAP:  
      `{{ $('Domain Available?').item.json.description[1].match(/https?:\/\/[^\s\)]+/)[0] }}`
    - Author: `"Grab Now"`
- **Inputs / Outputs:** Input from Gmail; output → `Update Sheet: Mark Available`.
- **Edge cases / failures:**
  - If `description[1]` is missing or does not contain a URL, `.match(...)[0]` will throw (common failure).
  - Discord bot permissions: missing `Send Messages` / `Embed Links`.
  - Wrong channel/guild IDs or bot not in guild.

---

### Block 5 — Marking State + Throttling + Loop Continuation
**Overview:** After notifying, the workflow updates the sheet so the domain is not checked again as “unavailable”, then waits 10 seconds before proceeding to the next domain.  
**Nodes Involved:**  
- `Update Sheet: Mark Available`
- `Wait 10 Seconds`

#### Node: Update Sheet: Mark Available
- **Type / Role:** Google Sheets node; updates an existing row by matching the `Domain` column.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `Domain`
  - Writes:
    - `Domain = {{ $('Loop Through Domains').item.json.Domain }}`
    - `isAvailable = yes`
- **Inputs / Outputs:** Input from Discord; output → `Wait 10 Seconds`.
- **Edge cases / failures:**
  - If duplicate domains exist in sheet, update behavior may be ambiguous (which row updates depends on connector behavior).
  - If the domain value differs by case/whitespace, matching may fail and no row updates.
  - OAuth/auth errors.

#### Node: Wait 10 Seconds
- **Type / Role:** Wait node; throttling and pacing; then continues loop.
- **Configuration:** Wait `10` seconds.
- **Connections:** Receives from:
  - IF false branch (domain not available)
  - Sheet update after notifications  
  Then outputs back to `Loop Through Domains` to fetch next batch item.
- **Edge cases / failures:**
  - Execution time increases linearly with number of domains.
  - If schedule overlaps, multiple runs could compete and send duplicates unless sheet updates occur quickly.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / setup notes |  |  | ## RDAP .my Domain Availability Monitor / How it works + Setup checklist |
| Schedule: Every 30 Minutes | Schedule Trigger | Periodic execution |  | Fetch Target Domains from Sheet | ## RDAP .my Domain Availability Monitor / How it works + Setup checklist |
| Fetch Target Domains from Sheet | Google Sheets | Read target domains where `isAvailable=no` | Schedule: Every 30 Minutes | Loop Through Domains | ## RDAP .my Domain Availability Monitor / How it works + Setup checklist |
| Loop Through Domains | Split In Batches | Iterate each domain row | Fetch Target Domains from Sheet; Wait 10 Seconds | RDAP: Check Domain Status | ## RDAP .my Domain Availability Monitor / How it works + Setup checklist |
| RDAP: Check Domain Status | HTTP Request | Query MYNIC RDAP domain endpoint | Loop Through Domains | Domain Available? | **Endpoint**: https://rdap.mynic.my/rdap/domain/{domain} / Available vs Registered response notes / No API key required |
| Domain Available? | IF | Detect “available for registration” | RDAP: Check Domain Status | Gmail: Send Availability Alert (true); Wait 10 Seconds (false) | **Endpoint**: https://rdap.mynic.my/rdap/domain/{domain} / Available vs Registered response notes / No API key required |
| Gmail: Send Availability Alert | Gmail | Email alert for available domain | Domain Available? (true) | Discord: Notify Available Domain |  |
| Discord: Notify Available Domain | Discord | Post Discord notification + embed link | Gmail: Send Availability Alert | Update Sheet: Mark Available |  |
| Update Sheet: Mark Available | Google Sheets | Mark domain as `isAvailable=yes` | Discord: Notify Available Domain | Wait 10 Seconds |  |
| Wait 10 Seconds | Wait | Throttle and continue loop | Domain Available? (false); Update Sheet: Mark Available | Loop Through Domains |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Google Sheet**
   1. Create a spreadsheet (e.g., “Domain Target”).
   2. In `Sheet1`, create columns: `Domain` (string), `isAvailable` (string).
   3. Add domains like `example.my` with `isAvailable` set to `no`.

2. **Add Trigger**
   1. Add node **Schedule Trigger** named `Schedule: Every 30 Minutes`.
   2. Set interval to **every 30 minutes**.

3. **Add Google Sheets reader**
   1. Add **Google Sheets** node named `Fetch Target Domains from Sheet`.
   2. Credentials: configure **Google Sheets OAuth2** (Google Cloud project, consent, OAuth client; authorize in n8n).
   3. Select the spreadsheet and sheet.
   4. Add a filter: `isAvailable` equals `no`.
   5. Connect: `Schedule: Every 30 Minutes` → `Fetch Target Domains from Sheet`.

4. **Add loop**
   1. Add **Split In Batches** node named `Loop Through Domains`.
   2. Leave defaults (or set batch size to 1 if you want strict pacing).
   3. Connect: `Fetch Target Domains from Sheet` → `Loop Through Domains`.

5. **Add RDAP HTTP request**
   1. Add **HTTP Request** node named `RDAP: Check Domain Status`.
   2. URL: `https://rdap.mynic.my/rdap/domain/{{ $json.Domain }}`
   3. Response: JSON.
   4. Enable “Never Error” (so non-200 responses don’t stop execution).
   5. Connect: `Loop Through Domains` → `RDAP: Check Domain Status`.

6. **Add availability IF**
   1. Add **IF** node named `Domain Available?`.
   2. Condition (String → contains):
      - Left: `{{ $json.description[0] }}`
      - Right: `is available for registration`
   3. Connect: `RDAP: Check Domain Status` → `Domain Available?`.

7. **Add Gmail alert**
   1. Add **Gmail** node named `Gmail: Send Availability Alert`.
   2. Credentials: configure **Gmail OAuth2**; authorize the sending account.
   3. Set:
      - To: your recipient email
      - Subject: `Sniper Report! {{ $('Loop Through Domains').item.json.Domain }} is Available`
      - Body: HTML (your template). Replace any hard-coded domain text with the expression above if desired.
      - Disable attribution if available.
   4. Connect: `Domain Available?` **true** → `Gmail: Send Availability Alert`.

8. **Add Discord notification**
   1. Add **Discord** node named `Discord: Notify Available Domain`.
   2. Credentials: configure **Discord Bot** credential; ensure bot is added to the server and has channel permissions.
   3. Configure:
      - Resource: Message
      - Guild and Channel: choose target
      - Content: `{{ $('Loop Through Domains').item.json.Domain }} is available to buy!`
      - Embed title (expression):  
        `{{ $('Domain Available?').item.json.description[1].match(/https?:\/\/[^\s\)]+/)[0] }}`
   4. Connect: `Gmail: Send Availability Alert` → `Discord: Notify Available Domain`.

9. **Add sheet update**
   1. Add **Google Sheets** node named `Update Sheet: Mark Available`.
   2. Operation: `update`
   3. Matching column: `Domain`
   4. Set fields:
      - `Domain = {{ $('Loop Through Domains').item.json.Domain }}`
      - `isAvailable = yes`
   5. Connect: `Discord: Notify Available Domain` → `Update Sheet: Mark Available`.

10. **Add wait + loop back**
   1. Add **Wait** node named `Wait 10 Seconds` with amount `10 seconds`.
   2. Connect:
      - `Update Sheet: Mark Available` → `Wait 10 Seconds`
      - `Domain Available?` **false** → `Wait 10 Seconds`
      - `Wait 10 Seconds` → `Loop Through Domains` (to proceed with next item)

11. **(Optional) Add Sticky Notes**
   - Add notes describing the RDAP endpoint and setup checklist to match the original workflow documentation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Endpoint: `https://rdap.mynic.my/rdap/domain/{domain}`; “available” response includes `description[0] = "is available for registration"` and `description[1]` contains registrar list link; no API key required (public API). | MYNIC RDAP behavior note (from sticky note). |
| Workflow logic: schedule → fetch `isAvailable=no` → loop domains → RDAP check → if available send Gmail + Discord → update sheet → wait to reduce rate issues → continue loop. | High-level operation note (from sticky note). |
| Branding link included in email template: `https://khmuhtadin.com` | Email footer reference. |
| Registrar list referenced in email button: `https://rdap.mynicregistry.my/registrars/` | Used as “register now” action link in the email template. |