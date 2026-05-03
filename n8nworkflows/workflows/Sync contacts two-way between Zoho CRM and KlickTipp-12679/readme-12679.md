Sync contacts two-way between Zoho CRM and KlickTipp

https://n8nworkflows.xyz/workflows/sync-contacts-two-way-between-zoho-crm-and-klicktipp-12679


# Sync contacts two-way between Zoho CRM and KlickTipp

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow implements a **two-way contact synchronization** between **Zoho CRM** and **KlickTipp**. It reacts to events from either system (via webhooks/triggers) and mirrors contact creation, update, deletion, subscription status changes, and segmentation tags.

**Target use cases:**
- Keep Zoho CRM contact records aligned with KlickTipp subscribers (fields + IDs).
- Automate opt-out/unsubscribe propagation from Zoho to KlickTipp.
- Sync segmentation between Zoho tags and KlickTipp tags.
- Maintain cross-system identifiers (`KlickTipp ID` in Zoho and `Zoho | Contact ID` in KlickTipp custom field).

### 1.1 Logical Blocks

1. **Zoho → KlickTipp: Contact registered via Form (DOI subscribe)**
2. **Zoho → KlickTipp: Contact created/updated (create/update/unsubscribe routing)**
3. **Zoho → KlickTipp: Contact deleted (delete in KlickTipp)**
4. **Zoho → KlickTipp: Segmentation tagging in KlickTipp (based on Zoho tags)**
5. **KlickTipp → Zoho: Triggered by KlickTipp “tagged” event (create/update contact in Zoho)**
6. **KlickTipp → Zoho: Segmentation tagging in Zoho (based on KlickTipp tag IDs)**
7. **Documentation / Sticky Notes (in-canvas guidance)**

---

## 2. Block-by-Block Analysis

### Block 1 — Zoho → KlickTipp: Contact registered via Form (DOI subscribe)
**Overview:**  
Handles contacts submitted via a Zoho form. If an email exists, it subscribes the contact in KlickTipp using **DOI** list settings and writes the created KlickTipp subscriber ID back into Zoho.

**Nodes involved:**
- **Contact registered via Form in Zoho CRM** (Webhook)
- **Check for email address2** (Filter)
- **Create Contact with DOI** (KlickTipp)
- **Add KlickTipp Contact ID2** (Zoho CRM)

#### Node: Contact registered via Form in Zoho CRM
- **Type / role:** Webhook trigger; entry point for “form registration” events from Zoho workflows.
- **Config:** `POST` webhook, path is empty in JSON (must be set in UI or generated).
- **Input/Output:** No input; outputs incoming payload as `{{$json.body...}}`.
- **Edge cases:**
  - Zoho workflow misconfigured URL/method → never triggers.
  - Payload schema differences → downstream expressions may fail if `body.email` etc. missing.

#### Node: Check for email address2
- **Type / role:** Filter; ensures `body.email` exists before attempting KlickTipp API calls.
- **Key expression:** `={{ $json.body.email }} exists`
- **Connections:** From webhook → to **Create Contact with DOI**.
- **Edge cases:** Email present but invalid format still passes (exists check only).

#### Node: Create Contact with DOI
- **Type / role:** KlickTipp node (`subscriber.subscribe`) with DOI behavior (implemented via list/tag config).
- **Config choices:**
  - Operation: **Subscribe** subscriber
  - `email = {{$json.body.email}}`
  - `listId = 358895`
  - `tagId = 13939731` (applied at subscribe)
  - Field mapping includes Zoho ID into KlickTipp custom field `field226649` plus name/phone/address/birthday fields.
- **Outputs:** KlickTipp response includes subscriber `id`, plus echoed fields like `field226649`.
- **Edge cases:**
  - KlickTipp credential/auth failure.
  - DOI list requires confirmation—subscriber may be “pending” depending on KlickTipp behavior.
  - If subscriber already exists, API may return error or update depending on KlickTipp rules.

#### Node: Add KlickTipp Contact ID2
- **Type / role:** Zoho CRM `contact.update` to store KlickTipp subscriber ID into Zoho custom field.
- **Config choices:**
  - `contactId = {{$json.field226649}}` (Zoho contact ID previously stored into KlickTipp field)
  - Custom field update: `KlickTipp_ID = {{$json.id}}`
