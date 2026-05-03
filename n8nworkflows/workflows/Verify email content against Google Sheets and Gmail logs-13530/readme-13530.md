Verify email content against Google Sheets and Gmail logs

https://n8nworkflows.xyz/workflows/verify-email-content-against-google-sheets-and-gmail-logs-13530


# Verify email content against Google Sheets and Gmail logs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Verify email content against Google Sheets and Gmail logs  
**Workflow name (in JSON):** “My workflow”  
**Purpose:** Automate email QA by fetching unread Gmail messages from a specific sender, extracting key elements (From/Reply-To, Subject, Preheader, Body), comparing them to “Expected” values stored in Google Sheets, and writing back “Actual” + “Result” (match/mismatch) to Google Sheets.

### 1.1 Trigger & Email Retrieval
- Manual execution starts the workflow.
- Gmail “getAll” retrieves unread emails from a configured sender (including spam/trash).

### 1.2 Expected Content Source (Google Sheets)
- Loads expected values (rows containing an `Expected` column) from a Google Sheet.

### 1.3 HTML Parsing (Preheader)
- Extracts preheader text from the email HTML via a CSS selector (`div#emailPreHeader`).

### 1.4 Data Consolidation & Validation
- Merges three streams (Gmail message, expected rows, extracted preheader).
- JavaScript cleans/decodes email HTML, determines which “actual” field to compare, and outputs a per-row QA result.

### 1.5 Logging Back to Google Sheets
- Appends or updates rows in Google Sheets with `Expected`, `Actual`, `Result` (and a `Label` if present).

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Gmail Intake
**Overview:** Starts execution manually and pulls candidate emails to validate (unread, from a specific sender).  
**Nodes involved:**  
- When clicking ‘Execute workflow’
- Get many messages

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point for ad-hoc runs.
- **Configuration:** No parameters.
- **Connections:**
  - **Output →** Get many messages
- **Edge cases / failures:**
  - None typical; only runs when manually executed.

#### Node: Get many messages
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — fetches email messages.
- **Operation:** `getAll`
- **Key configuration choices:**
  - `simple: false` (returns a richer message structure)
  - Filters:
    - `sender: user@example.com`
    - `readStatus: unread`
    - `includeSpamTrash: true`
- **Credentials:** Gmail OAuth2 (“Gmail account”)
- **Connections (notable):**
  - **Output →** Extracting Preheader From HTML (index 0)
  - **Output →** Combine HTML, Expected Content and Preheader (index 1)
  - **Output →** Load Expected Content (index 0) *(this is unusual; see edge cases)*
- **Edge cases / failures:**
  - OAuth token expiry / missing Gmail scopes.
  - Large inbox queries may paginate; “getAll” can be slow or hit API limits.
  - Multiple unread messages: downstream code later selects only the *first matching* Gmail item (see Block 4), so results can be misleading if >1 email is returned.
  - Sender filter exactness: aliases or display-names may not match as expected.

---

### Block 2 — Load Expected Content (Google Sheets)
**Overview:** Reads expected QA checkpoints from Google Sheets (rows that include an `Expected` value).  
**Nodes involved:**  
- Load Expected Content

#### Node: Load Expected Content
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — intended to read expected rows.
- **Configuration (as provided):**
  - `documentId` is set via a Google Sheets URL (`https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>/edit`)
  - `sheetName` is set to an **expression containing JavaScript + googleapis code**.
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account”)
- **Connections:**
  - **Input ←** Get many messages *(workflow wiring)*
  - **Output →** Combine HTML, Expected Content and Preheader (index 0)
- **Version-specific notes:**
  - Node typeVersion `4.7`.
- **Important issue / edge cases:**
  - The `sheetName` parameter is misused: it contains code that looks like it belongs in a Code node, not in a Sheets node parameter. In n8n, `sheetName` should be a literal sheet/tab name (e.g., `Sheet1`) or a simple expression resolving to a name/ID—not a block of Node.js using `googleapis`.
  - Because of this, the node may fail validation or execution, or it may simply pass an invalid sheet name to the Google Sheets API.
  - Also, connecting **Gmail output into a Sheets “read” node** is not required unless you intentionally want to loop sheets reads per email item. As configured, if Gmail returns N messages, Sheets could run N times (depending on execution mode), creating duplication.

