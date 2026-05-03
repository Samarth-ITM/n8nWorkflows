Lead collection with SendPulse and GPT-generated welcome emails/SMS

https://n8nworkflows.xyz/workflows/lead-collection-with-sendpulse-and-gpt-generated-welcome-emails-sms-11868


# Lead collection with SendPulse and GPT-generated welcome emails/SMS

## 1. Workflow Overview

**Purpose:** Collect lead data from a website form (name + email and/or phone), ensure the lead is stored in the appropriate SendPulse mailing list(s), then generate and send:
- a **personalized welcome email** (AI-generated) via **SendPulse SMTP** (through an MCP tool), and
- a **personalized welcome SMS** (AI-generated) via **SendPulse SMS API**.

**Target use cases:**
- “Contact us” / registration forms where you want immediate, personalized onboarding messages.
- Simple CRM-lite capture into SendPulse address books, with automated email + SMS follow-up.

**Logical blocks:**
1.1 **Webhook intake & configuration**  
1.2 **SendPulse access token caching (Data Table) + refresh**  
1.3 **Fetch SendPulse mailing lists**  
1.4 **Email path: list existence → list create if missing → add contact → AI email → SMTP send**  
1.5 **SMS path: list existence → list create if missing → add phone → AI SMS → SMS send**

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception & Workflow Configuration
**Overview:** Receives POSTed lead data and maps it into workflow variables used across subsequent nodes (API URLs, list names, sender identity, and lead fields).  
**Nodes involved:** `New Customer Registration`, `Workflow Configuration`

#### Node: New Customer Registration
- **Type / role:** Webhook (trigger). Entry point for new lead submissions.
- **Key configuration:**
  - **Path:** `customer-registration`
  - **Method:** `POST`
- **Inputs/Outputs:**
  - **Output:** The webhook payload is expected at `$json.body` (e.g., `$json.body.name`, `$json.body.email`, `$json.body.phone`).
  - **Next:** `Workflow Configuration`
- **Edge cases / failures:**
  - Missing `body` fields (name/email/phone) will cause downstream “Exists …?” checks to route differently.
  - If the form sends data in a different structure (not in `body`), expressions in `Workflow Configuration` will resolve to `undefined`.
- **Version specifics:** Webhook node v2.1.

#### Node: Workflow Configuration
- **Type / role:** Set node. Centralized configuration + mapping lead fields from the webhook.
- **Key configuration choices:**
  - Defines constants:
    - `sendPulseApiUrl` = `https://api.sendpulse.com`
    - `clientId`, `clientSecret` (placeholders)
    - mailing list names: `mailingListWithEmails`, `mailingListWithPhones`
    - email sender identity: `senderName`, `senderEmail`
    - SMS sender + routing: `smsSender`, `routeCountryCode`, `routeType`
  - Maps lead fields from webhook:
    - `name` = `{{ $json.body.name }}`
    - `email` = `{{ $json.body.email }}`
    - `phone` = `{{ $json.body.phone }}`
- **Inputs/Outputs:**
  - **Input:** Webhook output.
  - **Output:** A single item containing configuration + extracted lead data.
  - **Next:** `Generate Hash`
- **Edge cases / failures:**
  - If `clientId/clientSecret` are not replaced, token retrieval will fail (401/invalid_client).
  - If `senderEmail/smsSender` are invalid/unapproved in SendPulse, send operations can fail.
- **Version specifics:** Set node v3.4.

---

### 1.2 Token Management (Cache + Refresh)
**Overview:** Computes a stable storage key from SendPulse credentials, attempts to load an existing token from an n8n Data Table, refreshes it if expired (or missing), then outputs a usable access token.  
**Nodes involved:** `Generate Hash`, `Get Token from Storage`, `Is Token Expired?`, `Get New Access Token`, `Save Token to Storage`, `Set Access Token`

#### Node: Generate Hash
- **Type / role:** Code node. Generates an MD5 hash used as a lookup key in the token table.
- **Configuration (interpreted):**
  - Reads `clientId` and `clientSecret` from the incoming item.
  - Computes `storageKey = md5(clientId + ':' + clientSecret)`.
  - Outputs original JSON + `storageKey`.
- **Key variables/expressions:** Uses Node.js `crypto` module.
- **Inputs/Outputs:**
  - **Input:** `Workflow Configuration`
  - **Output:** Adds `storageKey`
  - **Next:** `Get Token from Storage`
- **Edge cases / failures:**
  - If `clientId` or `clientSecret` missing, the hash becomes deterministic but meaningless; token calls will fail later.
- **Version specifics:** Code node v2.

