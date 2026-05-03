Build an omnichannel OTP verification flow with WhatsApp, Telegram, email and Postgres

https://n8nworkflows.xyz/workflows/build-an-omnichannel-otp-verification-flow-with-whatsapp--telegram--email-and-postgres-14209


# Build an omnichannel OTP verification flow with WhatsApp, Telegram, email and Postgres

# 1. Workflow Overview

This workflow implements an omnichannel OTP-based user verification flow in n8n. It accepts inbound messages from multiple channels—primarily WhatsApp, Telegram, and email—normalizes them into a shared structure, checks whether the channel is already linked to an existing verified user, and if not, walks the user through email capture and OTP validation. Once the OTP is verified, the workflow consolidates the verified channel into a single global user identity stored in Postgres.

Typical use cases include:

- verifying a user across messaging channels before onboarding,
- linking multiple channels to a single account,
- handling first-contact onboarding for messaging users,
- validating ownership of an email before associating it with a channel identity.

The workflow is logically organized into the following blocks:

## 1.1 Input Reception and Channel Normalization
Receives a POST webhook payload, detects whether the source is Telegram, WhatsApp, or email, and maps all inputs to a common format: `channel`, `platform_user_id`, and `message`.

## 1.2 Existing Channel Validation
Checks whether the incoming channel/user pair already exists in `user_channels`. If yes, it fetches the associated user and immediately returns a “channel already verified” response path.

## 1.3 Session Initialization and Session Lookup
If the channel is not yet linked, the workflow creates an onboarding session if needed and retrieves the current onboarding state from `onboarding_sessions`.

## 1.4 Email Capture and Validation
If the session is waiting for an email, the workflow extracts an email address from the incoming message using regex. If no email is found, it prepares a prompt asking the user to provide one.

## 1.5 OTP Generation, Storage, and Email Delivery
Once an email is extracted, the workflow updates the session, generates a 6-digit OTP, stores it in `user_otps`, sends it by email, and moves the onboarding step to `waiting_for_otp`.

## 1.6 OTP Verification
If the session is already in `waiting_for_otp`, the workflow determines whether the incoming message looks like an OTP code, validates it against non-expired unused records in `user_otps`, and handles invalid-code responses.

## 1.7 Identity Consolidation and Completion
When the OTP is valid, the workflow marks it as used, marks the session as verified, upserts a verified user record by email, links the current channel in `user_channels`, and returns a success response.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Input Reception and Channel Normalization

### Overview
This block receives inbound events through a webhook and converts channel-specific payloads into a universal internal structure. It is the mandatory entry point for every execution and determines the rest of the branching logic.

### Nodes Involved
- `Webhook`
- `Universal normalization`

### Node Details

#### Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; external HTTP entry point.
- **Configuration choices:**  
  - Method: `POST`
  - Path: `8e947f01-96d1-47ed-bc4d-e28bcc71821a`
- **Key expressions or variables used:** None in configuration.
- **Input and output connections:**  
  - Input: none
  - Output: `Universal normalization`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Incorrect HTTP method
  - Payload shape not matching any expected channel format
  - Upstream source not configured to send JSON body
- **Sub-workflow reference:** None

#### Universal normalization
- **Type and technical role:** `n8n-nodes-base.code`; transforms incoming payloads to a common schema.
- **Configuration choices:**  
  - Reads `const data = $json.body || {};`
  - Supports:
    - Telegram-like payloads via `data.message`
    - WhatsApp Meta Cloud API payloads via `data.entry`
    - simple email payloads via `data.from && data.subject`
  - Returns:
    - `channel`
    - `platform_user_id`
    - `message`
- **Key expressions or variables used:**  
  - `data.message.from?.id?.toString()`
  - `data.entry[0].changes[0].value.messages[0].from`
  - `data.entry[0].changes[0].value.messages[0].text?.body`
  - `data.text || data.body`
- **Input and output connections:**  
  - Input: `Webhook`
  - Output: `Check if the channel already exists`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - WhatsApp payload missing nested `messages[0]` path may throw runtime errors
  - Email payload may not include `text` or `body`
  - Unknown payloads fall back to:
    - `channel = "unknown"`
    - `platform_user_id = "unknown"`
    - `message = JSON.stringify(data)`
- **Sub-workflow reference:** None

---

## 2.2 Block: Existing Channel Validation

### Overview
This block checks whether the incoming channel identity is already associated with a verified user. If it is, the workflow skips onboarding and prepares a continuation response.

### Nodes Involved
- `Check if the channel already exists`
- `If`
- `Execute a SQL query3`
- `Channel already verified`

### Node Details

#### Check if the channel already exists
- **Type and technical role:** `n8n-nodes-base.postgres`; lookup in `user_channels`.
- **Configuration choices:**  
  - Query:
    - selects `global_user_id`
    - filters by `channel` and `platform_user_id`
  - `alwaysOutputData: true`, so execution continues even if no row is returned.
- **Key expressions or variables used:**  
  - `={{ $json.channel }} {{ $json.platform_user_id }}`
- **Input and output connections:**  
  - Input: `Universal normalization`
  - Output: `If`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - The query replacement format appears to rely on whitespace-separated values rather than an explicit array; this can be fragile depending on the Postgres node parsing behavior
  - Database connectivity, credential, or schema errors
  - Missing unique index or table mismatch
- **Sub-workflow reference:** None

#### If
- **Type and technical role:** `n8n-nodes-base.if`; determines whether the query returned any fields.
- **Configuration choices:**  
  - Condition checks:
    - `Object.keys($json).length > 0`
- **Key expressions or variables used:**  
  - `={{ Object.keys($json).length }}`
- **Input and output connections:**  
  - Input: `Check if the channel already exists`
  - True: `Execute a SQL query3`
  - False: `Execute a SQL query`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Because `alwaysOutputData` is enabled upstream, this logic assumes empty result objects for “not found”; behavior depends on actual output shape from the Postgres node
- **Sub-workflow reference:** None

