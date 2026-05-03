Sync contacts, consent, and segments between KlickTipp and Pipedrive

https://n8nworkflows.xyz/workflows/sync-contacts--consent--and-segments-between-klicktipp-and-pipedrive-13233


# Sync contacts, consent, and segments between KlickTipp and Pipedrive

## 1. Workflow Overview

**Title:** Sync contacts, consent, and segments between KlickTipp and Pipedrive  
**Purpose:** Implements a near real-time **two-way synchronization** between **KlickTipp subscribers** and **Pipedrive Persons**, including:
- Core contact fields (name, email, birthday, IDs)
- Consent/marketing status (Subscribed / No consent / Unsubscribed)
- Segmentation mapping (**KlickTipp Tags ↔ Pipedrive Labels**)
- Deletion handling (Pipedrive delete → KlickTipp delete)

### 1.1 Logical Blocks (functional grouping)

1. **KlickTipp → Pipedrive: Trigger + Fetch full contact**
   - Triggered by KlickTipp tagging webhook; fetches subscriber data needed to decide create/update/unsubscribe.

2. **KlickTipp → Pipedrive: Route by existence & subscription**
   - Uses stored Pipedrive Person ID inside KlickTipp to determine create vs update, and handles unsubscribed state.

3. **KlickTipp → Pipedrive: Transfer contact data + write back ID**
   - Creates/updates the Pipedrive Person and then writes the resulting Pipedrive Person ID back into KlickTipp.

4. **KlickTipp → Pipedrive: Segmentation (Tags → Labels)**
   - If subscriber has specific tags, patch Pipedrive labels accordingly.

5. **Pipedrive → KlickTipp: Trigger + marketing status fetch**
   - Triggered by Pipedrive webhook; fetches person to read `marketing_status` reliably.

6. **Pipedrive → KlickTipp: Route changes (SOI/DOI/update/unsubscribe/delete)**
   - Routes based on marketing status, “change_source”, and delete action; prevents UI-only path from handling API/deletion events.

7. **Pipedrive → KlickTipp: Apply changes + Segmentation (Labels → Tags)**
   - Creates/updates/unsubscribes/deletes subscriber in KlickTipp; then applies tags based on label IDs.

---

## 2. Block-by-Block Analysis

### Block A — KlickTipp → Pipedrive (Trigger & fetch contact)
**Overview:** Starts when KlickTipp fires a webhook/tagging event. Fetches the full subscriber record so later nodes can inspect tags, status, and stored Pipedrive ID.  
**Nodes involved:**  
- **Contact tagged in KlickTipp** (Trigger)  
- **Get contact data for tagging** (KlickTipp Get)

#### Node: Contact tagged in KlickTipp
- **Type / role:** `klicktippTrigger` (community) — inbound webhook trigger from KlickTipp.
- **Config:** Webhook path is set to a UUID-like string; KlickTipp configured to call it via campaign webhook/activation tag.
- **Outputs:** Subscriber payload including fields like `id`, `email`, `fullname`, `status`, `fieldBirthday`, `tags`, and custom fields such as `field227685` if present.
- **Connections:** → **Get contact data for tagging**
- **Edge cases:**
  - Webhook not configured in KlickTipp → no executions.
  - Missing expected fields (e.g., `fieldBirthday`) → downstream expressions must tolerate nulls (some do not).
  - Credential mismatch (trigger uses “DEMO KlickTipp account”) vs action nodes may use another.

#### Node: Get contact data for tagging
- **Type / role:** `klicktipp` (community) — `subscriber.get` to fetch full subscriber by ID.
- **Config:** Fetch by `subscriberId = {{ $('Contact tagged in KlickTipp').item.json.id }}` with `identifierType = id`.
- **Outputs:** Full subscriber including `tags` array and custom field `field227685` (used as “Pipedrive Person ID”).
- **Connections:** → **Check subscription**
- **Failure modes:**
  - Invalid/expired KlickTipp API session → auth error.
  - Subscriber not found (rare if trigger payload stale) → node fails.

---

