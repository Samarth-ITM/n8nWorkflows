Automatically track certification changes with ScrapeGraphAI, GitLab and Rocket.Chat

https://n8nworkflows.xyz/workflows/automatically-track-certification-changes-with-scrapegraphai--gitlab-and-rocket-chat-12154


# Automatically track certification changes with ScrapeGraphAI, GitLab and Rocket.Chat

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automatically track certification changes with ScrapeGraphAI, GitLab and Rocket.Chat  
**Workflow name (internal):** Certification Requirement Tracker with Rocket.Chat and GitLab

**Purpose:**  
Runs on a schedule to scrape certification requirement pages, compare the latest scraped data to the previously stored version in GitLab, and then either (a) update GitLab and notify a chat channel when changes are detected, or (b) log a “no change” event in GitLab.

**Primary use cases:**
- Compliance monitoring for certification renewals and updated requirements
- Audit-friendly tracking of changes over time using GitLab history/issues
- Automated alerting when official certification bodies change requirements

### 1.1 Logical Blocks (by node dependency)

1. **Input & Iteration**
   - Schedule trigger → build list of certifications → process each in sequence
2. **Scraping**
   - Scrape each target webpage into structured JSON
3. **Error Gate**
   - Detect scrape errors and (intended) alert path vs. continue path
4. **State Retrieval & Comparison**
   - Fetch previous JSON from GitLab → merge current+previous → compare in Code
5. **Conditional Actions**
   - If changed: prepare file → commit updated JSON to GitLab → craft message → send chat message  
   - If not changed: create a GitLab issue as a log entry

> Important mismatch: despite the title/sticky notes mentioning **Rocket.Chat**, the final messaging node is a **Slack** node (“Send a message”). Also, the IF branch for scrape errors is not wired to any alert node.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Iteration

**Overview:**  
Starts the workflow on a fixed schedule, generates the list of certification URLs to track, then iterates over them one-by-one.

**Nodes involved:**
- Daily Trigger
- Certification URL Config
- Split In Batches

#### Node: Daily Trigger
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — workflow entry point.
- **Configuration:** Runs every **24 hours** (`hoursInterval: 24`).
- **Inputs/outputs:** No inputs; outputs to **Certification URL Config**.
- **Version notes:** typeVersion `1.1`.
- **Failure modes / edge cases:**
  - If n8n instance timezone differs from expectations, “daily” timing may drift from business hours.
  - If the instance is down at trigger time, scheduled execution may be missed depending on n8n setup.

#### Node: Certification URL Config
- **Type / role:** `Code` (`n8n-nodes-base.code`) — defines tracked certifications (IDs + URLs).
- **Configuration choices:** Hardcoded list in JavaScript returning two items:
  - `certId: "pmp"`, `url: https://www.pmi.org/certifications/pmp`
  - `certId: "cissp"`, `url: https://www.isc2.org/Certifications/CISSP`
- **Key variables:**
  - Produces `item.json.certId` and `item.json.url` used downstream (scraping and GitLab path).
- **Inputs/outputs:** Input from trigger; output to **Split In Batches**.
- **Version notes:** typeVersion `2`.
- **Failure modes / edge cases:**
  - Invalid URL / blocked page / bot protection can later cause scraping failure.
  - Duplicate `certId` would cause overwriting the same GitLab file.

#### Node: Split In Batches
- **Type / role:** `Split In Batches` — processes one certification per iteration.
- **Configuration choices:** Default options (batch size defaults to 1 in typical n8n behavior).
- **Inputs/outputs:** Input from URL config; output to **Scrape Requirement Data**.
- **Version notes:** typeVersion `3`.
- **Failure modes / edge cases:**
  - If you expect looping to continue automatically, ensure the node is configured/connected properly for “next batch” behavior. In this workflow JSON, only the forward path is shown; no explicit “continue” connection is present.

---

### Block 2 — Scraping

**Overview:**  
Scrapes each certification webpage via ScrapeGraphAI using a natural-language prompt and returns structured JSON fields.

**Nodes involved:**
- Scrape Requirement Data

