Report spam and phishing URLs from IMAP mailboxes to Spamhaus

https://n8nworkflows.xyz/workflows/report-spam-and-phishing-urls-from-imap-mailboxes-to-spamhaus-12846


# Report spam and phishing URLs from IMAP mailboxes to Spamhaus

## 1. Workflow Overview

**Purpose:** Monitor IMAP folders containing spam/phishing sample emails, extract URLs from each email body, filter and de-duplicate them, then submit remaining URLs to the **Spamhaus Submission API** with context (threat type + reason).

**Target use cases:**
- Security operations / abuse reporting teams forwarding spam/phishing samples into dedicated IMAP folders.
- Automated URL reporting to Spamhaus based on real inbound email content.

### 1.1 Trigger & Email Intake
Reads new emails from one or more IMAP folders (“spam” and optionally “phishing”) and converts each email into a standardized internal structure.

### 1.2 Per-Email Loop Orchestration
Processes emails one-by-one using a batch loop so each email’s URLs can be extracted and submitted in a controlled sequence.

### 1.3 URL Extraction, Cleanup, De-duplication
Parses email text for URLs/domains, normalizes them, removes obvious noise (images/tracking), and de-duplicates the URL list for the run.

### 1.4 URL Filtering & Submission to Spamhaus
Applies regex-based allow/deny rules and submits each URL to Spamhaus with threat context.

### 1.5 Run-Level Aggregation & Optional Hooks
Aggregates submissions (primarily to count how many were reported) and exposes no-op nodes intended as extension points.

---

## 2. Block-by-Block Analysis

### Block 1 — Triggers & Initial Context
**Overview:** Watches IMAP folders and assigns per-email metadata (threat type and reason) that will later be attached to Spamhaus submissions.

**Nodes involved:**
- **Spam Trigger** (active)
- **Phishing Trigger** (disabled)
- **initial config spam**
- **initial phish config** (disabled)

#### Node: Spam Trigger
- **Type / role:** `Email Read IMAP` trigger; polls an IMAP mailbox folder for messages.
- **Key configuration:**
  - **Mailbox:** `spamhaus/spam`
  - **Format:** `resolved` (n8n resolves common email fields; body text becomes accessible like `$json.text`)
  - **IMAP search config:** `["SEEN"]` (reads messages matching SEEN flag; note this is atypical if you intend *unseen* processing)
  - **Track Last Message ID:** enabled (prevents reprocessing same message across runs, using stored state)
- **Credentials:** IMAP credential named **IMAP**
- **Outputs:** to **initial config spam**
- **Failure/edge cases:**
  - IMAP auth failures, folder not found, connectivity/timeouts
  - Using `["SEEN"]` may miss new/unseen messages depending on mailbox behavior; confirm desired semantics

#### Node: Phishing Trigger (disabled)
- **Type / role:** `Email Read IMAP` trigger for phishing samples.
- **Key configuration:**
  - **Mailbox:** `spamhaus/phishing`
  - Same IMAP options as Spam Trigger
- **Credentials:** IMAP credential **IMAP**
- **Outputs:** to **initial phish config**
- **Failure/edge cases:** same as above; additionally, because node is disabled, phishing reporting will not run unless enabled.

#### Node: initial config spam
- **Type / role:** `Set`; wraps raw email and adds reporting context.
- **Configuration choices:**
  - Creates fields:
    - `email` = entire trigger JSON (`={{ $json }}`)
    - `threat_type` = `"spam"`
    - `reason.url` = `"url used in spam email"` (stored as nested field via dot notation)
- **Inputs:** from **Spam Trigger**
- **Outputs:** to **Loop over each email**
- **Edge cases:**
  - If downstream expects `email.text`, ensure IMAP node produces `text` field in resolved format.

#### Node: initial phish config (disabled)
- **Type / role:** `Set`; same as spam config but with phishing context.
- **Configuration choices:**
  - `threat_type` = `"phish"`
  - `reason.url` = `"url used in phish email"`
- **Inputs:** from **Phishing Trigger**
- **Outputs:** to **Loop over each email**
- **Edge cases:** same as initial config spam.

---

### Block 2 — Per-Email Loop Orchestration
**Overview:** Controls processing one email at a time. The loop also provides a “run hook” output each iteration.

**Nodes involved:**
- **Loop over each email**
- **add run specific job**

#### Node: Loop over each email
- **Type / role:** `Split In Batches`; iterates over incoming emails.
- **Configuration choices:** defaults (batch size not explicitly set; n8n default is typically 1 unless configured)
- **Inputs:** from **initial config spam** and/or **initial phish config**
- **Outputs / connections:**
  - **Output 0** → **add run specific job**
  - **Output 1** → **extract all URLs**