### Block B — KlickTipp → Pipedrive (Route by existence & subscription)
**Overview:** Decides whether to create a new Pipedrive Person, update an existing one, or mark as unsubscribed based on KlickTipp subscriber status and presence of stored Pipedrive ID in KlickTipp.  
**Nodes involved:**  
- **Check subscription** (Switch)

#### Node: Check subscription
- **Type / role:** `switch` — routing logic.
- **Config choices:**
  - **“Person does not exist”**: true if `!$json.field227685` AND `$json.status !== 'Unsubscribed'`
  - **“Person exists”**: true if `!!$json.field227685` AND `$json.status !== 'Unsubscribed'`
  - **“Person exists → unsubscribe”**: true if `!!$json.field227685` AND `$json.status === 'Unsubscribed'`
  - `allMatchingOutputs = false` ensures only one path.
- **Input:** Output of **Get contact data for tagging** (so `$json` is KlickTipp subscriber object).
- **Connections:**
  - Person does not exist → **Create a person**
  - Person exists → **Update a person**
  - Person exists → unsubscribe → **Update a person3**
- **Edge cases:**
  - If `field227685` is present but not a valid Pipedrive Person ID → update calls will fail.
  - Status values must match exactly (`'Unsubscribed'`); if KlickTipp changes casing/labels, routing may break.

---

### Block C — KlickTipp → Pipedrive (Create/update person + consent mapping)
**Overview:** Creates or updates the Pipedrive Person with mapped fields and consent (`marketing_status`). Converts birthday from KlickTipp Unix seconds into Pipedrive’s `YYYY-MM-DD`.  
**Nodes involved:**  
- **Create a person**  
- **Update a person**  
- **Update a person3**

#### Node: Create a person
- **Type / role:** `pipedrive` — create Person.
- **Config (interpreted):**
  - Name from KlickTipp: `fullname`
  - Email: `email`
  - Phone: `fieldMobilePhone`
  - Custom properties:
    - `95cac42f...` = birthday formatted `YYYY-MM-DD` in `Europe/Berlin`
    - `7effe3f...` = KlickTipp subscriber ID
  - `marketing_status`:
    - If KlickTipp `status === 'Subscribed'` → `subscribed`
    - Else → `no_consent`
- **Connections:** → **Add Pipedrive Person ID to KlickTipp contact**
- **Edge cases / failures:**
  - Birthday conversion handles missing timestamp by returning `""` (safe).
  - Pipedrive validation errors (duplicate email rules, invalid custom field keys).
  - Phone field may not exist in KlickTipp payload → may create empty phone entry.

#### Node: Update a person
- **Type / role:** `pipedrive` — update Person.
- **Config:**
  - `personId = {{ $('Get contact data for tagging').item.json.field227685 }}`
  - Updates name/email + same custom fields and consent logic as create.
  - Birthday conversion does **not** guard missing `fieldBirthday` (it directly multiplies). If null, may produce `Invalid Date`.
- **Connections:** → **Check relevant segment**
- **Failure modes:**
  - Missing/invalid Person ID → 404 or validation error.
  - If `fieldBirthday` is empty → date conversion may yield `"Invalid Date"` string.

#### Node: Update a person3
- **Type / role:** `pipedrive` — update Person for unsubscribed case.
- **Config differences:**
  - Sets `marketing_status = unsubscribed` explicitly.
  - Otherwise similar to **Update a person** (same birthday conversion risk).
- **Connections:** → **Check relevant segment**

---

### Block D — KlickTipp → Pipedrive (Persist ID link back to KlickTipp)
**Overview:** After creating a Person in Pipedrive, writes the new Pipedrive Person ID back to the KlickTipp subscriber custom field `field227685`. This forms the permanent link for future sync operations.  
**Nodes involved:**  
- **Add Pipedrive Person ID to KlickTipp contact**

#### Node: Add Pipedrive Person ID to KlickTipp contact
- **Type / role:** `klicktipp` (community) — `subscriber.update` by email lookup.
- **Config:**
  - Updates `field227685 = {{ $json.id }}` where `$json.id` is the output from **Create a person**.
  - Lookup by email: `{{ $('Contact tagged in KlickTipp').item.json.email }}`
