Receive Meta Ads ad account webhooks, log to Sheets, and alert in Slack

https://n8nworkflows.xyz/workflows/receive-meta-ads-ad-account-webhooks--log-to-sheets--and-alert-in-slack-12223


# Receive Meta Ads ad account webhooks, log to Sheets, and alert in Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow receives **Meta Ads Webhooks** for the **Ad Account** object (`ad_account`), handles Meta’s **verification handshake**, acknowledges every webhook delivery immediately, **logs events into Google Sheets** (one tab per event type), and sends a **compact Slack summary** (count of events by type).

**Target use cases:**
- Monitor operational/quality signals from Meta Ads ad accounts (e.g., creative fatigue, recommendations, issues).
- Build an auditable event log (Sheets) + lightweight alerting (Slack).
- Handle webhook payloads containing arrays (multiple `entry` and `changes` items).

### Logical blocks
1.1 **Webhook reception & routing (verification vs event)**  
1.2 **Verification handshake (token + challenge response)**  
1.3 **Event-type routing and immediate acknowledgements**  
1.4 **Payload flattening (Split Out) for array-safe processing**  
1.5 **Google Sheets logging per event type**  
1.6 **Cross-event merge + summarization + Slack notification**

---

## 2. Block-by-Block Analysis

### 2.1 Webhook reception & routing (verification vs event)

**Overview:**  
Receives HTTP requests from Meta, then separates verification calls from actual webhook deliveries. This prevents Meta subscription setup from being treated as an event.

**Nodes involved:**
- **Webhook: Meta (ad_account)**
- **Route: verification vs webhook**

#### Node: Webhook: Meta (ad_account)
- **Type / role:** `Webhook` trigger node (entry point).
- **Key configuration (interpreted):**
  - **Path:** `meta-ads-ad-account-webhook` (forms the callback URL).
  - **Methods:** multiple methods enabled (Meta verification is typically GET; deliveries are POST).
  - **Response mode:** **“responseNode”** → the workflow must explicitly respond using Respond to Webhook nodes.
- **Key data used later:** `$json.query` (verification params), `$json.body` (webhook payload).
- **Connections:**
  - Output → **Route: verification vs webhook** (note: JSON shows two identical connections; functionally it’s still a single path).
- **Failure/edge cases:**
  - If no Respond node is reached, Meta may see timeouts and retry.
  - If Meta signs requests (optional), this workflow does not validate signatures.
- **Version notes:** Type version **2.1**; responseMode “responseNode” requires downstream Respond nodes.

#### Node: Route: verification vs webhook
- **Type / role:** `Switch` router.
- **Configuration choices:**
  - Rule 1 (**Verification**): `$json.query?.['hub.mode'] == 'subscribe'`
  - Rule 2 (**AdAccountData**): `$json.body?.object == 'ad_account'`
  - Outputs are renamed to “Verification” and “AdAccountData”.
- **Connections:**
  - Verification output → **Verify: token**
  - AdAccountData output → **Route: field**
- **Failure/edge cases:**
  - If Meta sends payloads without `body.object` or with unexpected object type, no route matches → no acknowledgement sent.
  - If `hub.mode` is absent on verification, handshake fails.

---

### 2.2 Verification handshake (token + challenge response)

**Overview:**  
Handles Meta’s subscription verification by checking the verify token and returning `hub.challenge`.

**Nodes involved:**
- **Verify: token**
- **Respond: challenge**

#### Node: Verify: token
- **Type / role:** `If` condition gate.
- **Configuration choices:**
  - Condition: `$json.query?.['hub.verify_token'] == 'YOUR_VERIFY_TOKEN'`
- **Connections:**
  - True output → **Respond: challenge**
  - False output → not connected (no response is sent on failure).
- **Failure/edge cases:**
  - **Important:** If the token mismatches, Meta still needs an HTTP response (typically 403). Here, the false branch is not handled, which can cause a timeout or generic failure during setup.
  - Hard-coded token must match the one set in Facebook Developers.
- **Version notes:** Type version **2.2**.

#### Node: Respond: challenge
- **Type / role:** `Respond to Webhook` (ends the verification request properly).
- **Configuration choices:**
  - Respond with **text**
  - Body: `{{$json.query['hub.challenge']}}`
