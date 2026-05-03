Sync CRM contacts with Mailchimp and Pipedrive

https://n8nworkflows.xyz/workflows/sync-crm-contacts-with-mailchimp-and-pipedrive-13225


# Sync CRM contacts with Mailchimp and Pipedrive

## 1. Workflow Overview

**Title:** Sync CRM contacts with Mailchimp and Pipedrive  
**Workflow name (internal):** CRM Contact Sync with Mailchimp and Pipedrive

**Purpose:**  
This workflow receives contact change events from a “source CRM” via webhook, enriches each contact ID into a full profile via the source CRM API, normalizes fields into a canonical schema, then upserts the person in **Pipedrive** (create if missing, update if found). After a successful Pipedrive operation, it prepares a small payload and sends it to **Mailchimp** to keep marketing audiences aligned.

**Primary use cases:**
- Keeping Pipedrive as the operational CRM while another CRM (HubSpot/Salesforce/custom) is the system-of-record for contact changes.
- Feeding marketing tooling (Mailchimp) with “new vs updated” contact events via tags.

### 1.1 Trigger & Input Fan-out
Receives webhook payload(s) and splits potentially batched contact IDs into individual items.

### 1.2 Source CRM Enrichment & Normalization
Fetches each contact profile from the source CRM REST API and normalizes vendor-specific fields into a stable schema.

### 1.3 Pipedrive Match + Create/Update
Searches Pipedrive for a person by email, determines existence, then routes to create or update.

### 1.4 Mailchimp Notification (Success Path)
Unifies both success branches (create/update), builds a Mailchimp-ready payload, and sends the sync notification.

> Note: A sticky note mentions an “Error Trigger”/error-routing mechanism, but **no Error Trigger node or error path exists in the provided JSON**. As-is, errors will fail the execution without the described Mailchimp alert routing.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Input Fan-out

**Overview:**  
Accepts inbound POST webhook calls containing one or multiple contact identifiers, then iterates over contacts one-by-one to process them deterministically downstream.

**Nodes involved:**
- Incoming Contact Webhook
- Split Contacts

#### Node: Incoming Contact Webhook
- **Type / role:** `Webhook` (entry point trigger)
- **Configuration (interpreted):**
  - HTTP Method: **POST**
  - Path: `crm-contact-sync` (endpoint becomes `{n8n-base-url}/webhook/crm-contact-sync` depending on environment)
- **Key expressions/variables:**
  - Expects `contactId` to exist somewhere in the incoming body per item (downstream uses `$json.contactId`)
- **Connections:**
  - Output → **Split Contacts**
- **Version:** typeVersion **1**
- **Edge cases / failures:**
  - If the payload does not contain `contactId` (or the structure is not iterable as items), downstream nodes will fail or fetch the wrong URL.
  - Webhook authentication/verification is not configured; endpoint may be publicly callable unless protected by n8n instance controls.

#### Node: Split Contacts
- **Type / role:** `SplitInBatches` (fan-out / batching control)
- **Configuration (interpreted):**
  - Uses default batching options (batch size not explicitly set in parameters)
  - Intended to iterate through an array of contacts or items produced by the webhook
- **Connections:**
  - Input ← **Incoming Contact Webhook**
  - Output → **Fetch Contact Details (Source CRM)**
- **Version:** typeVersion **3**
- **Edge cases / failures:**
  - If the webhook posts a single object rather than an array of items, behavior depends on how n8n materializes incoming data into items. You may need a Set/Code node to convert `contacts[]` into items before splitting.
  - Without explicit batch size, large batches may increase execution time and API rate-limit risk.

---

### Block 2 — Source CRM Enrichment & Normalization

**Overview:**  
For each contact ID, fetches the full contact record from the source CRM API and normalizes fields into a shared schema used by Pipedrive and Mailchimp logic.

**Nodes involved:**
- Fetch Contact Details (Source CRM)
- Normalize Contact Data

#### Node: Fetch Contact Details (Source CRM)
- **Type / role:** `HTTP Request` (REST enrichment call)
- **Configuration (interpreted):**
  - Method: default (not shown; typically GET for a simple URL fetch)
  - URL: `https://api.sourcecrm.com/contacts/{{$json.contactId}}`
  - Authentication: **predefinedCredentialType** using **httpHeaderAuth**
    - This implies an n8n credential of type “HTTP Header Auth” is required (e.g., API key/bearer token in headers).
- **Key expressions/variables:**
  - `{{$json.contactId}}` interpolated into URL path
- **Connections:**
  - Input ← **Split Contacts**
  - Output → **Normalize Contact Data**