- **Connections:** → **Check relevant segment**
- **Edge cases:**
  - If KlickTipp email changed between trigger and update → lookup may fail.
  - If multiple subscribers share same email (should not) → unpredictable.
  - Uses a different KlickTipp credential (“DEMO KlickTipp account”) than some other KlickTipp nodes; ensure it points to same workspace.

---

### Block E — KlickTipp → Pipedrive Segmentation (Tags → Labels)
**Overview:** If KlickTipp subscriber has specific tag IDs, the workflow patches Pipedrive Person `label_ids` accordingly.  
**Nodes involved:**  
- **Check relevant segment** (Switch)  
- **Assign Label Customer** (HTTP PATCH)  
- **Assign Label ABC** (HTTP PATCH)

#### Node: Check relevant segment
- **Type / role:** `switch` — segmentation routing based on KlickTipp tags array.
- **Config:**
  - Checks if `$('Get contact data for tagging').item.json.tags` **contains**:
    - `14054258` → route “Label Customer exists…”
    - `14054294` → route “Label ABC exists…”
  - `allMatchingOutputs = true` → can assign multiple labels if multiple tags are present.
- **Connections:**
  - Customer branch → **Assign Label Customer**
  - ABC branch → **Assign Label ABC**
- **Edge cases:**
  - If `tags` is missing or not an array → “contains” may error or return false.

#### Node: Assign Label Customer
- **Type / role:** `httpRequest` — direct PATCH to Pipedrive v2 API.
- **Config:**
  - URL uses domain `https://klicktipp3.pipedrive.com/api/v2/persons/{{ personId }}`
  - Person ID expression:
    - Prefer `$('Contact tagged in KlickTipp').item.json.field227685` (existing link)
    - Else fallback to `$('Create a person').item.json.id`
  - JSON body: `{"label_ids":[14]}`
  - Auth: predefined credential type `pipedriveApi`
- **Connections:** none
- **Failure modes:**
  - Domain hardcoded (`klicktipp3.pipedrive.com`) must match your company domain.
  - PATCH overwrites labels to exactly `[14]` (may remove other labels). Same for ABC node.

#### Node: Assign Label ABC
- **Same as above**, but body is `{"label_ids":[27]}`.

---

### Block F — Pipedrive → KlickTipp (Trigger & determine marketing status)
**Overview:** When a Pipedrive Person changes, fetch the current Person to read `marketing_status`, then route based on change source, deletion, existence of linked KlickTipp ID, and status.  
**Nodes involved:**  
- **Changes in Pipedrive1** (Trigger)  
- **Get marketing status** (Pipedrive Get)  
- **Check subscription1** (Switch)  
- **Filter non UI events and deletions** (Filter)

#### Node: Changes in Pipedrive1
- **Type / role:** `pipedriveTrigger` — webhook on Person events.
- **Config:** `entity = person` (captures create/update/delete).
- **Connections:** → **Get marketing status**
- **Edge cases:**
  - Webhook setup in Pipedrive must be active and pointing to n8n.
  - Delete events provide `previous` data; ensure downstream nodes use correct properties.

#### Node: Get marketing status
- **Type / role:** `pipedrive` — get Person by ID.
- **Config:** `personId = {{ $json.data.id }}`
- **Error handling:** `onError = continueRegularOutput` (important: continues even if fetch fails).
- **Connections:** → **Check subscription1**
- **Failure modes:**
  - If the person was deleted, GET may 404; node continues, but `marketing_status` becomes null → routing must tolerate null.

#### Node: Check subscription1
- **Type / role:** `switch` — main Pipedrive→KlickTipp router.
- **Config / outputs (allMatchingOutputs = true):**
  - **Single Opt-In:** no KlickTipp ID in Pipedrive custom field `7effe3f...` AND marketing_status != `unsubscribed`
  - **Form submission → DOI:** `meta.change_source == api`
  - **Update only:** KlickTipp ID exists AND marketing_status != `unsubscribed`
  - **Unsubscribe and update:** KlickTipp ID exists AND marketing_status == `unsubscribed`
  - **Contact deletion:** `meta.action == delete`
