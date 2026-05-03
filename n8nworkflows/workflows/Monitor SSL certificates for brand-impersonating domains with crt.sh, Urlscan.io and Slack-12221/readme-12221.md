Monitor SSL certificates for brand-impersonating domains with crt.sh, Urlscan.io and Slack

https://n8nworkflows.xyz/workflows/monitor-ssl-certificates-for-brand-impersonating-domains-with-crt-sh--urlscan-io-and-slack-12221


# Monitor SSL certificates for brand-impersonating domains with crt.sh, Urlscan.io and Slack

## 1. Workflow Overview

**Purpose:**  
This workflow monitors public SSL certificate transparency logs (via **crt.sh**) for newly issued certificates containing brand-like domains, filters/whitelists legitimate domains, runs suspicious domains through **Urlscan.io** to obtain a report + screenshot, and posts actionable alerts to **Slack**.

**Primary use cases:**
- Brand impersonation / typosquatting detection (phishing lookalike domains)
- Early warning when attackers obtain TLS certificates for deceptive domains
- Lightweight security monitoring that provides visual evidence (screenshots) for triage

### 1.1 Scheduling & Data Collection
Runs every hour and queries crt.sh for certificates matching a wildcard pattern.

### 1.2 Normalization, Filtering & Deduplication
Splits multi-domain certificate entries into single domains, then filters to “recent + not whitelisted + unique”.

### 1.3 Scanning & Enrichment (Urlscan.io)
For each suspicious domain: trigger a Urlscan.io scan, wait for completion, then fetch final results including screenshot URL/report.

### 1.4 Alerting & Looping
Send a Slack message with domain, urlscan report link, and screenshot; then loop back to process the next suspicious domain.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & crt.sh Query
**Overview:** Triggers hourly execution and pulls certificate log matches from crt.sh using a wildcard search for the monitored brand/domain pattern.  
**Nodes involved:** `Schedule (Every Hour)`, `Poll crt.sh (SSL Logs)`

#### Node: Schedule (Every Hour)
- **Type / Role:** `scheduleTrigger` — periodic workflow entry point.
- **Configuration (interpreted):** Runs on an interval of **1 hour**.
- **Inputs/Outputs:**  
  - **Output →** `Poll crt.sh (SSL Logs)`
- **Edge cases / failures:** None typical; if n8n instance is down, missed schedules won’t run until recovery (unless using queue/retry strategy outside this workflow).

#### Node: Poll crt.sh (SSL Logs)
- **Type / Role:** `httpRequest` — fetches JSON search results from crt.sh.
- **Configuration choices:**
  - URL (expression): `https://crt.sh/?q=%.testdomain.com&output=json`
    - Uses `%` wildcard to match subdomains (crt.sh query syntax).
  - **Retry on Fail:** enabled
  - **maxTries:** 5
  - **waitBetweenTries:** 5000 ms
- **Key variables/expressions:** none beyond the fixed URL expression.
- **Inputs/Outputs:**  
  - **Input ←** `Schedule (Every Hour)`  
  - **Output →** `Split Out`
- **Edge cases / failures:**
  - crt.sh commonly returns **502/504** or intermittent failures; retry mitigates this.
  - Response may be empty, non-JSON, or rate-limited; downstream nodes may receive 0 items.
  - Data shape may change; the workflow expects fields like `name_value` and sometimes `entry_timestamp`.

---

### Block 2 — Split, Filter, Deduplicate
**Overview:** Converts crt.sh results into one domain per item, removes older entries, excludes known legitimate domains, and prevents repeated scanning of the same domain within a run.  
**Nodes involved:** `Split Out`, `Filter & Deduplicate`, `Split In Batches`

#### Node: Split Out
- **Type / Role:** `splitOut` — expands an array/string-like field into multiple items.
- **Configuration choices:**
  - **Field to split out:** `name_value`
  - This is intended to handle certificates containing multiple SANs/domains.
- **Inputs/Outputs:**  
  - **Input ←** `Poll crt.sh (SSL Logs)`  
  - **Output →** `Filter & Deduplicate`