- **Connections:** none (terminal response).
- **Failure/edge cases:**
  - If query param missing, response will be empty and verification fails.
- **Version notes:** Type version **1.4**.

---

### 2.3 Event-type routing and immediate acknowledgements

**Overview:**  
Routes `ad_account` webhook deliveries by `field` and immediately acknowledges receipt to Meta (required to prevent retries), returning a small JSON payload describing the type.

**Nodes involved:**
- **Route: field**
- **Respond Creative Fatigue**
- **Respond Ad Recommendations**
- **Respond Async Creation**
- **Respond In Process**
- **Respond Product Set Issue**
- **Respond With Issues**

#### Node: Route: field
- **Type / role:** `Switch` router by webhook “field”.
- **Configuration choices:** checks:
  - `$json.body.entry[0].changes[0].field` equals one of:
    - `creative_fatigue`
    - `ad_recommendations`
    - `ads_async_creation_request`
    - `in_process_ad_objects`
    - `product_set_issue`
    - `with_issues_ad_objects`
- **Connections:** each output goes to its corresponding Respond node.
- **Failure/edge cases:**
  - If Meta sends multiple entries/changes, using `[0]` can misroute if the first item differs from subsequent ones. (Later Split Out handles arrays, but routing occurs *before* splitting.)
  - If payload structure differs (missing arrays), expressions can evaluate to `undefined` and no route matches.

#### Node group: Respond (per field)
Each is a `RespondToWebhook` node returning JSON:
- **Respond Creative Fatigue** → `{status:"received", field:<field>, webhook_type:"creative_fatigue"}`
- **Respond Ad Recommendations** → `webhook_type:"ad_recommendations"`
- **Respond Async Creation** → `webhook_type:"ads_async_creation_request"`
- **Respond In Process** → `webhook_type:"in_process_ad_objects"`
- **Respond Product Set Issue** → `webhook_type:"product_set_issue"`
- **Respond With Issues** → `webhook_type:"with_issues_ad_objects"`

Common characteristics:
- **Type / role:** `Respond to Webhook` acknowledgement.
- **Connections:** each continues to its split/log branch (see next blocks).
- **Failure/edge cases:**
  - If these nodes do not execute (routing miss), Meta delivery may be retried.
- **Version notes:** all are type version **1.4**.

---

### 2.4 Payload flattening (Split Out) for array-safe processing

**Overview:**  
Meta may send multiple `entry` items and multiple `changes` per entry. These nodes flatten arrays into individual items so each log row corresponds to one change.

**Nodes involved:**
- **Split Out Accounts CF** → **Split Out Changes CF**
- **Split Out Accounts AdRec** → **Split Out Changes AdRec**
- **Split Out Accounts AsCr** → **Split Out Changes AsCr**
- **Split Out Accounts Pr** → **Split Out Changes Pr**
- **Split Out Accounts SetIssue** → **Split Out Changes SetIssue**
- **Split Out Accounts Issues** → **Split Out Changes SetIssue1** (name mismatch, but functionally “changes for Issues”)

#### Split nodes (general behavior)
- **Type / role:** `Split Out` turns an array field into multiple items.
- **Key configuration:**
  - `fieldToSplitOut` is set to either `body.entry` or `changes` depending on stage.
  - Many nodes set **Include: allOtherFields** to keep context fields available after splitting.

#### Notable configuration details & edge cases
- **Split Out Accounts CF**
  - Splits `body.entry` and includes all other fields.
  - Output retains the split entry under a special key name (`body.entry`), which later appears in expressions as `['body.entry']` and sometimes escaped.
- **Split Out Changes CF**
  - Uses `fieldToSplitOut: "['body.entry'].changes"` (escaped in JSON).
  - **Risk:** the unusual path increases chance of expression/path mistakes if edited.
- **Split Out Changes AdRec/AsCr/Pr/SetIssue/SetIssue1**
  - All split `changes` after splitting entries.
  - **Risk:** If `changes` is missing or not an array, Split Out may error or output nothing depending on node behavior/version.

Connections (per branch):
- Each Respond node → Split Out Accounts (per type) → Split Out Changes (per type) → Google Sheets log node.

---

### 2.5 Google Sheets logging per event type

