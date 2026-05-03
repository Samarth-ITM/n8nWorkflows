Feedspace Testimonials to HubSpot CRM Contacts

https://n8nworkflows.xyz/workflows/feedspace-testimonials-to-hubspot-crm-contacts-13027


# Feedspace Testimonials to HubSpot CRM Contacts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow receives new testimonials from **Feedspace** via a webhook and syncs them into **HubSpot CRM** by:
1) creating/updating a **Contact** (upsert by email), and  
2) creating a **Note** associated with that contact containing the testimonial details (text + optional video/audio links).

**Target use cases:**
- Automatically centralize customer feedback collected in Feedspace inside HubSpot.
- Keep HubSpot contacts up-to-date while attaching structured testimonial context for Sales/CS teams.

### 1.1 Documentation & On-canvas Guidance (Sticky Notes)
Explains the workflow’s intent, setup requirements, and the functional sections.

### 1.2 Input Reception & Normalization
Receives the webhook payload from Feedspace and extracts a normalized set of fields (email, name, comment, rating, media URLs, feed metadata).

### 1.3 Validation & Conditional Routing
Checks if an email exists; if not, responds to the webhook with an error.

### 1.4 HubSpot Contact Upsert
Creates or updates a HubSpot contact using the email, and maps basic profile fields.

### 1.5 HubSpot Note Creation & Webhook Response
Builds a formatted note body, creates a HubSpot Note associated to the contact, then responds “success” to the webhook call.

---

## 2. Block-by-Block Analysis

### Block 1 — Documentation & Section Labels
**Overview:** Provides human-readable context and organizes the canvas into sections. No runtime impact.  
**Nodes involved:**  
- 📋 Workflow Overview (Sticky Note)  
- 🔗 Trigger Section (Sticky Note)  
- 🏢 HubSpot Section (Sticky Note)

#### Node: 📋 Workflow Overview
- **Type / role:** Sticky Note; documentation.
- **Configuration choices:** Contains overview text + embedded YouTube reference.
- **Connections:** None.
- **Edge cases:** None.

Sticky note content (preserved):
- “## 📋 Feedspace Testimonials to HubSpot CRM Contacts”
- YouTube: `@[youtube](AtR84TFFiSI)`
- Setup steps: HubSpot App Token, webhook URL in Feedspace, activate workflow.

#### Node: 🔗 Trigger Section
- **Type / role:** Sticky Note; visual grouping for trigger/data processing.
- **Connections / edge cases:** None.

#### Node: 🏢 HubSpot Section
- **Type / role:** Sticky Note; visual grouping for HubSpot integration.
- **Connections / edge cases:** None.

---

### Block 2 — Webhook Trigger & Data Normalization
**Overview:** Receives the Feedspace event via POST and transforms potentially nested webhook data into a consistent flat schema used by subsequent nodes.  
**Nodes involved:**  
- Receive Testimonial (Webhook)  
- Extract Testimonial Data (Code)

#### Node: Receive Testimonial
- **Type / role:** Webhook trigger; entry point.
- **Configuration (interpreted):**
  - **HTTP Method:** POST  
  - **Path:** a generated UUID-like path segment (unique endpoint)
  - **Response mode:** “Respond with Respond to Webhook node” (the workflow must end with a Respond node on each path).
- **Outputs:** Sends webhook request data to “Extract Testimonial Data”.
- **Edge cases / failures:**
  - Feedspace sends unexpected schema or empty body.
  - If downstream does not respond (miswired), requests can hang/time out depending on n8n/webhook settings.
  - If webhook URL is changed, Feedspace integration will break until updated.

#### Node: Extract Testimonial Data
- **Type / role:** Code node (JavaScript); normalize payload fields.
- **Configuration (interpreted):**
  - Reads the first input item: `const input = $input.first().json;`
  - Treats webhook payload as `input.body || input`
  - Extracts nested objects:
    - `data = body.data || {}`
    - `response = data.response || {}`
  - Derives:
    - `name` defaulting to `"Unknown"`
    - `firstName` and `lastName` by splitting name on spaces
  - Emits a single normalized JSON object with fields such as:
    - `email`, `name`, `firstName`, `lastName`, `phone`, `position`, `organization`, `photo`
    - `rating`, `ratingType`, `comment`
    - `videoUrl`, `audioUrl`, `thumbnailUrl`
    - `feedId`, `feedUrl`, `formName`, `formUrl`, `projectName`, `projectLogo`, `createdAt`
    - `consent`, `liveMode`