- **Credentials:** Zoho OAuth2.
- **Edge cases:**
  - If `field226649` not present, update fails.
  - If Zoho field API name differs from `KlickTipp_ID` or permissions missing → 400/403.

---

### Block 2 — Zoho → KlickTipp: Contact created/updated routing (create/update/unsubscribe)
**Overview:**  
Triggered when a Zoho contact is created or updated. It routes records into three actions: **new contact subscribe (SOI)**, **update existing subscriber**, or **unsubscribe** when Zoho indicates opt-out.

**Nodes involved:**
- **Contact created or updated in Zoho CRM** (Webhook)
- **Check Subscription** (Switch)
- **Check for email address** (Filter)
- **Create Contact with SOI** (KlickTipp)
- **Add KlickTipp Contact ID** (Zoho CRM)
- **Tag contact for first transactional mail** (KlickTipp contact-tagging)
- **Check for email address1** (Filter)
- **Unsubscribe contact** (KlickTipp)
- **Check for KlickTipp ID1** (Filter)
- **Update contact** (KlickTipp)
- **Get contact data for tagging1** (Zoho CRM) *(also used by Block 4)*

#### Node: Contact created or updated in Zoho CRM
- **Type / role:** Webhook trigger from Zoho workflows for create/update events.
- **Config:** POST webhook.
- **Edge cases:** Same as other webhooks; ensure Zoho sends `body` with expected keys.

#### Node: Check Subscription
- **Type / role:** Switch router; decides flow based on whether the Zoho record has a KlickTipp ID and whether email opt-out is enabled.
- **Rules (interpreted):**
  1. **New Contact:** `body.klicktippId` is empty
  2. **Unsubscribed:** `body.emailOptOut` is true
  3. **Update Contact:** `body.klicktippId` not empty
- **Options:** loose type validation enabled.
- **Edge cases:**
  - If Zoho sends `emailOptOut` as `"true"` string, loose validation helps but can still be inconsistent.
  - Misrouted if `klicktippId` exists but invalid.

#### Node: Check for email address (New Contact branch)
- **Type / role:** Filter; email must exist for subscription.
- **Connection:** From **Check Subscription: New Contact** → **Create Contact with SOI**.

#### Node: Create Contact with SOI
- **Type / role:** KlickTipp `subscriber.subscribe` (single opt-in style by configuration).
- **Config choices:**
  - `listId = 364353`
  - Maps Zoho fields into KlickTipp fields including `field226649 = Zoho contact id`
- **Connections:** Output to **Add KlickTipp Contact ID** and **Get contact data for tagging1** (parallel).
- **Edge cases:** Similar to DOI node; if subscriber exists, may error.

#### Node: Add KlickTipp Contact ID
- **Type / role:** Zoho CRM `contact.update` to store KlickTipp subscriber ID.
- **Config:** update Zoho custom field `KlickTipp_ID = {{$json.id}}` using `contactId = {{$json.field226649}}`.
- **Connection:** → **Tag contact for first transactional mail**
- **Edge cases:** same as Add KlickTipp Contact ID2.

#### Node: Tag contact for first transactional mail
- **Type / role:** KlickTipp `contact-tagging` to attach a tag immediately after creation.
- **Config:** `tagId = ["13879733"]`, `email = {{ $('Create Contact with SOI').item.json.email }}`
- **Edge cases:**
  - Uses cross-node reference to `Create Contact with SOI`. If multiple items processed, ensure item pairing is correct (n8n item linking usually works but can break with merges).

#### Node: Check for email address1 (Unsubscribed branch)
- **Type / role:** Filter; email required to unsubscribe by email.
- **Connection:** → **Unsubscribe contact**

#### Node: Unsubscribe contact
- **Type / role:** KlickTipp `subscriber.unsubscribe` using email.
- **Config:** `email = {{$json.body.email}}`
- **Edge cases:** If email not found in KlickTipp, API may return “not found” or success with no-op.

#### Node: Check for KlickTipp ID1 (Update Contact branch)
- **Type / role:** Filter; requires `body.klicktippId` exists.
- **Connection:** → **Update contact**

#### Node: Update contact
- **Type / role:** KlickTipp `subscriber.update` by subscriber ID.
- **Config:**
  - `subscriberId = {{$json.body.klicktippId}}`, `identifierType = id`
  - Maps Zoho fields to KlickTipp fields (name, phones, address, birthday, etc.)