#### Execute a SQL query3
- **Type and technical role:** `n8n-nodes-base.postgres`; fetches the verified user's details.
- **Configuration choices:**  
  - Query:
    - `SELECT global_user_id, email FROM users WHERE global_user_id = $1 LIMIT 1;`
  - Uses the `global_user_id` from the previous lookup.
- **Key expressions or variables used:**  
  - `={{ $json.global_user_id }}`
- **Input and output connections:**  
  - Input: `If` true branch
  - Output: `Channel already verified`
- **Version-specific requirements:** Type version `2.6`, `alwaysOutputData: true`.
- **Edge cases or potential failure types:**  
  - User record missing even though `user_channels` entry exists
  - Database auth or SQL errors
- **Sub-workflow reference:** None

#### Channel already verified
- **Type and technical role:** `n8n-nodes-base.set`; creates a response message.
- **Configuration choices:**  
  - Sets `response_message` to:
    - `✅ Channel already verified for ${$json.email}. Continuamos con tu perfil.`
  - Keeps other fields.
- **Key expressions or variables used:**  
  - Template string using `$json.email`
- **Input and output connections:**  
  - Input: `Execute a SQL query3`
  - Output: none
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If the user record has no email, the message may render with `undefined`
- **Sub-workflow reference:** None

---

## 2.3 Block: Session Initialization and Session Lookup

### Overview
If the channel is not already linked, this block creates an onboarding session if one does not exist and then retrieves the session state. That state controls whether the workflow asks for an email or expects an OTP.

### Nodes Involved
- `Execute a SQL query`
- `Search onboarding_session`
- `If1`

### Node Details

#### Execute a SQL query
- **Type and technical role:** `n8n-nodes-base.postgres`; inserts a session row if absent.
- **Configuration choices:**  
  - Inserts into `onboarding_sessions`
  - Default step: `waiting_for_email`
  - Uses `ON CONFLICT (channel, platform_user_id) DO NOTHING`
- **Key expressions or variables used:**  
  - `={{ [ $json.channel ?? $node["Universal normalization"].json.channel, $json.platform_user_id ?? $node["Universal normalization"].json.platform_user_id ] }}`
- **Input and output connections:**  
  - Input: `If` false branch
  - Output: `Search onboarding_session`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Requires a unique constraint on `(channel, platform_user_id)`
  - Insert may silently do nothing if session already exists, which is expected
- **Sub-workflow reference:** None

#### Search onboarding_session
- **Type and technical role:** `n8n-nodes-base.postgres`; fetches current onboarding step and stored email.
- **Configuration choices:**  
  - Query:
    - `SELECT step, email FROM onboarding_sessions WHERE channel = $1 AND platform_user_id = $2;`
- **Key expressions or variables used:**  
  - `={{ [ $node["Universal normalization"].json.channel, $node["Universal normalization"].json.platform_user_id ] }}`
- **Input and output connections:**  
  - Input: `Execute a SQL query`
  - Output: `If1`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - If the previous insert failed unexpectedly, this may return no row
  - Step values outside the expected set can break branching assumptions
- **Sub-workflow reference:** None

#### If1
- **Type and technical role:** `n8n-nodes-base.if`; routes based on session step.
- **Configuration choices:**  
  - True when `step === "waiting_for_email"`
  - False branch is effectively the “not waiting_for_email” path, used for OTP handling
- **Key expressions or variables used:**  
  - `={{ $json.step }}`
- **Input and output connections:**  
  - Input: `Search onboarding_session`
  - True: `Extract email`
  - False: `IF_step- waiting_otp`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - If `step` is null or absent, execution goes to false branch and may not behave as intended
- **Sub-workflow reference:** None

---

## 2.4 Block: Email Capture and Validation

### Overview
When the session is waiting for an email, this block attempts to extract one from the user's message. If extraction fails, it prepares a prompt asking the user to provide their email address.

### Nodes Involved
- `Extract email`
- `If4`
- `Greeting 1 - send your email`

### Node Details

#### Extract email
- **Type and technical role:** `n8n-nodes-base.code`; regex-based email extraction.
- **Configuration choices:**  
  - Reads the normalized message from `Universal normalization`
  - Uses a case-insensitive regex to detect an email
  - Returns `extracted_email` in lowercase or `null`
- **Key expressions or variables used:**  
  - `$node["Universal normalization"].json.message`
  - regex: `/[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}/i`
- **Input and output connections:**  
  - Input: `If1` true branch
  - Output: `If4`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Regex is robust for common addresses but not exhaustive for every RFC-valid email
  - Multiple emails in a message: only the first match is used
- **Sub-workflow reference:** None

#### If4
- **Type and technical role:** `n8n-nodes-base.if`; checks if extraction succeeded.
- **Configuration choices:**  
  - True if `extracted_email` is not empty
- **Key expressions or variables used:**  
  - `={{ $json.extracted_email }}`
- **Input and output connections:**  
  - Input: `Extract email`
  - True: `If2- extracted_email`
  - False: `Greeting 1 - send your email`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - `null` and empty string are both treated as missing email
- **Sub-workflow reference:** None

#### Greeting 1 - send your email
- **Type and technical role:** `n8n-nodes-base.set`; prepares a user-facing prompt.
- **Configuration choices:**  
  - Sets `response_message` to:
    - `👋 Welcome to Zyntha ☕ To continue, I need your email address to create or verify your profile.`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: `If4` false branch
  - Output: none
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** None significant
- **Sub-workflow reference:** None

---

## 2.5 Block: OTP Generation, Storage, and Email Delivery

### Overview
This block receives a valid extracted email, stores it in the onboarding session, generates a time-limited OTP, sends it by email, updates the session state, and prepares a confirmation response.

### Nodes Involved
- `If2- extracted_email`
- `Execute a SQL query5`
- `Generate OTP`
- `Save OTP`
- `Send email`
- `waiting_for_otp`
- `If2`
- `response_message1 - verification code sent`

### Node Details

#### If2- extracted_email
- **Type and technical role:** `n8n-nodes-base.if`; secondary non-empty check before proceeding.
- **Configuration choices:**  
  - Checks `$('Extract email').item.json.extracted_email` is not empty
