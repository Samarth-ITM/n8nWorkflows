Escalate negative Facebook Page reviews to Slack and Supabase tickets

https://n8nworkflows.xyz/workflows/escalate-negative-facebook-page-reviews-to-slack-and-supabase-tickets-13273


# Escalate negative Facebook Page reviews to Slack and Supabase tickets

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow ingests Facebook Page review events via an HTTP webhook, normalizes the payload into a standard internal schema, detects negative reviews (rating ≤ 2), alerts a Slack channel, and attempts to create a tracking ticket (“case”) in Supabase. If Supabase case creation fails, it sends a fallback Slack alert containing the error.

**Target use cases:**
- Customer support escalation for poor Facebook Page reviews
- Lightweight incident/ticket creation in Supabase for follow-up
- Real-time notifications to a support Slack channel

### 1.1 Input Reception & Normalization
Receives the webhook request and maps incoming fields (`body.*`) into normalized fields (`rating`, `review_text`, etc.).

### 1.2 Negative Review Detection & Slack Escalation
Filters to only negative reviews (≤ 2 stars) and posts an alert to Slack.

### 1.3 Ticket Creation in Supabase & Failure Escalation
Attempts to create a Supabase record; if an error is detected in the Supabase node output, posts a failure alert to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization
**Overview:**  
Receives Facebook review payloads via webhook and standardizes key fields into a consistent schema used by downstream nodes.

**Nodes involved:**
- Facebook Page Review Trigger
- Global Configuration

#### Node: Facebook Page Review Trigger
- **Type / role:** Webhook (trigger). Entry point for external HTTP POST events.
- **Configuration (interpreted):**
  - HTTP Method: `POST`
  - Path: `/facebook-reviews`
  - Webhook is identified internally by a webhook ID (used by n8n).
- **Key data expectations:**
  - Expects the incoming payload to have a `body` object with:
    - `reviewer_name`
    - `rating`
    - `review_text`
    - `page_name`
- **Connections:**
  - **Output →** Global Configuration
- **Version specifics:** `typeVersion 2.1` (Webhook node behavior differs slightly across versions; ensure you’re on a compatible n8n version supporting Webhook v2+).
- **Edge cases / failures:**
  - Missing `body` or missing fields will cause downstream expressions like `$json.body.rating` to resolve to `undefined` (may break numeric comparisons later).
  - Non-numeric `rating` can cause the IF condition (number comparison) to fail or evaluate unexpectedly.

#### Node: Global Configuration
- **Type / role:** Set node (data mapping/normalization).
- **Configuration (interpreted):**
  - Creates a normalized object containing:
    - `reviewer_name` = `{{$json.body.reviewer_name}}`
    - `rating` (number) = `{{$json.body.rating}}`
    - `review_text` = `{{$json.body.review_text}}`
    - `page_name` = `{{$json.body.page_name}}`
  - “Include: none” indicates it outputs only the explicitly set fields (drops all other incoming data).
- **Key expressions / variables:**
  - Uses `$json.body.*` paths.
- **Connections:**
  - **Input ←** Facebook Page Review Trigger
  - **Output →** Check Negative Review (≤ 2 Stars)
- **Version specifics:** `typeVersion 3` (Set node UI/behavior can differ between versions; confirm field typing is set correctly for `rating`).
- **Edge cases / failures:**
  - If `body` is absent, every mapped field becomes empty/undefined; the IF node may treat rating as null/NaN.
  - If `rating` is a string like `"2"`, Set attempts to store it as a number; depending on n8n coercion rules, it may work or result in NaN—validate payload typing.

**Sticky note context (applies to nodes in this block):**
- “Review Intake & Configuration”: central place to adapt field mappings for different payload structures.

---

### Block 2 — Negative Review Detection & Slack Escalation
**Overview:**  
Filters reviews to only those with rating ≤ 2 and sends a Slack message notifying support with key review details.

**Nodes involved:**
- Check Negative Review (≤ 2 Stars)
- Slack – New Negative Review Alert

#### Node: Check Negative Review (≤ 2 Stars)
- **Type / role:** IF node (routing/filtering).
- **Configuration (interpreted):**
  - Condition: `{{$json.rating}} <= 2` using a strict number comparison.