#### Node: Get Token from Storage
- **Type / role:** Data Table node (get). Retrieves cached token record by `hash`.
- **Configuration choices:**
  - **Operation:** `get`
  - **Filter:** `hash == {{ $json.storageKey }}`
  - **Limit:** 1
  - **Data table:** `tokens` (ID cached as `l03FRqX5p37JB7a6`)
- **Inputs/Outputs:**
  - **Input:** From `Generate Hash`
  - **Output:** Token record item if found; otherwise typically an empty result (behavior depends on n8n Data Table implementation).
  - **Next:** `Is Token Expired?`
- **Edge cases / failures:**
  - Data Table missing, wrong schema, or permissions issue → node error.
  - If no row exists, downstream `Is Token Expired?` must handle missing `tokenExpiry/accessToken` safely (see next node).
- **Version specifics:** Data Table node v1.

#### Node: Is Token Expired?
- **Type / role:** IF node. Decides whether to refresh token.
- **Configuration (interpreted):**
  - Condition group (AND):
    1) `tokenExpiry != "null"` (string comparison)  
    2) `Date.now() >= new Date(tokenExpiry).getTime()`
- **Routing:**
  - **True branch (token expired):** `Get New Access Token`
  - **False branch (not expired or missing):** `Set Access Token`
- **Important nuance / edge case:**
  - If `tokenExpiry` is missing/undefined, the first condition (`notEquals "null"`) is likely **true**, and `new Date(undefined).getTime()` yields `NaN`, making the second condition **false**; overall AND becomes false → routes to **Set Access Token** with potentially missing `accessToken`.  
  - In other words: **a missing token may incorrectly be treated as “not expired”**, causing failures later when Authorization header is built.
- **Version specifics:** IF node v2.2.

#### Node: Get New Access Token
- **Type / role:** HTTP Request. Fetches a new OAuth access token from SendPulse.
- **Configuration (interpreted):**
  - **POST** `{{ sendPulseApiUrl }}/oauth/access_token`
  - Form/body parameters:
    - `grant_type=client_credentials`
    - `client_id={{ clientId }}`
    - `client_secret={{ clientSecret }}`
- **Inputs/Outputs:**
  - **Input:** From `Is Token Expired?` true
  - **Output:** Expects `access_token` in response JSON
  - **Next:** `Save Token to Storage`
- **Edge cases / failures:**
  - 401/400 if credentials wrong.
  - SendPulse downtime/timeouts.
- **Version specifics:** HTTP Request v4.3.

#### Node: Save Token to Storage
- **Type / role:** Data Table node (upsert). Stores token + expiry timestamp.
- **Configuration (interpreted):**
  - **Operation:** `upsert` matching on `hash`
  - Writes columns:
    - `hash = {{ $('Generate Hash').item.json.storageKey }}`
    - `accessToken = {{ $json.access_token }}`
    - `tokenExpiry = {{ new Date(Date.now() + 3600000).toISOString() }}` (1 hour)
- **Inputs/Outputs:**
  - **Input:** Token response from `Get New Access Token`
  - **Output:** Upsert result (and/or the written data depending on n8n behavior)
  - **Next:** `Set Access Token`
- **Edge cases / failures:**
  - If SendPulse token TTL differs from 1 hour, the expiry might be inaccurate.
  - Data table schema mismatch (“tokenExpiry” column typo in note: “stirng”) can cause confusion when creating the table manually.
- **Version specifics:** Data Table node v1.

#### Node: Set Access Token
- **Type / role:** Set node. Normalizes the token field used later.
- **Configuration:** `accessToken = {{ $json.accessToken }}`
- **Inputs/Outputs:**
  - **Input:** Either from `Save Token to Storage` (fresh token) or from `Is Token Expired?` false (cached record)
  - **Output:** `{ accessToken: ... }`
  - **Next:** `Get Mailing List`
- **Edge cases / failures:**
  - If upstream item doesn’t contain `accessToken` (missing Data Table row scenario), downstream API calls will 401 due to empty bearer token.

---

### 1.3 Retrieve SendPulse Mailing Lists
**Overview:** Calls SendPulse to retrieve all address books (mailing lists), used by both email and phone flows to find or create required lists.  
**Nodes involved:** `Get Mailing List`

#### Node: Get Mailing List
- **Type / role:** HTTP Request. Fetches SendPulse email address books.
- **Configuration (interpreted):**
  - **GET** `{{ sendPulseApiUrl }}/addressbooks`
  - Header: `Authorization: Bearer {{ $json.accessToken }}`
- **Inputs/Outputs:**
  - **Input:** `Set Access Token`
  - **Output:** List(s) of mailing lists; downstream code nodes use `$('Get Mailing List').all()` to iterate.
  - **Next:** `Exists email?`