- **Connection:** → **Get contact data for tagging1**
- **Edge cases:**
  - If `klicktippId` is stale/deleted → update fails.
  - Partial field values may overwrite existing data with empty strings if Zoho sends empty.

#### Node: Get contact data for tagging1
- **Type / role:** Zoho CRM `contact.get` to fetch full Zoho contact for segmentation decisions.
- **Config:** `contactId = {{ $('Contact created or updated in Zoho CRM').item.json.body.id }}`
- **Connection:** → **Check relevant Segment** (Block 4)
- **Edge cases:** If webhook payload doesn’t include `body.id`, lookup fails.

---

### Block 3 — Zoho → KlickTipp: Contact deleted (delete in KlickTipp)
**Overview:**  
When a Zoho contact is deleted, this block deletes the corresponding KlickTipp subscriber—if a KlickTipp ID is provided.

**Nodes involved:**
- **Contact deleted in Zoho CRM** (Webhook)
- **Check for KlickTipp ID** (Filter)
- **Delete contact in KlickTipp** (KlickTipp)

#### Node: Contact deleted in Zoho CRM
- **Type / role:** Webhook trigger for deletion events.
- **Edge cases:** Zoho deletion workflows must send the KlickTipp ID (or you must fetch it before delete).

#### Node: Check for KlickTipp ID
- **Type / role:** Filter; requires `body.klicktippId` exists.
- **Connection:** → **Delete contact in KlickTipp**
- **Edge cases:** If Zoho cannot provide KlickTipp ID on deletion, deletion won’t happen.

#### Node: Delete contact in KlickTipp
- **Type / role:** KlickTipp `subscriber.delete` by ID.
- **Config:** `subscriberId = {{$json.body.klicktippId}}`, `identifierType = id`
- **Edge cases:** Already-deleted subscriber → not found.

---

### Block 4 — Zoho → KlickTipp: Segmentation tagging (based on Zoho tags)
**Overview:**  
After creating/updating a Zoho contact (or after creating in KlickTipp), the workflow inspects Zoho contact tags and applies corresponding KlickTipp tags.

**Nodes involved:**
- **Get contact data for tagging1** (Zoho CRM get) *(from Block 2)*
- **Check relevant Segment ** (Switch)
- **Tag contact in KlickTipp** (KlickTipp contact-tagging)
- **Tag contact in KlickTipp1** (KlickTipp contact-tagging)

#### Node: Check relevant Segment 
- **Type / role:** Switch; routes based on whether Zoho contact contains specific tags.
- **Config choices:**
  - Option: `allMatchingOutputs = true` (can tag multiple segments)
  - Rule “Online store”: checks if any `Tag[].name` equals “online store” (case-insensitive)
    - Expression: `={{ ($json.Tag ?? []).some(t => (t?.name || '').toLowerCase() === 'online store') }}`
  - Rule “ABC”: similar check for `abc`
- **Connections:** outputs to respective tagger nodes.
- **Edge cases:**
  - Zoho tag data structure might differ (`Tags` vs `Tag`), breaking expression.
  - If tag names differ in spelling/case, rule won’t match.

#### Node: Tag contact in KlickTipp
- **Type / role:** KlickTipp `contact-tagging` add tag.
- **Config:** `email = {{$json.email}}`, `tagId = ["13879589"]` (“Online Store”)
- **Edge cases:** If `$json.email` not present from Zoho get response, tagging fails.

#### Node: Tag contact in KlickTipp1
- **Type / role:** KlickTipp `contact-tagging` add tag.
- **Config:** `tagId = ["13879590"]` (“ABC”)
- **Edge cases:** Same as above.

---

### Block 5 — KlickTipp → Zoho: Triggered by KlickTipp “tagged” event (create/update)
**Overview:**  
Triggered by KlickTipp when a subscriber is tagged (typically using the activation tag “Zoho | Send contact”). Depending on whether a Zoho contact ID exists in KlickTipp (`field226649`), it creates or updates the Zoho contact. Then it writes the Zoho ID back into KlickTipp.