- **Connections:**
  - Single Opt-In → **Filter non UI events and deletions**
  - Form submission → DOI → **Create contact with DOI**
  - Update only → **Update contact changes in KlickTipp**
  - Unsubscribe and update → **Unsubscribe contact**
  - Contact deletion → **Delete contact**
- **Edge cases:**
  - Because `allMatchingOutputs = true`, a single event could match multiple routes if conditions overlap (notably `change_source=api` plus other conditions). In this workflow, DOI path only checks `change_source`, so it may run alongside update/unsubscribe depending on event shape.

#### Node: Filter non UI events and deletions
- **Type / role:** `filter` — ensures SOI create path only runs for UI changes.
- **Config:** Pass only if:
  - `meta.change_source == app`
  - `meta.action != delete`
- **Connections:** → **Create contact with SOI**
- **Edge cases:**
  - If Pipedrive changes the values for `change_source`, SOI creation might never run.

---

### Block G — Pipedrive → KlickTipp (Create/update/unsubscribe/delete)
**Overview:** Executes the routed actions in KlickTipp (SOI/DOI subscribe, update subscriber fields, unsubscribe, delete). Handles birthday conversion from Pipedrive `YYYY-MM-DD` to Unix seconds.  
**Nodes involved:**  
- **Create contact with SOI**  
- **Create contact with DOI**  
- **Update contact changes in KlickTipp**  
- **Unsubscribe contact** → **Update contact changes in KlickTipp1**  
- **Delete contact**

#### Node: Create contact with SOI
- **Type / role:** `klicktipp` — `subscriber.subscribe` (Single Opt-In list).
- **Config:**
  - Email from `Changes in Pipedrive1.data.emails[0].value`
  - listId: `364353`
  - tagId: `14029585` (applied at subscribe)
  - Fields mapped: first_name, last_name, birthday (converted), field227685 (Pipedrive ID)
  - Birthday conversion: reads Pipedrive custom field `95cac42f...` (YYYY-MM-DD) → Unix seconds using `Date.parse(bday + "T00:00:00Z")/1000`
- **Connections:** → **Add KlickTipp contact ID to Pipedrive**
- **Edge cases:**
  - If email array empty → expression fails.
  - If birthday is missing → returns `""` (safe).
  - Subscription may fail if email already exists; KlickTipp behavior depends on API (may update or error).

#### Node: Create contact with DOI
- **Type / role:** `klicktipp` — `subscriber.subscribe` (Double Opt-In list).
- **Config:** Similar to SOI but listId `358895`; does **not** include birthday field in this node (only first/last + field227685).
- **Connections:** → **Add KlickTipp contact ID to Pipedrive1**
- **Edge cases:** Same email array risk.

#### Node: Update contact changes in KlickTipp
- **Type / role:** `klicktipp` — `subscriber.update` by subscriber ID.
- **Config:**
  - Uses `subscriberId` from Pipedrive custom field `7effe3f...` (stored KlickTipp ID).
  - Updates first/last, birthday (converted to Unix seconds), and field227685 (Pipedrive ID).
  - Email parameter set to current Pipedrive email (used by node, but update is keyed by ID).
- **Connections:** → **Check relevant segment2**
- **Failure modes:**
  - If stored KlickTipp ID missing/invalid → update fails.
  - If Pipedrive birthday malformed → Date.parse returns NaN → produces `NaN` (would become `null`/invalid in KlickTipp).

#### Node: Unsubscribe contact
- **Type / role:** `klicktipp` — `subscriber.unsubscribe` by email.
- **Config:** Email from `Changes in Pipedrive1.data.emails[0].value`
- **Error handling:** `onError = continueRegularOutput`
- **Connections:** → **Update contact changes in KlickTipp1**
- **Edge cases:**
  - Unsubscribing a non-existing email may error; node continues anyway.

#### Node: Update contact changes in KlickTipp1
- **Type / role:** `klicktipp` — `subscriber.update` by subscriber ID (same mapping as Update contact changes in KlickTipp).
- **Role:** Ensures core fields still get synchronized even when the contact is unsubscribed.
- **Connections:** → **Check relevant segment2**