- **Version:** typeVersion **4**
- **Edge cases / failures:**
  - Missing/empty `contactId` → invalid URL and likely 404 or request error.
  - Auth header misconfigured/expired → 401/403.
  - Rate limiting / timeouts from the source CRM.
  - Response shape differs from what normalization expects (e.g., nested properties).

#### Node: Normalize Contact Data
- **Type / role:** `Code` (data shaping / canonical schema)
- **Configuration (interpreted):**
  - JavaScript transforms the source CRM response into:
    - `contactId`, `email`, `firstName`, `lastName`, `phone`, `company`, `updatedAt`
  - It attempts multiple field paths to support different CRM payload shapes:
    - Direct fields: `src.email`, `src.firstName`, etc.
    - HubSpot-like: `src.properties?.email?.value`, etc.
- **Key logic (important variables):**
  - `const src = $input.item.json;`
  - `updatedAt` fallback: `src.updatedAt || src.properties?.lastmodifieddate || new Date().toISOString()`
- **Connections:**
  - Output → **Search Person by Email**
  - Output → **Combine Contact & Search** (as input index 0 to merge)
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - If `email` is empty, Pipedrive search will be ineffective and may return many/none depending on Pipedrive behavior; downstream may create duplicates or update unintended records.
  - `lastmodifieddate` may be an object or differently formatted; stored as-is.

---

### Block 3 — Pipedrive Match + Create/Update

**Overview:**  
Looks up a person in Pipedrive by email, merges the search result with normalized contact data, then routes to either create a new person or update an existing one.

**Nodes involved:**
- Search Person by Email
- Combine Contact & Search
- Determine Existence & Prepare Data
- Is Person Existing?
- Update Person
- Create Person
- Prepare Updated Notification
- Prepare New Notification

#### Node: Search Person by Email
- **Type / role:** `Pipedrive` (search)
- **Configuration (interpreted):**
  - Resource: **person**
  - Operation: **search**
  - Term: `={{ $json.email }}`
  - Return All: **true** (returns all matches)
- **Connections:**
  - Input ← **Normalize Contact Data**
  - Output → **Combine Contact & Search** (as input index 1)
- **Version:** typeVersion **1**
- **Edge cases / failures:**
  - Missing/invalid Pipedrive credentials.
  - Empty email term may return unexpected results or no results.
  - Multiple people with same email: code selects the *first* match only.

#### Node: Combine Contact & Search
- **Type / role:** `Merge` (combine two streams)
- **Configuration (interpreted):**
  - Mode: **mergeByIndex**
  - Input 0: normalized contact
  - Input 1: Pipedrive search response
  - Output: a single item with properties combined
- **Connections:**
  - Inputs ← **Normalize Contact Data**, **Search Person by Email**
  - Output → **Determine Existence & Prepare Data**
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - `mergeByIndex` assumes both inputs emit the same number of items in the same order. Here, it’s effectively 1-to-1 per contact, so it should work, but any mismatch can produce incorrect pairing.
  - Field name collisions: whichever side writes the same keys last will overwrite (depends on merge behavior). This matters because the next node expects a `data.items[]` structure from the search response.

#### Node: Determine Existence & Prepare Data
- **Type / role:** `Code` (routing decision preparation)
- **Configuration (interpreted):**
  - Checks for search results at `item.data.items`
  - Sets:
    - `exists` (boolean)
    - `personId` (from first match)
  - Returns `{ ...item, exists, personId }`
- **Key logic:**
  - Existence condition:
    - `if (item.data && item.data.items && item.data.items.length > 0) ...`
  - First match:
    - `personId = item.data.items[0].item.id;`
- **Connections:**
  - Output → **Is Person Existing?**
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - If Pipedrive search response format changes (or merge overwrote `data`), `exists` will be false and the workflow will create duplicates.
  - When multiple matches exist, always uses the first; no disambiguation.

#### Node: Is Person Existing?
- **Type / role:** `IF` (branching)
- **Configuration (interpreted):**
  - Condition: boolean `={{ $json.exists }}`
  - **True branch (exists = true):** Update
  - **False branch:** Create
- **Connections:**
  - True → **Update Person**
  - False → **Create Person**
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - Non-boolean `exists` values (e.g., string `"false"`) can lead to wrong branching depending on coercion.

#### Node: Update Person
- **Type / role:** `Pipedrive` (update)
- **Configuration (interpreted):**
  - Resource: **person**
  - Operation: **update**
  - Person ID: `={{ $json.personId }}`
  - `updateFields`: empty object in JSON (so no fields are actually updated unless configured in UI)
- **Connections:**
  - Output → **Prepare Updated Notification**