- **Edge cases / failures:**
  - 401 if token missing/expired.
  - API response shape changes can break downstream code that expects `item.json.name` and `item.json.id`.
- **Version specifics:** HTTP Request v4.3.

---

### 1.4 Email Processing (List + Contact + AI Email + SMTP)
**Overview:** If an email exists, ensure the “email” mailing list exists, add the email contact, then generate a welcome email via an AI agent and send it via SendPulse SMTP tool (MCP).  
**Nodes involved:** `Exists email?`, `Search Mailing Lists With Emails`, `Exists Mailing List With Emails?`, `Create Mailing List With Emails`, `Add Email to Mailing List`, `OpenAI Chat Model`, `SendPulse MCP Client`, `Generate And Send Welcome Email`

#### Node: Exists email?
- **Type / role:** IF node. Gate email flow.
- **Condition:** “exists” check on `{{ $('Workflow Configuration').item.json.email }}`
- **Routing:**
  - **True:** `Search Mailing Lists With Emails`
  - **False:** `Exists phone?` (skips email path entirely)
- **Edge cases:**
  - If email is an empty string, “exists” behavior depends on n8n semantics; may still count as existing. Consider adding “not empty” validation if needed.

#### Node: Search Mailing Lists With Emails
- **Type / role:** Code node. Finds target email list by name.
- **Configuration (interpreted):**
  - Reads all items from `Get Mailing List`.
  - Compares `item.json.name` to configured `mailingListWithEmails`.
  - Outputs:
    - `isMailingList: true/false`
    - `mailingListId: foundList.id or null`
- **Inputs/Outputs:**
  - **Input:** Triggered after `Exists email?` true
  - **Output:** Single decision item
  - **Next:** `Exists Mailing List With Emails?`
- **Edge cases:**
  - If `Get Mailing List` returns paginated data (not handled), list might not be found.

#### Node: Exists Mailing List With Emails?
- **Type / role:** IF node. Determines whether to create list.
- **Condition:** `{{ $json.isMailingList }} == true`
- **Routing:**
  - **True:** `Add Email to Mailing List`
  - **False:** `Create Mailing List With Emails`

#### Node: Create Mailing List With Emails
- **Type / role:** HTTP Request. Creates an address book for emails.
- **Configuration:**
  - **POST** `{{ sendPulseApiUrl }}/addressbooks`
  - Body: `bookName = {{ mailingListWithEmails }}`
  - Header: intended to be `Authorization: Bearer ...`
- **Important issue (likely bug):**
  - Header parameter name is configured as `=Authorization` (leading `=`). This may result in a malformed header key and SendPulse returning **401**.  
  - It should be exactly `Authorization`.
- **Inputs/Outputs:**
  - **Input:** From “false” branch of `Exists Mailing List With Emails?`
  - **Output:** Newly created list object (expects `id`)
  - **Next:** `Add Email to Mailing List`

#### Node: Add Email to Mailing List
- **Type / role:** HTTP Request. Adds the lead email to an address book.
- **Configuration (interpreted):**
  - **POST** `{{ sendPulseApiUrl }}/addressbooks/{{ $json.id || $json.mailingListId }}/emails`
  - JSON body includes:
    - email: `{{ email }}`
    - variables: `{ name: {{ name }} }`
  - Header: `Authorization: Bearer {{ accessToken }}`
- **Inputs/Outputs:**
  - **Input:** Either list creation response (with `id`) or search result (with `mailingListId`)
  - **Output:** SendPulse add result
  - **Next:** `Generate And Send Welcome Email`
- **Edge cases / failures:**
  - Duplicate emails: SendPulse may return conflict or ignore; behavior depends on API.
  - Invalid email format → 400.

#### Node: OpenAI Chat Model
- **Type / role:** LangChain OpenAI chat model. Provides LLM to both AI agents.
- **Configuration:**
  - Model: `gpt-4.1-mini`
  - No special options enabled.
- **Connections:**
  - Feeds as `ai_languageModel` into:
    - `Generate And Send Welcome Email`
    - `Generate Welcome SMS`
- **Edge cases / failures:**
  - Missing OpenAI credentials → execution error.
  - Rate limits / timeouts from OpenAI.
- **Version specifics:** node v1.3.

#### Node: SendPulse MCP Client
- **Type / role:** MCP client tool provider for LangChain agent (exposes SendPulse SMTP send tool).
- **Configuration:**
  - Endpoint: `https://mcp.sendpulse.com/mcp`
  - Included tool: `smtp_emails_send`
  - Auth: “multipleHeadersAuth” credential (must be configured)
