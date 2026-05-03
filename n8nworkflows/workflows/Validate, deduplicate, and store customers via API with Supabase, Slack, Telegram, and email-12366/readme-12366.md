Validate, deduplicate, and store customers via API with Supabase, Slack, Telegram, and email

https://n8nworkflows.xyz/workflows/validate--deduplicate--and-store-customers-via-api-with-supabase--slack--telegram--and-email-12366


# Validate, deduplicate, and store customers via API with Supabase, Slack, Telegram, and email

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** (Retail) Customer Cleanup API → Supabase and send Notification  
**Goal:** Expose a synchronous HTTP API endpoint that receives customer data, **validates and normalizes it**, **prevents duplicates** using Supabase (by email), **stores new customers**, and **notifies** internal channels (Slack + Telegram) and the customer (Gmail).  
**Primary use cases:** onboarding API for retail/customer systems, data quality enforcement at ingestion, deduplication and audit of invalid submissions.

### 1.1 API Intake (Synchronous Webhook)
Receives a POST request and keeps the connection open until a dedicated “Respond to Webhook” node returns a response.

### 1.2 Validation & Normalization
Cleans names/email/phone and returns a structured `{ valid, errors, data }` payload. Routes to either validation error response or deduplication logic.

### 1.3 Deduplication & Persistence (Supabase)
Checks if a customer already exists (email equality). If duplicate, returns a “user exists” response; otherwise creates a new row in `customers`.

### 1.4 Responses & Notifications
Returns API responses and sends notifications:
- Slack DM/message to a selected user
- Telegram message to a chat
- Gmail email to the customer (on success path)

---

## 2. Block-by-Block Analysis

### Block 1 — API & Validation Layer

**Overview:** Accepts inbound customer payload from a webhook request, standardizes/cleans fields, performs field-level validation, and produces a canonical object for the rest of the workflow.

**Nodes involved:**
- **Customer API1** (Webhook)
- **Validate & Clean Data** (Code)
- **Is Validation Passed?** (IF)
- **API Validation Error** (Respond to Webhook)

#### Node: Customer API1
- **Type / role:** `Webhook` — entry point; synchronous API endpoint.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `customer/validate-and-store`
  - **Response mode:** `responseNode` (workflow must end with a Respond to Webhook node on all paths)
- **Input → Output:** Starts the workflow → outputs to **Validate & Clean Data**.
- **Edge cases / failures:**
  - Missing/invalid JSON body; downstream code expects `$json.body` (may become `{}`).
  - If any execution path does not reach a Respond to Webhook node, the request can hang until timeout.

#### Node: Validate & Clean Data
- **Type / role:** `Code` — transforms and validates incoming payload.
- **Key configuration choices (interpreted):**
  - Reads request payload as `const input = $json.body || {};`
  - Produces:
    - `valid`: boolean
    - `errors`: array of `{ field, message }`
    - `data`: normalized `{ first_name, last_name, email, phone }`
- **Key logic & expressions:**
  - **Name normalization:** trims, lowercases, then capitalizes word initials.
  - **Email normalization:** trim + lowercase; regex validation; max length 254.
  - **Phone normalization:** removes non-numeric/non-plus; if 10 digits → assumes India and prepends `+91`; requires leading `+` and length 11–16.
  - **Name validation regex:** `/^[a-zA-Z\\s'-]+$/` (rejects accented letters and many non-Latin names).
- **Input → Output:** From webhook → to **Is Validation Passed?**.
- **Edge cases / failures:**
  - Assumes Indian numbering when 10 digits; international customers may fail validation unless they send `+<countrycode>`.
  - Names with accents (e.g., “José”) or non-English alphabets will be rejected.
  - If body fields use different keys than `first_name/last_name/email/phone`, they’ll be treated as missing.

#### Node: Is Validation Passed?
- **Type / role:** `IF` — routes valid vs invalid payloads.
- **Key configuration:**
  - Condition: `{{$json.valid}} isTrue`
- **Branches:**
  - **True:** → **Check Existing User**
  - **False:** → **API Validation Error**
- **Edge cases / failures:**
  - If `valid` is missing or not boolean, strictness can cause unexpected routing; here it is always set by the Code node.

#### Node: API Validation Error
- **Type / role:** `Respond to Webhook` — returns an HTTP response for invalid input.
- **Key configuration (as provided):**
  - Options present but no explicit status code/body shown in parameters.