- **Key expressions / variables:**
  - `leftValue = {{$json.rating}}`
  - `rightValue = 2`
- **Connections:**
  - **Input ←** Global Configuration
  - **True output (main index 0) →** Slack – New Negative Review Alert
  - **False output:** not connected (workflow ends for non-negative reviews).
- **Version specifics:** `typeVersion 2` and condition `typeValidation: strict` (important: strict typing means null/undefined/non-number may not behave as intended).
- **Edge cases / failures:**
  - If `rating` is missing/undefined, the condition may fail validation or evaluate false; negative reviews could be missed if payload is malformed.
  - If rating scale differs (e.g., 0–10), logic would need adjusting.

#### Node: Slack – New Negative Review Alert
- **Type / role:** Slack node (message posting to a channel).
- **Configuration (interpreted):**
  - Operation: Send a message to a selected channel (channel ID is not set in the JSON export; must be configured).
  - Message text includes templated fields:
    - Page name, reviewer name, rating, review text
- **Key expressions / variables used in message:**
  - `{{ $json.page_name }}`
  - `{{ $json.reviewer_name }}`
  - `{{ $json.rating }}`
  - `{{ $json.review_text }}`
- **Connections:**
  - **Input ←** Check Negative Review (≤ 2 Stars) [true path]
  - **Output →** Create Support Case (Supabase)
- **Credentials:**
  - Uses Slack API credentials (“Slack account 31”).
- **Version specifics:** `typeVersion 2`
- **Edge cases / failures:**
  - Missing channel selection (`channelId.value` empty) will cause runtime failure until configured.
  - Slack auth/token scope issues (e.g., missing `chat:write`) will fail message posting.
  - Message formatting assumes fields are present; missing values will render blank.

**Sticky note context (applies to nodes in this block):**
- “Negative Review Detection & Alert”: only proceed for rating ≤ 2 and notify support immediately.

---

### Block 3 — Ticket Creation in Supabase & Failure Escalation
**Overview:**  
Creates a Supabase record in `cases`. If Supabase returns an error, the workflow detects it and sends a Slack alert with the error details.

**Nodes involved:**
- Create Support Case (Supabase)
- Check Case Creation Failure
- Slack – Case Creation Failed Alert

#### Node: Create Support Case (Supabase)
- **Type / role:** Supabase node (insert row into a table).
- **Configuration (interpreted):**
  - Table: `cases`
  - Fields inserted:
    - `rating` = `{{ $('Global Configuration').item.json.rating }}`
    - `review_text` = `{{ $('Global Configuration').item.json.review_text }}`
    - `reviewer_name` = `{{ $('Global Configuration').item.json.reviewer_name }}`
    - `page_name` = `{{ $('Global Configuration').item.json.page_name }}`
  - `onError: continueRegularOutput` and `alwaysOutputData: true`:
    - The workflow continues even if Supabase insert fails.
    - Output will still be produced, typically including an `error` property when failing.
- **Key expressions / variables:**
  - Uses cross-node reference: `$('Global Configuration').item.json.*`
    - This is robust to upstream changes as long as that node exists and produces expected fields.
- **Connections:**
  - **Input ←** Slack – New Negative Review Alert
  - **Output →** Check Case Creation Failure
- **Credentials:**
  - Supabase API credentials (“Supabase account 6”).
- **Version specifics:** `typeVersion 1`
- **Edge cases / failures:**
  - Schema mismatch: if `cases` table or columns don’t exist, insert fails.
  - RLS (Row Level Security) enabled without correct policies can cause insert failures.
  - Credential issues (wrong URL/key) cause auth failures.
  - Network/timeouts may return transient errors; currently no retry logic is implemented.

#### Node: Check Case Creation Failure
- **Type / role:** IF node (detect error condition).
- **Configuration (interpreted):**
  - Condition group requires BOTH:
    1) `{{$json.error}}` is **not empty**
    2) `{{$json.rating}}` is **empty**
  - This appears intended to identify a Supabase error payload (where `error` exists and the original case fields are not present on the output item).
