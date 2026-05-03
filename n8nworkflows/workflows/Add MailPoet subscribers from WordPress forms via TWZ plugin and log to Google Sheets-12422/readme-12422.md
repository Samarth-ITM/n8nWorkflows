Add MailPoet subscribers from WordPress forms via TWZ plugin and log to Google Sheets

https://n8nworkflows.xyz/workflows/add-mailpoet-subscribers-from-wordpress-forms-via-twz-plugin-and-log-to-google-sheets-12422


# Add MailPoet subscribers from WordPress forms via TWZ plugin and log to Google Sheets

## 1. Workflow Overview

**Purpose:**  
Automate subscriber creation in **MailPoet (via TWZ WordPress REST API)** from an **n8n Form Trigger** submission, avoid duplicates (Google Sheets + MailPoet check), **log the subscriber** to **Google Sheets**, then show a **completion message** to the user.

**Primary use cases:**
- Capture leads/review requests from a form and push them into MailPoet.
- Prevent duplicate subscriber creation.
- Maintain an external log/audit trail in Google Sheets.

### 1.1 Input Reception (Form)
Receives first name, last name, and email from an n8n-hosted form and redirects the user after submit.

### 1.2 Duplicate Prevention – Google Sheets Pre-check
Looks in Google Sheets first (intended to detect existing entries) before calling MailPoet.

### 1.3 Duplicate Prevention – MailPoet API Check (TWZ plugin)
Queries WordPress/TWZ endpoint to see whether the email is already a MailPoet subscriber.

### 1.4 Conditional Add + Logging + Completion
If not found in MailPoet, adds subscriber via TWZ endpoint, appends a row to Google Sheets, then returns a completion message. If found, skips creation and goes directly to completion.

---

## 2. Block-by-Block Analysis

### Block 1 — Form Submission Intake
**Overview:** Collects the user’s name and email from an n8n Form Trigger and initiates the workflow.  
**Nodes Involved:** `On form submission`

#### Node: On form submission
- **Type / Role:** `Form Trigger` — entry point, collects form fields, starts execution on submit.
- **Key configuration:**
  - **Form title:** “Let's Start Your Project”
  - **Fields (required):**
    - `FIRST NAME` (text)
    - `LAST NAME` (text)
    - `YOUR EMAIL` (email)
  - **Response behavior:** redirect to `https://example.com`
  - **Attribution:** disabled (`appendAttribution: false`)
- **Key data produced:** The submission values are exposed in `$json`, accessed later as:
  - `$json["FIRST NAME"]`, `$json["LAST NAME"]`, `$json["YOUR EMAIL"]`
- **Connections:**
  - **Output →** `Check Google Sheet for Existing Subscriber`
- **Version notes:** Form Trigger `typeVersion 2.3` (behavior depends on n8n Forms capability in your instance).
- **Potential failures / edge cases:**
  - Field naming is case/space-sensitive; downstream expressions will break if labels change.
  - Email validation occurs at the form field level, but still consider normalization (case/trim) downstream if needed.

**Sticky note coverage:**  
“Trigger: Form Submission … listens to your WordPress form submission … collects First Name, Last Name, Email.”  
*(Note: This workflow uses n8n Form Trigger, not a WordPress-native form webhook.)*

---

### Block 2 — Google Sheets Pre-check (Existing Subscriber)
**Overview:** Intended to check whether the subscriber already exists in Google Sheets to prevent duplicates before calling MailPoet.  
**Nodes Involved:** `Check Google Sheet for Existing Subscriber`, `Check If Subscriber Already Exists`

#### Node: Check Google Sheet for Existing Subscriber
- **Type / Role:** `Google Sheets` — reads data from a spreadsheet (configured, but filtering is effectively empty).
- **Key configuration:**
  - **Document:** Google Sheet with ID `1FvjhY3IsLOaB-chqUp3v8EhFdRCtND3-h5km27b4JK8` (shown cached name: “titles”)
  - **Sheet tab:** `Sheet1` (`gid=0`)
  - **Filters UI:** present but contains an empty object (`values: [ {} ]`), so **no actual filter condition is defined**.