#### Node: Delete contact
- **Type / role:** `klicktipp` — `subscriber.delete` by email lookup.
- **Config:** `lookupEmail = {{ $('Changes in Pipedrive1').item.json.previous.emails[0].value }}`
  - Uses **previous email** to handle cases where email changed before deletion.
- **Error handling:** `alwaysOutputData = true` to not break downstream execution (though it has no outgoing connections here).
- **Edge cases:**
  - If `previous.emails[0]` missing (depending on webhook payload) → expression error.
  - If contact already deleted in KlickTipp → API error.

---

### Block H — Pipedrive → KlickTipp Segmentation (Labels → Tags) + transactional tagging
**Overview:** After Pipedrive→KlickTipp create/update/unsubscribe updates, assigns KlickTipp tags based on Pipedrive label IDs. Also tags newly created SOI contacts for a first transactional mail.  
**Nodes involved:**  
- **Check relevant segment2** (Switch)  
- **Tag contact in KlickTipp2**  
- **Tag contact in KlickTipp3**  
- **Tag contact for first transactional mail**  
- **Add KlickTipp contact ID to Pipedrive**  
- **Add KlickTipp contact ID to Pipedrive1**

#### Node: Add KlickTipp contact ID to Pipedrive
- **Type / role:** `pipedrive` — update Person to store KlickTipp subscriber ID.
- **Config:** `personId = Changes in Pipedrive1.data.id`
- **Updates:** custom property `7effe3f... = {{ $json.id }}` (where `$json.id` is KlickTipp subscriber ID returned by SOI subscribe)
- **Connections:** → **Check relevant segment2** AND → **Tag contact for first transactional mail**
- **Edge cases:** If KlickTipp subscribe response doesn’t include `id`, mapping fails.

#### Node: Tag contact for first transactional mail
- **Type / role:** `klicktipp` — add tag.
- **Config:** tagId `[14152568]` applied to Pipedrive email.
- **Connections:** none
- **Edge cases:** Runs in parallel with segmentation; may tag even if segmentation fails.

#### Node: Add KlickTipp contact ID to Pipedrive1
- **Type / role:** `pipedrive` — update Person (DOI path).
- **Config:** same custom property update as above.
- **Connections:** none (not followed by segmentation in this workflow as wired).

#### Node: Check relevant segment2
- **Type / role:** `switch` — reads Pipedrive `label_ids` and tags accordingly in KlickTipp.
- **Config:**
  - Normalizes label IDs from `Changes in Pipedrive1.data.label_ids` which may be array or comma-string.
  - Branch 1: label 14 present → **Tag contact in KlickTipp2** (tag 14054258)
  - Branch 2: label 27 present → **Tag contact in KlickTipp3** (tag 14054294)
  - `allMatchingOutputs = true` supports multiple tags.
- **Connections:** to tag nodes.
- **Edge cases:**
  - If `label_ids` missing, normalization returns empty list (safe).
  - If `label_ids` uses unexpected format, tags may not apply.

#### Node: Tag contact in KlickTipp2 / Tag contact in KlickTipp3
- **Type / role:** `klicktipp` — contact-tagging.
- **Config:** Uses email from Pipedrive webhook payload; applies tag list:
  - KlickTipp Tag `14054258` (Customer)
  - KlickTipp Tag `14054294` (ABC)
- **Connections:** none
- **Failure modes:** Email missing/invalid; contact not found.

---