- **Input → Output:** Receives invalid validation payload → forwards to **Slack Notify** (and then Telegram).
- **Edge cases / failures:**
  - If not configured to return a 4xx status (commonly 400) and error body, API consumers won’t get actionable feedback.
  - Some n8n setups stop execution after responding; in this workflow it continues to Slack/Telegram (works, but be aware of response timing).

**Sticky note covering this block:**
- **“API & Validation Layer”** — describes centralized validation/cleanup and gating logic.

---

### Block 2 — Duplicate Check & Database Logic (Supabase)

**Overview:** Checks for an existing customer row by email to prevent duplicates, then either returns a duplicate response or creates a new customer record.

**Nodes involved:**
- **Check Existing User** (Supabase)
- **User Already Exists?** (IF)
- **Create User** (Supabase)
- **API User Exists** (Respond to Webhook)

#### Node: Check Existing User
- **Type / role:** `Supabase` — read query for deduplication.
- **Key configuration choices:**
  - **Operation:** Get All (from table)
  - **Table:** `customers`
  - **Filter condition:** `email eq {{$json.data.email}}`
  - **alwaysOutputData:** true (continues even if no rows are found)
- **Inputs / outputs:**
  - Input: validated payload (`data.email`)
  - Output: list/collection of matching rows (format depends on Supabase node)
  - Next: → **User Already Exists?**
- **Credentials:** Supabase API credential (service role key recommended for server-side writes).
- **Edge cases / failures:**
  - Auth errors (bad URL/key), RLS policies blocking select.
  - Email column not indexed: potential latency at scale.
  - Output shape differences across node versions can break downstream “empty check”.

#### Node: User Already Exists?
- **Type / role:** `IF` — decides duplicate vs new user path.
- **Configuration warning (important):**
  - The stored condition in JSON appears unusual: it checks an “empty” operation with a constant `leftValue: true`. As written, it may not actually evaluate the Supabase result correctly.
  - Intended logic is likely: “if Supabase returned any rows → duplicate; else → create”.
- **Branches (as connected):**
  - **Output 0 (true branch):** → **API User Exists**
  - **Output 1 (false branch):** → **Create User**
- **Edge cases / failures:**
  - If the IF condition is misconfigured, duplicates may be inserted or new users may be blocked.
  - Strongly recommended to explicitly check something like:
    - `{{ $items().length > 0 }}` or
    - `{{ $json.length > 0 }}` depending on Supabase output format.

#### Node: Create User
- **Type / role:** `Supabase` — inserts a new row into `customers`.
- **Key configuration choices:**
  - **Table:** `customers`
  - **Fields mapped using expressions referencing the validation node:**
    - `first_name`: `{{ $('Is Validation Passed?').item.json.data.first_name }}`
    - `last_name`: `{{ $('Is Validation Passed?').item.json.data.last_name }}`
    - `email`: `{{ $('Is Validation Passed?').item.json.data.email }}`
    - `phone`: `{{ $('Is Validation Passed?').item.json.data.phone }}`
    - `valid`: `{{ $('Is Validation Passed?').item.json.valid }}`
    - `error`: `{{ $('Is Validation Passed?').item.json.errors }}`
- **Input → Output:** From IF (non-duplicate path) → to **API Success Response**.
- **Edge cases / failures:**
  - DB constraint violations (unique email) if deduplication fails/races.
  - Column type mismatch: `error` is an array; table must support JSON/JSONB (recommended) or text serialization.
  - RLS policies blocking insert.

#### Node: API User Exists
- **Type / role:** `Respond to Webhook` — returns an HTTP response when a duplicate is detected.
- **Key configuration (as provided):**
  - No explicit status code/body shown (commonly should be **409 Conflict**).
- **Input → Output:** Duplicate path → continues to **Slack Notify**.
- **Edge cases / failures:**
  - Without a clear response body, clients won’t know how to resolve duplication (e.g., login instead of signup).

**Sticky note covering this block:**
- **“Duplicate Check & Database Logic”** — explains Supabase check, duplicate prevention, routing.

---

### Block 3 — Notifications & API Responses

**Overview:** Produces final API responses and sends internal (Slack/Telegram) notifications on both success and failure/duplicate paths; emails the customer on successful creation.

**Nodes involved:**
- **API Success Response** (Respond to Webhook)
- **Send Notifiaction to User** (Gmail)
- **Slack Notify** (Slack)
- **Telegram Notify** (Telegram)