- **Key expressions or variables used:**  
  - `={{ $('Extract email').item.json.extracted_email }}`
- **Input and output connections:**  
  - Input: `If4` true branch
  - True: `Execute a SQL query5`
  - False: none
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Redundant with `If4`, but harmless
  - Expression depends on linked item context
- **Sub-workflow reference:** None

#### Execute a SQL query5
- **Type and technical role:** `n8n-nodes-base.postgres`; stores email in session and switches step.
- **Configuration choices:**  
  - Updates `onboarding_sessions`
  - Sets:
    - `email = $1`
    - `step = 'waiting_for_otp'`
- **Key expressions or variables used:**  
  - `={{ [ $json.extracted_email, $node["Universal normalization"].json.channel, $node["Universal normalization"].json.platform_user_id ] }}`
- **Input and output connections:**  
  - Input: `If2- extracted_email`
  - Output: `Generate OTP`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - If no session row exists, update affects zero rows
  - Requires the prior session initialization to have succeeded
- **Sub-workflow reference:** None

#### Generate OTP
- **Type and technical role:** `n8n-nodes-base.code`; generates one-time code and expiration.
- **Configuration choices:**  
  - Generates a 6-digit numeric code using `Math.random`
  - Sets expiration to current time + 5 minutes
- **Key expressions or variables used:**  
  - `otp_code`
  - `otp_expires_at`
- **Input and output connections:**  
  - Input: `Execute a SQL query5`
  - Output: `Save OTP`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - `Math.random` is sufficient for many onboarding scenarios but not cryptographically secure
  - Simultaneous requests can generate multiple valid OTPs unless controlled in the database or business logic
- **Sub-workflow reference:** None

#### Save OTP
- **Type and technical role:** `n8n-nodes-base.postgres`; persists the generated OTP.
- **Configuration choices:**  
  - Inserts into `user_otps`
  - Stores:
    - `channel`
    - `platform_user_id`
    - `otp_code`
    - `expires_at`
- **Key expressions or variables used:**  
  - `{{ $node["Universal normalization"].json.channel }}`
  - `{{ $node["Universal normalization"].json.platform_user_id }}`
  - `{{ $json.otp_code }}`
  - `{{ $json.otp_expires_at }}`
- **Input and output connections:**  
  - Input: `Generate OTP`
  - Output: `Send email`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Query replacement is multiline rather than a single explicit array; may need normalization depending on n8n/Postgres behavior
  - Table schema must support `is_used` defaulting to false
- **Sub-workflow reference:** None

#### Send email
- **Type and technical role:** `n8n-nodes-base.emailSend`; sends the OTP by SMTP.
- **Configuration choices:**  
  - To: extracted email
  - From: `user@example.com`
  - Subject: `Tu código de verificación`
  - HTML body includes the OTP and 5-minute expiry notice in English and Spanish
- **Key expressions or variables used:**  
  - `{{ $('Generate OTP').item.json.otp_code }}`
  - `={{ $('Extract email').item.json.extracted_email }}`
- **Input and output connections:**  
  - Input: `Save OTP`
  - Output: `waiting_for_otp`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**  
  - SMTP auth failure
  - Sender address rejected
  - Email delivered to spam
  - Missing/invalid email address
- **Sub-workflow reference:** None

#### waiting_for_otp
- **Type and technical role:** `n8n-nodes-base.postgres`; confirms session step is `waiting_for_otp`.
- **Configuration choices:**  
  - Updates `onboarding_sessions`
  - Sets `step = 'waiting_for_otp'`
- **Key expressions or variables used:**  
  - `={{ $node["Universal normalization"].json.channel }}`
  - `{{ $node["Universal normalization"].json.platform_user_id }}`
- **Input and output connections:**  
  - Input: `Send email`
  - Output: `If2`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Query replacement formatting may need to be converted to explicit array syntax
- **Sub-workflow reference:** None

#### If2
- **Type and technical role:** `n8n-nodes-base.if`; checks that the step is now `waiting_for_otp`.
- **Configuration choices:**  
  - Condition: `step === "waiting_for_otp"`
  - In this workflow, the false branch leads to the confirmation message
- **Key expressions or variables used:**  
  - `={{ $json.step }}`
- **Input and output connections:**  
  - Input: `waiting_for_otp`
  - True: no connection
  - False: `response_message1 - verification code sent`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - The branching is counterintuitive: the response is connected to the false output
  - Since the update query output may not include `step`, this can route unexpectedly
- **Sub-workflow reference:** None

#### response_message1 - verification code sent
- **Type and technical role:** `n8n-nodes-base.set`; prepares OTP sent response.
- **Configuration choices:**  
  - Sets `response_message` to:
    - `I sent you a verification code. Enter it here to continue.`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: `If2` false branch
  - Output: none
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** None significant
- **Sub-workflow reference:** None

---

## 2.6 Block: OTP Verification

### Overview
If the session is already waiting for an OTP, this block decides whether the incoming message is likely a code and validates it against the database. Invalid or expired codes lead to a failure response.

### Nodes Involved
- `IF_step- waiting_otp`
- `IF_resend_otp`
- `Execute a SQL query1`
- `IF_code_exists?`
- `code does not exist`

### Node Details

#### IF_step- waiting_otp
- **Type and technical role:** `n8n-nodes-base.if`; routes only sessions currently expecting an OTP.
- **Configuration choices:**  
  - True when `step === "waiting_for_otp"`
- **Key expressions or variables used:**  
  - `={{ $json.step }}`
- **Input and output connections:**  
  - Input: `If1` false branch
  - True: `IF_resend_otp`
  - False: none
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - If session step is `verified` or anything unexpected, the flow ends silently
- **Sub-workflow reference:** None

#### IF_resend_otp
- **Type and technical role:** `n8n-nodes-base.if`; distinguishes numeric OTP-looking messages from other content.
- **Configuration choices:**  
  - Uses regex `/^\d{4,6}$/`
  - True if message is a 4-to-6 digit code
  - In this workflow, the validation query is connected to the false branch