- **Edge cases / failures:**
  - If `name_value` is missing or not splittable as expected, it may output unexpected items or none.
  - crt.sh `name_value` can contain newline-separated domains; behavior depends on node implementation/version.

#### Node: Filter & Deduplicate
- **Type / Role:** `code` — custom JavaScript filtering, recency check, wildcard cleanup, and deduplication.
- **Configuration choices (logic):**
  - Whitelist array:  
    `myDomains = ["yourdomain.com","testdomain.com","google.com"]`
  - Recency window: `hoursAgo = 4000` (cutoff date = now − 4000 hours)
  - For each item:
    - Reads domain from `item.json.name_value`
    - Reads certificate timestamp from `item.json.entry_timestamp` (or uses current time as fallback)
    - Removes wildcard prefix `*.` from domain
    - Conditions to alert:
      - `certDate > cutoffDate` (recent)
      - not in `myDomains`
      - not already seen in this run (Set-based)
      - not empty
  - Returns items shaped as: `{ domain: cleanDomain, date: certDate }`
- **Key expressions/variables used:** `items`, `myDomains`, `hoursAgo`, `entry_timestamp`, `name_value`, `seenDomains`
- **Inputs/Outputs:**  
  - **Input ←** `Split Out`  
  - **Output →** `Split In Batches`
- **Edge cases / failures:**
  - If `entry_timestamp` is absent or unparsable, fallback uses “now”, which can create false “new” classifications during testing.
  - Whitelist is exact-match only; subdomains of whitelisted roots are *not* automatically excluded unless they appear exactly or you add pattern logic.
  - Internationalized domains/punycode or whitespace/newline artifacts may require additional normalization.
  - The `hoursAgo` is very large (~166 days). If the intent is near-real-time detection, reduce it.

#### Node: Split In Batches
- **Type / Role:** `splitInBatches` — processes suspicious domains one-by-one (or in batches) and supports looping.
- **Configuration choices:** Defaults (no batch size explicitly set in parameters shown).
- **Inputs/Outputs:**  
  - **Input ←** `Filter & Deduplicate` and loop-back from `Alert Slack`
  - **Output (main index 1) →** `Perform a scan`  
  - **Output (main index 0)** is unused here (commonly used as “done/no items” path).
- **Operational behavior note:** The workflow relies on a **loop**: after Slack alert, it feeds back into `Split In Batches` to continue with the next item.
- **Edge cases / failures:**
  - If the loop-back connection is removed, only the first domain may be processed.
  - If the incoming item shape changes, downstream expressions referencing `domain` may fail.

---

### Block 3 — Urlscan.io Scan & Retrieval
**Overview:** Submits each suspicious domain to Urlscan.io, waits for processing, then retrieves the full scan result including a screenshot URL.  
**Nodes involved:** `Perform a scan`, `Wait for Scan`, `Get a scan`

#### Node: Perform a scan
- **Type / Role:** `urlScanIo` — creates a scan task on Urlscan.io.
- **Configuration choices:**
  - URL: `={{ $json.domain }}` (expects current item has `domain`)
- **Inputs/Outputs:**  
  - **Input ←** `Split In Batches` (main output index 1)  
  - **Output →** `Wait for Scan`
- **Edge cases / failures:**
  - Requires Urlscan.io credentials/API key configured in n8n.
  - Urlscan may reject malformed URLs; consider ensuring scheme (`http/https`) if needed.
  - Rate limits/quota limitations can cause failures.
  - If a domain does not resolve, Urlscan can still create a scan but may produce limited results.

#### Node: Wait for Scan
- **Type / Role:** `wait` — pauses execution to allow scan completion.
- **Configuration choices:**
  - Wait duration: **30 seconds** (note: sticky note mentions 90 seconds; the node is configured for 30s)
- **Inputs/Outputs:**  
  - **Input ←** `Perform a scan`  
  - **Output →** `Get a scan`
- **Edge cases / failures:**
  - If 30 seconds is insufficient, `Get a scan` may return “not found” / incomplete data. Consider increasing to 60–120s or implementing polling/retry.
  - Wait nodes can hold executions; high volume can increase memory/queue usage.