#### Node: API Success Response
- **Type / role:** `Respond to Webhook` — returns success to API caller after insert.
- **Key configuration (as provided):**
  - No explicit status code/body shown (commonly should be **201 Created** with created record or ID).
- **Input → Output:** From **Create User** → then to **Send Notifiaction to User**.
- **Edge cases / failures:**
  - If response is sent before insert details are included, clients may not get created record metadata.

#### Node: Send Notifiaction to User
- **Type / role:** `Gmail` — sends confirmation email to the customer.
- **Key configuration:**
  - **Subject:** “Your account has been created”
  - **Body (text):** `Hello {{ $json.first_name }} {{ $json.last_name }}, Your customer profile has been successfully created. Thank you.`
  - **Send To:** currently empty (`sendTo: ""`)
- **Input → Output:** From **API Success Response** → to **Slack Notify**.
- **Critical issue / edge cases:**
  - With `sendTo` blank, the node will fail or send nowhere. It should likely be `{{ $('Is Validation Passed?').item.json.data.email }}` or use the created user email from Supabase output.
  - Referencing `$json.first_name` here depends on what **API Success Response** outputs; Respond to Webhook nodes often do not pass through the original item as expected. Safer to reference the validation node explicitly.

#### Node: Slack Notify
- **Type / role:** `Slack` — internal notification.
- **Key configuration choices:**
  - Sends text: `{{ $json.valid ? 'Customer stored successfully' : 'Customer validation failed' }}`
  - Targets a specific user (selected list value `U09SEALBNAE`, cached name “n8n_demo”)
- **Input → Output:** Triggered after **API Validation Error**, **API User Exists**, and **Send Notifiaction to User** → outputs to **Telegram Notify**.
- **Edge cases / failures:**
  - Expression depends on `$json.valid` being present on all incoming paths (duplicate and success paths may not include `valid` unless explicitly mapped).
  - Slack auth/token scope issues, user not found, rate limiting.

#### Node: Telegram Notify
- **Type / role:** `Telegram` — internal notification.
- **Key configuration:**
  - Chat ID: `123456789`
  - Text: `{{ $json.valid ? 'Customer stored in Supabase' : 'Customer validation failed' }}`
- **Input → Output:** From **Slack Notify**; terminal node (no further connections).
- **Edge cases / failures:**
  - Same `$json.valid` availability issue as Slack.
  - Chat ID may be wrong; bot must be allowed in the chat.