- **Key expressions or variables used:**  
  - `={{ /^\d{4,6}$/.test(($json.message || '').trim()) }}`
- **Input and output connections:**  
  - Input: `IF_step- waiting_otp`
  - True: none
  - False: `Execute a SQL query1`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Branching appears inverted: OTP validation is executed on the false output, not the true one
  - This likely requires correction for production use
- **Sub-workflow reference:** None

#### Execute a SQL query1
- **Type and technical role:** `n8n-nodes-base.postgres`; validates the provided OTP.
- **Configuration choices:**  
  - Selects from `user_otps`
  - Filters by:
    - `channel`
    - `platform_user_id`
    - `otp_code`
    - `is_used = false`
    - `expires_at > now()`
  - Orders newest first and limits to one row
  - `alwaysOutputData: true`
- **Key expressions or variables used:**  
  - `={{ [ $('Universal normalization').item.json.channel, $('Universal normalization').item.json.platform_user_id, (($('Universal normalization').item.json.message || '').trim()) ] }}`
- **Input and output connections:**  
  - Input: `IF_resend_otp` false branch
  - Output: `IF_code_exists?`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Expired or already used OTP returns no valid result
  - Multiple simultaneous OTPs: only the latest matching unused valid one is selected
- **Sub-workflow reference:** None

#### IF_code_exists?
- **Type and technical role:** `n8n-nodes-base.if`; checks whether a valid OTP row was found.
- **Configuration choices:**  
  - True when `id` is not empty
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
- **Input and output connections:**  
  - Input: `Execute a SQL query1`
  - True: `valid OTP`
  - False: `code does not exist`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Assumes result rows include `id`
  - If query output shape changes, this test may fail
- **Sub-workflow reference:** None

#### code does not exist
- **Type and technical role:** `n8n-nodes-base.set`; invalid OTP response.
- **Configuration choices:**  
  - Sets `response_message` to:
    - `❌ Invalid code. Please try again with the correct code.`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: `IF_code_exists?` false branch
  - Output: none
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** None significant
- **Sub-workflow reference:** None

---

## 2.7 Block: Identity Consolidation and Completion

### Overview
Once an OTP is validated, this block marks it as used, marks the session as verified, ensures a verified user exists for the captured email, links the incoming channel to that user, and prepares a completion response.

### Nodes Involved
- `valid OTP`
- `Execute a SQL query2`
- `Get verified session email`
- `If email en session`
- `Missing email response`
- `Upsert user by email`
- `Upsert user_channel`
- `Verification completed response`

### Node Details

#### valid OTP
- **Type and technical role:** `n8n-nodes-base.postgres`; marks the OTP as consumed.
- **Configuration choices:**  
  - Updates `user_otps`
  - Sets `is_used = true`
  - Filters by the ID found in `Execute a SQL query1`
- **Key expressions or variables used:**  
  - `={{ $('Execute a SQL query1').item.json.id }}`
- **Input and output connections:**  
  - Input: `IF_code_exists?` true branch
  - Output: `Execute a SQL query2`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - If OTP row has already been marked used by another concurrent execution, update may affect zero rows
- **Sub-workflow reference:** None

#### Execute a SQL query2
- **Type and technical role:** `n8n-nodes-base.postgres`; marks onboarding session as verified.
- **Configuration choices:**  
  - Updates `onboarding_sessions`
  - Sets `step = 'verified'`
- **Key expressions or variables used:**  
  - `={{ [ $('Universal normalization').item.json.channel, $('Universal normalization').item.json.platform_user_id ] }}`
- **Input and output connections:**  
  - Input: `valid OTP`
  - Output: `Get verified session email`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - No matching session row means no state update
- **Sub-workflow reference:** None

#### Get verified session email
- **Type and technical role:** `n8n-nodes-base.postgres`; retrieves session email after verification.
- **Configuration choices:**  
  - `SELECT email FROM onboarding_sessions WHERE channel = $1 AND platform_user_id = $2 LIMIT 1`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**  
  - `={{ [ $('Universal normalization').item.json.channel, $('Universal normalization').item.json.platform_user_id ] }}`
- **Input and output connections:**  
  - Input: `Execute a SQL query2`
  - Output: `If email en session`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Session may be verified but missing stored email
- **Sub-workflow reference:** None

#### If email en session
- **Type and technical role:** `n8n-nodes-base.if`; verifies email presence before creating/linking user.
- **Configuration choices:**  
  - True when `email` is not empty
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
- **Input and output connections:**  
  - Input: `Get verified session email`
  - True: `Upsert user by email`
  - False: `Missing email response`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Missing email prevents identity consolidation
- **Sub-workflow reference:** None

#### Missing email response
- **Type and technical role:** `n8n-nodes-base.set`; fallback response if session email cannot be found.
- **Configuration choices:**  
  - Sets `response_message` to:
    - `⚠️ I couldn’t find the session email. Please send your email to continue.`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: `If email en session` false branch
  - Output: none
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** None significant
- **Sub-workflow reference:** None

#### Upsert user by email
- **Type and technical role:** `n8n-nodes-base.postgres`; creates or verifies a canonical user record.
- **Configuration choices:**  
  - Inserts into `users (email, verified)`
  - On conflict by `email`, updates:
    - `verified = true`
    - `updated_at = now()`
  - Returns `global_user_id, email`
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
- **Input and output connections:**  
  - Input: `If email en session` true branch
  - Output: `Upsert user_channel`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Requires unique constraint on `users.email`
- **Sub-workflow reference:** None

#### Upsert user_channel
- **Type and technical role:** `n8n-nodes-base.postgres`; links the verified channel to the canonical user.
- **Configuration choices:**  
  - Inserts `(global_user_id, channel, platform_user_id)`
  - On conflict `(channel, platform_user_id)` updates `global_user_id`
  - Returns linked channel data
- **Key expressions or variables used:**  
  - `={{ [ $json.global_user_id, $('Universal normalization').item.json.channel, $('Universal normalization').item.json.platform_user_id ] }}`