**Nodes involved:**
- **Contact Tagged in KlickTipp** (KlickTipp Trigger)
- **Check Subscription in Zoho** (Switch)
- **Check for last name in KlickTipp** (Filter) → create path
- **Create a contact in Zoho** (Zoho CRM)
- **Add Zoho CRM Contact ID to KlickTipp Contact** (KlickTipp update)
- **Check for Zoho contact ID** (Filter) → update path
- **Update a contact in Zoho** (Zoho CRM)
- **Get contact data for tagging** (KlickTipp get) *(continues into Block 6)*

#### Node: Contact Tagged in KlickTipp
- **Type / role:** KlickTipp Trigger node; entry point from KlickTipp webhook system.
- **Config:** Uses KlickTipp credentials; webhookId empty in JSON (generated by n8n).
- **Edge cases:** Trigger requires KlickTipp-side webhook/tag setup (see sticky note “Documentation”).

#### Node: Check Subscription in Zoho
- **Type / role:** Switch; decides if Zoho contact should be created or updated.
- **Rules:**
  - **New Contact:** `field226649` is empty (no Zoho ID stored in KlickTipp)
  - **Update Contact:** `field226649` not empty
- **Edge cases:** If `field226649` populated with wrong ID, updates wrong Zoho record.

#### Node: Check for last name in KlickTipp
- **Type / role:** Filter; requires `fieldLastName` exists (Zoho requires last name for contacts).
- **Connection:** New Contact branch → **Create a contact in Zoho**
- **Edge cases:** Contacts without last name will not sync to Zoho unless you implement a fallback.

#### Node: Create a contact in Zoho
- **Type / role:** Zoho CRM `contact.create`.
- **Config choices:**
  - `lastName = {{$json.fieldLastName}}`
  - Sets common fields from KlickTipp fields: email, phones, fax, first name, address, etc.
  - Saves KlickTipp subscriber id into Zoho custom field `KlickTipp_ID = {{$json.id}}`
  - Converts birthday epoch seconds to `YYYY-MM-DD` using:  
    `{{ new Date($json.fieldBirthday * 1000).toLocaleDateString('en-CA') }}`
- **Connection:** → **Add Zoho CRM Contact ID to KlickTipp Contact**
- **Edge cases:**
  - If `fieldBirthday` is empty/non-numeric → date expression yields “Invalid Date”.
  - Zoho API may reject invalid date formats or missing required fields.

#### Node: Add Zoho CRM Contact ID to KlickTipp Contact
- **Type / role:** KlickTipp `subscriber.update` (by email lookup) to store Zoho ID into KlickTipp custom field.
- **Config:**
  - Updates `field226649 = {{$json.id}}` (Zoho contact id returned by create)
  - Uses `lookupEmail = {{ $('Contact Tagged in KlickTipp').item.json.email }}`
- **Connection:** → **Get contact data for tagging**
- **Edge cases:**
  - If email differs between trigger payload and subscriber record, lookup may update wrong/no subscriber.
  - If multiple items, cross-node reference can mis-pair.

#### Node: Check for Zoho contact ID
- **Type / role:** Filter; requires `field226649` exists and `fieldLastName` not empty.
- **Connection:** Update branch → **Update a contact in Zoho**
- **Edge cases:** Prevents updating Zoho when required data missing, but also blocks partial updates.

#### Node: Update a contact in Zoho
- **Type / role:** Zoho CRM `contact.update`.
- **Config:** Similar mapping to create; updates the Zoho record identified by `contactId = {{$json.field226649}}`.
- **Connection:** → **Get contact data for tagging**
- **Edge cases:** Invalid Zoho ID → 404/400; birthday conversion issues.

---

### Block 6 — KlickTipp → Zoho: Segmentation tagging in Zoho (based on KlickTipp tag IDs)
**Overview:**  
After creating/updating the Zoho contact, the workflow fetches the subscriber’s tags from KlickTipp and applies corresponding tags in Zoho using direct Zoho API calls.

**Nodes involved:**
- **Get contact data for tagging** (KlickTipp get)
- **Check relevant Segment 1** (Switch)
- **Tag Contact in Zoho CRM** (HTTP Request)
- **Tag Contact in Zoho CRM1** (HTTP Request)

#### Node: Get contact data for tagging
- **Type / role:** KlickTipp `subscriber.get` by email.
- **Config:** `lookupEmail = {{ $('Contact Tagged in KlickTipp').item.json.email }}`
- **Output:** includes `tags` array with tag IDs.
- **Edge cases:** If the trigger event doesn’t include email, lookup fails.