- **Connections:**
  - Provides `ai_tool` to `Generate And Send Welcome Email`
- **Edge cases / failures:**
  - Wrong MCP credentials/headers → tool call failure.
  - MCP endpoint unreachable.

#### Node: Generate And Send Welcome Email
- **Type / role:** LangChain Agent. Generates email content and sends it via MCP SMTP tool.
- **Configuration (interpreted):**
  - Prompt includes rules (friendly, concise, non-spammy), personalization, and sender/recipient details from `Workflow Configuration`.
  - Instructs agent to **send the created email via SMTP** (so it should call the MCP tool).
- **Inputs/Outputs:**
  - **Input:** From `Add Email to Mailing List`
  - **Uses:** `OpenAI Chat Model` + `SendPulse MCP Client` tool
  - **Output:** Agent result (not used further in this workflow)
- **Edge cases / failures:**
  - If the agent does not call the tool (prompt/tool-selection mismatch), no email is sent.
  - Invalid sender email/domain may be rejected by SendPulse SMTP tool/policy.

---

### 1.5 SMS Processing (List + Phone + AI SMS + Send)
**Overview:** If a phone exists, ensure the “phone” mailing list exists, add the phone number with variables, generate a welcome SMS via AI, then send it via SendPulse SMS API.  
**Nodes involved:** `Exists phone?`, `Search Mailing Lists With Phone Numbers`, `Is Exists Mailing List With Phone Numbers`, `Create mailing list with phones`, `Add Phone Number to Mailing List`, `Generate Welcome SMS`, `Send SMS to Client`

#### Node: Exists phone?
- **Type / role:** IF node. Gate SMS flow.
- **Condition:** “exists” check on `{{ phone }}`
- **Routing:**
  - **True:** `Search Mailing Lists With Phone Numbers`
  - **False:** (no downstream node; workflow effectively ends after email path if any)
- **Edge cases:** Same as email—may want validation for empty string / phone format.

#### Node: Search Mailing Lists With Phone Numbers
- **Type / role:** Code node. Finds target phone list by configured name.
- **Configuration (interpreted):**
  - Iterates `$('Get Mailing List').all()` (note: uses email address books endpoint results).
  - Compares to `mailingListWithPhones`.
  - Outputs `isMailingList` + `mailingListId`.
- **Edge cases:**
  - If SMS address books are separate from email address books in SendPulse, searching `/addressbooks` may not reflect SMS lists (depends on SendPulse account model).

#### Node: Is Exists Mailing List With Phone Numbers
- **Type / role:** IF node. Create list if missing.
- **Condition:** `isMailingList == true`
- **Routing:**
  - **True:** `Add Phone Number to Mailing List`
  - **False:** `Create mailing list with phones`

#### Node: Create mailing list with phones
- **Type / role:** HTTP Request. Creates an address book (intended for phone list).
- **Configuration:**
  - **POST** `{{ sendPulseApiUrl }}/addressbooks ` (note trailing space in URL)
  - Body: `bookName = {{ mailingListWithPhones }}`
  - Header: `Authorization: Bearer {{ accessToken }}`
- **Important issue (likely bug):**
  - The URL contains a **trailing space** after `/addressbooks `. This can cause request failure (404 or invalid URL) depending on n8n normalization.
- **Next:** `Add Phone Number to Mailing List`

#### Node: Add Phone Number to Mailing List
- **Type / role:** HTTP Request. Adds phone number + variables into SendPulse SMS address book.
- **Configuration (interpreted):**
  - **POST** `{{ sendPulseApiUrl }}/sms/numbers/variables`
  - JSON body:
    - `addressBookId: {{ $json.id || $json.mailingListId }}`
    - `phones` map: `{ "<phone>": [[{ name:"Name", type:"string", value:"<name>" }]] }`
  - Header: `Authorization: Bearer {{ accessToken }}`
- **Inputs/Outputs:**
  - **Input:** list id from either create or search
  - **Output:** API result
  - **Next:** `Generate Welcome SMS`
- **Edge cases / failures:**
  - Phone format must match SendPulse expectations (E.164 or local—depends on SendPulse).
  - If addressBookId is wrong type (string vs number), request may fail.

#### Node: Generate Welcome SMS
- **Type / role:** LangChain Agent. Generates final SMS text only.
- **Configuration:** Prompt instructs short, professional SMS; output should be message body only.
- **Connections:**
  - Uses `OpenAI Chat Model` (as language model)
  - Output goes to `Send SMS to Client`
- **Edge cases / failures:**
  - If the agent returns extra formatting/quotes/newlines, SMS body may be suboptimal.
  - Token limits unlikely, but still possible.