- **Input and output connections:**  
  - Input: `Upsert user by email`
  - Output: `Verification completed response`
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Requires unique constraint on `(channel, platform_user_id)`
- **Sub-workflow reference:** None

#### Verification completed response
- **Type and technical role:** `n8n-nodes-base.set`; final success response.
- **Configuration choices:**  
  - Sets `response_message` to:
    - `✅ Verification completed. This channel has been linked to your user.`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: `Upsert user_channel`
  - Output: none
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** None significant
- **Sub-workflow reference:** None

---

## 2.8 Documentation and Annotation Nodes

### Overview
These nodes are visual documentation embedded in the canvas. They do not execute business logic but are important for context and reconstruction.

### Nodes Involved
- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`
- `Sticky Note4`
- `Sticky Note5`
- `Sticky Note6`
- `Sticky Note7`
- `Sticky Note8`
- `Sticky Note9`
- `Sticky Note10`
- `Sticky Note11`

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas annotation.
- **Configuration choices:** Content: `## the user exists`
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note1
- **Type and technical role:** sticky note.
- **Configuration choices:** Content: `## the user does not exist`
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note2
- **Type and technical role:** sticky note.
- **Configuration choices:** General workflow explanation and setup instructions.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note3
- **Type and technical role:** sticky note.
- **Configuration choices:** Section 1 annotation about channel detection and normalization.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note4
- **Type and technical role:** sticky note.
- **Configuration choices:** Section 2 annotation about existing channel validation.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note5
- **Type and technical role:** sticky note.
- **Configuration choices:** Section 3 annotation about session initialization.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note6
- **Type and technical role:** sticky note.
- **Configuration choices:** Section 4 annotation about email extraction and validation.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note7
- **Type and technical role:** sticky note.
- **Configuration choices:** Section 5 annotation about OTP generation and delivery.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note8
- **Type and technical role:** sticky note.
- **Configuration choices:** Annotation for OTP/code validation area, though its title still says Section 4.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note9
- **Type and technical role:** sticky note.
- **Configuration choices:** Section 7 annotation about identity consolidation.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note10
- **Type and technical role:** sticky note.
- **Configuration choices:** Database prerequisite warning.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