#### Node: Check relevant Segment 1
- **Type / role:** Switch; routes based on KlickTipp tag IDs.
- **Config:**
  - `allMatchingOutputs = true`
  - “Online store” if `$json.tags` contains `13879589`
  - “ABC” if contains `13879590`
- **Edge cases:** If tags come as strings vs numbers inconsistently, `contains` may fail.

#### Node: Tag Contact in Zoho CRM
- **Type / role:** HTTP Request to Zoho CRM tags endpoint.
- **Config choices:**
  - `POST https://www.zohoapis.eu/crm/v3/Contacts/actions/add_tags`
  - Auth: predefined credential type `zohoOAuth2Api`
  - JSON body uses `{{$json.field226649}}` and tag name `"Online Store"`
- **Edge cases:**
  - Zoho tag must exist or Zoho must be allowed to create tags implicitly (behavior depends on Zoho settings).
  - Region endpoint is EU; using wrong region breaks auth.

#### Node: Tag Contact in Zoho CRM1
- **Type / role:** Same as above but adds `"ABC"` tag.
- **Edge cases:** Same as above.

---

### Block 7 — Documentation / Sticky Notes (canvas guidance)
**Overview:**  
Provides embedded operational guidance and a visual structure (steps 1–7). These do not execute but are important for maintainability and setup.