**Overview:**  
Appends a row to a dedicated sheet tab for each event type. Each row includes parsed fields and a stored “raw_body_json” snapshot for traceability.

**Nodes involved:**
- **Log Creative Fatigue**
- **Log Ad Recommendations**
- **Log Async Creation**
- **Log In-process ad objects**
- **Log product set issues**
- **Log with-issues ad objects**

All Google Sheets nodes:
- **Type / role:** `Google Sheets` node, **operation = Append**
- **Mapping:** “Define below” with explicit column mapping, matching on `id` column (though Append doesn’t dedupe by itself).
- **Credential:** `KH | Google Sheets` (OAuth2)
- **Edge cases:**
  - Auth expiration / missing permissions.
  - Sheet/tab name missing → append fails.
  - Column names in sheet must match configured schema.
  - Data type conversion is disabled; everything is sent largely as-is.

#### Node: Log Creative Fatigue
- **Sheet/tab:** `creative_fatigue`
- **Document:** Spreadsheet “Meta Ads Dispatcher Logs” (ID `1VGlkRBHo...`)
- **Key expressions:**
  - `date`: `{{$now.toFormat('dd/MM/yyyy')}}`
  - `id`: `{{$json['body.entry'].id}}`
  - Values pulled from `['body.entry'].changes`.value.* (escaped paths)
  - `raw_body_json`: `{{ $('Respond Creative Fatigue').item.json.body }}`
- **Connections:** Output → **Set: event field**
- **Failure/edge cases:**
  - The complex escaped paths can break if upstream Split Out changes structure.
  - `raw_body_json` references the Respond node body, not the original webhook body; it’s a small acknowledgement payload, not the raw Meta payload.

#### Node: Log Ad Recommendations
- **Sheet/tab:** `ad_recommendations`
- **Document:** same “Meta Ads Dispatcher Logs”
- **Key expressions:**
  - `date`: `{{ new Date($json.time * 1000).toLocaleString() }}`
  - `raw_body_json`: `{{ $('Respond Ad Recommendations').item.json.body }}`
  - `ad_object_ids`, `recommendation_type`, `recommendation_message` from `$json.changes.value.*`
- **Connections:** Output → **Set: event field**
- **Edge cases:** same “raw_body_json” caveat.

#### Node: Log Async Creation
- **Sheet/tab:** `ads_async_creation_request`
- **Document:** same “Meta Ads Dispatcher Logs”
- **Key expressions:**
  - `result`: `{{$json.changes.value.result.result_id}}`
  - `status`: `{{$json.changes.value.status}}`
  - `raw_body_json`: `{{ $('Respond Async Creation').item.json.body }}`
- **Connections:** Output → **Set: event field**
- **Edge cases:** nested path `result.result_id` may be absent.

#### Node: Log In-process ad objects
- **Sheet/tab:** `in_process_ad_objects`
- **Document:** **`YOUR_SPREADSHEET_ID` placeholder** (different from other logs)
- **Key expressions:** level/id/status_name from `$json.changes.value.*`
- **Connections:** Output → **Set: event field**
- **Failure/edge cases:**
  - Will fail until `YOUR_SPREADSHEET_ID` is replaced with a real spreadsheet ID.

#### Node: Log product set issues
- **Sheet/tab:** `product_set_issue`
- **Document:** **`YOUR_SPREADSHEET_ID` placeholder**
- **Key expressions:** `type`, `description`, `product_set_id`, `recommended_action`, `ad_account_id`
- **Connections:** Output → **Set: event field**
- **Failure/edge cases:** will fail until spreadsheet ID is set.

#### Node: Log with-issues ad objects
- **Sheet/tab:** `with_issues_ad_objects`
- **Document:** **`YOUR_SPREADSHEET_ID` placeholder**
- **Key expressions:**
  - `error_code`, `error_message`, `error_summary`, `level`
  - **Potential bug:** `ad_account_id` is set to `{{$json.changes.value.id}}` (looks like it should be an account id but uses `id`).
- **Connections:** Output → **Set: event field**
- **Failure/edge cases:** will fail until spreadsheet ID is set.

---

### 2.6 Cross-event merge + summarization + Slack notification