#### Node: Scrape Requirement Data
- **Type / role:** `ScrapeGraphAI` (`n8n-nodes-scrapegraphai.scrapegraphAi`) — extracts structured fields from a web page.
- **Configuration choices:**
  - **websiteUrl:** `={{ $json.url }}`
  - **userPrompt:** “Extract the certification name, full requirement description, last updated date, and renewal interval in years… Return JSON with keys: certName, requirementText, lastUpdated, renewalIntervalYears.”
- **Inputs/outputs:** Input from Split In Batches; output to **Scrape Error?**.
- **Version notes:** typeVersion `1` (community node; ensure installed and compatible with your n8n version).
- **Failure modes / edge cases:**
  - Website blocks scraping, requires JS rendering, or rate-limits.
  - LLM extraction inconsistencies: missing keys, wrong types (e.g., `renewalIntervalYears` as string).
  - Node may return an `error` field or throw an execution error (the workflow only checks `$json.error`, which may not exist if the node fails hard).

---

### Block 3 — Error Gate

**Overview:**  
Checks whether the scraping step produced an error flag. If no error is detected, proceeds to fetch previous data from GitLab and compare.

**Nodes involved:**
- Scrape Error?

#### Node: Scrape Error?
- **Type / role:** `IF` — conditional routing based on a boolean expression.
- **Configuration choices:**
  - Condition: `={{ $json.error }}` equals `true`
- **Connections:**
  - **True branch (error):** not connected to anything (no alert is actually sent).
  - **False branch (no error):** connected to **Fetch Previous Data** and also directly to **Merge Current & Previous** (input index 0).
- **Version notes:** typeVersion `2`.
- **Failure modes / edge cases:**
  - If ScrapeGraphAI returns errors differently (e.g., nested error object), `$json.error` may be undefined and evaluate false, incorrectly continuing.
  - Because the **true** branch is not wired, scrape errors will silently stop for that item with no notification.
  - The IF node’s false branch sends data to both Fetch and Merge; this is intentional to provide “current” to the merge’s input 0.

---

### Block 4 — State Retrieval & Comparison

**Overview:**  
Loads the previous JSON snapshot for the same certification from GitLab, merges it with the current scrape result, and computes whether anything changed.

**Nodes involved:**
- Fetch Previous Data
- Merge Current & Previous
- Detect Changes

#### Node: Fetch Previous Data
- **Type / role:** `GitLab` — fetch a file from a GitLab repository.
- **Configuration choices:**
  - Resource: **file**
  - Operation: **get**
  - `filePath`: `={{ '/certifications/' + $json.certId + '.json' }}`
  - (Branch not specified in parameters for “get”; behavior depends on node defaults—often defaults to the default branch.)
- **Inputs/outputs:** Input from IF (no-error path); output to **Merge Current & Previous** (input index 1).
- **Version notes:** typeVersion `1`.
- **Failure modes / edge cases:**
  - Auth/permission error to repo.
  - File doesn’t exist on first run → GitLab node may throw an error rather than returning empty content (depends on implementation). The subsequent Code node expects `previous.json` to be content; missing file should be handled (currently only handles JSON parse errors, not GitLab 404 failures).
  - Wrong repository/project selected in credentials.

#### Node: Merge Current & Previous
- **Type / role:** `Merge` — combine two inputs into one item by position.
- **Configuration choices:**
  - Mode: **mergeByPosition**
  - Input 0: current scraped item (from IF false branch)
  - Input 1: previous file content (from GitLab get)
- **Inputs/outputs:** Outputs merged item to **Detect Changes**.
- **Version notes:** typeVersion `2`.
- **Failure modes / edge cases:**
  - If one side produces zero items (e.g., GitLab get failed), merge may output nothing or misalign items.
  - If multiple items pass through unexpectedly, position-based merge can pair the wrong items.

#### Node: Detect Changes
- **Type / role:** `Code` — parses previous JSON and compares to current scrape.
- **Configuration choices (interpreted):**
  - Reads both inputs via `$input.all()` and assumes `[current, previous]`.
  - `scraped = current.json`
  - `prevContent = previous.json`
  - Attempts `JSON.parse(prevContent)`; if parsing fails, treats as `null`.
  - Change detection: `changed = !prevParsed || JSON.stringify(scraped) !== JSON.stringify(prevParsed)`
  - Returns merged output:
    - All scraped fields
    - `certId: current.json.certId`
    - `changed` boolean
    - `prevData` (parsed previous JSON or null)
