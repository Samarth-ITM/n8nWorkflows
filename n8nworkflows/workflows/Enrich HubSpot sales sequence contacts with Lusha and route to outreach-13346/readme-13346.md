Enrich HubSpot sales sequence contacts with Lusha and route to outreach

https://n8nworkflows.xyz/workflows/enrich-hubspot-sales-sequence-contacts-with-lusha-and-route-to-outreach-13346


# Enrich HubSpot sales sequence contacts with Lusha and route to outreach

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Sales Engagement Enrichment with Lusha*  
**Stated title:** *Enrich HubSpot sales sequence contacts with Lusha and route to outreach*

**Purpose:**  
Automatically enrich HubSpot contacts when they are added to a HubSpot Sales Sequence, using Lusha to retrieve verified contact details (email, phone, seniority, etc.). If enrichment yields a valid email, the workflow creates/exports a prospect to Outreach and updates the HubSpot contact. If not, it logs the skipped contact and alerts an SDR via Slack.

**Typical use cases:**
- SDR/AE outbound ops: auto-enrich newly sequenced contacts and feed them into an engagement platform (Outreach/Salesloft).
- Data hygiene: update CRM contact fields with enriched phone/title/industry.
- Ops monitoring: alert on failed enrichments for manual remediation.

### Logical blocks
**1.1 Input Reception (HubSpot trigger)**  
Detects sequence enrollment by monitoring a HubSpot contact property change.

**1.2 Enrichment & Prospect Normalization (Lusha + Code)**  
Calls Lusha to enrich by email, then builds a clean “prospect record” used downstream.

**1.3 Validation, Routing, Update & Notifications (IF + Outreach + HubSpot + Slack)**  
Routes valid prospects to Outreach and updates HubSpot; routes invalid prospects to logging and Slack notification.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (HubSpot trigger)

**Overview:**  
Starts the workflow when a HubSpot contact’s sequence enrollment count changes, indicating the contact was added to a sequence.

**Nodes involved:**
- **Contact Added to Sequence** (HubSpot Trigger)

#### Node: Contact Added to Sequence
- **Type / Role:** `hubspotTrigger` — webhook trigger on HubSpot contact events.
- **Configuration (interpreted):**
  - Event: `contact.propertyChange`
  - Property watched: `hs_sequences_actively_enrolled_count`
  - Uses HubSpot OAuth2 credentials.
- **Key fields/variables:**
  - The trigger output includes contact identifiers and `properties` (e.g., `properties.email`, `properties.firstname`, etc.).
- **Connections:**
  - **Output →** Enrich with Lusha
- **Version requirements:** Node `typeVersion: 1`.
- **Edge cases / failures:**
  - **OAuth scope/permission issues:** insufficient scopes to subscribe to events or read contact properties.
  - **Property not present:** if the portal doesn’t have `hs_sequences_actively_enrolled_count` or sequences aren’t enabled, events may not fire as expected.
  - **Webhook lifecycle:** if the webhook is deleted/disabled in HubSpot, trigger stops firing.
  - **No email present:** downstream Lusha enrichment is by email; missing/invalid CRM email will likely cause Lusha lookup failure/empty results.

---

### 2.2 Enrichment & Prospect Normalization (Lusha + Code)

**Overview:**  
Enriches the contact via Lusha (search by email) and then constructs a normalized prospect payload with fallback logic from HubSpot when Lusha fields are missing.

**Nodes involved:**
- **Enrich with Lusha** (Lusha community node)
- **Build Prospect Record** (Code)

#### Node: Enrich with Lusha
- **Type / Role:** `@lusha-org/n8n-nodes-lusha.lusha` — enrich a single contact via Lusha API.
- **Configuration (interpreted):**
  - Operation: `enrichSingle`
  - Search by: `email`
  - Email input expression: `{{ $json.properties.email }}`
  - Additional options: none set.
  - Uses Lusha API credential.
- **Connections:**
  - **Input ←** Contact Added to Sequence
  - **Output →** Build Prospect Record
- **Version requirements:**
  - Node `typeVersion: 1`
  - Requires installing the community node: `@lusha-org/n8n-nodes-lusha`.