- **Credentials:** `TWZ Google Sheets account` (OAuth2).
- **Connections:**
  - **Input ←** `On form submission`
  - **Output →** `Check If Subscriber Already Exists`
- **Potential failures / edge cases:**
  - OAuth scope/permission issues (403), expired token refresh issues.
  - If the intent is “lookup by email”, the node currently does not do it; it may return many rows or an unexpected structure, making the subsequent IF logic unreliable.
  - Spreadsheet structure does not appear aligned with subscriber fields (see logging block—sheet columns look unrelated to subscribers).

#### Node: Check If Subscriber Already Exists
- **Type / Role:** `IF` — conditional branching (intended to decide whether to proceed with MailPoet check/add).
- **Key configuration (interpreted):**
  - Condition uses **Object → notExists** with **empty leftValue** and empty rightValue.
  - As configured, the condition does not reference any field from the Google Sheets output.
- **Connections:**
  - **Input ←** `Check Google Sheet for Existing Subscriber`
  - **True/False outputs:**
    - One branch → `Get Subscriber From  MailPoet`
    - Other branch → `Completion Message to User`
- **Potential failures / edge cases:**
  - Because `leftValue` is empty, the IF result may be deterministic/unintended (depending on n8n evaluation), causing the workflow to always go one way.
  - If Google Sheets returns multiple items, an IF node evaluates per item; you can get mixed branching and multiple downstream calls unless you consolidate/limit results.

**Sticky note coverage:**  
“Check Existing Subscribers … checks your Google Sheet or MailPoet API … avoid duplicate entries.”

---

### Block 3 — MailPoet Lookup via TWZ WordPress REST API
**Overview:** Queries the TWZ MailPoet endpoint to check whether the email already exists as a MailPoet subscriber.  
**Nodes Involved:** `Get Subscriber From  MailPoet`, `Check If Subscriber Exists In MailPoet`

#### Node: Get Subscriber From  MailPoet
- **Type / Role:** `HTTP Request` — GET subscriber info from WordPress/TWZ REST endpoint.
- **Key configuration:**
  - **URL (expression):**  
    `http://example.com/wp-json/twz-mailpoet/v1/subscriber/{{ $json["YOUR EMAIL"] }}`
  - **Authentication:** `genericCredentialType` → `httpHeaderAuth`
- **Credentials:**
  - Uses `httpHeaderAuth` credential named **Local Test Wordpress**
  - Also lists `httpBasicAuth` credential **TWZ Wordpress Access** (present in node credentials but not selected as auth method; likely unused).
- **Connections:**
  - **Input ←** `Check If Subscriber Already Exists`
  - **Output →** `Check If Subscriber Exists In MailPoet`
- **Potential failures / edge cases:**
  - URL should ideally be HTTPS; HTTP may be blocked or insecure.
  - If email contains special characters, consider URL encoding (the node as written does not explicitly encode).
  - 401/403 if WP auth headers are wrong or TWZ endpoint permissions are restricted.
  - 404/500 if the endpoint is missing/misconfigured.
  - Response shape must include a field named `subscriber` for the next IF to work.

#### Node: Check If Subscriber Exists In MailPoet
- **Type / Role:** `IF` — branches based on whether the MailPoet response indicates a subscriber exists.
- **Key configuration (interpreted):**
  - Condition: **String → notExists** on `leftValue: "subscriber"`
  - This appears intended to check whether `$json.subscriber` exists, but it currently references the literal string `"subscriber"` (not an expression like `{{$json.subscriber}}`).
- **Connections:**
  - **Input ←** `Get Subscriber From  MailPoet`
  - **One branch →** `Add Subscriber to MailPoet`
  - **Other branch →** `Completion Message to User`
- **Potential failures / edge cases:**
  - If the condition is not correctly referencing the response property, the workflow may always add or always skip.
  - If the API returns errors in a different JSON shape, the IF condition won’t behave as expected.

**Sticky note coverage:**  
Included in “Conditional Branch: IF Node … If subscriber does not exist → Add subscriber … If exists → Skip…”

---