- **Connections:** Output to **Requirement Changed?**
- **Version notes:** typeVersion `2`.
- **Failure modes / edge cases:**
  - **GitLab “get file” content format:** Many GitLab file APIs return file metadata and base64 content, not raw JSON string. If `previous.json` is an object (not a string), `JSON.parse(prevContent)` will fail.
  - `JSON.stringify` comparison is order-sensitive for object keys; ScrapeGraphAI could reorder fields or add noise, producing “changed” even when semantically identical.
  - `scraped` likely contains `url` and `certId` from upstream? Actually ScrapeGraphAI might output only extracted keys and drop `certId`; the code re-adds `certId` from `current.json.certId`, but if ScrapeGraphAI overwrote/removed it, `current.json.certId` could be missing (depends on node output behavior).

---

### Block 5 — Conditional Actions (Storage + Notifications)

**Overview:**  
Branches based on `changed`. If true, updates the JSON file in GitLab and sends a chat message. If false, logs an issue in GitLab.

**Nodes involved:**
- Requirement Changed?
- Prepare GitLab File
- Save Updated Requirement
- Craft Alert Message
- Send a message
- Log No-Change Issue

#### Node: Requirement Changed?
- **Type / role:** `IF` — checks `$json.changed`.
- **Configuration:** Boolean condition `={{ $json.changed }}` equals `true`.
- **Connections:**
  - True → **Prepare GitLab File**
  - False → **Log No-Change Issue**
- **Version notes:** typeVersion `2`.
- **Failure modes / edge cases:**
  - If `changed` is missing (undefined), it will evaluate false and route to “no change”.

#### Node: Prepare GitLab File
- **Type / role:** `Set` — intended to construct `filePath`, file content, and commit message.
- **Configuration choices:** Not configured (no fields defined).
- **Connections:** To **Save Updated Requirement**.
- **Version notes:** typeVersion `3`.
- **Failure modes / edge cases:**
  - As-is, `Save Updated Requirement` expects `filePath` and `commitMsg` in `$json` but they are never set here; GitLab edit will fail.
  - Also missing is the actual file content parameter (GitLab “edit file” requires content); this node does not provide it.

#### Node: Save Updated Requirement
- **Type / role:** `GitLab` — edits (creates/updates) a file in repo.
- **Configuration choices:**
  - Resource: **file**
  - Operation: **edit**
  - Branch: `main`
  - `filePath`: `={{ $json.filePath }}`
  - `commitMessage`: `={{ $json.commitMsg }}`
  - (No visible “content” parameter set in this JSON; typically required.)
- **Connections:** To **Craft Alert Message**
- **Version notes:** typeVersion `1`.
- **Failure modes / edge cases:**
  - Missing/empty `filePath` or `commitMsg` due to Set node not configured.
  - Missing file content parameter will cause API validation failure.
  - Permission/branch protection may reject commits to `main`.

#### Node: Craft Alert Message
- **Type / role:** `Set` — intended to prepare a message payload for chat notification.
- **Configuration choices:** Not configured (no fields defined).
- **Connections:** To **Send a message**
- **Version notes:** typeVersion `3`.
- **Failure modes / edge cases:**
  - With no fields set, the Slack node may send an empty/default message (or fail, depending on required fields).

#### Node: Send a message
- **Type / role:** `Slack` (`n8n-nodes-base.slack`) — posts a message to Slack.
- **Configuration choices:** Only `otherOptions` present; channel/text not shown (likely relies on node defaults or missing config).
- **Connections:** Terminal node.
- **Version notes:** typeVersion `2.3`.
- **Failure modes / edge cases:**
  - Missing required Slack parameters (channel, text) will fail.
  - Credential/auth errors; workspace/channel permissions.

> Note: This contradicts the workflow title and sticky notes that say Rocket.Chat. If Rocket.Chat is required, replace this node with the Rocket.Chat node.

#### Node: Log No-Change Issue
- **Type / role:** `GitLab` — creates an issue used as an audit log for “no change”.
- **Configuration choices:**
  - Title: `No change for {{$json.certId}} on {{$now.toFormat('yyyy-LL-dd')}}`
  - Labels: empty
  - Assignee IDs: empty
  - (Resource/operation not explicitly shown in the snippet, but implied “Issue → Create” by the parameters.)