**Nodes involved (sticky notes):**
- **Documentation**
- **1. Get contact data.**
- **2. Route by subscription.**
- **3. Filter contacts.**
- **4. Transfer contact data.**
- **5. Save contact ID**
- **6. Get full contact.**
- **7. Segmentation.**

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Documentation | Sticky Note | In-canvas documentation |  |  | Community Node Disclaimer: This workflow uses KlickTipp community nodes. / Introduction / How it works / Setup Instructions / Customization (full text in note) |
| 1. Get contact data. | Sticky Note | Visual step label |  |  | ## 1. Get contact data. |
| 2. Route by subscription. | Sticky Note | Visual step label |  |  | ## 2. Route by subscription. |
| 3. Filter contacts. | Sticky Note | Visual step label |  |  | ## 3. Filter contacts. |
| 4. Transfer contact data. | Sticky Note | Visual step label |  |  | ## 4. Transfer contact data. |
| 5. Save contact ID | Sticky Note | Visual step label |  |  | ## 5. Save contact ID |
| 6. Get full contact. | Sticky Note | Visual step label |  |  | ## 6. Get full contact. |
| 7. Segmentation. | Sticky Note | Visual step label |  |  | ## 7. Segmentation. |
| Contact registered via Form in Zoho CRM | Webhook | Entry: Zoho form registration event |  | Check for email address2 | ## 1. Get contact data. |
| Check for email address2 | Filter | Guard: require email before DOI subscribe | Contact registered via Form in Zoho CRM | Create Contact with DOI | ## 3. Filter contacts. |
| Create Contact with DOI | KlickTipp | Subscribe (DOI list/tag) from Zoho form contact | Check for email address2 | Add KlickTipp Contact ID2 | ## 4. Transfer contact data. |
| Add KlickTipp Contact ID2 | Zoho CRM | Persist KlickTipp ID into Zoho custom field | Create Contact with DOI |  | ## 5. Save contact ID |
| Contact created or updated in Zoho CRM | Webhook | Entry: Zoho contact create/update event |  | Check Subscription | ## 1. Get contact data. |
| Check Subscription | Switch | Route: new vs unsubscribed vs update | Contact created or updated in Zoho CRM | Check for email address; Check for email address1; Check for KlickTipp ID1 | ## 2. Route by subscription. |
| Check for email address | Filter | Guard for SOI creation | Check Subscription (New Contact) | Create Contact with SOI | ## 3. Filter contacts. |
| Create Contact with SOI | KlickTipp | Subscribe (SOI list) from Zoho contact | Check for email address | Add KlickTipp Contact ID; Get contact data for tagging1 | ## 4. Transfer contact data. |
| Add KlickTipp Contact ID | Zoho CRM | Persist KlickTipp ID into Zoho custom field | Create Contact with SOI | Tag contact for first transactional mail | ## 5. Save contact ID |
| Tag contact for first transactional mail | KlickTipp | Apply transactional tag after creation | Add KlickTipp Contact ID |  | ## 7. Segmentation. |
| Check for email address1 | Filter | Guard for unsubscribe | Check Subscription (Unsubscribed) | Unsubscribe contact | ## 3. Filter contacts. |
| Unsubscribe contact | KlickTipp | Unsubscribe subscriber by email | Check for email address1 |  | ## 4. Transfer contact data. |
| Check for KlickTipp ID1 | Filter | Guard for update by KlickTipp ID | Check Subscription (Update Contact) | Update contact | ## 3. Filter contacts. |
| Update contact | KlickTipp | Update subscriber by ID using Zoho fields | Check for KlickTipp ID1 | Get contact data for tagging1 | ## 4. Transfer contact data. |
| Get contact data for tagging1 | Zoho CRM | Fetch full Zoho contact for segmentation | Create Contact with SOI; Update contact | Check relevant Segment  | ## 6. Get full contact. |
| Check relevant Segment  | Switch | Decide KlickTipp tags based on Zoho tags | Get contact data for tagging1 | Tag contact in KlickTipp; Tag contact in KlickTipp1 | ## 7. Segmentation. |
| Tag contact in KlickTipp | KlickTipp | Apply “Online Store” tag in KlickTipp | Check relevant Segment  |  | ## 7. Segmentation. |
| Tag contact in KlickTipp1 | KlickTipp | Apply “ABC” tag in KlickTipp | Check relevant Segment  |  | ## 7. Segmentation. |
| Contact deleted in Zoho CRM | Webhook | Entry: Zoho contact deletion event |  | Check for KlickTipp ID | ## 1. Get contact data. |
| Check for KlickTipp ID | Filter | Guard for delete by KlickTipp ID | Contact deleted in Zoho CRM | Delete contact in KlickTipp | ## 3. Filter contacts. |
| Delete contact in KlickTipp | KlickTipp | Delete subscriber by ID | Check for KlickTipp ID |  | ## 4. Transfer contact data. |
| Contact Tagged in KlickTipp | KlickTipp Trigger | Entry: KlickTipp tagging event |  | Check Subscription in Zoho | ## 1. Get contact data. |
| Check Subscription in Zoho | Switch | Route: create vs update Zoho contact | Contact Tagged in KlickTipp | Check for last name in KlickTipp; Check for Zoho contact ID | ## 2. Route by subscription. |
| Check for last name in KlickTipp | Filter | Guard: Zoho requires last name | Check Subscription in Zoho (New Contact) | Create a contact in Zoho | ## 3. Filter contacts. |
| Create a contact in Zoho | Zoho CRM | Create Zoho contact from KlickTipp data | Check for last name in KlickTipp | Add Zoho CRM Contact ID to KlickTipp Contact | ## 4. Transfer contact data. |
| Add Zoho CRM Contact ID to KlickTipp Contact | KlickTipp | Write Zoho ID into KlickTipp custom field | Create a contact in Zoho | Get contact data for tagging | ## 5. Save contact ID |
| Check for Zoho contact ID | Filter | Guard: update only if Zoho ID exists + last name | Check Subscription in Zoho (Update Contact) | Update a contact in Zoho | ## 3. Filter contacts. |
| Update a contact in Zoho | Zoho CRM | Update Zoho contact from KlickTipp data | Check for Zoho contact ID | Get contact data for tagging | ## 4. Transfer contact data. |
| Get contact data for tagging | KlickTipp | Fetch subscriber tags for Zoho segmentation | Add Zoho CRM Contact ID to KlickTipp Contact; Update a contact in Zoho | Check relevant Segment 1 | ## 6. Get full contact. |
| Check relevant Segment 1 | Switch | Decide Zoho tags based on KlickTipp tag IDs | Get contact data for tagging | Tag Contact in Zoho CRM; Tag Contact in Zoho CRM1 | ## 7. Segmentation. |
| Tag Contact in Zoho CRM | HTTP Request | Add “Online Store” tag in Zoho | Check relevant Segment 1 |  | ## 7. Segmentation. |
| Tag Contact in Zoho CRM1 | HTTP Request | Add “ABC” tag in Zoho | Check relevant Segment 1 |  | ## 7. Segmentation. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1. Create **Zoho OAuth2** credential in n8n (`zohoOAuth2Api`):
      - Use your Zoho API Console **Client ID** and **Client Secret**.
      - Ensure the Zoho region matches your account (this workflow uses `zohoapis.eu`).
   2. Create **KlickTipp API** credential (community node):
      - Configure username/password (API access required).