### Block 4 — Add Subscriber + Log to Google Sheets + Completion
**Overview:** Creates a new MailPoet subscriber through TWZ endpoint, logs details to Google Sheets, and returns a completion page/message.  
**Nodes Involved:** `Add Subscriber to MailPoet`, `Log Subscriber to Google Sheet`, `Completion Message to User`

#### Node: Add Subscriber to MailPoet
- **Type / Role:** `HTTP Request` — POST to create subscriber in MailPoet via WordPress/TWZ endpoint.
- **Key configuration:**
  - **URL:** `http://example.com/wp-json/twz-mailpoet/v1/subscribers`
  - **Body:** JSON (sent as JSON)
    - `email`: `{{$json["YOUR EMAIL"]}}`
    - `first_name`: `{{$json["FIRST NAME"]}}`
    - `last_name`: `{{$json["LAST NAME"]}}`
  - **Authentication:** `httpHeaderAuth` (generic credential type)
- **Credentials:** `Local Test Wordpress` (HTTP Header Auth).
- **Connections:**
  - **Input ←** `Check If Subscriber Exists In MailPoet`
  - **Output →** `Log Subscriber to Google Sheet`
- **Potential failures / edge cases:**
  - 409/duplicate error if endpoint rejects existing emails (especially if earlier checks fail).
  - 400 if required fields missing or if the endpoint expects a different schema.
  - 401/403 auth issues.
  - If the endpoint is rate-limited, add retry/backoff options in HTTP node.

#### Node: Log Subscriber to Google Sheet
- **Type / Role:** `Google Sheets` — append a new row.
- **Key configuration:**
  - **Operation:** Append
  - **Document:** same ID as above
  - **Sheet tab:** `Sheet1` (`gid=0`)
  - **Columns schema shown:** `title`, `used`, `url`, `METATITLE`, `METADESCRIPTION`, `audio`
  - **Mapping:** “defineBelow” but **no actual mapping values are provided** in the `columns.value` object.
- **Credentials:** `TWZ Google Sheets account` (OAuth2).
- **Connections:**
  - **Input ←** `Add Subscriber to MailPoet`
  - **Output →** `Completion Message to User`
- **Potential failures / edge cases:**
  - Will append blank/incorrect rows unless mapping is configured to include email/name fields.
  - Spreadsheet column mismatch can cause runtime errors (depending on node configuration and sheet headers).
  - Multiple items could append multiple rows unexpectedly.

#### Node: Completion Message to User
- **Type / Role:** `Form` — completion step shown to the user.
- **Key configuration:**
  - **Operation:** Completion
  - **Title:** “Thank you for your review!”
- **Connections:**
  - **Inputs ←** from:
    - `Log Subscriber to Google Sheet`
    - `Check If Subscriber Already Exists` (skip branch)
    - `Check If Subscriber Exists In MailPoet` (skip branch)
  - **Output:** none
- **Potential failures / edge cases:**
  - If the workflow branches incorrectly, user may see completion without any action taken (or after a failed action unless errors are handled).