#### Node: Get a scan
- **Type / Role:** `urlScanIo` — retrieves scan results by scan ID.
- **Configuration choices:**
  - Operation: `get`
  - Scan ID: `={{ $json.scanId }}`
    - Assumes `Perform a scan` output includes `scanId`.
- **Inputs/Outputs:**  
  - **Input ←** `Wait for Scan`  
  - **Output →** `Alert Slack`
- **Edge cases / failures:**
  - If scan not ready, may return partial response or error.
  - If `$json.scanId` missing due to upstream change/error, expression fails.
  - Urlscan response fields may differ; the Slack step expects screenshot URL presence.

---

### Block 4 — Slack Alerting & Loop Control
**Overview:** Posts a formatted Slack alert including the suspicious domain, urlscan report, and screenshot; then loops back to process the next domain.  
**Nodes involved:** `Alert Slack`

#### Node: Alert Slack
- **Type / Role:** `slack` — posts a message to a channel.
- **Configuration choices:**
  - Mode: `channel`
  - Channel ID: `C0917N0QN2C` (cached name shown: `content-creation-agent`)
  - Message text (expression):
    - Domain from batch node: `{{ $node["Split In Batches"].json["domain"] }}`
    - Report link: `{{ $('Perform a scan').item.json.result }}`
    - Screenshot: `{{ $json.task.screenshotURL }}`
  - `unfurl_media: true` to preview screenshot/media
- **Inputs/Outputs:**  
  - **Input ←** `Get a scan`  
  - **Output →** `Split In Batches` (loop-back)
- **Credentials / Auth note:**
  - Node shows a `webhookId` field set to `YOUR_SLACK_WEBHOOK_HERE`, but it is configured as a Slack node with channel selection (typically **Slack OAuth/Bot token**). Ensure the correct Slack credential type is configured in n8n.
- **Edge cases / failures:**
  - Missing permissions (chat:write, channel access) or wrong channel ID leads to auth/permission errors.
  - If `$json.task.screenshotURL` is absent (scan failed/not ready), message may contain blank screenshot link.
  - If `Perform a scan` result field path changes, the report link expression may break.
  - Looping continues until batches are exhausted; ensure `Split In Batches` is correctly configured to terminate.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule (Every Hour) | scheduleTrigger | Hourly trigger / entry point | — | Poll crt.sh (SSL Logs) | ## Phishing Lookout: Brand Impersonation Monitor<br>This workflow monitors SSL certificate logs to find and scan new domains that might be impersonating your brand.<br><br>## Background<br>In modern cybersecurity, Brand Impersonation (or "Typosquatting") is quite common in phishing attacks. Attackers register domains that look nearly identical to a trusted brand—such as .xyz-n8n.io, n8n.i0, etc. instead of the legitimate— to deceive users into revealing sensitive credentials or downloading malware.<br><br>## How it works<br>Monitor: Checks crt.sh every hour for new SSL certificates matching your brand keywords.<br><br>Process: Uses a Split Out node to handle multi-domain certificates and a Filter node to ignore your own legitimate domains.<br><br>Scan: Automatically sends suspicious domains to Urlscan.io for a headless browser scan and screenshot.<br><br>Triage: Implements a 90-second Wait to allow the scan to finish before fetching results.<br><br>Alert: Sends a Slack message with the domain name, report link, and an image of the supposedly suspicious site trying to mimic your site login page, etc. for phishing.<br><br>## Setup Steps<br>Credentials: Connect your Urlscan.io API key and Slack bot token.<br><br>Configuration: Update the "Poll crt.sh" node.<br>In URL https://crt.sh/?q=%.testdomain.com&output=json, use your specific brand name (e.g., %.yourbrand.com or .yourdomain.com instead of .testdomain.com).<br><br>Whitelist: Add your real domains to the myDomains list in the Filter & Deduplicate code node to prevent false alerts. Alternatively, you may also NOT opt to include your own domain for testing purposes to check how the Workflow behaves and outputs. In such case, obviously, your domain and sub-domains also are highlighted as Suspicious (as received in Slack Alerts)<br><br>Looping: Ensure the Alert Slack node output is connected back to the Split In Batches input to process all found domains. |