- **Important behavior note:** In n8n, Split In Batches typically has:
  - one output for the current batch items
  - one output to continue/loop
  This workflow uses both outputs; ensure the wiring matches the intended semantics in your n8n version.
- **Failure/edge cases:**
  - If batch configuration or output semantics differ by n8n version, the loop can stall or skip URL extraction.

#### Node: add run specific job
- **Type / role:** `No Operation`; placeholder for run-level logic.
- **Inputs:** from **Loop over each email** (first output)
- **Outputs:** none (it’s a dead-end by design)
- **Typical use:** metrics, notifications, cleanup, calling a follow-up workflow.
- **Edge cases:** none (no-op).

---

### Block 3 — URL Extraction + Run De-duplication
**Overview:** Extracts URLs/domains from each email text, normalizes them, removes obvious noise, and then clears/handles deduplication before splitting into one-URL-per-item.

**Nodes involved:**
- **extract all URLs**
- **de-duplicate URLs**
- **split URLs to array**

#### Node: extract all URLs
- **Type / role:** `Code` (JavaScript); parses URLs from email body text.
- **Key logic (interpreted):**
  - Regex finds:
    - `http(s)://...` links
    - bare domains like `example.com/path` (then coerces to `http://example.com/...`)
  - Normalizes by:
    - trimming
    - stripping trailing brackets/quotes/whitespace and trailing punctuation (`.,;:`)
  - Removes noise:
    - image extensions: `.png/.jpg/.jpeg/.gif/.webp/.svg`
    - URLs containing `/pixel/tracking/`
  - De-duplicates within the node via `new Set(...)`
- **Key variables/fields:**
  - Reads `item.json.email.text`
  - Outputs: `{ urls: [...] }`
- **Inputs:** from **Loop over each email** (second output)
- **Outputs:** to **de-duplicate URLs**
- **Failure/edge cases:**
  - If `email.text` is missing (HTML-only emails, different IMAP node output), extraction may yield empty list.
  - Regex can overmatch (e.g., domains in signatures, punctuation-heavy text).
  - Normalization adds `http://` to bare domains (may be undesirable if you need original scheme).

#### Node: de-duplicate URLs
- **Type / role:** `Remove Duplicates`; here used in a non-standard way.
- **Configuration choices:**
  - **Operation:** `clearDeduplicationHistory`
- **Inputs:** from **extract all URLs**
- **Outputs:** to **split URLs to array**
- **Important behavior note:** `clearDeduplicationHistory` clears stored dedupe state; it does not itself remove duplicates unless used with a dedupe operation. In this workflow, duplicates are already removed in the Code node (`Set`), so this node effectively resets history (useful if you later switch to stateful dedupe across runs).
- **Failure/edge cases:**
  - If you intended to remove duplicates across items/runs, configuration likely needs adjustment (choose a field-based dedupe operation).

#### Node: split URLs to array
- **Type / role:** `Split Out`; converts the `urls` array into individual items.
- **Configuration choices:**
  - Field to split: `urls`
- **Inputs:** from **de-duplicate URLs**
- **Outputs:** to **filter out URLs that match regexes**
- **Edge cases:**
  - If `urls` is empty or missing, it may output no items (downstream nodes won’t run).

---

### Block 4 — Filtering + Spamhaus Submission
**Overview:** Filters out non-relevant URLs based on regex rules, constructs the Spamhaus API payload, and submits each URL.

**Nodes involved:**
- **filter out URLs that match regexes**
- **create item for spamhaus**
- **Spamhaus submit url**

#### Node: filter out URLs that match regexes
- **Type / role:** `Filter`; excludes URLs matching unwanted patterns.
- **Configuration choices:**
  - Condition: `$json.urls` **notRegex** `/(privacy|imprint|impressum)/i`
  - (Case sensitive setting shown as true, but regex has `/i`; the regex itself is case-insensitive.)
- **Inputs:** from **split URLs to array**
- **Outputs:** to **create item for spamhaus**
- **Edge cases:**
  - Only one regex rule is configured; you may want to expand to exclude common benign destinations (unsubscribe, known CDNs, etc.).
  - If `$json.urls` is not a string (e.g., unexpected structure), regex evaluation may fail under strict validation.

#### Node: create item for spamhaus
- **Type / role:** `Set`; builds the API submission object.
- **Configuration choices (payload fields):**
  - `source.object` = `={{ $json.urls }}` (the URL string for this item)
  - `threat_type` = `={{ $('Loop over each email').item.json.threat_type }}`
  - `reason` = `={{ $('Loop over each email').item.json.reason.url }}`