---

### Block 3 — Extract Preheader from Email HTML
**Overview:** Parses the email HTML to extract the preheader text used for validation.  
**Nodes involved:**  
- Extracting Preheader From HTML

#### Node: Extracting Preheader From HTML
- **Type / role:** HTML (`n8n-nodes-base.html`) — extracts data from HTML by CSS selector.
- **Operation:** `extractHtmlContent`
- **Key configuration:**
  - `dataPropertyName: "html"` (expects the HTML input to be in `$json.html`)
  - Extraction values:
    - key: `preheader`
    - selector: `div#emailPreHeader`
- **Connections:**
  - **Input ←** Get many messages
  - **Output →** Combine HTML, Expected Content and Preheader (index 2)
- **Edge cases / failures:**
  - Gmail node typically provides HTML in fields like `textAsHtml`, not necessarily `$json.html`. If the incoming item does not contain `$json.html`, extraction may yield empty `preheader`.
  - Selector fragility: if the template changes (no `div#emailPreHeader`), preheader becomes empty and comparisons may mismatch.
  - If multiple messages are returned, this runs per message item; later merge/code logic must be consistent about which message is being validated.

---

### Block 4 — Merge Inputs & Compute QA Results
**Overview:** Combines expected rows, email content, and preheader extraction; then computes `Actual` and `Result` for each expected check.  
**Nodes involved:**  
- Combine HTML, Expected Content and Preheader
- Extract Actual Content and Results

#### Node: Combine HTML, Expected Content and Preheader
- **Type / role:** Merge (`n8n-nodes-base.merge`) — merges multiple input streams into one item list.
- **Mode:** 3 inputs (`numberInputs: 3`)
- **Connections:**
  - **Input 0 ←** Load Expected Content
  - **Input 1 ←** Get many messages
  - **Input 2 ←** Extracting Preheader From HTML
  - **Output →** Extract Actual Content and Results
- **Edge cases / failures:**
  - Merge semantics with 3 inputs can be non-intuitive in n8n: depending on merge mode defaults, item pairing may not align as you expect (especially when one input has many rows and another has 1 email).
  - If Gmail returns multiple messages, you may merge expected rows with multiple emails unintentionally.

#### Node: Extract Actual Content and Results
- **Type / role:** Code (`n8n-nodes-base.code`) — performs content cleaning, field extraction, and comparisons.
- **Key logic (interpreted):**
  - Defines helpers:
    - `cleanHtml(str)`: removes `<style>`, `<head>`, `<footer>`, strips tags, normalizes whitespace.
    - `decodeEntities(str)`: replaces `&#x9;`, `&amp;`, `&nbsp;`.
  - Builds `excelRows` as `items.filter(i => i.json["Expected"])`.
  - Chooses a single Gmail object:
    - `const gmail = items.find(i => i.json.subject || i.json.from || i.json.textAsHtml || i.json.text)?.json || {};`
    - Extracts:
      - `fromAddress = gmail.from?.value?.[0]?.address || ""`
      - `replyToAddress = gmail.replyTo?.value?.[0]?.address || ""`
      - `subjectLine = (gmail.subject || "").replace(/^TEST\s*\|\s*/i, "").trim()`
  - Chooses preheader:
    - `const preheaderItem = items.find(i => i.json.preheader)?.json || {}`
    - `preheader = preheaderItem.preheader || ""`
  - Computes `bodyCopy`:
    - from `gmail.textAsHtml` cleaned & decoded, else `gmail.text`.
  - For each expected row:
    - If `expected.includes("@")` → compare against `fromAddress || replyToAddress`
    - Else if contains “subject” → compare to `subjectLine`
    - Else if contains “preheader” → compare to `preheader`
    - Else → compare to `bodyCopy`
    - Output: `{ Expected, Actual: actualValue, Result: "MATCH"|"MISMATCH" }`