- **Connections:**
  - **Input ←** Create Support Case (Supabase)
  - **True output →** Slack – Case Creation Failed Alert
  - **False output:** not connected (workflow ends on success).
- **Version specifics:** `typeVersion 2`, strict validation.
- **Edge cases / failures:**
  - Error detection logic is somewhat fragile:
    - If Supabase returns an error but still includes some fields, the second condition (`rating empty`) could be false, preventing the failure alert.
    - If a success response includes an `error` field for any reason, it could false-trigger (unlikely but possible).
  - Consider simplifying to just `notEmpty($json.error)` if Supabase reliably populates it on failure.

#### Node: Slack – Case Creation Failed Alert
- **Type / role:** Slack node (fallback error notification).
- **Configuration (interpreted):**
  - Sends message to a channel (channel ID not set; must be configured).
  - Message includes the same review details plus `{{ $json.error }}`.
- **Key expressions / variables used in message:**
  - `{{ $json.page_name }}`
  - `{{ $json.reviewer_name }}`
  - `{{ $json.rating }}`
  - `{{ $json.review_text }}`
  - `{{ $json.error }}`
- **Connections:**
  - **Input ←** Check Case Creation Failure [true path]
  - **Output:** none
- **Credentials:**
  - Slack API credentials (“Slack account 31”).
- **Version specifics:** `typeVersion 2`
- **Edge cases / failures:**
  - If the Supabase error output does not include the original review fields, Slack message will have blanks for page/reviewer/rating/review text. (This is likely why the Supabase node uses `alwaysOutputData`, but the IF condition suggests the fields may still be missing.)

**Sticky note context (applies to nodes in this block):**
- “Case Creation & Error Handling”: create Supabase case; if it fails, ensure visibility via Slack.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documentation / operator guidance | — | — | Facebook Page Negative Review Watchdog; Setup steps for Webhook, field mapping, Slack, Supabase, activation |
| Trigger & Filter | Sticky Note | Block label for intake/config | — | — | Review Intake & Configuration; central configuration point for adapting payload |
| Alert & Storage | Sticky Note | Block label for detection/alert | — | — | Negative Review Detection & Alert; only proceed when rating ≤ 2; send Slack alert |
| Error Handling | Sticky Note | Block label for ticket creation/error path | — | — | Case Creation & Error Handling; create Supabase record; Slack fallback on failure |
| Facebook Page Review Trigger | Webhook | Receives review payload via HTTP POST | — | Global Configuration | Review Intake & Configuration; central configuration point for adapting payload |
| Global Configuration | Set | Normalizes incoming payload fields | Facebook Page Review Trigger | Check Negative Review (≤ 2 Stars) | Review Intake & Configuration; central configuration point for adapting payload |
| Check Negative Review (≤ 2 Stars) | IF | Filters to negative reviews (rating ≤ 2) | Global Configuration | Slack – New Negative Review Alert | Negative Review Detection & Alert; only proceed when rating ≤ 2; send Slack alert |
| Slack – New Negative Review Alert | Slack | Posts negative review message to Slack channel | Check Negative Review (≤ 2 Stars) | Create Support Case (Supabase) | Negative Review Detection & Alert; only proceed when rating ≤ 2; send Slack alert |
| Create Support Case (Supabase) | Supabase | Inserts a “case” row for tracking | Slack – New Negative Review Alert | Check Case Creation Failure | Case Creation & Error Handling; create Supabase record; Slack fallback on failure |
| Check Case Creation Failure | IF | Detects Supabase insert failure | Create Support Case (Supabase) | Slack – Case Creation Failed Alert | Case Creation & Error Handling; create Supabase record; Slack fallback on failure |
| Slack – Case Creation Failed Alert | Slack | Sends Slack alert with Supabase error | Check Case Creation Failure | — | Case Creation & Error Handling; create Supabase record; Slack fallback on failure |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: “Facebook Page Negative Review Watchdog → Slack Escalation + Supabase Ticket” (or your preferred name).
   - Keep execution order setting default (this workflow uses `executionOrder: v1`).

2) **Add Webhook trigger**
   - Add node: **Webhook**
   - Configure:
     - **HTTP Method:** POST  
     - **Path:** `facebook-reviews`
   - Deploy URL will be used by your Facebook review source/webhook provider.