- **Version:** typeVersion **1**
- **Edge cases / failures:**
  - If `personId` is null/empty → request failure.
  - As configured, no update fields are supplied; execution may succeed but change nothing (depending on Pipedrive API behavior and n8n node implementation).
  - Required: Pipedrive credentials.

#### Node: Create Person
- **Type / role:** `Pipedrive` (create)
- **Configuration (interpreted):**
  - Resource: **person**
  - Operation: default create (operation not explicitly set; node uses “create” when not set in many versions, but this is version-dependent)
  - Name: `={{ $json.firstName + ' ' + $json.lastName }}`
  - `additionalFields`: empty
- **Connections:**
  - Output → **Prepare New Notification**
- **Version:** typeVersion **1**
- **Edge cases / failures:**
  - If first/last name empty, creates a person with a blank/odd name string.
  - Email is not mapped here; without mapping email into additional fields, the created person may not have an email (depends on what fields are configured in the node UI but not reflected in this JSON).
  - Required: Pipedrive credentials.

#### Node: Prepare Updated Notification
- **Type / role:** `Code` (decorate payload for downstream notification)
- **Configuration (interpreted):**
  - Adds `notificationType: 'updated'` to the item
- **Connections:**
  - Output → **Merge Notifications** (input index 0)
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - If upstream Pipedrive node returns a different structure, this node still spreads `...d` and continues, potentially losing normalized fields if they are not present anymore.

#### Node: Prepare New Notification
- **Type / role:** `Code` (decorate payload for downstream notification)
- **Configuration (interpreted):**
  - Adds `notificationType: 'new'`
- **Connections:**
  - Output → **Merge Notifications** (input index 1)
- **Version:** typeVersion **2**
- **Edge cases / failures:** same as above.

---

### Block 4 — Mailchimp Notification (Success Path)

**Overview:**  
Converges create/update success paths, builds a small Mailchimp payload (including a “New/Updated” tag), then sends it to Mailchimp.

**Nodes involved:**
- Merge Notifications
- Build Mailchimp Payload
- Send Sync Notification

#### Node: Merge Notifications
- **Type / role:** `Merge` (converge branches)
- **Configuration (interpreted):**
  - Mode: **mergeByIndex**
  - Input 0: updated branch items
  - Input 1: new branch items
- **Connections:**
  - Inputs ← **Prepare Updated Notification**, **Prepare New Notification**
  - Output → **Build Mailchimp Payload**
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - `mergeByIndex` is usually not the best “union” for mutually exclusive branches. If only one branch runs, the merge node’s behavior can be surprising (depending on n8n merge mode semantics). Many workflows use “Pass-through” or “Wait”/“Append” patterns instead.
  - If this node doesn’t receive both inputs, it may not emit output (workflow could stall after create/update).

#### Node: Build Mailchimp Payload
- **Type / role:** `Code` (Mailchimp payload shaping)
- **Configuration (interpreted):**
  - Extracts: `email, firstName, lastName, notificationType`
  - Outputs:
    - `email`
    - `firstName`
    - `lastName`
    - `tag`: `'New'` if new else `'Updated'`
- **Connections:**
  - Output → **Send Sync Notification**
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - Missing email → Mailchimp upsert will fail or create unusable member.
  - Tag logic assumes only `new` or `updated`; other values become `Updated`.

#### Node: Send Sync Notification
- **Type / role:** `Mailchimp` (audience member upsert/notification)
- **Configuration (interpreted):**
  - The JSON shows:
    - `mergeFieldsUi.mergeFieldsValues` contains two empty objects (placeholders), implying merge fields were intended but not actually mapped.
  - List/Audience, operation, and member identification are **not visible** in the JSON snippet; they must be set in node UI/credentials.
- **Connections:**
  - Input ← **Build Mailchimp Payload**
  - Output: none (terminal)
- **Version:** typeVersion **1**
- **Edge cases / failures:**
  - Missing Mailchimp credentials / wrong audience ID.
  - Member upsert requires email hashing in some endpoints; the node handles it when configured properly.
  - Merge fields and tags must exist/configured in Mailchimp (or Mailchimp may ignore/raise errors depending on endpoint).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Incoming Contact Webhook | Webhook | Entry point; receives contact IDs via POST | — | Split Contacts | **Trigger & Source**: This section ingests webhook events… normalises into a shared format… |