#### Node: Send SMS to Client
- **Type / role:** HTTP Request. Sends SMS via SendPulse.
- **Configuration (interpreted):**
  - **POST** `{{ sendPulseApiUrl }}/sms/send`
  - JSON body:
    - `sender`: `{{ smsSender }}`
    - `phones`: `[ "{{ phone }}" ]`
    - `body`: `{{ $json.output }}` (expects agent output field named `output`)
    - `route`: `{ "{{ routeCountryCode }}": "{{ routeType }}" }`
  - Header: `Authorization: Bearer {{ accessToken }}`
- **Edge cases / failures:**
  - If the agent output field is not exactly `$json.output` (depends on agent node output schema), the SMS body may be empty.
  - Sender not approved or insufficient SMS balance → SendPulse error.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Customer Registration | Webhook | Receives lead submission | — | Workflow Configuration | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Workflow Configuration | Set | Defines config + maps webhook fields | New Customer Registration | Generate Hash | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Workflow Configuration | Set | Defines config + maps webhook fields | New Customer Registration | Generate Hash | ## 1. Workflow Configuration\nConfigure workflow variables |
| Generate Hash | Code | Builds storageKey for token cache | Workflow Configuration | Get Token from Storage | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Generate Hash | Code | Builds storageKey for token cache | Workflow Configuration | Get Token from Storage | ## 2. Get access token\nSendPulse token management logic with caching in the Data Table |
| Get Token from Storage | Data Table | Loads cached token by hash | Generate Hash | Is Token Expired? | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Get Token from Storage | Data Table | Loads cached token by hash | Generate Hash | Is Token Expired? | ## 2. Get access token\nSendPulse token management logic with caching in the Data Table |
| Is Token Expired? | IF | Decides refresh vs reuse token | Get Token from Storage | Get New Access Token; Set Access Token | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Is Token Expired? | IF | Decides refresh vs reuse token | Get Token from Storage | Get New Access Token; Set Access Token | ## 2. Get access token\nSendPulse token management logic with caching in the Data Table |
| Get New Access Token | HTTP Request | Requests SendPulse OAuth token | Is Token Expired? (true) | Save Token to Storage | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Get New Access Token | HTTP Request | Requests SendPulse OAuth token | Is Token Expired? (true) | Save Token to Storage | ## 2. Get access token\nSendPulse token management logic with caching in the Data Table |
| Save Token to Storage | Data Table | Upserts token + expiry | Get New Access Token | Set Access Token | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Save Token to Storage | Data Table | Upserts token + expiry | Get New Access Token | Set Access Token | ## 2. Get access token\nSendPulse token management logic with caching in the Data Table |
| Set Access Token | Set | Normalizes `accessToken` field | Is Token Expired? (false) / Save Token to Storage | Get Mailing List | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Get Mailing List | HTTP Request | Fetches SendPulse address books | Set Access Token | Exists email? | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS; How it works + setup checklist (see note content) |
| Get Mailing List | HTTP Request | Fetches SendPulse address books | Set Access Token | Exists email? | ## 3. Get mailing list\nGetting a list of all mailing lists from the Sendpulse Email service |
| Exists email? | IF | Gates email flow | Get Mailing List | Search Mailing Lists With Emails; Exists phone? | ## 4. Email Processing\nChecking the existence of a mailing list for emails, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome email via AI and SendPulse MCP |
| Search Mailing Lists With Emails | Code | Finds email list by name | Exists email? (true) | Exists Mailing List With Emails? | ## 4. Email Processing\nChecking the existence of a mailing list for emails, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome email via AI and SendPulse MCP |
| Exists Mailing List With Emails? | IF | Create list if missing | Search Mailing Lists With Emails | Add Email to Mailing List; Create Mailing List With Emails | ## 4. Email Processing\nChecking the existence of a mailing list for emails, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome email via AI and SendPulse MCP |
| Create Mailing List With Emails | HTTP Request | Creates email address book | Exists Mailing List With Emails? (false) | Add Email to Mailing List | ## 4. Email Processing\nChecking the existence of a mailing list for emails, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome email via AI and SendPulse MCP |
| Add Email to Mailing List | HTTP Request | Adds email contact to list | Exists Mailing List With Emails? (true) / Create Mailing List With Emails | Generate And Send Welcome Email | ## 4. Email Processing\nChecking the existence of a mailing list for emails, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome email via AI and SendPulse MCP |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM for agents | — | Generate And Send Welcome Email; Generate Welcome SMS |  |
| SendPulse MCP Client | MCP Client Tool (LangChain) | Provides SMTP send tool | — | Generate And Send Welcome Email |  |
| Generate And Send Welcome Email | Agent (LangChain) | Generates + sends welcome email via MCP tool | Add Email to Mailing List | — | ## 4. Email Processing\nChecking the existence of a mailing list for emails, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome email via AI and SendPulse MCP |
| Exists phone? | IF | Gates SMS flow | Exists email? (false) / (implicitly after email path gating) | Search Mailing Lists With Phone Numbers | ## 5. SMS Processing\nChecking the existence of a mailing list for phone numbers, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome SMS via AI and SendPulse SMS API |
| Search Mailing Lists With Phone Numbers | Code | Finds phone list by name | Exists phone? (true) | Is Exists Mailing List With Phone Numbers | ## 5. SMS Processing\nChecking the existence of a mailing list for phone numbers, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome SMS via AI and SendPulse SMS API |
| Is Exists Mailing List With Phone Numbers | IF | Create list if missing | Search Mailing Lists With Phone Numbers | Add Phone Number to Mailing List; Create mailing list with phones | ## 5. SMS Processing\nChecking the existence of a mailing list for phone numbers, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome SMS via AI and SendPulse SMS API |
| Create mailing list with phones | HTTP Request | Creates phone address book | Is Exists Mailing List With Phone Numbers (false) | Add Phone Number to Mailing List | ## 5. SMS Processing\nChecking the existence of a mailing list for phone numbers, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome SMS via AI and SendPulse SMS API |
| Add Phone Number to Mailing List | HTTP Request | Adds phone + variables to SMS address book | Is Exists Mailing List With Phone Numbers (true) / Create mailing list with phones | Generate Welcome SMS | ## 5. SMS Processing\nChecking the existence of a mailing list for phone numbers, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome SMS via AI and SendPulse SMS API |
| Generate Welcome SMS | Agent (LangChain) | Generates SMS text | Add Phone Number to Mailing List | Send SMS to Client | ## 5. SMS Processing\nChecking the existence of a mailing list for phone numbers, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome SMS via AI and SendPulse SMS API |
| Send SMS to Client | HTTP Request | Sends SMS via SendPulse | Generate Welcome SMS | — | ## 5. SMS Processing\nChecking the existence of a mailing list for phone numbers, creating a list if necessary, adding a contact to the mailing list, automatically generating and sending a personalized welcome SMS via AI and SendPulse SMS API |
| Sticky Note4 | Sticky Note | Documentation (not executed) | — | — | # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS (contains “How it works” + setup checklist) |
| Sticky Note5 | Sticky Note | Documentation (not executed) | — | — | ## 2. Get access token\nSendPulse token management logic with caching in the Data Table |
| Sticky Note6 | Sticky Note | Documentation (not executed) | — | — | ## 1. Workflow Configuration\nConfigure workflow variables |
| Sticky Note8 | Sticky Note | Documentation (not executed) | — | — | ## 3. Get mailing list\nGetting a list of all mailing lists from the Sendpulse Email service |
| Sticky Note9 | Sticky Note | Documentation (not executed) | — | — | ## 4. Email Processing… |
| Sticky Note10 | Sticky Note | Documentation (not executed) | — | — | ## 5. SMS Processing… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** (inactive by default while configuring).