- **Inputs:** from **filter out URLs that match regexes**
- **Outputs:** to **Spamhaus submit url**
- **Important dependency:** Uses **cross-node item reference** to `Loop over each email` to pick the current email’s context. This relies on n8n’s item linking behaving as expected in your execution mode.
- **Edge cases:**
  - If multiple emails are processed and item linkage is lost/misaligned, threat_type/reason may be incorrect or undefined.
  - If `reason.url` is absent, expression resolves to empty and may violate Spamhaus API requirements (if any).

#### Node: Spamhaus submit url
- **Type / role:** `HTTP Request`; calls Spamhaus submission endpoint.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://submit.spamhaus.org/portal/api/v1/submissions/add/url`
  - **Body:** JSON (stringified via `={{ $json.toJsonString() }}`)
  - **Auth:** `httpHeaderAuth` via generic credential type (header-based API key/token)
- **Credentials:** HTTP Header Auth credential named **Spamhaus**
- **Inputs:** from **create item for spamhaus**
- **Outputs:** to **aggregate all into a single list**
- **Failure/edge cases:**
  - 401/403 if header auth misconfigured/expired
  - 429 rate limiting (bulk URL submissions)
  - API validation errors if payload fields don’t match Spamhaus expectations
  - Using `toJsonString()` creates a JSON string; most APIs expect an object. If Spamhaus expects JSON object, consider sending `$json` directly (depends on node/body settings and n8n version).

---

### Block 5 — Aggregation + Extension Hooks
**Overview:** Collects results mainly to count reported URLs and provides a placeholder for per-email follow-up actions, then signals the loop to continue.

**Nodes involved:**
- **aggregate all into a single list**
- **add email specific job**

#### Node: aggregate all into a single list
- **Type / role:** `Aggregate`; combines item data.
- **Notes (in node):** “This is solely to count how many URLs were reported.”
- **Configuration choices:**
  - Aggregate mode: “aggregate all item data” (collect all incoming items into a single item)
- **Inputs:** from **Spamhaus submit url**
- **Outputs:** to:
  - **Loop over each email** (to continue the batch loop)
  - **add email specific job**
- **Edge cases:**
  - Aggregating all item data can grow large if many URLs are submitted; may increase memory usage.
  - If the intention is just a count, a lighter-weight approach is to increment a counter or use “Item Lists”/metrics.

#### Node: add email specific job
- **Type / role:** `No Operation`; placeholder for per-email logic after all URLs for that email are submitted.
- **Inputs:** from **aggregate all into a single list**
- **Outputs:** none
- **Typical use:** label/move email, notify analysts, write to DB, etc.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Spam Trigger | Email Read IMAP | Trigger: read spam samples from IMAP folder | — | initial config spam | # Triggers |
| Phishing Trigger | Email Read IMAP | Trigger: read phishing samples from IMAP folder (currently disabled) | — | initial phish config | # Triggers |
| initial config spam | Set | Add context (threat type + reason) and wrap email | Spam Trigger | Loop over each email | # Report URLs to Spamhaus (full note applies to workflow) |
| initial phish config | Set | Add context for phishing (disabled) | Phishing Trigger | Loop over each email | # Report URLs to Spamhaus (full note applies to workflow) |
| Loop over each email | Split In Batches | Iterate through emails one by one | initial config spam, initial phish config | add run specific job; extract all URLs | # Process each email individually |
| add run specific job | No Operation | Placeholder for run-level tasks | Loop over each email | — | # Process each email individually |
| extract all URLs | Code | Extract + normalize URLs from email text | Loop over each email | de-duplicate URLs | # Process each email individually |
| de-duplicate URLs | Remove Duplicates | Clears dedupe history (duplicates already handled in Code) | extract all URLs | split URLs to array | # Process each email individually |
| split URLs to array | Split Out | Convert URLs array into one item per URL | de-duplicate URLs | filter out URLs that match regexes | # Process each email individually |
| filter out URLs that match regexes | Filter | Exclude benign/common URLs by regex | split URLs to array | create item for spamhaus | # Process each email individually |
| create item for spamhaus | Set | Build Spamhaus submission payload | filter out URLs that match regexes | Spamhaus submit url | # Process each email individually |
| Spamhaus submit url | HTTP Request | Submit URL to Spamhaus API | create item for spamhaus | aggregate all into a single list | # Process each email individually |
| aggregate all into a single list | Aggregate | Aggregate results (used for counting) + drive loop continuation | Spamhaus submit url | Loop over each email; add email specific job | # Process each email individually |
| add email specific job | No Operation | Placeholder for per-email post-processing | aggregate all into a single list | — | # Process each email individually |
| Sticky Note | Sticky Note | Comment | — | — | # Triggers |
| Sticky Note1 | Sticky Note | Comment | — | — | # Report URLs to Spamhaus (contains setup/behavior description) |
| Sticky Note2 | Sticky Note | Comment | — | — | # Process each email individually |

---