| Split Contacts | SplitInBatches | Iterates over batched webhook items | Incoming Contact Webhook | Fetch Contact Details (Source CRM) | **Trigger & Source**: This section ingests webhook events… normalises into a shared format… |
| Fetch Contact Details (Source CRM) | HTTP Request | Enrich contact ID into full contact profile | Split Contacts | Normalize Contact Data | **Trigger & Source**: This section ingests webhook events… normalises into a shared format… |
| Normalize Contact Data | Code | Canonical field mapping for downstream | Fetch Contact Details (Source CRM) | Search Person by Email; Combine Contact & Search | **Trigger & Source**: This section ingests webhook events… normalises into a shared format… |
| Search Person by Email | Pipedrive | Look up person candidates by email | Normalize Contact Data | Combine Contact & Search | **Pipedrive Sync**: Nodes in this block handle… Search Person by Email… create/update… |
| Combine Contact & Search | Merge | Combine normalized contact + Pipedrive search result | Normalize Contact Data; Search Person by Email | Determine Existence & Prepare Data | **Pipedrive Sync**: Nodes in this block handle… Search Person by Email… create/update… |
| Determine Existence & Prepare Data | Code | Decide create vs update; extract personId | Combine Contact & Search | Is Person Existing? | **Pipedrive Sync**: Nodes in this block handle… Search Person by Email… create/update… |
| Is Person Existing? | IF | Branch to create or update | Determine Existence & Prepare Data | Update Person; Create Person | **Pipedrive Sync**: Nodes in this block handle… Search Person by Email… create/update… |
| Update Person | Pipedrive | Update existing person | Is Person Existing? (true) | Prepare Updated Notification | **Pipedrive Sync**: Nodes in this block handle… Search Person by Email… create/update… |
| Create Person | Pipedrive | Create new person | Is Person Existing? (false) | Prepare New Notification | **Pipedrive Sync**: Nodes in this block handle… Search Person by Email… create/update… |
| Prepare Updated Notification | Code | Mark notification type = updated | Update Person | Merge Notifications | **Mailchimp Notification & Error Handling**: Once Pipedrive confirms… tag (New/Updated)… Error Trigger path… *(note: not present in workflow)* |
| Prepare New Notification | Code | Mark notification type = new | Create Person | Merge Notifications | **Mailchimp Notification & Error Handling**: Once Pipedrive confirms… tag (New/Updated)… Error Trigger path… *(note: not present in workflow)* |
| Merge Notifications | Merge | Converge create/update branch | Prepare Updated Notification; Prepare New Notification | Build Mailchimp Payload | **Mailchimp Notification & Error Handling**: Once Pipedrive confirms… tag (New/Updated)… Error Trigger path… *(note: not present in workflow)* |
| Build Mailchimp Payload | Code | Shape data for Mailchimp | Merge Notifications | Send Sync Notification | **Mailchimp Notification & Error Handling**: Once Pipedrive confirms… tag (New/Updated)… Error Trigger path… *(note: not present in workflow)* |
| Send Sync Notification | Mailchimp | Upsert/notify Mailchimp audience | Build Mailchimp Payload | — | **Mailchimp Notification & Error Handling**: Once Pipedrive confirms… tag (New/Updated)… Error Trigger path… *(note: not present in workflow)* |
| Overview | Sticky Note | Documentation | — | — | ## How it works… ## Setup steps… |
| Trigger & Source | Sticky Note | Documentation | — | — | ## Trigger & Source… |
| Pipedrive Sync | Sticky Note | Documentation | — | — | ## Pipedrive Sync… |
| Notification & Error Handling | Sticky Note | Documentation | — | — | ## Mailchimp Notification & Error Handling… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **CRM Contact Sync with Mailchimp and Pipedrive**
   - (Optional) Set **Settings → Execution order** to `v1` to match the workflow.

2. **Add “Incoming Contact Webhook”**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: `crm-contact-sync`
   - Save the node and copy the **Test URL** / **Production URL**.
   - In your source CRM, configure a webhook to POST to that URL with payload items containing `contactId`.

3. **Add “Split Contacts”**
   - Node type: **Split In Batches**
   - Leave options default (or set a batch size if you expect large batches).
   - Connect: **Incoming Contact Webhook → Split Contacts**

4. **Add “Fetch Contact Details (Source CRM)”**
   - Node type: **HTTP Request**
   - Method: **GET**
   - URL: `https://api.sourcecrm.com/contacts/{{$json.contactId}}`
   - Authentication:
     - Choose **Predefined Credential Type**
     - Credential type: **HTTP Header Auth**
     - Create credential (example):
       - Header Name: `Authorization`
       - Header Value: `Bearer <YOUR_SOURCE_CRM_TOKEN>`
   - Connect: **Split Contacts → Fetch Contact Details (Source CRM)**