**Overview:**  
After each logging branch, the workflow sets a common `field` attribute, merges items from different event types, summarizes counts by `field`, and posts a Slack message per summarized group.

**Nodes involved:**
- **Set: event field**
- **Summarize**
- **Slack: send summary**

#### Node: Set: event field
- **Type / role:** `Set` (adds a normalized `field` for summarization).
- **Configuration choices:**
  - Adds `field` as string:
    - `{{ $('Split Out Changes CF').item.json['[\'body.entry\'].changes'].field }}`
  - **Include other fields:** enabled.
- **Connections:** Output → **Summarize**
- **Failure/edge cases (important):**
  - This node **references only the Creative Fatigue branch** (`Split Out Changes CF`) regardless of which branch triggered it.
  - When Ad Recommendations / Async / Issues branches flow into this node, that referenced node may not exist in the current execution path → `field` could become `undefined` or expression may fail depending on n8n behavior and run data availability.
  - More robust approach would set `field` from the *current item* (e.g., `$json.changes.field` or `$json.field` depending on structure) rather than referencing another node.
- **Version notes:** type version **3.4**.

#### Node: Summarize
- **Type / role:** `Summarize` aggregator.
- **Configuration choices:**
  - **Split by:** `field`
  - **Summarize:** `id` (and implicitly provides `count`)
- **Connections:** Output → **Slack: send summary**
- **Failure/edge cases:**
  - If `field` is missing/undefined, grouping may collapse into a single undefined group.

#### Node: Slack: send summary
- **Type / role:** `Slack` message sender.
- **Configuration choices:**
  - Sends formatted text including:
    - `Event type: {{$json.field}}`
    - `Count: {{$json.count}}`
    - Details link: `https://docs.google.com/spreadsheets/d/{{ $env.GOOGLE_SHEETS_ID }}/edit`
  - Channel selected by ID: `C0122GQH15RTA` (must exist and bot must have access).
- **Credentials:** not shown in JSON (Slack node typically uses Slack OAuth2 credentials or a Slack app token depending on instance setup).
- **Failure/edge cases:**
  - `$env.GOOGLE_SHEETS_ID` must be defined in n8n environment variables; otherwise link is broken (and may not match the actual sheet IDs used above).
  - Slack auth errors, missing channel permission, rate limits.