- **Connections:** Terminal node.
- **Version notes:** typeVersion `1`.
- **Failure modes / edge cases:**
  - Repeated daily issues may spam the project; consider using commits or a single rolling issue instead.
  - If `certId` missing, title becomes malformed.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation block |  |  | (content) How it works + setup steps (ScrapeGraphAI/GitLab/Rocket.Chat) |
| Section – Data Collection | Sticky Note | Documentation block |  |  | (content) Data Collection & Scraping description |
| Section – Change Detection | Sticky Note | Documentation block |  |  | (content) Comparison Logic description |
| Section – Actions | Sticky Note | Documentation block |  |  | (content) Conditional Actions & Storage description |
| Daily Trigger | Schedule Trigger | Periodic workflow start | — | Certification URL Config | Section – Data Collection: Data Collection & Scraping (applies to this cluster) |
| Certification URL Config | Code | Defines certId/url list | Daily Trigger | Split In Batches | Section – Data Collection: Data Collection & Scraping |
| Split In Batches | SplitInBatches | Serializes processing per cert | Certification URL Config | Scrape Requirement Data | Section – Data Collection: Data Collection & Scraping |
| Scrape Requirement Data | ScrapeGraphAI | Scrape & extract structured data | Split In Batches | Scrape Error? | Section – Data Collection: Data Collection & Scraping |
| Scrape Error? | IF | Gate on scrape error flag | Scrape Requirement Data | Fetch Previous Data; Merge Current & Previous | Section – Data Collection: Data Collection & Scraping |
| Fetch Previous Data | GitLab (File Get) | Load previous snapshot from repo | Scrape Error? (false) | Merge Current & Previous | Section – Change Detection: Comparison Logic |
| Merge Current & Previous | Merge | Pair current scrape with previous snapshot | Scrape Error? (false); Fetch Previous Data | Detect Changes | Section – Change Detection: Comparison Logic |
| Detect Changes | Code | Parse/compare; set changed flag | Merge Current & Previous | Requirement Changed? | Section – Change Detection: Comparison Logic |
| Requirement Changed? | IF | Branch on changed | Detect Changes | Prepare GitLab File; Log No-Change Issue | Section – Actions: Conditional Actions & Storage |
| Prepare GitLab File | Set | Build filePath/content/commit message (intended) | Requirement Changed? (true) | Save Updated Requirement | Section – Actions: Conditional Actions & Storage |
| Save Updated Requirement | GitLab (File Edit) | Commit updated JSON to GitLab | Prepare GitLab File | Craft Alert Message | Section – Actions: Conditional Actions & Storage |
| Craft Alert Message | Set | Build chat alert payload (intended) | Save Updated Requirement | Send a message | Section – Actions: Conditional Actions & Storage |
| Send a message | Slack | Send notification to chat | Craft Alert Message | — | Section – Actions: Conditional Actions & Storage (note: says Rocket.Chat but node is Slack) |
| Log No-Change Issue | GitLab (Issue Create) | Audit log when no change | Requirement Changed? (false) | — | Section – Actions: Conditional Actions & Storage |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: “Certification Requirement Tracker with Rocket.Chat and GitLab” (or your preferred name).

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Set interval: every **24 hours** (or adjust cadence).

3. **Add Code node: “Certification URL Config”**
   - Node: **Code**
   - Paste and adapt logic to output items like:
     - `certId` (unique identifier)
     - `url` (certification page URL)
   - Ensure each returned item is `{ json: { certId, url } }`.

4. **Connect:** Schedule Trigger → Certification URL Config

5. **Add Split In Batches**
   - Node: **Split In Batches**
   - Keep defaults (or set Batch Size = 1 explicitly).
6. **Connect:** Certification URL Config → Split In Batches

7. **Add ScrapeGraphAI node: “Scrape Requirement Data”**
   - Node: **ScrapeGraphAI**
   - Credentials: add **ScrapeGraphAI credential** in n8n Credentials.
   - Set **Website URL** to expression: `{{$json.url}}`
   - Set **Prompt** to extract:
     - `certName`
     - `requirementText`
     - `lastUpdated`
     - `renewalIntervalYears`
8. **Connect:** Split In Batches → Scrape Requirement Data