### Sticky Notes (documentation-only nodes)
These nodes do not execute logic but annotate the canvas:
- **Sticky Note2:** “## 1. Get contact data.”
- **Sticky Note6:** “## 2. Route by subscription.”
- **Sticky Note9:** “## 3. Filter contacts.”
- **Sticky Note7:** “## 4. Transfer contact data.”
- **Sticky Note8:** “## 5. Save contact ID”
- **Sticky Note10:** “## 6. Segmentation.”
- **Sticky Note13:** Community node disclaimer + full functional explanation and setup notes (content preserved in Section 5).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Contact tagged in KlickTipp | klicktippTrigger | Entry point (KlickTipp → Pipedrive) via webhook/tag event | — | Get contact data for tagging | ## 1. Get contact data. |
| Get contact data for tagging | klicktipp | Fetch full KlickTipp subscriber by ID | Contact tagged in KlickTipp | Check subscription | ## 1. Get contact data. |
| Check subscription | switch | Route: create/update/unsubscribe Pipedrive based on KlickTipp status + stored Pipedrive ID | Get contact data for tagging | Create a person; Update a person; Update a person3 | ## 2. Route by subscription. |
| Create a person | pipedrive | Create Pipedrive Person from KlickTipp data | Check subscription | Add Pipedrive Person ID to KlickTipp contact | ## 4. Transfer contact data. |
| Update a person | pipedrive | Update existing Pipedrive Person from KlickTipp data | Check subscription | Check relevant segment | ## 4. Transfer contact data. |
| Update a person3 | pipedrive | Update person + set marketing_status=unsubscribed | Check subscription | Check relevant segment | ## 4. Transfer contact data. |
| Add Pipedrive Person ID to KlickTipp contact | klicktipp | Write back Pipedrive Person ID into KlickTipp custom field | Create a person | Check relevant segment | ## 5. Save contact ID |
| Check relevant segment | switch | Tags → Labels routing (KlickTipp tags determine Pipedrive labels) | Update a person; Update a person3; Add Pipedrive Person ID to KlickTipp contact | Assign Label Customer; Assign Label ABC | ## 6. Segmentation. |
| Assign Label Customer | httpRequest | PATCH Pipedrive label_ids=[14] | Check relevant segment | — | ## 6. Segmentation. |
| Assign Label ABC | httpRequest | PATCH Pipedrive label_ids=[27] | Check relevant segment | — | ## 6. Segmentation. |
| Changes in Pipedrive1 | pipedriveTrigger | Entry point (Pipedrive → KlickTipp) on Person changes | — | Get marketing status |  |
| Get marketing status | pipedrive | Fetch Person to read marketing_status | Changes in Pipedrive1 | Check subscription1 |  |
| Check subscription1 | switch | Route: SOI/DOI/update/unsubscribe/delete (Pipedrive → KlickTipp) | Get marketing status | Filter non UI events and deletions; Create contact with DOI; Update contact changes in KlickTipp; Unsubscribe contact; Delete contact |  |
| Filter non UI events and deletions | filter | Allow only UI-driven SOI creation; block API + deletions | Check subscription1 | Create contact with SOI | ## 3. Filter contacts. |
| Create contact with SOI | klicktipp | Subscribe in SOI list + map fields (incl. birthday conversion) | Filter non UI events and deletions | Add KlickTipp contact ID to Pipedrive | ## 4. Transfer contact data. |
| Create contact with DOI | klicktipp | Subscribe in DOI list (API/form submissions) | Check subscription1 | Add KlickTipp contact ID to Pipedrive1 | ## 4. Transfer contact data. |
| Add KlickTipp contact ID to Pipedrive | pipedrive | Store KlickTipp subscriber ID into Pipedrive custom field | Create contact with SOI | Check relevant segment2; Tag contact for first transactional mail | ## 5. Save contact ID |
| Add KlickTipp contact ID to Pipedrive1 | pipedrive | Store KlickTipp subscriber ID (DOI path) | Create contact with DOI | — | ## 5. Save contact ID |
| Update contact changes in KlickTipp | klicktipp | Update KlickTipp subscriber fields from Pipedrive | Check subscription1 | Check relevant segment2 | ## 4. Transfer contact data. |
| Unsubscribe contact | klicktipp | Unsubscribe KlickTipp subscriber by email | Check subscription1 | Update contact changes in KlickTipp1 | ## 2. Route by subscription. |
| Update contact changes in KlickTipp1 | klicktipp | Update fields after unsubscribe (keep data aligned) | Unsubscribe contact | Check relevant segment2 | ## 4. Transfer contact data. |
| Delete contact | klicktipp | Delete KlickTipp subscriber using previous email | Check subscription1 | — |  |
| Check relevant segment2 | switch | Labels → Tags routing (Pipedrive labels determine KlickTipp tags) | Update contact changes in KlickTipp; Update contact changes in KlickTipp1; Add KlickTipp contact ID to Pipedrive | Tag contact in KlickTipp2; Tag contact in KlickTipp3 | ## 6. Segmentation. |
| Tag contact in KlickTipp2 | klicktipp | Apply tag 14054258 based on label 14 | Check relevant segment2 | — | ## 6. Segmentation. |
| Tag contact in KlickTipp3 | klicktipp | Apply tag 14054294 based on label 27 | Check relevant segment2 | — | ## 6. Segmentation. |
| Tag contact for first transactional mail | klicktipp | Apply transactional mail tag 14152568 | Add KlickTipp contact ID to Pipedrive | — |  |
| Sticky Note2 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note6 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note9 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note7 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note8 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note10 | stickyNote | Canvas annotation | — | — |  |
| Sticky Note13 | stickyNote | Canvas annotation / long-form notes | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Add **Pipedrive API** credentials in n8n (OAuth2 or API token depending on node support in your n8n version).  
   2. Add **KlickTipp API** credentials for the community nodes (ensure the same account/workspace is used consistently).