- **Connections:**
  - Input: Receive Testimonial
  - Output: Has Email?
- **Edge cases / failures:**
  - If Feedspace changes field names (e.g., `reviewer_email` missing), `email` may end up empty.
  - `name.split(' ')` may produce only first name; last name becomes empty string (handled).
  - Some fields are assumed objects (e.g., `data.feed_form?.name`); optional chaining prevents runtime error, but could yield empty strings.

---

### Block 3 — Validation & Routing
**Overview:** Ensures email is present before attempting HubSpot operations. If missing, immediately returns HTTP 400 to the webhook caller.  
**Nodes involved:**  
- Has Email? (IF)  
- Respond - No Email (Respond to Webhook)

#### Node: Has Email?
- **Type / role:** IF node; branching logic.
- **Configuration (interpreted):**
  - Condition checks that `{{$json.email}}` is not empty.
  - It includes two similar “not empty” checks combined with **OR**, effectively treating either condition as sufficient (functionally redundant but harmless).
- **Connections:**
  - Input: Extract Testimonial Data
  - **True branch:** Upsert Contact
  - **False branch:** Respond - No Email
- **Edge cases / failures:**
  - If `email` contains only whitespace, “isNotEmpty” may still pass depending on n8n’s string evaluation; consider trimming in the Code node for stricter validation.
  - If the email is malformed but non-empty, HubSpot may reject it or upsert may behave unexpectedly.

#### Node: Respond - No Email
- **Type / role:** Respond to Webhook; terminates the request on the “no email” path.
- **Configuration (interpreted):**
  - HTTP status: **400**
  - Header: `Content-Type: application/json`
  - Body (stringified JSON): `{ success: false, error: 'No email provided in testimonial' }`
- **Connections:**
  - Input: Has Email? (false branch)
  - Output: none (ends execution path)
- **Edge cases / failures:**
  - None typical; only fails if upstream wiring is broken or node disabled.

---

### Block 4 — HubSpot Contact Upsert
**Overview:** Creates or updates a HubSpot contact using the testimonial’s email and basic profile fields (first name, last name, job title).  
**Nodes involved:**  
- Upsert Contact (HubSpot)

#### Node: Upsert Contact
- **Type / role:** HubSpot node; CRM contact upsert.
- **Configuration (interpreted):**
  - **Authentication:** HubSpot **App Token**
  - **Operation:** Upsert contact by **Email**
  - **Mapped fields:**
    - Email: `{{$json.email}}`
    - First name: `{{$json.firstName}}`
    - Last name: `{{$json.lastName}}`
    - Job title: `{{$json.position}}`
- **Connections:**
  - Input: Has Email? (true branch)
  - Output: Prepare Note Content
- **Key outputs used downstream:**
  - `vid` (contact ID) is used as the association target for notes.
- **Edge cases / failures:**
  - Invalid/expired app token → 401/403.
  - HubSpot rate limiting (429).
  - HubSpot validation errors (e.g., invalid email format).
  - If HubSpot node output structure changes or `vid` is not present, the note association will fail later.

---

### Block 5 — Note Composition, Note Creation, and Success Response
**Overview:** Builds a formatted testimonial note, creates a HubSpot Note object associated to the contact, then returns a success response to the webhook caller.  
**Nodes involved:**  
- Prepare Note Content (Set)  
- Create Testimonial Note (HTTP Request)  
- Respond - Success (Respond to Webhook)

#### Node: Prepare Note Content
- **Type / role:** Set node (raw JSON output); constructs note payload inputs.
- **Configuration (interpreted):**
  - Creates:
    - `contactId`: from HubSpot upsert output `{{$json.vid}}`
    - `timestamp`: `Date.now()` (milliseconds)
    - `noteBody`: a formatted multi-line string including:
      - Feed type, form name, project name
      - Rating line only if rating exists
      - Comment text
      - Video/audio URLs if present
      - Feed URL
  - References fields from **Extract Testimonial Data** via item lookup, e.g.:
    - `$('Extract Testimonial Data').item.json.comment`
- **Connections:**
  - Input: Upsert Contact
  - Output: Create Testimonial Note
- **Edge cases / failures:**
  - If the workflow runs with multiple items (batch), the explicit `$('Extract Testimonial Data').item` referencing can mismatch items. This workflow appears designed for single webhook events (single item), so it is typically safe.
  - If `vid` is missing, `contactId` becomes invalid and the note creation will fail.