2. **Create three Zoho webhook entry nodes**
   1. Add **Webhook** node named **Contact registered via Form in Zoho CRM**
      - Method: `POST`
      - Path: set a unique path (e.g., `zoho-form-contact`)
   2. Add **Webhook** node named **Contact created or updated in Zoho CRM**
      - Method: `POST`
      - Path: e.g., `zoho-contact-upsert`
   3. Add **Webhook** node named **Contact deleted in Zoho CRM**
      - Method: `POST`
      - Path: e.g., `zoho-contact-delete`

3. **Zoho Form → KlickTipp (DOI) branch**
   1. Add **Filter** node **Check for email address2**
      - Condition: `{{$json.body.email}}` **exists**
   2. Add **KlickTipp** node **Create Contact with DOI**
      - Resource: `subscriber`
      - Operation: `subscribe`
      - Email: `{{$json.body.email}}`
      - List ID: `358895`
      - Tag ID: `13939731`
      - Data fields mapping:
        - `field226649 = {{$json.body.id}}` (Zoho contact ID)
        - map first/last name, phones, address, birthday, etc. as in workflow
   3. Add **Zoho CRM** node **Add KlickTipp Contact ID2**
      - Resource: `contact`, Operation: `update`
      - Contact ID: `{{$json.field226649}}`
      - Custom field: `KlickTipp_ID = {{$json.id}}`
   4. Connect: Webhook → Filter → KlickTipp subscribe → Zoho update

4. **Zoho Create/Update → KlickTipp routing**
   1. Add **Switch** node **Check Subscription**
      - Outputs:
        - “New Contact”: `{{$json.body.klicktippId}}` is empty
        - “Unsubscribed”: `{{$json.body.emailOptOut}}` is true
        - “Update Contact”: `{{$json.body.klicktippId}}` is not empty
   2. **New Contact path**
      - Add **Filter** **Check for email address**: `{{$json.body.email}}` exists
      - Add **KlickTipp** **Create Contact with SOI**
        - Resource: `subscriber`, Operation: `subscribe`
        - Email: `{{$json.body.email}}`
        - List ID: `364353`
        - Map Zoho fields and set `field226649 = {{$json.body.id}}`
      - Add **Zoho CRM** **Add KlickTipp Contact ID**
        - Update custom field `KlickTipp_ID = {{$json.id}}` for `contactId = {{$json.field226649}}`
      - Add **KlickTipp** **Tag contact for first transactional mail**
        - Resource: `contact-tagging`
        - Email: `{{ $('Create Contact with SOI').item.json.email }}`
        - tagId: `13879733`
      - Connect: Switch(New) → Filter → Create SOI → Zoho update → Tag
   3. **Unsubscribed path**
      - Add **Filter** **Check for email address1**: `{{$json.body.email}}` exists
      - Add **KlickTipp** **Unsubscribe contact**
        - Resource: `subscriber`, Operation: `unsubscribe`
        - Email: `{{$json.body.email}}`
      - Connect: Switch(Unsubscribed) → Filter → Unsubscribe
   4. **Update Contact path**
      - Add **Filter** **Check for KlickTipp ID1**: `{{$json.body.klicktippId}}` exists
      - Add **KlickTipp** **Update contact**
        - Resource: `subscriber`, Operation: `update`
        - Identifier type: `id`
        - subscriberId: `{{$json.body.klicktippId}}`
        - Email: `{{$json.body.email}}`
        - Map Zoho fields to KlickTipp fields
      - Connect: Switch(Update) → Filter → Update contact

5. **Zoho → KlickTipp segmentation**
   1. Add **Zoho CRM** node **Get contact data for tagging1**
      - Resource: `contact`, Operation: `get`
      - contactId: `{{ $('Contact created or updated in Zoho CRM').item.json.body.id }}`
   2. Feed into **Switch** node **Check relevant Segment **
      - Enable “All matching outputs”
      - Rule “Online store”: expression checks Zoho tags array for name “Online Store”
      - Rule “ABC”: expression checks for “ABC”
   3. Add two **KlickTipp contact-tagging** nodes:
      - **Tag contact in KlickTipp** tagId `13879589` using `email = {{$json.email}}`
      - **Tag contact in KlickTipp1** tagId `13879590` using `email = {{$json.email}}`
   4. Connect: Zoho get → Switch → taggers
   5. Ensure **Create Contact with SOI** and **Update contact** both connect to **Get contact data for tagging1** (as in workflow).