9. **Add IF node: “Scrape Error?”**
   - Condition (boolean): `{{$json.error}}` equals `true`
10. **Connect:** Scrape Requirement Data → Scrape Error?

11. **Add GitLab node: “Fetch Previous Data”**
   - Node: **GitLab**
   - Credentials: create a **GitLab credential** (PAT/OAuth) with access to your target project/repo.
   - Resource: **File**
   - Operation: **Get**
   - File path expression: `{{ '/certifications/' + $json.certId + '.json' }}`
   - Ensure project/repo is set in the node/credential as required.
12. **Connect:** Scrape Error? (false branch) → Fetch Previous Data

13. **Add Merge node: “Merge Current & Previous”**
   - Mode: **Merge By Position**
14. **Connect current input:** Scrape Error? (false branch) → Merge Current & Previous (Input 1 / index 0 in JSON)
15. **Connect previous input:** Fetch Previous Data → Merge Current & Previous (Input 2 / index 1)

16. **Add Code node: “Detect Changes”**
   - Implement:
     - Parse previous file content into JSON
     - Compare with current scrape
     - Output: `changed`, `certId`, and current fields
   - If your GitLab “get file” returns base64 content or a wrapper object, decode/extract content first (adjust code accordingly).
17. **Connect:** Merge Current & Previous → Detect Changes

18. **Add IF node: “Requirement Changed?”**
   - Condition (boolean): `{{$json.changed}}` equals `true`
19. **Connect:** Detect Changes → Requirement Changed?

20. **Changed = true branch: prepare commit**
   - Add **Set** node: “Prepare GitLab File”
   - Configure fields you must set (minimum):
     - `filePath`: `{{ '/certifications/' + $json.certId + '.json' }}`
     - `commitMsg`: e.g. `{{ 'Update requirements for ' + $json.certId + ' on ' + $now.toFormat('yyyy-LL-dd') }}`
     - `content`: `{{ JSON.stringify($json, null, 2) }}` (or a subset containing only scraped fields)
   - Connect: Requirement Changed? (true) → Prepare GitLab File

21. **Add GitLab node: “Save Updated Requirement”**
   - Resource: **File**
   - Operation: **Edit** (or “Create/Update” depending on node capabilities)
   - Branch: `main`
   - `filePath`: `{{$json.filePath}}`
   - Commit message: `{{$json.commitMsg}}`
   - Content/body: map from `{{$json.content}}` (exact parameter name depends on the GitLab node UI)
   - Connect: Prepare GitLab File → Save Updated Requirement

22. **Add Set node: “Craft Alert Message”**
   - Create a `text` field (and optionally channel) describing:
     - cert name/id
     - what changed (you may extend Detect Changes to generate a diff)
   - Connect: Save Updated Requirement → Craft Alert Message

23. **Add chat node**
   - If you truly want Rocket.Chat: add **Rocket.Chat** node and map message text/channel.
   - If using Slack (as in provided JSON): add **Slack** node “Send a message”, configure:
     - Channel
     - Text: `{{$json.text}}`
   - Connect: Craft Alert Message → Send a message

24. **Changed = false branch: log audit event**
   - Add GitLab node: “Log No-Change Issue”
   - Resource: **Issue**
   - Operation: **Create**
   - Title: `No change for {{$json.certId}} on {{$now.toFormat('yyyy-LL-dd')}}`
   - Connect: Requirement Changed? (false) → Log No-Change Issue

25. **(Recommended) Handle scrape errors**
   - Add a Rocket.Chat/Slack node on **Scrape Error? (true)** branch to alert with `certId`, `url`, and error details.
   - Without this, scrape errors produce no notifications.

26. **Enable workflow**
   - Turn the workflow **Active** and verify GitLab/Chat messages on the next scheduled run.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes describe Rocket.Chat alerts, but the implemented messaging node is Slack (“Send a message”). | Align the node choice (Rocket.Chat vs Slack) with intended behavior. |
| Scrape error branch is not connected to any alert action. | Add a notification node on the IF true branch to avoid silent failures. |
| Change detection uses `JSON.stringify` equality. | Consider semantic comparison (ignore whitespace, normalize dates, compare selected fields only). |
| GitLab file “get” response format may not be raw JSON text. | You may need to extract/decode content before `JSON.parse`. |