#### Node: Create Testimonial Note
- **Type / role:** HTTP Request; calls HubSpot CRM Notes API directly.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.hubapi.com/crm/v3/objects/notes`
  - **Authentication:** predefined credential type `hubspotAppToken` (HubSpot App Token credential selected)
  - **JSON body:**
    - `properties.hs_note_body`: JSON-stringified note body text
    - `properties.hs_timestamp`: timestamp from previous node
    - `associations`: associates the note to a contact:
      - `to.id`: `contactId`
      - Association type: HubSpot-defined, `associationTypeId: 202` (Note → Contact association)
- **Connections:**
  - Input: Prepare Note Content
  - Output: Respond - Success
- **Edge cases / failures:**
  - 401/403 if token invalid or missing scopes.
  - 400 if association type id changes or invalid contact id.
  - 429 rate limits.
  - Network timeouts.
  - If `hs_note_body` is not properly stringified, HubSpot may reject it (this workflow does stringify it).

#### Node: Respond - Success
- **Type / role:** Respond to Webhook; terminates successful path.
- **Configuration (interpreted):**
  - HTTP status: **200**
  - Header: `Content-Type: application/json`
  - Body (stringified JSON) includes:
    - `success: true`
    - `contactId`: pulled from `$('Upsert Contact').item.json.vid`
    - `noteId`: `$json.id` (from HubSpot Notes API response)
    - message: `'Testimonial added to HubSpot'`
- **Connections:**
  - Input: Create Testimonial Note
  - Output: none (ends execution path)
- **Edge cases / failures:**
  - If the HTTP node doesn’t return `id`, `noteId` may be null/undefined.
  - As with earlier cross-node item references, multi-item execution could cause mismatches (unlikely with webhook single item).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Workflow Overview | Sticky Note | Workflow documentation / overview |  |  | ## 📋 Feedspace Testimonials to HubSpot CRM Contacts \| YouTube: `@[youtube](AtR84TFFiSI)` \| Setup: HubSpot App Token, add webhook URL to Feedspace, activate workflow |
| 🔗 Trigger Section | Sticky Note | Section label for trigger + processing |  |  | ### 🔗 Webhook Trigger and Data processing |
| 🏢 HubSpot Section | Sticky Note | Section label for HubSpot integration |  |  | ### 🏢 HubSpot Integration |
| Receive Testimonial | Webhook | Receives Feedspace testimonial via POST |  | Extract Testimonial Data | ### 🔗 Webhook Trigger and Data processing |
| Extract Testimonial Data | Code | Normalizes webhook payload to a flat schema | Receive Testimonial | Has Email? | ### 🔗 Webhook Trigger and Data processing |
| Has Email? | IF | Validates email presence and routes flow | Extract Testimonial Data | Upsert Contact (true), Respond - No Email (false) | ### 🔗 Webhook Trigger and Data processing |
| Upsert Contact | HubSpot | Create/update HubSpot contact by email | Has Email? (true) | Prepare Note Content | ### 🏢 HubSpot Integration |
| Prepare Note Content | Set | Builds formatted note content + association inputs | Upsert Contact | Create Testimonial Note | ### 🏢 HubSpot Integration |
| Create Testimonial Note | HTTP Request | Creates HubSpot Note and associates to Contact | Prepare Note Content | Respond - Success | ### 🏢 HubSpot Integration |
| Respond - No Email | Respond to Webhook | Returns 400 error to webhook caller | Has Email? (false) |  | ### 🔗 Webhook Trigger and Data processing |
| Respond - Success | Respond to Webhook | Returns 200 success + IDs to webhook caller | Create Testimonial Note |  | ### 🏢 HubSpot Integration |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **Feedspace Testimonials to HubSpot CRM Contacts**
   - Keep it **inactive** until testing is complete.

2) **Add Sticky Notes (optional but recommended)**
   - Add three Sticky Note nodes:
     - **📋 Workflow Overview**: paste the overview text and the YouTube reference `@[youtube](AtR84TFFiSI)`.
     - **🔗 Trigger Section**: “Webhook Trigger and Data processing”.
     - **🏢 HubSpot Section**: “HubSpot Integration”.

3) **Add the Webhook trigger**
   - Add node: **Webhook**
   - Name: **Receive Testimonial**
   - Configure:
     - **HTTP Method:** POST
     - **Path:** generate a unique path (n8n can auto-generate)
     - **Response mode:** use **Respond to Webhook** node
   - Copy the **Test URL** (later also the Production URL) to configure in **Feedspace** integrations.

4) **Add the Code node to normalize payload**
   - Add node: **Code**
   - Name: **Extract Testimonial Data**
   - Paste logic that:
     - reads `input.body || input`
     - extracts `data` and `data.response`
     - outputs normalized fields: email, name parts, comment, rating, media URLs, feed/project/form metadata, createdAt, consent, liveMode
   - Connect: **Receive Testimonial → Extract Testimonial Data**

5) **Add email validation**
   - Add node: **IF**
   - Name: **Has Email?**
   - Condition:
     - Check `email` **is not empty** (expression `{{$json.email}}`)
     - (Optional improvement) add a regex/email validation or trim whitespace in the Code node
   - Connect: **Extract Testimonial Data → Has Email?**

6) **Add “no email” webhook response**
   - Add node: **Respond to Webhook**
   - Name: **Respond - No Email**
   - Configure:
     - Response code: **400**
     - Header: `Content-Type: application/json`
     - Response body (text): stringified JSON indicating error (e.g., success false + message)
   - Connect: **Has Email? (false) → Respond - No Email**

7) **Configure HubSpot credentials**
   - In n8n, create credentials for **HubSpot App Token**:
     - Provide your **Private App Token** from HubSpot.
     - Ensure the app has permissions for CRM objects you use (Contacts, Notes, Associations as required by HubSpot).
   - You will use this credential in both the HubSpot node and the HTTP Request node (credential type: HubSpot App Token).

8) **Add HubSpot upsert contact**
   - Add node: **HubSpot**
   - Name: **Upsert Contact**
   - Authentication: **App Token**
   - Operation: **Upsert Contact** (by email)
   - Map fields:
     - Email: `{{$json.email}}`
     - First Name: `{{$json.firstName}}`
     - Last Name: `{{$json.lastName}}`
     - Job Title: `{{$json.position}}`
   - Connect: **Has Email? (true) → Upsert Contact**

9) **Add Set node to prepare note content**
   - Add node: **Set**
   - Name: **Prepare Note Content**
   - Mode: **Raw JSON output**
   - Create fields:
     - `contactId`: from HubSpot output contact id (in this workflow it uses `{{$json.vid}}`)
     - `timestamp`: `Date.now()`
     - `noteBody`: formatted multiline string that includes fields from **Extract Testimonial Data** (comment, rating, URLs, feedUrl, etc.)
   - Connect: **Upsert Contact → Prepare Note Content**

10) **Add HTTP Request to create HubSpot Note**
   - Add node: **HTTP Request**
   - Name: **Create Testimonial Note**
   - Method: **POST**
   - URL: `https://api.hubapi.com/crm/v3/objects/notes`
   - Authentication: **Predefined credential type → HubSpot App Token**
   - Body type: **JSON**
   - JSON body:
     - `properties.hs_note_body`: use JSON stringify around the note text
     - `properties.hs_timestamp`: timestamp value
     - `associations`: link to contact with HubSpot-defined association type id **202**
   - Connect: **Prepare Note Content → Create Testimonial Note**

11) **Add success response**
   - Add node: **Respond to Webhook**
   - Name: **Respond - Success**
   - Configure:
     - Response code: **200**
     - Header: `Content-Type: application/json`
     - Response body: JSON string containing:
       - `success: true`
       - `contactId`: from Upsert Contact output
       - `noteId`: from Create Testimonial Note response
       - message text
   - Connect: **Create Testimonial Note → Respond - Success**

12) **Test end-to-end**
   - Use the webhook **Test URL**, send a sample payload shaped like Feedspace’s event.
   - Confirm:
     - Missing email returns 400.
     - Valid email creates/updates a contact and creates an associated note.
   - Switch Feedspace to the **Production URL** and activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video reference embedded on the canvas | `@[youtube](AtR84TFFiSI)` |
| Setup requirements listed in the workflow note | Configure HubSpot App Token; add webhook URL to Feedspace integrations; activate workflow |
| AssociationTypeId used for Note → Contact | `associationTypeId: 202` (HubSpot-defined). If HubSpot changes association definitions, this may need updating in the HTTP Request node. |
| Single-item assumption | The workflow uses cross-node item references (e.g., `$('Extract Testimonial Data').item...`), which is safest when each execution handles exactly one testimonial (typical for webhook triggers). |