- **Edge cases / failures:**
  - **Missing email** from HubSpot trigger payload → may return error or empty enrichment.
  - **Rate limits / quota** on Lusha API → transient errors.
  - **Auth errors** (invalid API key).
  - **Data shape changes** (fields like `primaryEmail`, `primaryPhone`) depending on Lusha plan/response.

#### Node: Build Prospect Record
- **Type / Role:** `code` — transforms the Lusha response into a consistent internal “prospect record”.
- **Configuration (interpreted):**
  - Pulls the original HubSpot trigger item explicitly:
    - `const crm = $('Contact Added to Sequence').first().json;`
  - Uses current node input as Lusha payload:
    - `const lusha = $json;`
  - Outputs a single JSON object containing:
    - `contactId` (from HubSpot `objectId`)
    - `email` (HubSpot contact email)
    - name fields with fallback: Lusha → HubSpot → empty string
    - `verifiedEmail` = `lusha.primaryEmail` (or empty)
    - `phone` = `lusha.primaryPhone` (or empty)
    - job/company metadata
    - `companySize` derived as `lusha.companySize[1]` if array
    - `hasValidEmail` boolean = `!!lusha.primaryEmail`
    - `enrichedAt` timestamp
- **Key expressions / variables used:**
  - Cross-node reference: `$('Contact Added to Sequence').first().json`
  - Optional chaining for safety: `crm.properties?.email`
- **Connections:**
  - **Input ←** Enrich with Lusha
  - **Output →** Has Valid Email?
- **Version requirements:** Node `typeVersion: 2` (uses modern Code node runtime).
- **Edge cases / failures:**
  - If the HubSpot trigger output does not include `objectId`, `contactId` becomes undefined → downstream HubSpot update fails.
  - If Lusha returns unexpected types (e.g., `companySize` not array), mapping may degrade (handled partially).
  - If Lusha returns no `primaryEmail`, the record is routed to “skipped”.

---

### 2.3 Validation, Routing, Update & Notifications

**Overview:**  
Checks whether a verified email exists. If yes, it both (a) creates a prospect in Outreach and (b) updates the HubSpot contact with enriched fields. If no, it logs the skip and sends a Slack alert.

**Nodes involved:**
- **Has Valid Email?** (IF)
- **Add to Outreach Sequence** (HTTP Request)
- **Update HubSpot Contact** (HubSpot)
- **Log Skipped Contact** (Code)
- **Notify Skip on Slack** (Slack)

#### Node: Has Valid Email?
- **Type / Role:** `if` — branching based on enrichment validity.
- **Configuration (interpreted):**
  - Condition: `{{ $json.hasValidEmail }}` is `true`
  - Boolean operator: “true”
- **Connections:**
  - **Input ←** Build Prospect Record
  - **True branch →** Add to Outreach Sequence **and** Update HubSpot Contact (two parallel outputs from the true path)
  - **False branch →** Log Skipped Contact
- **Version requirements:** Node `typeVersion: 2`.
- **Edge cases / failures:**
  - If `hasValidEmail` is missing/non-boolean, condition may evaluate unexpectedly (but it is always set by the Code node here).

#### Node: Add to Outreach Sequence
- **Type / Role:** `httpRequest` — creates a Prospect in Outreach via REST API.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://api.outreach.io/api/v2/prospects`
  - Body: JSON API format (`application/vnd.api+json`)
  - Authentication: OAuth2 via “generic credential type” (`oAuth2Api`)
  - Payload mapping:
    - `firstName`, `lastName`
    - `emails` array containing `verifiedEmail`
    - `homePhones` array containing `phone`
    - `title`, `company`
- **Connections:**
  - **Input ←** Has Valid Email? (true branch)
  - **Output:** none (terminal)
- **Version requirements:** Node `typeVersion: 4`.
- **Edge cases / failures:**
  - **Auth failure** (OAuth misconfigured/expired token).
  - **Payload validation errors** (e.g., Outreach requires different phone/email fields; using `homePhones` may not match your schema).
  - **Duplicate handling:** Outreach may reject duplicates or create multiple prospects depending on account settings—no dedupe logic is present.
  - **API format:** must keep `Content-Type: application/vnd.api+json` for JSON:API compatibility.