| Poll crt.sh (SSL Logs) | httpRequest | Query crt.sh certificate transparency logs | Schedule (Every Hour) | Split Out | ## Poll crt.sh<br>- Searches the crt.sh database for new SSL certificates using wildcard queries (e.g., %.testdomain.com).<br>- Configured with Retry on Fail to handle intermittent <br>502/504 errors from the public database.<br>- Split items (domains) into separate items just in case |
| Split Out | splitOut | Expand multi-domain cert entries into separate items | Poll crt.sh (SSL Logs) | Filter & Deduplicate | ## Poll crt.sh<br>- Searches the crt.sh database for new SSL certificates using wildcard queries (e.g., %.testdomain.com).<br>- Configured with Retry on Fail to handle intermittent <br>502/504 errors from the public database.<br>- Split items (domains) into separate items just in case |
| Filter & Deduplicate | code | Whitelist filtering, recency filtering, remove wildcards, dedupe | Split Out | Split In Batches | ## Filter & Deduplicate<br>- Only allows domains with certificates registered within the last 'x' hours.<br>- Uses a myDomains list to exclude your company's legitimate domains from the scan. (Productive scenario)<br>- Deduplication: Ensures the same domain isn't scanned multiple times in the same run. |
| Split In Batches | splitInBatches | Iteration/loop control over suspicious domains | Filter & Deduplicate; Alert Slack | Perform a scan |  |
| Perform a scan | urlScanIo | Submit domain to Urlscan.io for scanning | Split In Batches | Wait for Scan | ## Perform scan, Wait, Get results in loop <br>Scan Trigger: Submits the suspicious URL to Urlscan.io for a safe, remote headless browser analysis.<br>Captures the uuid (Scan ID) needed to retrieve the results in the next step.<br>### Wait for Scan<br>Pauses the workflow for 90 seconds to allow the remote browser to finish loading the site and taking a screenshot.<br>Prevents "resource not found" errors during the retrieval step.<br>### Get a scan<br>Uses the Scan ID to fetch the final report, including the site "screenshot" and technical risk data. Gets visual evidence of the malicious site into the workflow. |
| Wait for Scan | wait | Delay to allow urlscan processing | Perform a scan | Get a scan | ## Perform scan, Wait, Get results in loop <br>Scan Trigger: Submits the suspicious URL to Urlscan.io for a safe, remote headless browser analysis.<br>Captures the uuid (Scan ID) needed to retrieve the results in the next step.<br>### Wait for Scan<br>Pauses the workflow for 90 seconds to allow the remote browser to finish loading the site and taking a screenshot.<br>Prevents "resource not found" errors during the retrieval step.<br>### Get a scan<br>Uses the Scan ID to fetch the final report, including the site "screenshot" and technical risk data. Gets visual evidence of the malicious site into the workflow. |
| Get a scan | urlScanIo | Retrieve urlscan report + screenshot data by scanId | Wait for Scan | Alert Slack | ## Perform scan, Wait, Get results in loop <br>Scan Trigger: Submits the suspicious URL to Urlscan.io for a safe, remote headless browser analysis.<br>Captures the uuid (Scan ID) needed to retrieve the results in the next step.<br>### Wait for Scan<br>Pauses the workflow for 90 seconds to allow the remote browser to finish loading the site and taking a screenshot.<br>Prevents "resource not found" errors during the retrieval step.<br>### Get a scan<br>Uses the Scan ID to fetch the final report, including the site "screenshot" and technical risk data. Gets visual evidence of the malicious site into the workflow. |
| Alert Slack | slack | Send security alert to Slack and continue loop | Get a scan | Split In Batches | ## Alert Slack<br>Notification: Sends an alert to the security channel containing the malicious site, domain, report link, and site screenshot.<br><br>Note, "Unfurl Media" is enabled to display the screenshot directly in the chat for instant triage. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it e.g. *Phishing Lookout and Brand Domain Monitor Workflow*.

2. **Add Trigger: “Schedule Trigger”**  
   - Node name: `Schedule (Every Hour)`  
   - Set **Rule → Interval → 1 hour**.