**Sticky note covering this block:**
- **“Notifications & API Responses”** — describes API responses + Slack/Telegram + email on success.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Customer API1 | Webhook | Synchronous API entrypoint (POST) | — | Validate & Clean Data | # API & Validation Layer  \n\n## Description\n**This section receives customer data via an API call and performs centralized validation and cleanup. It standardizes names, emails and phone numbers, aggregates field-specific errors and decides whether the request can proceed further in the workflow.** |
| Validate & Clean Data | Code | Normalize + validate fields; produce `{valid, errors, data}` | Customer API1 | Is Validation Passed? | # API & Validation Layer  \n\n## Description\n**This section receives customer data via an API call and performs centralized validation and cleanup. It standardizes names, emails and phone numbers, aggregates field-specific errors and decides whether the request can proceed further in the workflow.** |
| Is Validation Passed? | IF | Route valid vs invalid submissions | Validate & Clean Data | Check Existing User; API Validation Error | # API & Validation Layer  \n\n## Description\n**This section receives customer data via an API call and performs centralized validation and cleanup. It standardizes names, emails and phone numbers, aggregates field-specific errors and decides whether the request can proceed further in the workflow.** |
| API Validation Error | Respond to Webhook | Return validation error response to API client | Is Validation Passed? | Slack Notify | # Notifications & API Responses  \n\n## Description\n**This section handles final API responses and notifications. It sends success or failure responses back to the client and notifies internal teams via Slack and Telegram, while also emailing the customer when a new account is successfully created.** |
| Check Existing User | Supabase | Query `customers` by email for deduplication | Is Validation Passed? | User Already Exists? | # Duplicate Check & Database Logic\n\n## Description\n**This section checks Supabase to determine whether a customer already exists based on email. It prevents duplicate records, routes existing users to an appropriate API response and ensures only new, valid customers are stored in the database.** |
| User Already Exists? | IF | Route duplicate vs new customer creation | Check Existing User | API User Exists; Create User | # Duplicate Check & Database Logic\n\n## Description\n**This section checks Supabase to determine whether a customer already exists based on email. It prevents duplicate records, routes existing users to an appropriate API response and ensures only new, valid customers are stored in the database.** |
| API User Exists | Respond to Webhook | Return duplicate-user response to API client | User Already Exists? | Slack Notify | # Notifications & API Responses  \n\n## Description\n**This section handles final API responses and notifications. It sends success or failure responses back to the client and notifies internal teams via Slack and Telegram, while also emailing the customer when a new account is successfully created.** |
| Create User | Supabase | Insert new customer into `customers` | User Already Exists? | API Success Response | # Duplicate Check & Database Logic\n\n## Description\n**This section checks Supabase to determine whether a customer already exists based on email. It prevents duplicate records, routes existing users to an appropriate API response and ensures only new, valid customers are stored in the database.** |
| API Success Response | Respond to Webhook | Return success response to API client | Create User | Send Notifiaction to User | # Notifications & API Responses  \n\n## Description\n**This section handles final API responses and notifications. It sends success or failure responses back to the client and notifies internal teams via Slack and Telegram, while also emailing the customer when a new account is successfully created.** |
| Send Notifiaction to User | Gmail | Email customer after successful creation | API Success Response | Slack Notify | # Notifications & API Responses  \n\n## Description\n**This section handles final API responses and notifications. It sends success or failure responses back to the client and notifies internal teams via Slack and Telegram, while also emailing the customer when a new account is successfully created.** |
| Slack Notify | Slack | Internal Slack notification | API Validation Error; API User Exists; Send Notifiaction to User | Telegram Notify | # Notifications & API Responses  \n\n## Description\n**This section handles final API responses and notifications. It sends success or failure responses back to the client and notifies internal teams via Slack and Telegram, while also emailing the customer when a new account is successfully created.** |
| Telegram Notify | Telegram | Internal Telegram notification | Slack Notify | — | # Notifications & API Responses  \n\n## Description\n**This section handles final API responses and notifications. It sends success or failure responses back to the client and notifies internal teams via Slack and Telegram, while also emailing the customer when a new account is successfully created.** |
| Sticky Note | Sticky Note | Comment block: API & validation | — | — |  |
| Sticky Note1 | Sticky Note | Comment block: duplicate check & DB logic | — | — |  |
| Sticky Note2 | Sticky Note | Comment block: notifications & responses | — | — |  |
| Sticky Note3 | Sticky Note | Comment block: how it works + setup steps | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Webhook entry**
   1. Add node: **Webhook**
   2. Name: **Customer API1**
   3. Set **HTTP Method** = `POST`
   4. Set **Path** = `customer/validate-and-store`
   5. Set **Response Mode** = `Using Respond to Webhook node` (responseNode)

2. **Add validation/cleanup code**
   1. Add node: **Code**
   2. Name: **Validate & Clean Data**
   3. Paste logic that:
      - Reads `const input = $json.body || {};`
      - Normalizes `first_name`, `last_name`, `email`, `phone`
      - Produces `{ valid, errors, data }`
   4. Connect: **Customer API1 → Validate & Clean Data**

3. **Add validation gate**
   1. Add node: **IF**
   2. Name: **Is Validation Passed?**
   3. Condition: Boolean → Value1 `{{ $json.valid }}` → Operation `is true`
   4. Connect: **Validate & Clean Data → Is Validation Passed?**

4. **Validation error response path**
   1. Add node: **Respond to Webhook**
   2. Name: **API Validation Error**
   3. Configure recommended response:
      - **Status code:** 400
      - **Body:** include `{ valid: false, errors: {{$json.errors}}, data: {{$json.data}} }`
   4. Connect: **Is Validation Passed? (false) → API Validation Error**

5. **Supabase duplicate check path**
   1. Add node: **Supabase**
   2. Name: **Check Existing User**
   3. Credentials: create **Supabase API** credential:
      - Supabase URL
      - Service role key (or key with select/insert permissions)
   4. Operation: **Get All**
   5. Table: `customers`
   6. Filter: `email` **equals** `{{ $json.data.email }}`
   7. Enable “Always Output Data” (so the IF can decide even when empty)
   8. Connect: **Is Validation Passed? (true) → Check Existing User**