#### Sticky Note11
- **Type and technical role:** sticky note.
- **Configuration choices:** Security notice about OTP logging and expiration.
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | webhook | Receives POST requests from channels |  | Universal normalization | ## Section 1 — Channel Detection & Normalization<br>Receives incoming requests from multiple platforms and identifies the communication channel.<br>Normalizes user data into a unified structure before checking if the channel is already linked. |
| Universal normalization | code | Detects source channel and maps payload to common fields | Webhook | Check if the channel already exists | ## Section 1 — Channel Detection & Normalization<br>Receives incoming requests from multiple platforms and identifies the communication channel.<br>Normalizes user data into a unified structure before checking if the channel is already linked. |
| Check if the channel already exists | postgres | Looks up existing channel linkage in `user_channels` | Universal normalization | If | ## Section 1 — Channel Detection & Normalization<br>Receives incoming requests from multiple platforms and identifies the communication channel.<br>Normalizes user data into a unified structure before checking if the channel is already linked.<br>## Section 2 — Existing Channel Validation<br>- Checks if the incoming channel is already associated with a verified user.<br>- If found, skips the verification process and continues with the existing profile. |
| If | if | Branches on whether channel linkage exists | Check if the channel already exists | Execute a SQL query3; Execute a SQL query | ## the user exists<br>## the user does not exist |
| Execute a SQL query3 | postgres | Fetches user details for an already linked channel | If | Channel already verified | ## the user exists<br>## Section 2 — Existing Channel Validation<br>- Checks if the incoming channel is already associated with a verified user.<br>- If found, skips the verification process and continues with the existing profile. |
| Channel already verified | set | Prepares response for already verified channel | Execute a SQL query3 |  | ## the user exists<br>## Section 2 — Existing Channel Validation<br>- Checks if the incoming channel is already associated with a verified user.<br>- If found, skips the verification process and continues with the existing profile. |
| Execute a SQL query | postgres | Inserts onboarding session if absent | If | Search onboarding_session | ## the user does not exist<br>## Section 3 — Session Initialization<br>- Creates or retrieves the onboarding session for the current channel.<br>- Tracks the current step of the verification process. |
| Search onboarding_session | postgres | Retrieves current onboarding step and email | Execute a SQL query | If1 | ## the user does not exist<br>## Section 3 — Session Initialization<br>- Creates or retrieves the onboarding session for the current channel.<br>- Tracks the current step of the verification process. |
| If1 | if | Routes by onboarding session step | Search onboarding_session | Extract email; IF_step- waiting_otp | ## the user does not exist |
| Extract email | code | Extracts email from incoming message | If1 | If4 | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing. |
| If4 | if | Checks whether an email was extracted | Extract email | If2- extracted_email; Greeting 1 - send your email | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing. |
| Greeting 1 - send your email | set | Prepares prompt asking the user for email | If4 |  | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing. |
| If2- extracted_email | if | Confirms extracted email before OTP flow | If4 | Execute a SQL query5 | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing.<br>## Section 5 — OTP Generation & Delivery<br>- Generates a secure one-time password and stores it with an expiration timestamp.<br>- Sends the OTP via email and updates the session to waiting_for_otp.<br>- A confirmation message is sent to the email used for delivering the OTP. |
| Execute a SQL query5 | postgres | Stores email in session and changes step to `waiting_for_otp` | If2- extracted_email | Generate OTP | ## Section 5 — OTP Generation & Delivery<br>- Generates a secure one-time password and stores it with an expiration timestamp.<br>- Sends the OTP via email and updates the session to waiting_for_otp.<br>- A confirmation message is sent to the email used for delivering the OTP. |
| Generate OTP | code | Generates 6-digit OTP and expiration timestamp | Execute a SQL query5 | Save OTP | ## Section 5 — OTP Generation & Delivery<br>- Generates a secure one-time password and stores it with an expiration timestamp.<br>- Sends the OTP via email and updates the session to waiting_for_otp.<br>- A confirmation message is sent to the email used for delivering the OTP.<br>## Security Notice<br>- Do not log OTP codes in production environments.<br>- Always define an expiration time and mark codes as used after validation. |
| Save OTP | postgres | Saves OTP in database | Generate OTP | Send email | ## Section 5 — OTP Generation & Delivery<br>- Generates a secure one-time password and stores it with an expiration timestamp.<br>- Sends the OTP via email and updates the session to waiting_for_otp.<br>- A confirmation message is sent to the email used for delivering the OTP. |
| Send email | emailSend | Sends OTP email to extracted email address | Save OTP | waiting_for_otp | ## Section 5 — OTP Generation & Delivery<br>- Generates a secure one-time password and stores it with an expiration timestamp.<br>- Sends the OTP via email and updates the session to waiting_for_otp.<br>- A confirmation message is sent to the email used for delivering the OTP. |
| waiting_for_otp | postgres | Updates session step to `waiting_for_otp` | Send email | If2 | ## Section 5 — OTP Generation & Delivery<br>- Generates a secure one-time password and stores it with an expiration timestamp.<br>- Sends the OTP via email and updates the session to waiting_for_otp.<br>- A confirmation message is sent to the email used for delivering the OTP. |
| If2 | if | Checks session step after OTP dispatch | waiting_for_otp | response_message1 - verification code sent | ## Section 5 — OTP Generation & Delivery<br>- Generates a secure one-time password and stores it with an expiration timestamp.<br>- Sends the OTP via email and updates the session to waiting_for_otp.<br>- A confirmation message is sent to the email used for delivering the OTP. |
| response_message1 - verification code sent | set | Prepares confirmation that OTP was sent | If2 |  | ## Section 5 — OTP Generation & Delivery<br>- Generates a secure one-time password and stores it with an expiration timestamp.<br>- Sends the OTP via email and updates the session to waiting_for_otp.<br>- A confirmation message is sent to the email used for delivering the OTP. |
| IF_step- waiting_otp | if | Routes sessions currently waiting for OTP | If1 | IF_resend_otp | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing.<br>- A message is sent to the user if the code does not exist, asking them to verify it and send it again. |
| IF_resend_otp | if | Detects whether incoming message looks like a numeric OTP | IF_step- waiting_otp | Execute a SQL query1 | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing.<br>- A message is sent to the user if the code does not exist, asking them to verify it and send it again. |
| Execute a SQL query1 | postgres | Validates OTP against unused, non-expired records | IF_resend_otp | IF_code_exists? | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing.<br>- A message is sent to the user if the code does not exist, asking them to verify it and send it again.<br>## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| IF_code_exists? | if | Checks whether OTP validation returned a row | Execute a SQL query1 | valid OTP; code does not exist | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing.<br>- A message is sent to the user if the code does not exist, asking them to verify it and send it again. |
| code does not exist | set | Prepares invalid-code response | IF_code_exists? |  | ## Section 4 — Email Extraction & Validation<br>- Extracts a valid email from the user message using regex detection.<br>- If no email is found, prompts the user to provide one before continuing.<br>- A message is sent to the user if the code does not exist, asking them to verify it and send it again. |
| valid OTP | postgres | Marks OTP as used | IF_code_exists? | Execute a SQL query2 | ## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| Execute a SQL query2 | postgres | Marks onboarding session as verified | valid OTP | Get verified session email | ## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| Get verified session email | postgres | Retrieves session email for verified session | Execute a SQL query2 | If email en session | ## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| If email en session | if | Confirms verified session has an email | Get verified session email | Upsert user by email; Missing email response | ## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| Missing email response | set | Prepares fallback when session email is missing | If email en session |  | ## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| Upsert user by email | postgres | Creates or verifies canonical user by email | If email en session | Upsert user_channel | ## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| Upsert user_channel | postgres | Links current channel to global user | Upsert user by email | Verification completed response | ## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| Verification completed response | set | Prepares final success message | Upsert user_channel |  | ## Section 7 — Identity Consolidation<br>- Marks the OTP as used and updates the session as verified in the Supabase database.<br>- Links the verified channel to a single global user ID to ensure omnichannel continuity.<br>- A confirmation message is sent via email if the verification has been completed and successfully linked to the user. |
| Sticky Note | stickyNote | Canvas annotation |  |  |  |
| Sticky Note1 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note2 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note3 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note4 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note5 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note6 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note7 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note8 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note9 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note10 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note11 | stickyNote | Canvas annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a manual rebuild sequence for n8n.

## 4.1 Prepare the database first
1. Create a Postgres database or use Supabase Postgres.
2. Create at minimum these tables:
   - `users`
   - `user_channels`
   - `user_otps`
   - `onboarding_sessions`
3. Ensure these constraints exist:
   - `users.email` unique
   - `user_channels(channel, platform_user_id)` unique
   - `onboarding_sessions(channel, platform_user_id)` unique
4. Ensure `user_otps` includes at least:
   - `id`
   - `channel`
   - `platform_user_id`
   - `otp_code`
   - `expires_at`
   - `is_used` default `false`
   - `created_at`
5. Suggested fields:
   - `users(global_user_id uuid or bigserial primary key, email text unique, verified boolean, updated_at timestamp)`
   - `user_channels(global_user_id, channel text, platform_user_id text, unique(channel, platform_user_id))`
   - `onboarding_sessions(channel text, platform_user_id text, email text, step text, unique(channel, platform_user_id))`

## 4.2 Create credentials
6. In n8n, create a **Postgres** credential for your database.
7. Create an **SMTP** credential for email sending.
8. If you expose the webhook to Telegram/WhatsApp/email ingestion systems, configure those systems separately to call the n8n webhook URL.

## 4.3 Create the entry and normalization nodes
9. Add a **Webhook** node.
   - Name: `Webhook`
   - Method: `POST`
   - Path: `8e947f01-96d1-47ed-bc4d-e28bcc71821a`