2) **Add Webhook node**: *Webhook*  
   - Name: `New Customer Registration`  
   - Method: `POST`  
   - Path: `customer-registration`  
   - Save the webhook URL for your website form.

3) **Add Set node**: *Set*  
   - Name: `Workflow Configuration`  
   - Add fields (strings unless noted):
     - `sendPulseApiUrl`: `https://api.sendpulse.com`
     - `clientId`: your SendPulse Client ID
     - `clientSecret`: your SendPulse Client Secret
     - `mailingListWithEmails`: e.g. `Customer emails`
     - `mailingListWithPhones`: e.g. `Customer phone numbers`
     - `senderName`: (your sender display name)
     - `senderEmail`: (your sender email)
     - `smsSender`: (approved SendPulse SMS sender)
     - `routeCountryCode`: e.g. `UA`
     - `routeType`: e.g. `international`
     - `name`: expression `{{ $json.body.name }}`
     - `email`: expression `{{ $json.body.email }}`
     - `phone`: expression `{{ $json.body.phone }}`
   - Connect: `New Customer Registration` → `Workflow Configuration`

4) **Add Code node**: *Code*  
   - Name: `Generate Hash`  
   - Paste logic to compute MD5 from `clientId:clientSecret` and output `storageKey`. (Same behavior as provided workflow.)
   - Connect: `Workflow Configuration` → `Generate Hash`