- **Connections:**
  - **Input ←** Combine HTML, Expected Content and Preheader
  - **Output →** Log Content Checks to Excel
- **Edge cases / failures:**
  - **Multiple emails:** the code uses `items.find(...)` to pick the *first* Gmail-like item; if multiple emails are in play, comparisons may use the wrong email.
  - **Expected-row structure:** it assumes each expected row has a top-level `Expected` field. If Google Sheets returns rows as arrays or uses different column names, `excelRows` will be empty → outputs nothing.
  - **Comparison heuristic:** `expected.includes("@")` is a weak detector for “From address” checks; any expected value containing `@` (including body content) will route into address comparison.
  - **Subject detection:** looks for substring “subject” in the expected value, not in a label column. This can misclassify checks.
  - **No Label output:** downstream Sheets mapping expects `$json.Label`, but this code never sets `Label`, so it will be blank unless it already existed in the incoming items (it won’t, given `return results.map(r => ({ json: r }))`).
  - HTML cleaning removes `<footer>...</footer>` entirely; if footer validation is expected, it will likely never match.

---

### Block 5 — Write QA Results to Google Sheets
**Overview:** Persists validation output back into Google Sheets using an upsert keyed on the `Expected` column.  
**Nodes involved:**  
- Log Content Checks to Excel

#### Node: Log Content Checks to Excel
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — append or update QA results.
- **Operation:** `appendOrUpdate`
- **Key configuration choices:**
  - Mapping mode: “defineBelow”
  - **Matching column(s):** `Expected` (used as the upsert key)
  - Columns mapped:
    - `Label` = `{{$json.Label}}`
    - `Expected` = `{{$json.Expected}}`
    - `Actual` = `{{$json.Actual}}`
    - `Result` = `{{$json.Result}}`
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account”)
- **Connections:**
  - **Input ←** Extract Actual Content and Results
- **Edge cases / failures:**
  - Same misconfiguration pattern as “Load Expected Content”: `sheetName` contains code-like content and is unlikely to work as intended.
  - Upsert by `Expected` is risky: many checks could share identical expected strings (e.g., repeated phrases), causing overwrites.
  - If `Label` is blank (likely), it will write empty labels unless the sheet has defaults.
  - Permissions/sharing: the OAuth user must have edit access.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual start / entry point | — | Get many messages | ## Main Overview – Email Marketing Automation QA Workflow<br><br>This workflow automates the end‑to‑end content verification process for email QA by comparing expected content values stored in Google Sheets with the actual values extracted from the email HTML. It eliminates manual checks for key email elements—including From Address, Subject Line, Preheader, Body Copy, and Footer Content—and generates a clear Result (Pass/Fail) for each item. This ensures consistency, accuracy, and significantly reduces QA turnaround time.<br><br>**How it works**<br>1. Load all expected content values (From, Subject, Preheader, Body, Footer) from Google Sheets.<br>2. Extract actual content from the email HTML using HTML parsing + JavaScript logic.<br>3. Merge expected and actual entries using a unique key (such as Section ID).<br>4. Compare Expected vs Actual values to determine the Result (Pass/Fail).<br>5. Write the Actual Content + Result back into Google Sheets, forming a structured QA output.<br><br>**Setup steps**<br>* Connect Google Sheets (Expected Content Source).<br>* Configure Gmail / HTML Source to pull the email HTML.<br>* Update JS extraction logic to match your email template structure.<br>* Ensure proper row matching via Section ID / Labels. |