2) **Create entry trigger (KlickTipp → Pipedrive)**
   1. Add node: **KlickTipp Trigger** named **“Contact tagged in KlickTipp”**.  
      - Set a unique **Webhook Path** (n8n will generate one; copy it).  
      - Configure KlickTipp campaign/activation tag webhook to call this URL.
   2. Add node: **KlickTipp** named **“Get contact data for tagging”**  
      - Resource: **Subscriber**  
      - Operation: **Get**  
      - Identifier type: **ID**  
      - Subscriber ID: expression referencing trigger `id`.
   3. Connect: Trigger → Get.

3) **Route KlickTipp events**
   1. Add node: **Switch** named **“Check subscription”**
      - Create 3 boolean-expression rules:
        - No Pipedrive ID (custom field `field227685`) and status not Unsubscribed → “Person does not exist”
        - Pipedrive ID exists and status not Unsubscribed → “Person exists”
        - Pipedrive ID exists and status Unsubscribed → “Person exists → unsubscribe”
      - Set `allMatchingOutputs = false`.
   2. Connect: Get contact data for tagging → Check subscription.

4) **Create/update Pipedrive Person**
   1. Add node: **Pipedrive** named **“Create a person”** (Operation: Create, Resource: Person)
      - Map name/email/phone from KlickTipp trigger
      - Set custom fields:
        - Birthday custom field key (your Pipedrive field key) mapped from KlickTipp Unix timestamp → `YYYY-MM-DD` (timezone Europe/Berlin)
        - KlickTipp ID custom field key mapped from KlickTipp `id`
      - Set `marketing_status` based on KlickTipp status.
   2. Add node: **Pipedrive** named **“Update a person”** (Operation: Update)
      - `personId` from KlickTipp custom field `field227685`
      - Same mapping and `marketing_status` logic.
   3. Add node: **Pipedrive** named **“Update a person3”** (Operation: Update)
      - Same as update, but set `marketing_status = unsubscribed`.
   4. Connect Switch outputs to the corresponding nodes.

5) **Write back Pipedrive Person ID into KlickTipp**
   1. Add node: **KlickTipp** named **“Add Pipedrive Person ID to KlickTipp contact”**
      - Resource: Subscriber, Operation: Update
      - Lookup by email (use the KlickTipp email)
      - Set data field `field227685` to the created Pipedrive person `id`
   2. Connect: Create a person → Add Pipedrive Person ID…

6) **Segmentation: KlickTipp Tags → Pipedrive Labels**
   1. Add node: **Switch** named **“Check relevant segment”**
      - Conditions: `tags contains 14054258` and/or `tags contains 14054294`
      - `allMatchingOutputs = true`
   2. Add two nodes: **HTTP Request**
      - **Assign Label Customer**: PATCH `/api/v2/persons/{id}` body `{"label_ids":[14]}`
      - **Assign Label ABC**: PATCH body `{"label_ids":[27]}`
      - Use predefined credential type: `pipedriveApi`
      - Ensure the base domain matches your Pipedrive company domain.
   3. Connect: Update a person / Update a person3 / Add Pipedrive Person ID… → Check relevant segment → HTTP nodes.