5. **Add “Normalize Contact Data”**
   - Node type: **Code**
   - Paste the logic that maps incoming fields into:
     - `contactId, email, firstName, lastName, phone, company, updatedAt`
   - Connect: **Fetch Contact Details (Source CRM) → Normalize Contact Data**

6. **Add “Search Person by Email” (Pipedrive)**
   - Node type: **Pipedrive**
   - Credentials: configure Pipedrive API token/OAuth (as supported by your n8n)
   - Resource: **Person**
   - Operation: **Search**
   - Term: `{{$json.email}}`
   - Return All: enabled
   - Connect: **Normalize Contact Data → Search Person by Email**

7. **Add “Combine Contact & Search”**
   - Node type: **Merge**
   - Mode: **Merge By Index**
   - Connect:
     - **Normalize Contact Data → Combine Contact & Search** (Input 1 / index 0)
     - **Search Person by Email → Combine Contact & Search** (Input 2 / index 1)

8. **Add “Determine Existence & Prepare Data”**
   - Node type: **Code**
   - Logic:
     - Inspect `data.items[]` from the Pipedrive search response
     - Set `exists` boolean and `personId` from first match
   - Connect: **Combine Contact & Search → Determine Existence & Prepare Data**

9. **Add “Is Person Existing?”**
   - Node type: **IF**
   - Condition: **Boolean** value1 = `{{$json.exists}}` is true
   - Connect: **Determine Existence & Prepare Data → Is Person Existing?**

10. **Add “Update Person”**
   - Node type: **Pipedrive**
   - Resource: **Person**
   - Operation: **Update**
   - Person ID: `{{$json.personId}}`
   - Map the fields you want to update (recommended at least email/name/phone/company as applicable)
   - Connect: **Is Person Existing? (true) → Update Person**

11. **Add “Create Person”**
   - Node type: **Pipedrive**
   - Resource: **Person**
   - Operation: **Create**
   - Name: `{{$json.firstName + ' ' + $json.lastName}}`
   - Also map **email** (and other fields) in the node fields/additional fields so the created person is searchable next time.
   - Connect: **Is Person Existing? (false) → Create Person**

12. **Add “Prepare Updated Notification”**
   - Node type: **Code**
   - Add `notificationType: 'updated'` to the current item
   - Connect: **Update Person → Prepare Updated Notification**

13. **Add “Prepare New Notification”**
   - Node type: **Code**
   - Add `notificationType: 'new'`
   - Connect: **Create Person → Prepare New Notification**

14. **Add “Merge Notifications”**
   - Node type: **Merge**
   - If you want to match the JSON exactly: set mode **Merge By Index**
   - Connect:
     - **Prepare Updated Notification → Merge Notifications** (Input 1/index 0)
     - **Prepare New Notification → Merge Notifications** (Input 2/index 1)
   - Practical note: for mutually exclusive branches, consider using a merge mode that appends/pass-through to avoid waiting for both inputs.

15. **Add “Build Mailchimp Payload”**
   - Node type: **Code**
   - Output:
     - `email`, `firstName`, `lastName`
     - `tag`: `New` vs `Updated` based on `notificationType`
   - Connect: **Merge Notifications → Build Mailchimp Payload**

16. **Add “Send Sync Notification”**
   - Node type: **Mailchimp**
   - Credentials: configure Mailchimp API key/OAuth in n8n
   - Configure:
     - Audience/List ID (required)
     - Operation typically “Upsert member” / “Create or Update” (exact naming depends on node version)
     - Map email to the member identifier
     - Map merge fields (FNAME/LNAME) if used by your audience
     - Apply tag from `tag` if your chosen operation supports tagging (or use a separate “Tag member” operation if needed)
   - Connect: **Build Mailchimp Payload → Send Sync Notification**

17. **Testing**
   - Execute the workflow in test mode.
   - POST sample body to the webhook, e.g. an item containing `{ "contactId": "123" }` (or an array of items depending on your source CRM).
   - Verify:
     - Source CRM request resolves
     - Normalization yields a non-empty email
     - Pipedrive create/update happens as expected
     - Mailchimp member is created/updated and tagged

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works” + setup steps describing webhook → source CRM enrichment → normalize → Pipedrive create/update → Mailchimp mirror; plus a mention of error routing to a separate Mailchimp list | Sticky note **Overview** |
| The “Notification & Error Handling” note references an isolated **Error Trigger** and an alert-to-operations Mailchimp call, but these nodes/paths are **not present** in the actual workflow JSON. | Sticky note **Mailchimp Notification & Error Handling** (documentation mismatch vs implementation) |
| Disclaimer (provided by user): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | User-provided disclaimer/context |