| Get many messages | Gmail | Fetch unread emails from sender | When clicking ‘Execute workflow’ | Extracting Preheader From HTML; Combine HTML, Expected Content and Preheader; Load Expected Content | ## Main Overview – Email Marketing Automation QA Workflow<br><br>This workflow automates the end‑to‑end content verification process for email QA by comparing expected content values stored in Google Sheets with the actual values extracted from the email HTML. It eliminates manual checks for key email elements—including From Address, Subject Line, Preheader, Body Copy, and Footer Content—and generates a clear Result (Pass/Fail) for each item. This ensures consistency, accuracy, and significantly reduces QA turnaround time.<br><br>**How it works**<br>1. Load all expected content values (From, Subject, Preheader, Body, Footer) from Google Sheets.<br>2. Extract actual content from the email HTML using HTML parsing + JavaScript logic.<br>3. Merge expected and actual entries using a unique key (such as Section ID).<br>4. Compare Expected vs Actual values to determine the Result (Pass/Fail).<br>5. Write the Actual Content + Result back into Google Sheets, forming a structured QA output.<br><br>**Setup steps**<br>* Connect Google Sheets (Expected Content Source).<br>* Configure Gmail / HTML Source to pull the email HTML.<br>* Update JS extraction logic to match your email template structure.<br>* Ensure proper row matching via Section ID / Labels. |
| Load Expected Content | Google Sheets | Load expected QA checks | Get many messages | Combine HTML, Expected Content and Preheader | ## Content Check <br>Reads expected email elements from Google Sheets—From address, Subject line, Preheader, Body copy, and Footer content. Extracts actual values from HTML, merges inputs, compares expected vs actual to generate the Result, and logs Actual Content + Result back to Google Sheets. |
| Extracting Preheader From HTML | HTML | Extract preheader via CSS selector | Get many messages | Combine HTML, Expected Content and Preheader | ## Content Check <br>Reads expected email elements from Google Sheets—From address, Subject line, Preheader, Body copy, and Footer content. Extracts actual values from HTML, merges inputs, compares expected vs actual to generate the Result, and logs Actual Content + Result back to Google Sheets. |
| Combine HTML, Expected Content and Preheader | Merge | Consolidate Gmail + expected + preheader streams | Load Expected Content; Get many messages; Extracting Preheader From HTML | Extract Actual Content and Results | ## Content Check <br>Reads expected email elements from Google Sheets—From address, Subject line, Preheader, Body copy, and Footer content. Extracts actual values from HTML, merges inputs, compares expected vs actual to generate the Result, and logs Actual Content + Result back to Google Sheets. |
| Extract Actual Content and Results | Code | Clean/decode HTML, compute Actual + Result per expected row | Combine HTML, Expected Content and Preheader | Log Content Checks to Excel | ## Content Check <br>Reads expected email elements from Google Sheets—From address, Subject line, Preheader, Body copy, and Footer content. Extracts actual values from HTML, merges inputs, compares expected vs actual to generate the Result, and logs Actual Content + Result back to Google Sheets. |
| Log Content Checks to Excel | Google Sheets | Persist QA results (upsert by Expected) | Extract Actual Content and Results | — | ## Content Check <br>Reads expected email elements from Google Sheets—From address, Subject line, Preheader, Body copy, and Footer content. Extracts actual values from HTML, merges inputs, compares expected vs actual to generate the Result, and logs Actual Content + Result back to Google Sheets. |
| Sticky Note | Sticky Note | Documentation / overview | — | — | ## Main Overview – Email Marketing Automation QA Workflow<br><br>This workflow automates the end‑to‑end content verification process for email QA by comparing expected content values stored in Google Sheets with the actual values extracted from the email HTML. It eliminates manual checks for key email elements—including From Address, Subject Line, Preheader, Body Copy, and Footer Content—and generates a clear Result (Pass/Fail) for each item. This ensures consistency, accuracy, and significantly reduces QA turnaround time.<br><br>**How it works**<br>1. Load all expected content values (From, Subject, Preheader, Body, Footer) from Google Sheets.<br>2. Extract actual content from the email HTML using HTML parsing + JavaScript logic.<br>3. Merge expected and actual entries using a unique key (such as Section ID).<br>4. Compare Expected vs Actual values to determine the Result (Pass/Fail).<br>5. Write the Actual Content + Result back into Google Sheets, forming a structured QA output.<br><br>**Setup steps**<br>* Connect Google Sheets (Expected Content Source).<br>* Configure Gmail / HTML Source to pull the email HTML.<br>* Update JS extraction logic to match your email template structure.<br>* Ensure proper row matching via Section ID / Labels. |
| Sticky Note3 | Sticky Note | Documentation / block description | — | — | ## Content Check <br>Reads expected email elements from Google Sheets—From address, Subject line, Preheader, Body copy, and Footer content. Extracts actual values from HTML, merges inputs, compares expected vs actual to generate the Result, and logs Actual Content + Result back to Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it e.g. “Verify email content against Google Sheets and Gmail logs”.