- **Version notes:** type version **2.4**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook: Meta (ad_account) | Webhook | Receive Meta webhook calls (GET verification + POST events) | — | Route: verification vs webhook | ## Get Webhook  \n**Webhook: Meta (ad_account)** … **Responses for all webhooks** Meta expects an acknowledgement… |
| Route: verification vs webhook | Switch | Distinguish verification requests vs ad_account webhook payloads | Webhook: Meta (ad_account) | Verify: token; Route: field | ## Get Webhook … |
| Verify: token | If | Validate `hub.verify_token` matches configured token | Route: verification vs webhook (Verification) | Respond: challenge | ## Get Webhook … |
| Respond: challenge | Respond to Webhook | Return `hub.challenge` text for Meta verification | Verify: token | — | ## Get Webhook … |
| Route: field | Switch | Route ad_account webhooks by `changes[0].field` | Route: verification vs webhook (AdAccountData) | Respond Creative Fatigue; Respond Ad Recommendations; Respond Async Creation; Respond In Process; Respond Product Set Issue; Respond With Issues | ## Get Webhook … |
| Respond Creative Fatigue | Respond to Webhook | Acknowledge creative_fatigue webhook | Route: field | Split Out Accounts CF | ## Get Webhook … |
| Respond Ad Recommendations | Respond to Webhook | Acknowledge ad_recommendations webhook | Route: field | Split Out Accounts AdRec | ## Get Webhook … |
| Respond Async Creation | Respond to Webhook | Acknowledge ads_async_creation_request webhook | Route: field | Split Out Accounts AsCr | ## Get Webhook … |
| Respond In Process | Respond to Webhook | Acknowledge in_process_ad_objects webhook | Route: field | Split Out Accounts Pr | ## Get Webhook … |
| Respond Product Set Issue | Respond to Webhook | Acknowledge product_set_issue webhook | Route: field | Split Out Accounts SetIssue | ## Get Webhook … |
| Respond With Issues | Respond to Webhook | Acknowledge with_issues_ad_objects webhook | Route: field | Split Out Accounts Issues | ## Get Webhook … |
| Split Out Accounts CF | Split Out | Split `body.entry` array (creative fatigue) | Respond Creative Fatigue | Split Out Changes CF | ##  Log data to flat table … |
| Split Out Changes CF | Split Out | Split changes array for CF (nonstandard path) | Split Out Accounts CF | Log Creative Fatigue | ##  Log data to flat table … |
| Log Creative Fatigue | Google Sheets | Append creative_fatigue log row | Split Out Changes CF | Set: event field | ##  Log data to flat table … |
| Split Out Accounts AdRec | Split Out | Split `body.entry` array (ad recommendations) | Respond Ad Recommendations | Split Out Changes AdRec | ##  Log data to flat table … |
| Split Out Changes AdRec | Split Out | Split `changes` array (ad recommendations) | Split Out Accounts AdRec | Log Ad Recommendations | ##  Log data to flat table … |
| Log Ad Recommendations | Google Sheets | Append ad_recommendations log row | Split Out Changes AdRec | Set: event field | ##  Log data to flat table … |
| Split Out Accounts AsCr | Split Out | Split `body.entry` array (async creation) | Respond Async Creation | Split Out Changes AsCr | ##  Log data to flat table … |
| Split Out Changes AsCr | Split Out | Split `changes` array (async creation) | Split Out Accounts AsCr | Log Async Creation | ##  Log data to flat table … |
| Log Async Creation | Google Sheets | Append ads_async_creation_request log row | Split Out Changes AsCr | Set: event field | ##  Log data to flat table … |
| Split Out Accounts Pr | Split Out | Split `body.entry` array (in-process) | Respond In Process | Split Out Changes Pr | ##  Log data to flat table … |
| Split Out Changes Pr | Split Out | Split `changes` array (in-process) | Split Out Accounts Pr | Log In-process ad objects | ##  Log data to flat table … |
| Log In-process ad objects | Google Sheets | Append in_process_ad_objects log row | Split Out Changes Pr | Set: event field | ##  Log data to flat table … |
| Split Out Accounts SetIssue | Split Out | Split `body.entry` array (product set issue) | Respond Product Set Issue | Split Out Changes SetIssue | ##  Log data to flat table … |
| Split Out Changes SetIssue | Split Out | Split `changes` array (product set issue) | Split Out Accounts SetIssue | Log product set issues | ##  Log data to flat table … |
| Log product set issues | Google Sheets | Append product_set_issue log row | Split Out Changes SetIssue | Set: event field | ##  Log data to flat table … |
| Split Out Accounts Issues | Split Out | Split `body.entry` array (with issues) | Respond With Issues | Split Out Changes SetIssue1 | ##  Log data to flat table … |
| Split Out Changes SetIssue1 | Split Out | Split `changes` array (with issues) | Split Out Accounts Issues | Log with-issues ad objects | ##  Log data to flat table … |
| Log with-issues ad objects | Google Sheets | Append with_issues_ad_objects log row | Split Out Changes SetIssue1 | Set: event field | ##  Log data to flat table … |
| Set: event field | Set | Add normalized `field` value before aggregation | All Log* nodes | Summarize | ## Send data to message … |
| Summarize | Summarize | Count events grouped by `field` | Set: event field | Slack: send summary | ## Send data to message … |
| Slack: send summary | Slack | Send Slack message with event type and count | Summarize | — | ## Send data to message … |
| Sticky Note - Overview | Sticky Note | Documentation / setup notes | — | — | # Meta Ads webhook dispatcher (ad_account)… [meta-ads-webhook-tester](https://github.com/KhatkevichKirill/meta-ads-webhook-tester) … |
| Sticky Note7 | Sticky Note | Documentation for webhook/verification/routing | — | — | (same content as note) |
| Sticky Note6 | Sticky Note | Documentation for splitting and logging | — | — | (same content as note) |
| Sticky Note11 | Sticky Note | Documentation for summarizing and Slack | — | — | (same content as note) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the trigger**
1. Add node **Webhook** named **“Webhook: Meta (ad_account)”**  
   - Path: `meta-ads-ad-account-webhook`  
   - Response Mode: **Using “Respond to Webhook” node**  
   - Enable **multiple methods** (to accept GET + POST)