**Sticky note coverage:**  
“Logging & Completion … completion message … optional Slack notification/email alert.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Comment / documentation | — | — | ## Trigger: Form Submission; This node listens to your WordPress form submission.; It collects the following fields:; - First Name; - Last Name; - Email |
| Sticky Note1 | Sticky Note | Comment / documentation | — | — | ## Check Existing Subscribers; This node checks your Google Sheet or MailPoet API to see if the subscriber already exists.; This helps avoid duplicate entries in MailPoet. |
| Sticky Note2 | Sticky Note | Comment / documentation | — | — | ## Conditional Branch: IF Node; - If subscriber does not exist → Add subscriber via HTTP Request node; - If subscriber exists → Skip to completion message; This ensures that new users are only added once.; - Add users to subscribers ; - Logs the subscriber data into Google Sheet |
| Sticky Note3 | Sticky Note | Comment / documentation | — | — | ## Logging & Completion; - Sends the completion message to the user (“Thank you for your review!”); Optional: Add Slack notification or email alert to the admin. |
| Sticky Note4 | Sticky Note | Comment / documentation | — | — | ## Workflow Overview; This workflow automates the full subscriber management process:; 1. Trigger: WordPress form submission; 2. Check if subscriber exists; 3. Add new subscriber via TWZ MailPoet API; 4. Log subscriber data to Google Sheet; 5. Optional: Notify admin via Slack; 6. Show completion message to user; Use this workflow to streamline MailPoet subscriber automation. |
| On form submission | Form Trigger | Collect user input and start workflow | — | Check Google Sheet for Existing Subscriber | ## Trigger: Form Submission; This node listens to your WordPress form submission.; It collects the following fields:; - First Name; - Last Name; - Email |
| Check Google Sheet for Existing Subscriber | Google Sheets | Pre-check/lookup in sheet (currently unfiltered) | On form submission | Check If Subscriber Already Exists | ## Check Existing Subscribers; This node checks your Google Sheet or MailPoet API to see if the subscriber already exists.; This helps avoid duplicate entries in MailPoet. |
| Check If Subscriber Already Exists | IF | Branch based on existence result (currently misconfigured) | Check Google Sheet for Existing Subscriber | Get Subscriber From  MailPoet; Completion Message to User | ## Check Existing Subscribers; This node checks your Google Sheet or MailPoet API to see if the subscriber already exists.; This helps avoid duplicate entries in MailPoet. |
| Get Subscriber From  MailPoet | HTTP Request | Query TWZ MailPoet endpoint for subscriber | Check If Subscriber Already Exists | Check If Subscriber Exists In MailPoet | ## Conditional Branch: IF Node; - If subscriber does not exist → Add subscriber via HTTP Request node; - If subscriber exists → Skip to completion message; This ensures that new users are only added once.; - Add users to subscribers ; - Logs the subscriber data into Google Sheet |
| Check If Subscriber Exists In MailPoet | IF | Decide add vs skip (currently likely misconfigured) | Get Subscriber From  MailPoet | Add Subscriber to MailPoet; Completion Message to User | ## Conditional Branch: IF Node; - If subscriber does not exist → Add subscriber via HTTP Request node; - If subscriber exists → Skip to completion message; This ensures that new users are only added once.; - Add users to subscribers ; - Logs the subscriber data into Google Sheet |
| Add Subscriber to MailPoet | HTTP Request | Create subscriber via TWZ MailPoet endpoint | Check If Subscriber Exists In MailPoet | Log Subscriber to Google Sheet | ## Conditional Branch: IF Node; - If subscriber does not exist → Add subscriber via HTTP Request node; - If subscriber exists → Skip to completion message; This ensures that new users are only added once.; - Add users to subscribers ; - Logs the subscriber data into Google Sheet |
| Log Subscriber to Google Sheet | Google Sheets | Append log row (mapping currently empty / mismatched columns) | Add Subscriber to MailPoet | Completion Message to User | ## Conditional Branch: IF Node; - If subscriber does not exist → Add subscriber via HTTP Request node; - If subscriber exists → Skip to completion message; This ensures that new users are only added once.; - Add users to subscribers ; - Logs the subscriber data into Google Sheet |
| Completion Message to User | Form | Show completion page/message | Log Subscriber to Google Sheet; Check If Subscriber Already Exists; Check If Subscriber Exists In MailPoet | — | ## Logging & Completion; - Sends the completion message to the user (“Thank you for your review!”); Optional: Add Slack notification or email alert to the admin. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“N8N MailPoet Test”** (or your preferred name).
   - Keep workflow **inactive** until credentials and endpoints are verified.

2. **Add the trigger node**
   - Add node: **Form Trigger**
   - Configure:
     - **Form title:** `Let's Start Your Project`
     - **Fields:**
       - Text, label `FIRST NAME`, required
       - Text, label `LAST NAME`, required
       - Email, label `YOUR EMAIL`, required
     - **Response:** Redirect
       - Redirect URL: `https://example.com`
     - Disable attribution if desired.
   - This node becomes the workflow entry point.

3. **Add Google Sheets “pre-check” node**
   - Add node: **Google Sheets**
   - Credentials: connect **Google Sheets OAuth2** (select/create credential like `TWZ Google Sheets account`)
   - Configure:
     - Choose **Document** by ID: `1FvjhY3IsLOaB-chqUp3v8EhFdRCtND3-h5km27b4JK8`
     - Choose sheet/tab: `Sheet1 (gid=0)`
     - Configure an actual lookup/filter (recommended):
       - Filter where `email` equals `{{$json["YOUR EMAIL"]}}` (requires a column named `email`)
   - Connect: **Form Trigger → Google Sheets**