2) **Add Manual Trigger**
- Node: **Manual Trigger**
- No configuration.

3) **Add Gmail node to fetch messages**
- Node: **Gmail**
- Credentials: configure **Gmail OAuth2** (Google Cloud project / scopes as required by n8n Gmail node).
- Operation: **Get All**
- Set:
  - **Sender**: `user@example.com` (adjust)
  - **Read status**: `unread`
  - **Include spam/trash**: enabled
  - **Simple**: disabled (so you receive rich message fields including `from`, `replyTo`, `textAsHtml`, etc.)
- Connect: **Manual Trigger → Gmail**

4) **Add Google Sheets node to load expected content**
- Node: **Google Sheets**
- Credentials: configure **Google Sheets OAuth2** with access to the spreadsheet.
- Operation: use a proper “read” operation (commonly **Read** / **Get Many** depending on node UI version).
- Set:
  - **Document**: select spreadsheet (by ID or paste URL)
  - **Sheet name**: e.g. `Sheet1`
  - Ensure the sheet has headers including at least: `Expected` (optionally `Label`)
- **Important:** Do **not** paste Node.js/googleapis code into `sheetName`. Keep it a sheet tab name.
- Connect: ideally **Gmail → (optional)** but recommended: **Manual Trigger → Load Expected Content** (so it loads once), not per email.

5) **Add HTML node to extract preheader**
- Node: **HTML**
- Operation: **Extract HTML Content**
- Set:
  - **HTML property**: adjust to where Gmail HTML is stored. In many Gmail outputs it is `textAsHtml`, so you may need:
    - Either map/copy `textAsHtml` into a field named `html` before this node (via Set node),
    - Or configure the HTML node (if supported) to read from `textAsHtml` directly.
  - **CSS selector**: `div#emailPreHeader`
  - Output key: `preheader`
- Connect: **Gmail → HTML**

6) **Add Merge node (3 inputs)**
- Node: **Merge**
- Set: **Number of Inputs = 3**
- Connect:
  - **Input 0:** Load Expected Content → Merge
  - **Input 1:** Gmail → Merge
  - **Input 2:** HTML Preheader → Merge

7) **Add Code node to compute comparisons**
- Node: **Code**
- Paste/implement logic equivalent to:
  - Read expected rows (`Expected`)
  - Pick the target email fields (`from`, `replyTo`, `subject`, `textAsHtml/text`)
  - Use the extracted `preheader`
  - Create output items: `{ Label?, Expected, Actual, Result }`
- Connect: **Merge → Code**
- Recommendation to make logging reliable:
  - Include `Label` from the sheet row if present, e.g. `const label = row.json.Label;` and return it.

8) **Add Google Sheets node to write results**
- Node: **Google Sheets**
- Operation: **Append or Update**
- Set:
  - **Document**: same spreadsheet
  - **Sheet name**: results sheet/tab (e.g. `QA Results`) or same sheet if you want in-place updates
  - **Mapping:** map `Expected`, `Actual`, `Result` (and `Label` if used)
  - **Matching columns:** prefer a unique key (e.g. `Label` or `Section ID`) rather than `Expected`
- Connect: **Code → Log Results (Google Sheets)**

9) **Run & validate**
- Execute workflow manually.
- Ensure:
  - Gmail returns exactly the intended email(s).
  - Preheader extraction is non-empty (selector matches your template).
  - Sheets read returns structured objects with an `Expected` property.
  - Sheets write doesn’t overwrite unintended rows.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Main Overview – Email Marketing Automation QA Workflow…” (sticky note content) | Workflow-level intent and setup guidance (included in Sticky Note section above). |
| “Content Check … Reads expected email elements…” (sticky note content) | Describes the compare-and-log block purpose (included in Sticky Note3 section above). |