5) **Create Data Table** (n8n “Data Tables”) named `tokens` with columns:
   - `hash` (string)
   - `accessToken` (string)
   - `tokenExpiry` (string; ISO datetime)
   - Note: the sticky note contains a typo “stirng”; use **string**.

6) **Add Data Table node**: *Data Table*  
   - Name: `Get Token from Storage`  
   - Operation: `get`  
   - Data table: `tokens`  
   - Filter: `hash` equals `{{ $json.storageKey }}`  
   - Limit: 1  
   - Connect: `Generate Hash` → `Get Token from Storage`

7) **Add IF node**: *IF*  
   - Name: `Is Token Expired?`  
   - Conditions (AND):
     - `tokenExpiry` **not equals** `null` (ensure you are comparing properly; ideally check “exists” and “not empty”)
     - Number compare: `{{ Date.now() }}` **gte** `{{ new Date($json.tokenExpiry).getTime() }}`
   - Connect: `Get Token from Storage` → `Is Token Expired?`

8) **Add HTTP Request node**: *HTTP Request*  
   - Name: `Get New Access Token`  
   - Method: `POST`  
   - URL: `{{ $('Workflow Configuration').item.json.sendPulseApiUrl }}/oauth/access_token`  
   - Body parameters:
     - `grant_type=client_credentials`
     - `client_id={{ $('Workflow Configuration').item.json.clientId }}`
     - `client_secret={{ $('Workflow Configuration').item.json.clientSecret }}`
   - Connect: `Is Token Expired?` (true) → `Get New Access Token`

9) **Add Data Table node**: *Data Table*  
   - Name: `Save Token to Storage`  
   - Operation: `upsert`  
   - Match/filter: `hash = {{ $('Generate Hash').item.json.storageKey }}`
   - Columns to write:
     - `hash`: same as above
     - `accessToken`: `{{ $json.access_token }}`
     - `tokenExpiry`: `{{ new Date(Date.now() + 3600000).toISOString() }}`
   - Connect: `Get New Access Token` → `Save Token to Storage`

10) **Add Set node**: *Set*  
   - Name: `Set Access Token`  
   - Field: `accessToken = {{ $json.accessToken }}`  
   - Connect:
     - `Save Token to Storage` → `Set Access Token`
     - `Is Token Expired?` (false) → `Set Access Token`

11) **Add HTTP Request node**: *HTTP Request*  
   - Name: `Get Mailing List`  
   - Method: `GET`  
   - URL: `{{ $('Workflow Configuration').item.json.sendPulseApiUrl }}/addressbooks`  
   - Header: `Authorization: Bearer {{ $json.accessToken }}`  
   - Connect: `Set Access Token` → `Get Mailing List`

12) **Add IF node**: *IF*  
   - Name: `Exists email?`  
   - Condition: string “exists” on `{{ $('Workflow Configuration').item.json.email }}`  
   - Connect: `Get Mailing List` → `Exists email?`

13) **Email list lookup Code node**: *Code*  
   - Name: `Search Mailing Lists With Emails`  
   - Implement: search `$('Get Mailing List').all()` for list name == `mailingListWithEmails`, output `isMailingList` + `mailingListId`.  
   - Connect: `Exists email?` (true) → `Search Mailing Lists With Emails`

14) **Add IF node**: *IF*  
   - Name: `Exists Mailing List With Emails?`  
   - Condition: boolean equals `true` on `{{ $json.isMailingList }}`  
   - Connect: `Search Mailing Lists With Emails` → `Exists Mailing List With Emails?`

15) **Add HTTP Request node** (create email list):  
   - Name: `Create Mailing List With Emails`  
   - POST `{{ sendPulseApiUrl }}/addressbooks`  
   - Body: `bookName={{ mailingListWithEmails }}`  
   - Header: **Authorization**: `Bearer {{ $('Set Access Token').first().json.accessToken }}`  
   - Connect: `Exists Mailing List With Emails?` (false) → `Create Mailing List With Emails`

16) **Add HTTP Request node** (add email):  
   - Name: `Add Email to Mailing List`  
   - POST `{{ sendPulseApiUrl }}/addressbooks/{{ $json.id || $json.mailingListId }}/emails`  
   - Header: `Authorization: Bearer {{ accessToken }}`  
   - JSON body with email + variables.name  
   - Connect:
     - `Exists Mailing List With Emails?` (true) → `Add Email to Mailing List`
     - `Create Mailing List With Emails` → `Add Email to Mailing List`

17) **Add OpenAI Chat Model node (LangChain)**:  
   - Name: `OpenAI Chat Model`  
   - Model: `gpt-4.1-mini`  
   - Configure **OpenAI API credentials**.