10. Add a **Code** node.
    - Name: `Universal normalization`
    - Connect `Webhook -> Universal normalization`
    - Paste this logic conceptually:
      - Read `body`
      - If Telegram-like structure exists, set:
        - `channel = telegram`
        - `platform_user_id = message.from.id as string`
        - `message = message.text`
      - Else if WhatsApp Cloud API structure exists:
        - `channel = whatsapp`
        - `platform_user_id = messages[0].from`
        - `message = messages[0].text.body`
      - Else if email-like payload exists:
        - `channel = email`
        - `platform_user_id = from`
        - `message = text or body`
      - Else use fallback `unknown`
    - Return one item with:
      - `channel`
      - `platform_user_id`
      - `message`

## 4.4 Add existing-channel lookup
11. Add a **Postgres** node.
    - Name: `Check if the channel already exists`
    - Connect `Universal normalization -> Check if the channel already exists`
    - Operation: Execute Query
    - Query:
      ```sql
      SELECT global_user_id
      FROM user_channels
      WHERE channel = $1
      AND platform_user_id = $2;
      ```
    - Query replacements: use channel and platform_user_id from normalized data
    - Enable `Always Output Data`
12. Add an **If** node.
    - Name: `If`
    - Connect `Check if the channel already exists -> If`
    - Condition:
      - left expression: `Object.keys($json).length`
      - operation: greater than
      - right value: `0`
13. Add a **Postgres** node.
    - Name: `Execute a SQL query3`
    - Connect `If true -> Execute a SQL query3`
    - Query:
      ```sql
      SELECT global_user_id, email
      FROM users
      WHERE global_user_id = $1
      LIMIT 1;
      ```
    - Replacement: `global_user_id`
    - Enable `Always Output Data`
14. Add a **Set** node.
    - Name: `Channel already verified`
    - Connect `Execute a SQL query3 -> Channel already verified`
    - Add field:
      - `response_message` =
        `✅ Channel already verified for {{ $json.email }}. Continuamos con tu perfil.`

## 4.5 Add onboarding session initialization
15. Add a **Postgres** node.
    - Name: `Execute a SQL query`
    - Connect `If false -> Execute a SQL query`
    - Query:
      ```sql
      INSERT INTO onboarding_sessions (channel, platform_user_id, step)
      VALUES ($1, $2, 'waiting_for_email')
      ON CONFLICT (channel, platform_user_id)
      DO NOTHING;
      ```
    - Replacements:
      - normalized `channel`
      - normalized `platform_user_id`
16. Add a **Postgres** node.
    - Name: `Search onboarding_session`
    - Connect `Execute a SQL query -> Search onboarding_session`
    - Query:
      ```sql
      SELECT step, email
      FROM onboarding_sessions
      WHERE channel = $1
      AND platform_user_id = $2;
      ```
17. Add an **If** node.
    - Name: `If1`
    - Connect `Search onboarding_session -> If1`
    - Condition:
      - `step` equals `waiting_for_email`

## 4.6 Build the email-capture branch
18. Add a **Code** node.
    - Name: `Extract email`
    - Connect `If1 true -> Extract email`
    - Logic:
      - Read normalized `message`
      - Match email with regex
      - Return `extracted_email` lowercase or `null`
19. Add an **If** node.
    - Name: `If4`
    - Connect `Extract email -> If4`
    - Condition:
      - `extracted_email` is not empty
20. Add a **Set** node.
    - Name: `Greeting 1 - send your email`
    - Connect `If4 false -> Greeting 1 - send your email`
    - Set:
      - `response_message = 👋 Welcome to Zyntha ☕ To continue, I need your email address to create or verify your profile.`
21. Add another **If** node.
    - Name: `If2- extracted_email`
    - Connect `If4 true -> If2- extracted_email`
    - Condition:
      - extracted email is not empty

## 4.7 Build the OTP generation and dispatch branch
22. Add a **Postgres** node.
    - Name: `Execute a SQL query5`
    - Connect `If2- extracted_email true -> Execute a SQL query5`
    - Query:
      ```sql
      UPDATE onboarding_sessions
      SET email = $1,
          step = 'waiting_for_otp'
      WHERE channel = $2
      AND platform_user_id = $3;
      ```
23. Add a **Code** node.
    - Name: `Generate OTP`
    - Connect `Execute a SQL query5 -> Generate OTP`
    - Logic:
      - generate 6-digit string
      - set expiry to now + 5 minutes
      - return `otp_code`, `otp_expires_at`
24. Add a **Postgres** node.
    - Name: `Save OTP`
    - Connect `Generate OTP -> Save OTP`
    - Query:
      ```sql
      INSERT INTO user_otps (channel, platform_user_id, otp_code, expires_at)
      VALUES ($1, $2, $3, $4);
      ```
25. Add an **Email Send** node.
    - Name: `Send email`
    - Connect `Save OTP -> Send email`
    - Configure SMTP credential
    - To: extracted email
    - From: `user@example.com` or your verified sender
    - Subject: `Tu código de verificación`
    - HTML body should include the OTP and expiration message
26. Add a **Postgres** node.
    - Name: `waiting_for_otp`
    - Connect `Send email -> waiting_for_otp`
    - Query:
      ```sql
      UPDATE onboarding_sessions
      SET step = 'waiting_for_otp'
      WHERE channel = $1
      AND platform_user_id = $2;
      ```
27. Add an **If** node.
    - Name: `If2`
    - Connect `waiting_for_otp -> If2`
    - Condition:
      - `step` equals `waiting_for_otp`
28. Add a **Set** node.
    - Name: `response_message1 - verification code sent`
    - Connect from the same branch pattern as the imported workflow if you want exact reproduction, or preferably from the successful branch if you want corrected behavior
    - Set:
      - `response_message = I sent you a verification code. Enter it here to continue.`

## 4.8 Build the OTP verification branch
29. Add an **If** node.
    - Name: `IF_step- waiting_otp`
    - Connect `If1 false -> IF_step- waiting_otp`
    - Condition:
      - `step` equals `waiting_for_otp`