6. **Zoho Delete → KlickTipp delete**
   1. Add **Filter** **Check for KlickTipp ID**: `{{$json.body.klicktippId}}` exists
   2. Add **KlickTipp** node **Delete contact in KlickTipp**
      - Resource: `subscriber`, Operation: `delete`
      - identifierType: `id`
      - subscriberId: `{{$json.body.klicktippId}}`
   3. Connect: Delete webhook → Filter → KlickTipp delete

7. **KlickTipp → Zoho trigger + create/update**
   1. Add **KlickTipp Trigger** node **Contact Tagged in KlickTipp**
      - Configure KlickTipp credentials
      - In KlickTipp, configure a webhook that fires on an “activation tag” (per note: `Zoho | Send contact`) pointing to this trigger URL.
   2. Add **Switch** **Check Subscription in Zoho**
      - “New Contact” if `{{$json.field226649}}` is empty
      - “Update Contact” if `{{$json.field226649}}` is not empty
   3. New Contact path:
      - Add **Filter** **Check for last name in KlickTipp**: `{{$json.fieldLastName}}` exists
      - Add **Zoho CRM** **Create a contact in Zoho**
        - last name required: `{{$json.fieldLastName}}`
        - map fields from KlickTipp fields
        - set Zoho custom field `KlickTipp_ID = {{$json.id}}`
        - convert birthday: `new Date($json.fieldBirthday * 1000).toLocaleDateString('en-CA')`
      - Add **KlickTipp** **Add Zoho CRM Contact ID to KlickTipp Contact**
        - Operation: update subscriber
        - lookupEmail: `{{ $('Contact Tagged in KlickTipp').item.json.email }}`
        - set `field226649 = {{$json.id}}` (Zoho contact id)
   4. Update Contact path:
      - Add **Filter** **Check for Zoho contact ID**: `field226649` exists AND last name not empty
      - Add **Zoho CRM** **Update a contact in Zoho**
        - contactId: `{{$json.field226649}}`
        - map fields similarly
   5. Both create/update should connect to the next “get subscriber” node.

8. **KlickTipp → Zoho segmentation**
   1. Add **KlickTipp** node **Get contact data for tagging**
      - Resource: `subscriber`, Operation: `get`
      - lookupEmail: `{{ $('Contact Tagged in KlickTipp').item.json.email }}`
   2. Add **Switch** **Check relevant Segment 1**
      - Enable “All matching outputs”
      - “Online store” if `{{$json.tags}}` contains `13879589`
      - “ABC” if contains `13879590`
   3. Add **HTTP Request** node **Tag Contact in Zoho CRM**
      - POST `https://www.zohoapis.eu/crm/v3/Contacts/actions/add_tags`
      - Auth: predefined Zoho OAuth2 credential
      - JSON body with `ids: [{{$json.field226649}}]` and tag name `"Online Store"`
   4. Add **HTTP Request** node **Tag Contact in Zoho CRM1**
      - Same endpoint, tag name `"ABC"`
   5. Connect: KlickTipp get → switch → each HTTP tagger

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Community Node Disclaimer: This workflow uses KlickTipp community nodes. | From sticky note “Documentation” |
| This workflow creates a complete two-way synchronization between KlickTipp and Zoho CRM. | From sticky note “Documentation” |
| KlickTipp Preparation: create custom field `Zoho | Contact ID` and tags (`Zoho | Send contact`, `Zoho | Contact created`, `Zoho | Contact created via Form`, `Zoho | Online store`, `Zoho | ABC`, …). Configure webhook activation tag `Zoho | Send contact`. | From sticky note “Documentation” |
| Zoho CRM Preparation: create workflow rules (form-created, created/updated, deleted) calling the respective webhooks. Create custom field `KlickTipp ID`. | From sticky note “Documentation” |
| Credential configuration guidance: Zoho Client ID/Secret + KlickTipp username/password. | From sticky note “Documentation” |
| Customization: expand field mapping; add segmentation branches in “Check relevant Segment”. | From sticky note “Documentation” |