3) **Add normalization node (Global Configuration)**
   - Add node: **Set**
   - Configure **Fields** (and set “Include” to **None / Keep Only Set**):
     - `reviewer_name` (String) = `{{$json.body.reviewer_name}}`
     - `rating` (Number) = `{{$json.body.rating}}`
     - `review_text` (String) = `{{$json.body.review_text}}`
     - `page_name` (String) = `{{$json.body.page_name}}`
   - Connect: **Webhook → Set**

4) **Add negative review filter**
   - Add node: **IF**
   - Condition:
     - Type: **Number**
     - Value 1: `{{$json.rating}}`
     - Operation: **less than or equal**
     - Value 2: `2`
   - Connect: **Set → IF**

5) **Add Slack alert for negative reviews**
   - Add node: **Slack**
   - Credentials:
     - Connect Slack API credentials (OAuth/token) with permission to post messages (`chat:write`).
   - Operation: **Post message** (channel)
   - Select the target **Channel** (required; the exported workflow had an empty channelId).
   - Message text (example matching the workflow):
     - `🚨 *Negative Facebook Review Detected*`
     - `📄 Page: {{$json.page_name}}`
     - `👤 Reviewer: {{$json.reviewer_name}}`
     - `⭐ Rating: {{$json.rating}}`
     - `📝 Review: {{$json.review_text}}`
   - Connect: **IF (true) → Slack (negative review alert)**
   - Leave IF (false) unconnected (workflow ends for non-negative reviews).

6) **Add Supabase case creation**
   - Add node: **Supabase**
   - Credentials:
     - Provide Supabase project URL and service role key (or an API key permitted by your RLS policies).
   - Operation: **Insert** (or “Create” depending on node UI)
   - Table: `cases`
   - Map fields (use either `$json.*` directly or replicate the workflow’s cross-node references):
     - `rating` = `{{ $('Global Configuration').item.json.rating }}`
     - `review_text` = `{{ $('Global Configuration').item.json.review_text }}`
     - `reviewer_name` = `{{ $('Global Configuration').item.json.reviewer_name }}`
     - `page_name` = `{{ $('Global Configuration').item.json.page_name }}`
   - Error handling:
     - Set node option **On Error:** “Continue (regular output)”
     - Enable **Always Output Data** (so the next IF can inspect the output even on failure)
   - Connect: **Slack (negative review alert) → Supabase**

7) **Add failure detection IF**
   - Add node: **IF**
   - Conditions (AND):
     1) String → `{{$json.error}}` **is not empty**
     2) String → `{{$json.rating}}` **is empty**
   - Connect: **Supabase → IF (failure check)**

8) **Add Slack fallback alert for Supabase failure**
   - Add node: **Slack**
   - Use same Slack credentials as before.
   - Select the same or a different channel for failures.
   - Message text:
     - `❌ *Supabase Case Creation Failed*`
     - `📄 Page: {{$json.page_name}}`
     - `👤 Reviewer: {{$json.reviewer_name}}`
     - `⭐ Rating: {{$json.rating}}`
     - `📝 Review: {{$json.review_text}}`
     - `🧾 Error: {{$json.error}}`
   - Connect: **IF (true) → Slack (failure alert)**

9) **(Optional) Add sticky notes**
   - Add sticky notes to label:
     - Intake/configuration
     - Detection/alert
     - Case creation/error handling
   - This is optional for execution but helps maintenance.

10) **Activate**
   - Test by sending a POST request to the webhook with a JSON body containing the expected fields.
   - Activate the workflow after verifying Slack posting and Supabase inserts.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Configure the Webhook node and connect it to your Facebook Page review source or webhook provider.” | Operations note from workflow “Overview” sticky note |
| “Review the Global Configuration node and adjust field mappings if your incoming payload structure differs.” | Mapping/compatibility note from “Overview” |
| “Connect your Slack credentials and select the channel where alerts should be delivered.” | Slack setup note from “Overview” |
| “Connect your Supabase credentials and configure the table where support records should be stored.” | Supabase setup note from “Overview” |
| “Activate the workflow to begin monitoring Facebook Page reviews automatically.” | Deployment note from “Overview” |