4. **Add IF node to branch on Google Sheets result**
   - Add node: **IF**
   - Configure to detect “no rows returned”.
     - Common approach: check if a field from the first returned row exists.
     - Example pattern (implementation depends on your Sheets node output):  
       - Condition: `{{$json.email}}` **does not exist** (or check `{{$items().length}}` via a Code node, if needed).
   - Connect: **Google Sheets → IF**
   - Plan branches:
     - **If NOT found in sheet** → proceed to MailPoet lookup
     - **If found** → go to completion

5. **Add HTTP Request to fetch subscriber from MailPoet (via TWZ WP endpoint)**
   - Add node: **HTTP Request**
   - Method: GET (default)
   - URL (expression):  
     `http://example.com/wp-json/twz-mailpoet/v1/subscriber/{{$json["YOUR EMAIL"]}}`
   - Authentication:
     - Choose **Header Auth** (or whatever TWZ requires)
     - Create/select credential (e.g., `Local Test Wordpress`) with required header name/value (often `Authorization: Bearer ...` or an API key header).
   - Connect: **IF (not found) → HTTP Request**

6. **Add IF node to decide whether to create subscriber**
   - Add node: **IF**
   - Configure based on the real API response.
     - If TWZ returns `{ subscriber: ... }` when found, then check:
       - Condition: `{{$json.subscriber}}` **exists** → skip create
       - Condition: `{{$json.subscriber}}` **does not exist** → create
   - Connect: **MailPoet GET → IF**

7. **Add HTTP Request to create subscriber**
   - Add node: **HTTP Request**
   - Method: POST
   - URL: `http://example.com/wp-json/twz-mailpoet/v1/subscribers`
   - Send body: **JSON**
   - Body fields:
     - `email`: `{{$json["YOUR EMAIL"]}}`
     - `first_name`: `{{$json["FIRST NAME"]}}`
     - `last_name`: `{{$json["LAST NAME"]}}`
   - Authentication: same Header Auth credential as above.
   - Connect: **IF (subscriber not found) → POST create**

8. **Add Google Sheets append logging**
   - Add node: **Google Sheets**
   - Operation: **Append**
   - Select same document and sheet.
   - Ensure the sheet has appropriate headers, e.g.: `email`, `first_name`, `last_name`, `timestamp`, `source`
   - Map columns, for example:
     - `email` → `{{$json["YOUR EMAIL"]}}`
     - `first_name` → `{{$json["FIRST NAME"]}}`
     - `last_name` → `{{$json["LAST NAME"]}}`
     - `timestamp` → `{{$now}}`
   - Connect: **POST create → Sheets append**

9. **Add Completion page node**
   - Add node: **Form**
   - Operation: **Completion**
   - Completion title: `Thank you for your review!`
   - Connect completion from all end states:
     - **Sheets append → Completion**
     - **IF (found in sheet) → Completion**
     - **IF (found in MailPoet) → Completion**

10. **(Optional) Add admin notifications**
   - Insert a Slack/Email node between “append” and “completion”, or on error branches, depending on your ops needs.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Trigger: Form Submission… listens to your WordPress form submission…” | Sticky note describes WP form, but the actual trigger is an **n8n Form Trigger**. If you truly want WP form submissions, replace with **Webhook Trigger** and post from WordPress/TWZ. |
| TWZ MailPoet endpoints used: `/wp-json/twz-mailpoet/v1/subscriber/{email}` and `/wp-json/twz-mailpoet/v1/subscribers` | Configure/verify the WordPress plugin routes, permissions, and expected request/response schema. |
| Google Sheets logging schema mismatch | Current sheet columns shown (`title`, `used`, `url`, etc.) don’t match subscriber logging; update the sheet headers and mapping to avoid blank or invalid rows. |
| Optional Slack notification mentioned | Add a Slack node after successful append (or on errors) if required by ops. |