#### Node: Update HubSpot Contact
- **Type / Role:** `hubspot` — updates CRM fields with enriched values.
- **Configuration (interpreted):**
  - Resource: `contact`
  - Operation: `update`
  - `contactId`: `{{ $('Build Prospect Record').item.json.contactId }}`
  - Fields updated:
    - `phone`, `company`, `industry`, `jobtitle` from Build Prospect Record output
- **Connections:**
  - **Input ←** Has Valid Email? (true branch)
  - **Output:** none (terminal)
- **Version requirements:** Node `typeVersion: 2`.
- **Edge cases / failures:**
  - If `contactId` is undefined/invalid, HubSpot update will fail (404/validation).
  - Field names must match HubSpot internal property names (`jobtitle` is standard; custom portals may differ).
  - HubSpot rate limits can cause intermittent 429 errors; no retry strategy is configured here.

#### Node: Log Skipped Contact
- **Type / Role:** `code` — produces a structured “skipped” log record.
- **Configuration (interpreted):**
  - Outputs:
    - `status: 'skipped'`
    - `reason: 'No valid email found by Lusha'`
    - contact identifiers and name
    - `skippedAt` timestamp
- **Connections:**
  - **Input ←** Has Valid Email? (false branch)
  - **Output →** Notify Skip on Slack
- **Version requirements:** Node `typeVersion: 2`.
- **Edge cases / failures:**
  - If upstream fields (firstName/lastName/email) are missing, Slack message will be less informative but still works.

#### Node: Notify Skip on Slack
- **Type / Role:** `slack` — posts a message to a Slack channel for visibility.
- **Configuration (interpreted):**
  - Channel: `#enrichment-logs`
  - Message text includes name, email, and reason.
  - Uses Slack OAuth2 credential.
- **Connections:**
  - **Input ←** Log Skipped Contact
  - **Output:** none (terminal)
- **Version requirements:** Node `typeVersion: 2`.
- **Edge cases / failures:**
  - Slack OAuth token missing scopes (e.g., `chat:write`) → message fails.
  - Channel not found / bot not in channel → `channel_not_found` or similar.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Sales Engagement Enrichment | Sticky Note | Documentation/overview |  |  | ## Enrich Contacts Added to Sales Sequences with Lusha\n\n**Who it's for:** SDRs & AEs running outbound campaigns\n\n**What it does:** When a contact is added to a HubSpot sequence, Lusha enriches them with verified email, direct phone, and seniority. Valid contacts are pushed to your outreach tool; invalid ones are logged. CRM records are updated with enriched data and an SDR alert is sent for high-value prospects.\n\n### How it works\n1. HubSpot trigger fires when a contact property changes (sequence enrollment)\n2. Lusha enriches the contact with verified email and direct phone\n3. A code node builds the outreach-ready prospect record\n4. Valid contacts are pushed to Outreach/Salesloft + CRM is updated\n5. Invalid contacts are logged and an SDR is alerted on Slack\n\n### Setup\n1. Install the [Lusha community node](https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha)\n2. Add your Lusha API, HubSpot OAuth2, and Slack credentials\n3. Configure the HubSpot trigger property and Outreach API endpoint\n4. Set the Slack channel for skip notifications |