30. Add an **If** node.
    - Name: `IF_resend_otp`
    - Connect `IF_step- waiting_otp true -> IF_resend_otp`
    - Condition expression:
      - `/^\d{4,6}$/.test(($json.message || '').trim())`
31. Add a **Postgres** node.
    - Name: `Execute a SQL query1`
    - Connect according to the imported workflow from the **false** branch of `IF_resend_otp` if you need an exact rebuild
    - Recommended production correction: connect from the **true** branch instead
    - Query:
      ```sql
      SELECT *
      FROM user_otps
      WHERE channel = $1
      AND platform_user_id = $2
      AND otp_code = $3
      AND is_used = false
      AND expires_at > now()
      ORDER BY created_at DESC
      LIMIT 1;
      ```
    - Enable `Always Output Data`
32. Add an **If** node.
    - Name: `IF_code_exists?`
    - Connect `Execute a SQL query1 -> IF_code_exists?`
    - Condition:
      - `id` is not empty
33. Add a **Set** node.
    - Name: `code does not exist`
    - Connect `IF_code_exists? false -> code does not exist`
    - Set:
      - `response_message = ❌ Invalid code. Please try again with the correct code.`

## 4.9 Build the verification completion branch
34. Add a **Postgres** node.
    - Name: `valid OTP`
    - Connect `IF_code_exists? true -> valid OTP`
    - Query:
      ```sql
      UPDATE user_otps
      SET is_used = true
      WHERE id = $1;
      ```
35. Add a **Postgres** node.
    - Name: `Execute a SQL query2`
    - Connect `valid OTP -> Execute a SQL query2`
    - Query:
      ```sql
      UPDATE onboarding_sessions
      SET step = 'verified'
      WHERE channel = $1
      AND platform_user_id = $2;
      ```
36. Add a **Postgres** node.
    - Name: `Get verified session email`
    - Connect `Execute a SQL query2 -> Get verified session email`
    - Query:
      ```sql
      SELECT email
      FROM onboarding_sessions
      WHERE channel = $1
      AND platform_user_id = $2
      LIMIT 1;
      ```
    - Enable `Always Output Data`
37. Add an **If** node.
    - Name: `If email en session`
    - Connect `Get verified session email -> If email en session`
    - Condition:
      - `email` is not empty
38. Add a **Set** node.
    - Name: `Missing email response`
    - Connect `If email en session false -> Missing email response`
    - Set:
      - `response_message = ⚠️ I couldn’t find the session email. Please send your email to continue.`
39. Add a **Postgres** node.
    - Name: `Upsert user by email`
    - Connect `If email en session true -> Upsert user by email`
    - Query:
      ```sql
      INSERT INTO users (email, verified)
      VALUES ($1, true)
      ON CONFLICT (email)
      DO UPDATE SET verified = true, updated_at = now()
      RETURNING global_user_id, email;
      ```
40. Add a **Postgres** node.
    - Name: `Upsert user_channel`
    - Connect `Upsert user by email -> Upsert user_channel`
    - Query:
      ```sql
      INSERT INTO user_channels (global_user_id, channel, platform_user_id)
      VALUES ($1, $2, $3)
      ON CONFLICT (channel, platform_user_id)
      DO UPDATE SET global_user_id = EXCLUDED.global_user_id
      RETURNING global_user_id, channel, platform_user_id;
      ```
41. Add a **Set** node.
    - Name: `Verification completed response`
    - Connect `Upsert user_channel -> Verification completed response`
    - Set:
      - `response_message = ✅ Verification completed. This channel has been linked to your user.`

## 4.10 Add documentation notes
42. Add the sticky notes if you want the same visual organization:
    - general workflow intro/setup note
    - section notes for normalization, validation, session initialization, email extraction, OTP delivery, identity consolidation
    - database prerequisite note
    - security notice

## 4.11 Important implementation corrections for production
43. Prefer array-style query replacements consistently in every Postgres node.
44. Review `If2` after `waiting_for_otp`; the imported workflow sends the confirmation message from the false branch, which may be unintended.
45. Review `IF_resend_otp`; OTP validation is connected to the false branch even though the condition tests for numeric OTP format. In most implementations, the validation query should run from the true branch.
46. Add a final response delivery mechanism if you want the workflow to actively answer the originating channel. The current workflow mostly prepares `response_message` fields but does not include channel-specific outbound reply nodes for Telegram or WhatsApp.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Omnichannel OTP Verification & User Onboarding: This workflow centralizes user verification across WhatsApp, Telegram, and email using OTP validation. It detects the channel, manages onboarding sessions, generates secure OTP codes, and links verified channels to a single global user identity. | General workflow note |
| How to set up: 1. Import the workflow into n8n. 2. Configure Postgres or Supabase credentials. 3. Download and create the required database tables (`users`, `user_channels`, `user_otps`, `onboarding_sessions`). 4. Connect email and messaging channel credentials. 5. Test with new and returning users. | General setup note |
| Customization: Adjust OTP expiration time; modify onboarding steps or validation logic; add additional communication channels. | General customization note |
| DATABASE REQUIRED: This workflow depends on predefined Postgres tables. You must create `users`, `user_channels`, `user_otps`, and `onboarding_sessions` before running the flow. | Database prerequisite |
| Security Notice: Do not log OTP codes in production environments. Always define an expiration time and mark codes as used after validation. | Security guidance |

## Additional implementation observations
- The workflow has a single entry point: the `Webhook` node.
- There are no sub-workflows or Execute Workflow nodes.
- The workflow is inactive in the exported JSON (`active: false`).
- The workflow uses Postgres credentials named `Postgres account 2` and SMTP credentials named `SMTP account`.
- Several SQL replacement configurations are written in inconsistent formats; standardizing them to explicit arrays is strongly recommended for reliable reproduction.
- The workflow prepares internal `response_message` fields but does not contain dedicated outbound WhatsApp or Telegram response nodes, so an additional delivery layer may be required depending on your runtime architecture.