## 4. Reproducing the Workflow from Scratch

1. **Create IMAP credential**
   - In n8n Credentials: add **IMAP** credential (host, port, TLS, username/password or OAuth depending on provider).

2. **Add trigger for spam samples**
   - Add node: **Email Read IMAP** named **Spam Trigger**
   - Set:
     - *Mailbox*: `spamhaus/spam`
     - *Format*: `resolved`
     - *Options*:
       - Custom Email Config: `["SEEN"]`
       - Track Last Message ID: enabled
   - Select IMAP credential created in step 1.

3. **(Optional) Add trigger for phishing samples**
   - Add another **Email Read IMAP** named **Phishing Trigger**
   - Mailbox: `spamhaus/phishing`
   - Same options as above
   - Enable it if you want phishing reporting.

4. **Add initial context nodes**
   - Add **Set** node named **initial config spam** connected from **Spam Trigger**
     - Create fields:
       - `email` (Object) = expression `{{$json}}`
       - `threat_type` (String) = `spam`
       - `reason.url` (String) = `url used in spam email`
   - Add **Set** node named **initial phish config** connected from **Phishing Trigger**
     - `threat_type` = `phish`
     - `reason.url` = `url used in phish email`

5. **Add loop node**
   - Add **Split In Batches** named **Loop over each email**
   - Connect **initial config spam** → **Loop over each email**
   - Connect **initial phish config** → **Loop over each email** (if used)
   - Leave batch options default (or set batch size to 1 explicitly for clarity).

6. **Add run hook (optional)**
   - Add **No Operation** named **add run specific job**
   - Connect **Loop over each email (output 0)** → **add run specific job**

7. **Add URL extraction**
   - Add **Code** node named **extract all URLs**
   - Connect **Loop over each email (output 1)** → **extract all URLs**
   - Paste the JS logic (same as workflow) that:
     - reads `item.json.email.text`
     - outputs `{ urls: [...] }`

8. **Add de-duplication node**
   - Add **Remove Duplicates** node named **de-duplicate URLs**
   - Set operation to **Clear deduplication history**
   - Connect **extract all URLs** → **de-duplicate URLs**

9. **Split URLs into individual items**
   - Add **Split Out** node named **split URLs to array**
   - Field to split out: `urls`
   - Connect **de-duplicate URLs** → **split URLs to array**

10. **Add URL filter**
   - Add **Filter** node named **filter out URLs that match regexes**
   - Condition: String → `{{$json.urls}}` **does not match regex** `/(privacy|imprint|impressum)/i`
   - Connect **split URLs to array** → **filter out URLs that match regexes**

11. **Build Spamhaus payload**
   - Add **Set** node named **create item for spamhaus**
   - Add fields:
     - `source.object` (String) = `{{$json.urls}}`
     - `threat_type` (String) = `{{$('Loop over each email').item.json.threat_type}}`
     - `reason` (String) = `{{$('Loop over each email').item.json.reason.url}}`
   - Connect **filter out URLs that match regexes** → **create item for spamhaus**

12. **Configure Spamhaus credential**
   - Create credential: **HTTP Header Auth** named **Spamhaus**
   - Add required header(s) as per Spamhaus API portal (for example, an API key header).

13. **Submit to Spamhaus**
   - Add **HTTP Request** node named **Spamhaus submit url**
   - Configure:
     - Method: POST
     - URL: `https://submit.spamhaus.org/portal/api/v1/submissions/add/url`
     - Authentication: Generic Credential → **HTTP Header Auth** (select **Spamhaus**)
     - Body: JSON
       - Body value expression: `{{$json.toJsonString()}}` (or adjust to send the object if needed by API)
   - Connect **create item for spamhaus** → **Spamhaus submit url**

14. **Aggregate results**
   - Add **Aggregate** node named **aggregate all into a single list**
   - Set to aggregate all item data
   - Connect **Spamhaus submit url** → **aggregate all into a single list**

15. **Add per-email hook and close the loop**
   - Add **No Operation** node named **add email specific job**
   - Connect **aggregate all into a single list** → **add email specific job**
   - Also connect **aggregate all into a single list** → **Loop over each email** (this is the loop continuation line)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Report URLs to Spamhaus” sticky note explains intent, modular design, setup steps, URL normalization, noise removal, and optional run/email-level extension points. | Applies to the whole workflow |
| Spamhaus submission endpoint used: `https://submit.spamhaus.org/portal/api/v1/submissions/add/url` | Used by the HTTP Request node |
| The workflow currently has the phishing trigger and phishing config disabled; only the spam folder is active. | Operational behavior / deployment note |
| IMAP search uses `["SEEN"]`; confirm whether you intended to process seen vs unseen emails. | Avoid missing samples or reprocessing patterns |