| 🔔 1. CRM Trigger | Sticky Note | Documentation for trigger block |  |  | Fires when a HubSpot contact's sequence enrollment count changes — indicating they were added to a sales sequence.\n\n**Nodes:** HubSpot Property Change Trigger\n\n💡 Customize the property name to match your CRM setup. |
| 🔍 2. Enrich & Build Prospect | Sticky Note | Documentation for enrichment block |  |  | Lusha enriches the contact by email. A code node validates the enrichment and builds a clean prospect record for outreach.\n\n**Nodes:** Lusha Enrich → Build Prospect Record → Has Valid Email? (IF)\n\n📖 [Lusha API docs](https://www.lusha.com/docs/) |
| 📤 3. Route, Update & Notify | Sticky Note | Documentation for routing block |  |  | Valid prospects are pushed to Outreach and their CRM records are updated. Invalid contacts are logged and the SDR is notified on Slack.\n\n**Nodes:** Outreach HTTP + Update HubSpot / Log Skipped + Slack Alert\n\n💡 Replace the Outreach API call with Salesloft or your engagement tool. |
| Contact Added to Sequence | HubSpot Trigger | Entry point: detect sequence enrollment via property change | — | Enrich with Lusha | Fires when a HubSpot contact's sequence enrollment count changes — indicating they were added to a sales sequence.\n\n**Nodes:** HubSpot Property Change Trigger\n\n💡 Customize the property name to match your CRM setup. |
| Enrich with Lusha | Lusha (community) | Enrich a contact by email via Lusha API | Contact Added to Sequence | Build Prospect Record | Lusha enriches the contact by email. A code node validates the enrichment and builds a clean prospect record for outreach.\n\n**Nodes:** Lusha Enrich → Build Prospect Record → Has Valid Email? (IF)\n\n📖 [Lusha API docs](https://www.lusha.com/docs/) |
| Build Prospect Record | Code | Normalize Lusha + HubSpot into a single prospect record | Enrich with Lusha | Has Valid Email? | Lusha enriches the contact by email. A code node validates the enrichment and builds a clean prospect record for outreach.\n\n**Nodes:** Lusha Enrich → Build Prospect Record → Has Valid Email? (IF)\n\n📖 [Lusha API docs](https://www.lusha.com/docs/) |
| Has Valid Email? | IF | Branch based on presence of verified email | Build Prospect Record | (true) Add to Outreach Sequence; Update HubSpot Contact; (false) Log Skipped Contact | Lusha enriches the contact by email. A code node validates the enrichment and builds a clean prospect record for outreach.\n\n**Nodes:** Lusha Enrich → Build Prospect Record → Has Valid Email? (IF)\n\n📖 [Lusha API docs](https://www.lusha.com/docs/) |
| Add to Outreach Sequence | HTTP Request | Create Outreach prospect from normalized record | Has Valid Email? (true) | — | Valid prospects are pushed to Outreach and their CRM records are updated. Invalid contacts are logged and the SDR is notified on Slack.\n\n**Nodes:** Outreach HTTP + Update HubSpot / Log Skipped + Slack Alert\n\n💡 Replace the Outreach API call with Salesloft or your engagement tool. |
| Update HubSpot Contact | HubSpot | Update HubSpot contact fields with enriched data | Has Valid Email? (true) | — | Valid prospects are pushed to Outreach and their CRM records are updated. Invalid contacts are logged and the SDR is notified on Slack.\n\n**Nodes:** Outreach HTTP + Update HubSpot / Log Skipped + Slack Alert\n\n💡 Replace the Outreach API call with Salesloft or your engagement tool. |
| Log Skipped Contact | Code | Build a structured skip log payload | Has Valid Email? (false) | Notify Skip on Slack | Valid prospects are pushed to Outreach and their CRM records are updated. Invalid contacts are logged and the SDR is notified on Slack.\n\n**Nodes:** Outreach HTTP + Update HubSpot / Log Skipped + Slack Alert\n\n💡 Replace the Outreach API call with Salesloft or your engagement tool. |
| Notify Skip on Slack | Slack | Alert SDR/team about skipped enrichment | Log Skipped Contact | — | Valid prospects are pushed to Outreach and their CRM records are updated. Invalid contacts are logged and the SDR is notified on Slack.\n\n**Nodes:** Outreach HTTP + Update HubSpot / Log Skipped + Slack Alert\n\n💡 Replace the Outreach API call with Salesloft or your engagement tool. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: **Sales Engagement Enrichment with Lusha**

2. **Add HubSpot Trigger node**
   - Node type: **HubSpot Trigger**
   - Name: **Contact Added to Sequence**
   - Event: **contact.propertyChange**
   - Property: **hs_sequences_actively_enrolled_count**
   - Credentials: **HubSpot OAuth2**
     - Ensure scopes allow reading contacts and managing webhooks/subscriptions as needed by the trigger.

3. **Install and add Lusha community node**
   - Install package on your n8n instance:  
     - `@lusha-org/n8n-nodes-lusha` (community node)
   - Add node: **Lusha**
   - Name: **Enrich with Lusha**
   - Operation: **enrichSingle**
   - Search by: **email**
   - Email value (expression): `{{ $json.properties.email }}`
   - Credentials: **Lusha API** (API key/token as required by the node)

4. **Add Code node to build the prospect record**
   - Node type: **Code**
   - Name: **Build Prospect Record**
   - Paste logic that:
     - reads HubSpot trigger payload using: `$('Contact Added to Sequence').first().json`
     - reads Lusha response using: `$json`
     - outputs a normalized object with:
       - `contactId`, `firstName`, `lastName`, `verifiedEmail`, `phone`, `jobTitle`, `company`, etc.
       - `hasValidEmail` set from existence of `primaryEmail`
       - timestamps `enrichedAt`
   - Connect: **Enrich with Lusha → Build Prospect Record**

5. **Add IF node for validation**
   - Node type: **IF**
   - Name: **Has Valid Email?**
   - Condition:
     - Boolean condition checking `{{ $json.hasValidEmail }}` is **true**
   - Connect: **Build Prospect Record → Has Valid Email?**

6. **True branch: Add HTTP Request to create Outreach prospect**
   - Node type: **HTTP Request**
   - Name: **Add to Outreach Sequence**
   - Method: **POST**
   - URL: `https://api.outreach.io/api/v2/prospects`
   - Authentication: **OAuth2** (generic OAuth2 credential in n8n)
     - Configure token URL/auth URL per Outreach requirements (in the OAuth2 credential).
   - Headers:
     - `Content-Type: application/vnd.api+json`
   - Body type: **JSON**
   - JSON body mapping:
     - `firstName`, `lastName`
     - `emails: [verifiedEmail]`
     - `homePhones: [phone]`
     - `title`, `company`
   - Connect: **Has Valid Email? (true) → Add to Outreach Sequence**

7. **True branch: Add HubSpot node to update contact**
   - Node type: **HubSpot**
   - Name: **Update HubSpot Contact**
   - Resource: **contact**
   - Operation: **update**
   - Contact ID expression: `{{ $('Build Prospect Record').item.json.contactId }}`
   - Update fields (expressions):
     - `phone` = `{{ $('Build Prospect Record').item.json.phone }}`
     - `company` = `{{ $('Build Prospect Record').item.json.company }}`
     - `industry` = `{{ $('Build Prospect Record').item.json.industry }}`
     - `jobtitle` = `{{ $('Build Prospect Record').item.json.jobTitle }}`
   - Credentials: **HubSpot OAuth2**
   - Connect: **Has Valid Email? (true) → Update HubSpot Contact**

8. **False branch: Add Code node for logging**
   - Node type: **Code**
   - Name: **Log Skipped Contact**
   - Create a JSON output with:
     - `status: skipped`
     - `reason: No valid email found by Lusha`
     - contact/name/email and a `skippedAt` timestamp
   - Connect: **Has Valid Email? (false) → Log Skipped Contact**

9. **False branch: Add Slack node to notify**
   - Node type: **Slack**
   - Name: **Notify Skip on Slack**
   - Operation: **Post message** (or equivalent in your Slack node version)
   - Channel: `#enrichment-logs` (change to your channel)
   - Text: include variables like `{{ $json.name }}`, `{{ $json.email }}`, `{{ $json.reason }}`
   - Credentials: **Slack OAuth2** (needs `chat:write` and channel access)
   - Connect: **Log Skipped Contact → Notify Skip on Slack**

10. **(Optional but recommended) Add operational hardening**
   - Add retry/error workflows for Outreach/HubSpot 429/5xx.
   - Add dedupe logic (search existing Outreach prospect by email before POST).
   - Add a guard for missing HubSpot `properties.email` before calling Lusha.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Lusha community node package: `@lusha-org/n8n-nodes-lusha` | https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha |
| Lusha API documentation | https://www.lusha.com/docs/ |
| Replace the Outreach HTTP call with Salesloft or another engagement tool if needed | Referenced in workflow notes |
| Configure HubSpot trigger property to match your CRM setup (`hs_sequences_actively_enrolled_count`) | HubSpot trigger block note |
| Slack channel for skip notifications is `#enrichment-logs` (ensure bot/user is a member) | Slack node configuration / notes |