3. **Add HTTP Request node (crt.sh query)**  
   - Node name: `Poll crt.sh (SSL Logs)`  
   - Method: **GET**  
   - URL: `https://crt.sh/?q=%.YOURTARGETDOMAIN.com&output=json`  
     - Example: `https://crt.sh/?q=%.example.com&output=json`
   - Enable **Retry on Fail**  
   - Set retries to **5** and delay to **5000 ms** (or similar).

4. **Connect** `Schedule (Every Hour)` → `Poll crt.sh (SSL Logs)`.

5. **Add “Split Out” node**  
   - Node name: `Split Out`  
   - Field to split out: `name_value`  
   - Connect `Poll crt.sh (SSL Logs)` → `Split Out`.

6. **Add “Code” node**  
   - Node name: `Filter & Deduplicate`  
   - Paste logic equivalent to:
     - A whitelist array `myDomains` (put your legitimate domains here)
     - A recency cutoff (e.g. last N hours)
     - Remove `*.` prefix
     - Dedupe using a Set per execution
     - Output items shaped like `{ domain, date }`
   - Connect `Split Out` → `Filter & Deduplicate`.

7. **Add “Split In Batches” node**  
   - Node name: `Split In Batches`  
   - Keep defaults (or set batch size = 1 if you want strict one-at-a-time behavior).  
   - Connect `Filter & Deduplicate` → `Split In Batches`.

8. **Add Urlscan.io node to submit scan**  
   - Node name: `Perform a scan`  
   - Operation: (default scan/submit)  
   - URL field: `{{$json.domain}}`  
   - **Credentials:** Add Urlscan.io API key credentials in n8n and select them for this node.  
   - Connect `Split In Batches` **(output 1)** → `Perform a scan`.

9. **Add “Wait” node**  
   - Node name: `Wait for Scan`  
   - Wait: **30 seconds** (consider 60–120s in real use).  
   - Connect `Perform a scan` → `Wait for Scan`.

10. **Add Urlscan.io node to fetch results**  
   - Node name: `Get a scan`  
   - Operation: **Get**  
   - Scan ID: `{{$json.scanId}}`  
   - Connect `Wait for Scan` → `Get a scan`.

11. **Add Slack node for alerting**  
   - Node name: `Alert Slack`  
   - Resource/Operation: post message (standard Slack “message” action)
   - Select mode: **Channel**, choose the target channel.
   - Text (adapt as needed), using expressions similar to:
     - Domain: `{{$node["Split In Batches"].json["domain"]}}`
     - Report link: `{{ $('Perform a scan').item.json.result }}`
     - Screenshot URL: `{{$json.task.screenshotURL}}`
   - Enable **Unfurl media** if you want the screenshot preview.
   - **Credentials:** configure Slack OAuth2/Bot token credentials with permission to post to the channel.
   - Connect `Get a scan` → `Alert Slack`.

12. **Close the loop for batch processing**
   - Connect `Alert Slack` → `Split In Batches` (back into its input).  
   - This ensures all suspicious domains found in the run are processed sequentially.

13. **Test**
   - Temporarily remove your main domain from `myDomains` to confirm alerts fire, then re-add it to reduce false positives.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Phishing Lookout: Brand Impersonation Monitor — monitors SSL logs, scans suspicious domains with Urlscan.io, and alerts Slack with screenshot/report. | Sticky note “Phishing Lookout: Brand Impersonation Monitor” (in-workflow documentation) |
| Update crt.sh query: change `%.testdomain.com` to your brand/domain pattern. | Sticky note “Phishing Lookout…” and “Poll crt.sh” |
| Whitelist your legitimate domains in `myDomains` in the code node to avoid false alerts. | Sticky note “Phishing Lookout…” and “Filter & Deduplicate” |
| Wait duration mismatch: notes mention 90 seconds but node is configured for 30 seconds; increase if “resource not found” occurs when fetching urlscan results. | Sticky note “Perform scan, Wait, Get results in loop” vs node configuration |
| Ensure Slack alert loops back into Split In Batches to process all results. | Sticky note “Phishing Lookout…” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.