7) **Create entry trigger (Pipedrive → KlickTipp)**
   1. Add node: **Pipedrive Trigger** named **“Changes in Pipedrive1”**
      - Entity: **Person**
   2. Add node: **Pipedrive** named **“Get marketing status”**
      - Operation: Get Person by `data.id`
      - Set **On Error: Continue**
   3. Connect: Trigger → Get marketing status.

8) **Route Pipedrive events**
   1. Add node: **Switch** named **“Check subscription1”**
      - Rules:
        - Single Opt-In: no KlickTipp ID in Pipedrive custom field AND marketing_status != unsubscribed
        - Form submission → DOI: `meta.change_source == api`
        - Update only: KlickTipp ID exists AND marketing_status != unsubscribed
        - Unsubscribe and update: KlickTipp ID exists AND marketing_status == unsubscribed
        - Contact deletion: `meta.action == delete`
      - Set `allMatchingOutputs = true` (as in the workflow).
   2. Connect: Get marketing status → Check subscription1.

9) **UI-only filter for SOI creation**
   1. Add node: **Filter** named **“Filter non UI events and deletions”**
      - Conditions: `meta.change_source == app` AND `meta.action != delete`
   2. Connect: Check subscription1 (Single Opt-In output) → Filter → next step.

10) **Pipedrive → KlickTipp actions**
   1. Add node: **KlickTipp** named **“Create contact with SOI”**
      - Operation: Subscribe, listId = your SOI list
      - Map first/last, Pipedrive person ID into `field227685`
      - Convert birthday from Pipedrive date field to Unix seconds if you use birthday syncing
   2. Add node: **KlickTipp** named **“Create contact with DOI”**
      - Operation: Subscribe, listId = your DOI list
      - Map core fields; apply tag `14029585` if needed
   3. Add node: **KlickTipp** named **“Update contact changes in KlickTipp”** (Operation: Update by subscriber ID)
   4. Add node: **KlickTipp** named **“Unsubscribe contact”** (Operation: Unsubscribe by email; On Error: Continue)
   5. Add node: **KlickTipp** named **“Update contact changes in KlickTipp1”** and connect from Unsubscribe contact
   6. Add node: **KlickTipp** named **“Delete contact”** (Operation: Delete by lookupEmail = previous email)

11) **Store KlickTipp subscriber ID back into Pipedrive**
   1. Add node: **Pipedrive** named **“Add KlickTipp contact ID to Pipedrive”** (update person custom field to returned KlickTipp `id`)
   2. Add node: **Pipedrive** named **“Add KlickTipp contact ID to Pipedrive1”** for DOI path (same update)
   3. Connect:
      - Create contact with SOI → Add KlickTipp contact ID to Pipedrive
      - Create contact with DOI → Add KlickTipp contact ID to Pipedrive1

12) **Segmentation: Pipedrive Labels → KlickTipp Tags**
   1. Add node: **Switch** named **“Check relevant segment2”**
      - Normalize `label_ids` to array and check for 14 and 27
      - `allMatchingOutputs = true`
   2. Add nodes:
      - **Tag contact in KlickTipp2** (contact-tagging, tag 14054258)
      - **Tag contact in KlickTipp3** (contact-tagging, tag 14054294)
   3. Connect:
      - Update contact changes in KlickTipp → Check relevant segment2
      - Update contact changes in KlickTipp1 → Check relevant segment2
      - Add KlickTipp contact ID to Pipedrive → Check relevant segment2

13) **Optional: transactional mail tagging**
   1. Add node: **KlickTipp** named **“Tag contact for first transactional mail”** (tag 14152568)
   2. Connect: Add KlickTipp contact ID to Pipedrive → Tag contact for first transactional mail

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Community Node Disclaimer: This workflow uses KlickTipp community nodes. | Sticky Note13 |
| Full functional description of both directions (KlickTipp→Pipedrive and Pipedrive→KlickTipp), consent mapping, birthday conversion details, segmentation mapping (Tags↔Labels), and setup/customization guidance. | Sticky Note13 |
| disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided by user |