6. **IF for “already exists”**
   1. Add node: **IF**
   2. Name: **User Already Exists?**
   3. Configure explicitly (recommended) based on Supabase output:
      - If Supabase returns an array in `$json`: use **Expression** condition like `{{ $json.length > 0 }}`
      - Or use “Number of Items” approach: `{{ $items().length > 0 }}`
   4. Connect: **Check Existing User → User Already Exists?**

7. **Duplicate response**
   1. Add node: **Respond to Webhook**
   2. Name: **API User Exists**
   3. Recommended configuration:
      - **Status code:** 409
      - **Body:** `{ message: "User already exists", email: "{{ $json.email || $json.data?.email }}" }`
   4. Connect: **User Already Exists? (true) → API User Exists**

8. **Create new customer in Supabase**
   1. Add node: **Supabase**
   2. Name: **Create User**
   3. Operation: **Insert**
   4. Table: `customers`
   5. Map fields (recommended to reference the validation output deterministically):
      - `first_name` = `{{ $('Is Validation Passed?').item.json.data.first_name }}`
      - `last_name` = `{{ $('Is Validation Passed?').item.json.data.last_name }}`
      - `email` = `{{ $('Is Validation Passed?').item.json.data.email }}`
      - `phone` = `{{ $('Is Validation Passed?').item.json.data.phone }}`
      - `valid` = `{{ $('Is Validation Passed?').item.json.valid }}`
      - `error` = `{{ $('Is Validation Passed?').item.json.errors }}` (ensure JSON/JSONB column)
   6. Connect: **User Already Exists? (false) → Create User**

9. **Success response**
   1. Add node: **Respond to Webhook**
   2. Name: **API Success Response**
   3. Recommended configuration:
      - **Status code:** 201
      - **Body:** include created record fields (from Supabase insert output) or at least `{ message: "Created" }`
   4. Connect: **Create User → API Success Response**

10. **Email the customer (Gmail)**
   1. Add node: **Gmail**
   2. Name: **Send Notifiaction to User**
   3. Credentials: configure **Gmail OAuth2** in n8n (Google Cloud OAuth consent + scopes)
   4. Set:
      - **To:** `{{ $('Is Validation Passed?').item.json.data.email }}`
      - **Subject:** `Your account has been created`
      - **Email type:** Text
      - **Message:** `Hello {{ $('Is Validation Passed?').item.json.data.first_name }} {{ $('Is Validation Passed?').item.json.data.last_name }}, Your customer profile has been successfully created. Thank you.`
   5. Connect: **API Success Response → Send Notifiaction to User**

11. **Slack notification**
   1. Add node: **Slack**
   2. Name: **Slack Notify**
   3. Credentials: Slack API credential with permission to post messages
   4. Configure message text (recommended robust version):
      - `Customer processing result: {{ $('Is Validation Passed?').item.json.valid ? 'success' : 'failed' }}`
      - Optionally include email/phone
   5. Configure target (user/channel) as needed
   6. Connect:
      - **API Validation Error → Slack Notify**
      - **API User Exists → Slack Notify**
      - **Send Notifiaction to User → Slack Notify**

12. **Telegram notification**
   1. Add node: **Telegram**
   2. Name: **Telegram Notify**
   3. Credentials: Telegram bot token
   4. Set **Chat ID** (your chat/group)
   5. Message text (recommended robust version) similar to Slack
   6. Connect: **Slack Notify → Telegram Notify**

13. **Database prerequisites (Supabase)**
   - Create table `customers` with columns:
     - `first_name` (text), `last_name` (text), `email` (text, **unique**), `phone` (text)
     - `valid` (boolean)
     - `error` (json/jsonb) or text
   - Ensure RLS policies allow `select` and `insert` for the key used.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow behaves as a synchronous API: webhook waits until Respond to Webhook returns. | Sticky note “How It Works” |
| Validation normalizes whitespace, capitalization, email lowercase, and phone formatting (defaults to India `+91` when 10 digits). | Sticky note “How It Works” |
| Intended responses: 400 for validation errors, 409 for duplicates, 201 for success. | Sticky note “How It Works” (behavior described), but response nodes are not explicitly configured in JSON |
| Setup guidance: create Supabase project/table; configure Supabase service role key; configure Slack/Telegram/Gmail; test with Postman; activate and monitor. | Sticky note “How It Works” (Setup Steps) |