18) **Add MCP Client Tool node**:  
   - Name: `SendPulse MCP Client`  
   - Endpoint: `https://mcp.sendpulse.com/mcp`  
   - Include tool: `smtp_emails_send`  
   - Authentication: **Multiple Headers Auth** credential (configure headers per SendPulse MCP requirements).

19) **Add LangChain Agent node (email)**:  
   - Name: `Generate And Send Welcome Email`  
   - Prompt: include constraints + sender/recipient details from `Workflow Configuration`, and instruct to send via SMTP.
   - Connect:
     - Main: `Add Email to Mailing List` → `Generate And Send Welcome Email`
     - AI Language Model: `OpenAI Chat Model` → `Generate And Send Welcome Email`
     - AI Tool: `SendPulse MCP Client` → `Generate And Send Welcome Email`

20) **SMS gating IF node**:  
   - Name: `Exists phone?`  
   - Condition: string “exists” on `{{ phone }}`  
   - Connect: `Exists email?` (false) → `Exists phone?` (matches provided workflow behavior)

21) **Phone list lookup Code node**:  
   - Name: `Search Mailing Lists With Phone Numbers`  
   - Same logic as email, but target list name is `mailingListWithPhones`.  
   - Connect: `Exists phone?` (true) → this code node

22) **Add IF node**:  
   - Name: `Is Exists Mailing List With Phone Numbers`  
   - Condition: `isMailingList == true`  
   - Connect: phone search → this IF

23) **Add HTTP Request node** (create phone list):  
   - Name: `Create mailing list with phones`  
   - POST `{{ sendPulseApiUrl }}/addressbooks` (**no trailing space**)  
   - Header: `Authorization: Bearer {{ accessToken }}`  
   - Body: `bookName={{ mailingListWithPhones }}`  
   - Connect: IF (false) → create node

24) **Add HTTP Request node** (add phone variables):  
   - Name: `Add Phone Number to Mailing List`  
   - POST `{{ sendPulseApiUrl }}/sms/numbers/variables`  
   - Header: `Authorization: Bearer {{ accessToken }}`  
   - JSON body uses `addressBookId` from `id || mailingListId`, and maps phone to variables.  
   - Connect:
     - IF (true) → add phone node
     - create phone list → add phone node

25) **Add LangChain Agent node (SMS)**:  
   - Name: `Generate Welcome SMS`  
   - Prompt: generate SMS text only (no extra markup).  
   - Connect:
     - Main: `Add Phone Number to Mailing List` → `Generate Welcome SMS`
     - AI Language Model: `OpenAI Chat Model` → `Generate Welcome SMS`

26) **Add HTTP Request node (send SMS)**:  
   - Name: `Send SMS to Client`  
   - POST `{{ sendPulseApiUrl }}/sms/send`  
   - Header: `Authorization: Bearer {{ accessToken }}`  
   - JSON body uses:
     - `sender`, `phones`, `body` from agent output, `route` mapping  
   - Connect: `Generate Welcome SMS` → `Send SMS to Client`

27) **Test execution**:
   - Send a POST to the webhook with JSON like:
     - `{ "name": "Ada", "email": "ada@example.com", "phone": "+380..." }` (ensure it arrives under `body` as expected by your webhook/form)
   - Verify:
     - token row written to Data Table
     - mailing lists created/found
     - contact added
     - welcome email sent
     - SMS sent

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| # Automated Lead Collection to SendPulse and AI-Generated Welcome Emails/SMS — describes flow and setup checklist (Client ID/Secret, list names, sender identity, tokens Data Table schema, OpenAI key, webhook wiring, testing) | Included in Sticky Note4 inside the workflow canvas |
| “## 2. Get access token — SendPulse token management logic with caching in the Data Table” | Sticky Note5 |
| “## 1. Workflow Configuration — Configure workflow variables” | Sticky Note6 |
| “## 3. Get mailing list — Getting a list of all mailing lists from the Sendpulse Email service” | Sticky Note8 |
| “## 4. Email Processing … via AI and SendPulse MCP” | Sticky Note9 |
| “## 5. SMS Processing … via AI and SendPulse SMS API” | Sticky Note10 |
| Disclaimer (FR): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Provided in your request (contextual compliance statement) |

**Notable implementation issues to fix when reproducing:**
- `Create Mailing List With Emails` header key appears as `=Authorization` (should be `Authorization`).
- `Create mailing list with phones` URL contains a trailing space (`/addressbooks `) (should be `/addressbooks`).
- Token “missing row” scenario may route to `Set Access Token` with an undefined token; consider enhancing `Is Token Expired?` to treat missing token as “expired/needs refresh”.