2) **Add verification vs event router**
2. Add **Switch** named **“Route: verification vs webhook”** connected from the Webhook.  
   - Rule A (rename output “Verification”): `{{$json.query?.['hub.mode']}}` equals `subscribe`  
   - Rule B (rename output “AdAccountData”): `{{$json.body?.object}}` equals `ad_account`

3) **Build verification handshake**
3. Add **If** named **“Verify: token”** connected from Switch output “Verification”.  
   - Condition: `{{$json.query?.['hub.verify_token']}}` equals `YOUR_VERIFY_TOKEN`  
4. Add **Respond to Webhook** named **“Respond: challenge”** connected from the **true** output.  
   - Respond With: **Text**  
   - Body: `{{$json.query['hub.challenge']}}`  
5. (Recommended improvement) Add another **Respond to Webhook** on the **false** output returning **403** or a JSON error so Meta doesn’t time out.

4) **Route webhook events by field**
6. Add **Switch** named **“Route: field”** connected from Switch output “AdAccountData”.  
   - Evaluate: `{{$json.body.entry[0].changes[0].field}}`  
   - Add rules for:
     - `creative_fatigue`
     - `ad_recommendations`
     - `ads_async_creation_request`
     - `in_process_ad_objects`
     - `product_set_issue`
     - `with_issues_ad_objects`
   - Rename outputs accordingly (optional but matches this workflow)

5) **Acknowledge each event type**
7. For each switch output, add a **Respond to Webhook** node returning JSON acknowledgement:
   - Example body for creative fatigue:  
     `{{ { "status":"received", "field": $json.body.entry[0].changes[0].field, "webhook_type":"creative_fatigue" } }}`  
   - Repeat similarly for each event type with correct `webhook_type`.

6) **Flatten arrays for each event type**
8. After each Respond node, add:
   - **Split Out Accounts <Type>**: split field `body.entry`
   - **Split Out Changes <Type>**: split field `changes` (or the specific path used by your data shape)  
   - Ensure **Include all other fields** is enabled where you need context preserved.

7) **Create the Google Sheets log nodes**
9. Create a Google Spreadsheet with tabs named:
   - `creative_fatigue`, `ad_recommendations`, `ads_async_creation_request`, `in_process_ad_objects`, `product_set_issue`, `with_issues_ad_objects`
10. In n8n, add **Google Sheets** credentials (OAuth2) with edit access to the spreadsheet.
11. For each event type, add a **Google Sheets** node configured:
   - Operation: **Append**
   - Document: select your spreadsheet (by ID)
   - Sheet name: corresponding tab
   - Map columns to expressions matching your payload fields (as in the workflow)
   - Include a column like `raw_body_json` if desired (preferably store `{{$json.body}}` from the original webhook, not the respond node output).

8) **Summarize and send Slack**
12. Add **Set** node “Set: event field” after each log node (all log nodes connect into the same Set node).  
   - Set `field` based on the *current item* (recommended), e.g. from `$json.changes.field` or a carried-through value.
13. Add **Summarize** node:
   - Split by: `field`
   - Summarize: `id` (to get `count`)
14. Add **Slack** node “Slack: send summary”:
   - Configure Slack credentials (OAuth2 / Slack app) and ensure the bot has access to the target channel.
   - Choose Channel by ID (or name).
   - Message text uses: `{{$json.field}}` and `{{$json.count}}`
   - If using a sheet link, either hardcode the spreadsheet ID or set an environment variable like `GOOGLE_SHEETS_ID`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Meta expects acknowledgement for verification and for every webhook delivery. | Mentioned in sticky notes; implemented via Respond nodes. |
| Create Google Sheets tabs: `creative_fatigue`, `ad_recommendations`, `ads_async_creation_request`, `in_process_ad_objects`, `product_set_issue`, `with_issues_ad_objects`. | Logging design requirement. |
| Testing tool for realistic payloads (including arrays): meta-ads-webhook-tester | https://github.com/KhatkevichKirill/meta-ads-webhook-tester |
| Built by Kirill Khatkevich; LinkedIn | https://www.linkedin.com/in/kirill-khatkevich/ |

