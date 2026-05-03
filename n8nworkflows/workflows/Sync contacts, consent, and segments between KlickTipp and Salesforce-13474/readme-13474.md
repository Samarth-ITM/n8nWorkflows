Sync contacts, consent, and segments between KlickTipp and Salesforce

https://n8nworkflows.xyz/workflows/sync-contacts--consent--and-segments-between-klicktipp-and-salesforce-13474


# Sync contacts, consent, and segments between KlickTipp and Salesforce

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Sync contacts, consent, and segments between KlickTipp and Salesforce — Reference Document

## 1. Workflow Overview

This workflow implements a **two-way synchronization** between **KlickTipp subscribers** and **Salesforce Contacts**, including:
- **Contact creation/update sync** in both directions
- **Consent/marketing status sync** (KlickTipp status → Salesforce custom field)
- **Segmentation sync** between **KlickTipp Tags** and **Salesforce Topics**
- **Deletion handling**
  - Daily cleanup: if a Salesforce contact was deleted, delete the corresponding KlickTipp subscriber
  - GDPR deletion: if a subscriber requests deletion in KlickTipp, delete the linked Salesforce contact

### 1.1 KlickTipp → Salesforce (subscriber event webhook)
Triggered by KlickTipp, then creates/updates Salesforce contact, writes IDs back to KlickTipp, and maps tags → Salesforce topics.

### 1.2 Salesforce → KlickTipp (polling triggers for create/update)
Triggered by Salesforce polling on contact created/updated, normalizes data, prevents infinite loops, creates/updates KlickTipp subscriber, then syncs status + segments back.

### 1.3 Deletion sync: Salesforce deleted → KlickTipp delete (daily scheduled)
Daily query of deleted Salesforce contacts in last 24 hours; deletes linked subscribers in KlickTipp.

### 1.4 GDPR deletion: KlickTipp deletion request → Salesforce delete
Triggered by KlickTipp deletion link; deletes Salesforce record if a Salesforce ID is present.

---

## 2. Block-by-Block Analysis

### Block A — KlickTipp → Salesforce: contact upsert + segmentation
**Overview:** When KlickTipp triggers (tag/webhook), the workflow checks if the subscriber already has a Salesforce Contact ID stored. It then creates or updates the Salesforce contact, stores the Salesforce ID back into KlickTipp, and maps KlickTipp tags to Salesforce Topics.

**Nodes involved:**
- Contact tagged in KlickTipp
- Salesforce Lookup Switch
- Create Salesforce Contact
- Update Salesforce Contact
- Sync SF ID to KlickTipp
- Get KlickTipp Tags
- Segment Matcher (KT to SF)
- Assign SF Topic (Customer)
- Assign SF Topic (ABC)

#### Node: Contact tagged in KlickTipp
- **Type/role:** `klicktippTrigger` — entry point via webhook / activation tag.
- **Config:** Webhook path/id `c7f3e542-0814-4121-8b46-1cec97c277f7`.
- **Key output fields used elsewhere:** `id`, `email`, `status`, `fieldFirstName`, `fieldLastName`, `fieldBirthday`, `fieldCity`, `fieldMobilePhone`, `fieldStreet1`, and custom field `field227883` (Salesforce Contact ID in KlickTipp).
- **Outputs:** To **Salesforce Lookup Switch**.
- **Failure modes:** KlickTipp credential/webhook configuration issues; missing expected fields (custom fields not configured in KlickTipp).

#### Node: Salesforce Lookup Switch
- **Type/role:** `if` — route based on whether Salesforce ID is missing in KlickTipp.
- **Condition logic:** `={{ !$json.field227883 }}` is **true** when Salesforce ID is absent.
- **Routing:**
  - **True** → Create Salesforce Contact
  - **False** → Update Salesforce Contact
- **Failure/edge cases:** If `field227883` is present but invalid (deleted Contact ID), update will fail.

#### Node: Create Salesforce Contact
- **Type/role:** `salesforce` — create Contact in Salesforce.
- **Config highlights:**
  - `lastname` from KlickTipp `fieldLastName`
  - Additional fields: `email`, `firstName`, address/phone, postal code.
  - Writes custom fields:
    - `KlickTipp_ID__c` = KlickTipp subscriber `id`
    - `KlickTipp_marketing_status__c` = KlickTipp `status`
  - Birthdate conversion from KlickTipp UNIX seconds → Salesforce `yyyy-MM-dd` using Luxon `DateTime.fromMillis(Number(fieldBirthday)*1000, { zone: 'Europe/Berlin' })`
- **Outputs:** To **Sync SF ID to KlickTipp**
- **Failure modes:** Salesforce auth, validation rules (required fields), date parsing if `fieldBirthday` not numeric.

#### Node: Update Salesforce Contact
- **Type/role:** `salesforce` — update existing Contact.
- **Config highlights:**
  - `contactId` = `={{ $('Contact tagged in KlickTipp').item.json.field227883 }}`
  - Updates standard fields + `KlickTipp_marketing_status__c` from KlickTipp `status`.
  - Birthdate conversion same as above.
- **Outputs:** To **Get KlickTipp Tags**
- **Failure modes:** invalid/missing Contact ID, insufficient field permissions, birthdate conversion issues.

#### Node: Sync SF ID to KlickTipp
- **Type/role:** `klicktipp` — writes Salesforce ID into KlickTipp custom field for 2-way matching.
- **Config:**
  - Operation: subscriber update by `id`
  - Updates data field `field227883` = `={{ $json.id }}` (Salesforce created Contact ID).
  - Uses subscriberId = KlickTipp `id` from trigger.
- **Outputs:** To **Get KlickTipp Tags**
- **Failure modes:** Wrong KlickTipp credential/account, missing custom field in KlickTipp.

#### Node: Get KlickTipp Tags
- **Type/role:** `klicktipp` — fetch full subscriber including `tags`.
- **Config:** subscriber get by `id` from trigger.
- **Outputs:** To **Segment Matcher (KT to SF)**.
- **Failure modes:** subscriber not found; auth errors.

#### Node: Segment Matcher (KT to SF)
- **Type/role:** `switch` — routes based on presence of specific KlickTipp tag IDs.
- **Rules (allMatchingOutputs=true):**
  - Output “Contact has Tag Customer” if `tags` contains `14061012`
  - Output “Contact has Tag ABC” if `tags` contains `14061060`
- **Outputs:**
  - Customer → Assign SF Topic (Customer)
  - ABC → Assign SF Topic (ABC)
- **Edge cases:** `tags` may be empty/undefined depending on KlickTipp response; ensure it’s an array.

#### Node: Assign SF Topic (Customer)
- **Type/role:** `httpRequest` — create Salesforce `TopicAssignment`.
- **Config:**
  - POST to `/services/data/v60.0/sobjects/TopicAssignment`
  - Body includes `TopicId: "0TOfj000000GO9dGAG"`
  - `EntityId` is intended to be the Salesforce Contact ID.
- **Important issue:** Expression references `{{ $('Create a contact').item.json.id }}` but the actual node name is **Create Salesforce Contact**. This will fail unless renamed or corrected.
- **onError:** `continueRegularOutput` (errors won’t stop workflow).
- **Failure modes:** wrong instance URL, missing TopicId, duplicate assignment (Salesforce may reject), auth/permission issues.

#### Node: Assign SF Topic (ABC)
- Same as above, TopicId `"0TOfj000000GOG5GAO"`.
- Same **node-name reference issue** for `EntityId`.

---

### Block B — Salesforce → KlickTipp: create/update subscriber + status + segmentation
**Overview:** When Salesforce contacts are created/updated, the workflow maps Salesforce fields to KlickTipp fields, avoids loopback updates, then creates or updates the KlickTipp subscriber. It then syncs consent status back to Salesforce and maps Salesforce Topics → KlickTipp tags.

**Nodes involved:**
- New Salesforce Contact
- Updated Salesforce Contact
- Data Normalization & Mapping
- Ignore API Auto-Updates
- KlickTipp Lookup Switch
- Create KlickTipp Subscriber
- Update KlickTipp Subscriber
- Check Subscription Status
- Sync Marketing Status to SF
- Sync KlickTipp ID to SF
- Get Salesforce Topics
- Segment Matcher (SF to KT)
- Apply Default Tag
- Apply Tag: Customer
- Apply Tag: ABC

#### Node: New Salesforce Contact
- **Type/role:** `salesforceTrigger` — polling trigger on contact creation.
- **Config:** polls every minute (`mode: everyMinute`), `triggerOn: contactCreated`.
- **Outputs:** To **Data Normalization & Mapping**.
- **Failure modes:** Salesforce polling limits, auth expiry.

#### Node: Updated Salesforce Contact
- **Type/role:** `salesforceTrigger` — polling trigger on contact update.
- **Config:** every minute, `triggerOn: contactUpdated`.
- **Outputs:** To **Data Normalization & Mapping**.
- **Failure modes:** polling limits; may emit updates caused by this workflow (loop risk).

#### Node: Data Normalization & Mapping
- **Type/role:** `set` — canonical mapping between Salesforce schema and KlickTipp fields.
- **Key mappings:**
  - `KlickTipp_ID` = `KlickTipp_ID__c`
  - `Email` = `Email`
  - `First_name` = `FirstName`
  - `Last_name` = `LastName`
  - `Salesforce_ID` = `Id`
  - `Birthday` = converts Salesforce `Birthdate` to UNIX seconds (string), returns `''` if missing:
    - `new Date(birthday)` → `Math.floor(getTime()/1000)`
  - `KlickTipp_ID_exists` boolean: true if `KlickTipp_ID__c` is non-empty after trim
- **Outputs:** To **Ignore API Auto-Updates**.
- **Edge cases:** Salesforce `Birthdate` parsing depends on format; if invalid, Date becomes `Invalid Date` → `NaN`.

#### Node: Ignore API Auto-Updates
- **Type/role:** `filter` — prevents infinite sync loops.
- **Config:** allows records only if `LastModifiedById != 005fj00000AdU0UAAV` (the integration/API user).
- **Input source in expression:** `Updated Salesforce Contact` node output.
- **Outputs:** To **KlickTipp Lookup Switch**.
- **Edge cases:** For “New Salesforce Contact” events, `Updated Salesforce Contact` context might not exist; however n8n will still evaluate with available item context. If expression resolution fails, filtering may behave unexpectedly—consider referencing `$json.LastModifiedById` directly instead.

#### Node: KlickTipp Lookup Switch
- **Type/role:** `if` — decides create vs update in KlickTipp based on `KlickTipp_ID_exists`.
- **Condition:** checks boolean is **false** → “Contact ID is missing”
- **Routing:**
  - **True branch (ID missing)** → Create KlickTipp Subscriber
  - **False branch (ID exists)** → Update KlickTipp Subscriber
- **Failure modes:** If `KlickTipp_ID__c` exists but is wrong, update will fail.

#### Node: Create KlickTipp Subscriber
- **Type/role:** `klicktipp` — subscribe/create with Single Opt-In.
- **Config:**
  - operation: `subscriber.subscribe`
  - listId: `364353`
  - tagId: `14041983` (initial subscription/sync tag)
  - fields:
    - `fieldFirstName`, `fieldLastName`, `fieldBirthday` (UNIX seconds), `field227883` (Salesforce_ID)
- **Outputs:** To **Sync KlickTipp ID to SF**
- **Failure modes:** email already exists (behavior depends on KlickTipp), invalid list/tag IDs, missing required fields.

#### Node: Sync KlickTipp ID to SF
- **Type/role:** `salesforce` — write back newly created KlickTipp subscriber ID.
- **Config:**
  - updates Contact with:
    - `KlickTipp_ID__c` = KlickTipp subscriber `id` (from prior node)
    - `KlickTipp_marketing_status__c` = literal `"Subscribed"`
- **Outputs:** To **Get Salesforce Topics** and **Apply Default Tag** (parallel branches).
- **Edge cases:** If Salesforce update triggers another polling update, loop protection must work.

#### Node: Update KlickTipp Subscriber
- **Type/role:** `klicktipp` — update existing subscriber.
- **Config:**
  - updates by subscriber `id` = `KlickTipp_ID`
  - writes fields: first/last name, birthday, and **field227685** = Salesforce_ID
- **Potential issue:** Elsewhere the Salesforce ID field is `field227883`. Here it uses `field227685`, which may be a typo or different field. If incorrect, Salesforce ID will not be stored consistently.
- **Outputs:** To **Check Subscription Status**
- **Failure modes:** subscriber not found; wrong field IDs.

#### Node: Check Subscription Status
- **Type/role:** `klicktipp` — fetch subscriber to get current `status`.
- **Config:** get subscriber by `KlickTipp_ID`.
- **Outputs:** To **Sync Marketing Status to SF**
- **Failure modes:** subscriber not found, API limits.

#### Node: Sync Marketing Status to SF
- **Type/role:** `salesforce` — updates Salesforce consent/marketing status.
- **Config:** sets `KlickTipp_marketing_status__c` = `={{ $json.status }}`
- **Outputs:** To **Get Salesforce Topics**
- **Failure modes:** field permission issues.

#### Node: Get Salesforce Topics
- **Type/role:** `httpRequest` — query Salesforce TopicAssignment for the Contact.
- **Config:**
  - GET to `/services/data//v60.0/query` (note the double slash after `data`)
  - SOQL: `SELECT TopicId, Topic.Name FROM TopicAssignment WHERE EntityId = '{{ Salesforce_ID }}'`
- **Outputs:** To **Segment Matcher (SF to KT)**
- **Failure modes:** wrong instance URL, API version mismatch, TopicAssignment not enabled, permissions.

#### Node: Segment Matcher (SF to KT)
- **Type/role:** `switch` — routes based on whether Topics include given names.
- **Logic:** builds array of `Topic.Name` from records and checks `contains`.
- **Rules (allMatchingOutputs=true):**
  - “Topic Customer exists” if includes `"Customer"`
  - “Topic ABC exists” if includes `"ABC"`
- **Outputs:** Customer → Apply Tag: Customer; ABC → Apply Tag: ABC
- **Edge cases:** `records` may be empty; mapping code defensively filters.

#### Node: Apply Default Tag
- **Type/role:** `klicktipp` — applies base tag to ensure inclusion in sync segment.
- **Config:** resource `contact-tagging`, email = mapped Email, tagId `14152706`.
- **Input:** from **Sync KlickTipp ID to SF** (not from topic query).
- **Failure modes:** email not found in KlickTipp (depends on KlickTipp behavior), wrong tag ID.

#### Node: Apply Tag: Customer / Apply Tag: ABC
- **Type/role:** `klicktipp` — apply segmentation tags.
- **Config:**
  - Customer tagId `14061012`
  - ABC tagId `14061060`
- **Failure modes:** same as above.

---

### Block C — Daily cleanup: Salesforce deleted → KlickTipp delete
**Overview:** Once per day, query Salesforce for deleted contacts in last day and delete corresponding KlickTipp subscribers.

**Nodes involved:**
- Daily Cleanup Trigger
- Fetch Deleted SF Contacts
- Delete contact

#### Node: Daily Cleanup Trigger
- **Type/role:** `scheduleTrigger` — daily time-based trigger.
- **Config:** triggers at hour `8` (server timezone of n8n unless configured).
- **Outputs:** to **Fetch Deleted SF Contacts**.

#### Node: Fetch Deleted SF Contacts
- **Type/role:** `httpRequest` — SOQL queryAll against Salesforce for deleted contacts.
- **Config:**
  - URL: `/services/data//v59.0/queryAll` (double slash after `data`)
  - Query: `SELECT Id, Email, SystemModstamp, KlickTipp_ID__c FROM Contact WHERE IsDeleted = true AND Email != null AND SystemModstamp = LAST_N_DAYS:1 ORDER BY SystemModstamp DESC`
  - Auth: Salesforce OAuth2 credential (predefined)
- **Outputs:** to **Delete contact**
- **Edge cases / issues:**
  - Salesforce query returns `{ records: [...] }`. n8n HTTP Request returns **one item** with a `records` array, not one item per record—this matters because the next node uses `records[0]` only (see below).
  - The URL must be updated to your org domain and should not contain accidental double slashes.

#### Node: Delete contact
- **Type/role:** `klicktipp` — delete subscriber.
- **Config:** subscriber delete, `subscriberId = {{ $json.records[0].KlickTipp_ID__c }}`
- **onError:** continue regular output (won’t stop workflow).
- **Critical limitation:** only deletes the **first** record (`records[0]`). To delete all, you’d typically add **Item Lists / Split Out Items** (or Code node) to iterate over each record.
- **Failure modes:** missing `KlickTipp_ID__c`, subscriber already deleted, wrong credential/account (note: credential name differs from other KlickTipp nodes).

---

### Block D — GDPR deletion: KlickTipp request → Salesforce delete
**Overview:** When a subscriber requests deletion in KlickTipp, the workflow deletes the linked Salesforce Contact, but only if a Salesforce ID is present.

**Nodes involved:**
- GDPR Deletion Request Trigger
- Is SF Link Present?
- GDPR Delete Salesforce Contact

#### Node: GDPR Deletion Request Trigger
- **Type/role:** `klicktippTrigger` — event triggered by deletion link click.
- **Config:** webhookId `77e920ed-b328-431f-9e51-e27e279898da` (path not explicitly shown in parameters).
- **Outputs:** to **Is SF Link Present?**
- **Failure modes:** webhook not registered/active, missing Salesforce ID field in payload.

#### Node: Is SF Link Present?
- **Type/role:** `filter` — ensures Salesforce delete only runs when `field227883` exists.
- **Condition:** checks existence of `={{ $json.field227883 }}`
- **Outputs:** to **GDPR Delete Salesforce Contact**
- **Edge cases:** if Salesforce ID stored in a different field (see earlier inconsistency), deletion may be skipped incorrectly.

#### Node: GDPR Delete Salesforce Contact
- **Type/role:** `salesforce` — delete Contact.
- **Config:** `contactId = {{ $('GDPR Deletion Request Trigger').item.json.field227883 }}`
- **Failure modes:** delete permission missing; record already deleted; Salesforce may soft-delete depending on org settings (API “delete” typically soft-deletes unless hard delete is used).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Documentation Sticky | stickyNote | Documentation panel |  |  | (content) How it works + Setup steps (see note in section 5) |
| Sticky Note2 | stickyNote | Visual grouping: “1. Get contact data.” |  |  | ## 1. Get contact data. |
| Sticky Note | stickyNote | Visual grouping: “2. Filter” |  |  | ## 2. Filter |
| Sticky Note6 | stickyNote | Visual grouping: “3. Route by subscription.” |  |  | ## 3. Route by subscription. |
| Sticky Note7 | stickyNote | Visual grouping: “4. Transfer contact data.” |  |  | ## 4. Transfer contact data. |
| Sticky Note8 | stickyNote | Visual grouping: “5. Save IDs & Status” |  |  | ## 5. Save IDs & Status |
| Sticky Note15 | stickyNote | Visual grouping: “6. Get full contact” |  |  | ## 6. Get full contact |
| Sticky Note10 | stickyNote | Visual grouping: “7. Segmentation.” |  |  | ## 7. Segmentation. |
| Contact tagged in KlickTipp | klicktippTrigger | Entry point: KlickTipp → SF sync | — | Salesforce Lookup Switch | ## 1. Get contact data. |
| Salesforce Lookup Switch | if | Decide create vs update in Salesforce | Contact tagged in KlickTipp | Create Salesforce Contact; Update Salesforce Contact | ## 3. Route by subscription. |
| Create Salesforce Contact | salesforce | Create Salesforce Contact from KlickTipp | Salesforce Lookup Switch | Sync SF ID to KlickTipp | ## 4. Transfer contact data. |
| Update Salesforce Contact | salesforce | Update Salesforce Contact from KlickTipp | Salesforce Lookup Switch | Get KlickTipp Tags | ## 4. Transfer contact data. |
| Sync SF ID to KlickTipp | klicktipp | Store Salesforce ID back in KlickTipp | Create Salesforce Contact | Get KlickTipp Tags | ## 5. Save IDs & Status |
| Get KlickTipp Tags | klicktipp | Fetch subscriber tags | Sync SF ID to KlickTipp; Update Salesforce Contact | Segment Matcher (KT to SF) | ## 6. Get full contact |
| Segment Matcher (KT to SF) | switch | Map KlickTipp tag IDs → SF topics | Get KlickTipp Tags | Assign SF Topic (Customer); Assign SF Topic (ABC) | ## 7. Segmentation. |
| Assign SF Topic (Customer) | httpRequest | Create TopicAssignment “Customer” | Segment Matcher (KT to SF) | — | ## 7. Segmentation. |
| Assign SF Topic (ABC) | httpRequest | Create TopicAssignment “ABC” | Segment Matcher (KT to SF) | — | ## 7. Segmentation. |
| New Salesforce Contact | salesforceTrigger | Entry point: SF contact created | — | Data Normalization & Mapping | ## 1. Get contact data. |
| Updated Salesforce Contact | salesforceTrigger | Entry point: SF contact updated | — | Data Normalization & Mapping | ## 1. Get contact data. |
| Data Normalization & Mapping | set | Map/normalize SF → KlickTipp fields | New Salesforce Contact; Updated Salesforce Contact | Ignore API Auto-Updates | ## 1. Get contact data. |
| Ignore API Auto-Updates | filter | Prevent sync loop (ignore integration user updates) | Data Normalization & Mapping | KlickTipp Lookup Switch | ## 2. Filter |
| KlickTipp Lookup Switch | if | Decide create vs update in KlickTipp | Ignore API Auto-Updates | Create KlickTipp Subscriber; Update KlickTipp Subscriber | ## 3. Route by subscription. |
| Create KlickTipp Subscriber | klicktipp | Subscribe/create subscriber in KlickTipp | KlickTipp Lookup Switch | Sync KlickTipp ID to SF | ## 4. Transfer contact data. |
| Update KlickTipp Subscriber | klicktipp | Update existing KlickTipp subscriber | KlickTipp Lookup Switch | Check Subscription Status | ## 4. Transfer contact data. |
| Check Subscription Status | klicktipp | Fetch subscriber status for consent sync | Update KlickTipp Subscriber | Sync Marketing Status to SF | ## 5. Save IDs & Status |
| Sync KlickTipp ID to SF | salesforce | Save KlickTipp ID + status into Salesforce | Create KlickTipp Subscriber | Get Salesforce Topics; Apply Default Tag | ## 5. Save IDs & Status |
| Sync Marketing Status to SF | salesforce | Update SF marketing status from KlickTipp status | Check Subscription Status | Get Salesforce Topics | ## 5. Save IDs & Status |
| Get Salesforce Topics | httpRequest | Fetch Salesforce Topics for segmentation | Sync KlickTipp ID to SF; Sync Marketing Status to SF | Segment Matcher (SF to KT) | ## 6. Get full contact |
| Segment Matcher (SF to KT) | switch | Map Salesforce topic names → KlickTipp tags | Get Salesforce Topics | Apply Tag: Customer; Apply Tag: ABC | ## 7. Segmentation. |
| Apply Default Tag | klicktipp | Apply base sync tag | Sync KlickTipp ID to SF | — | ## 7. Segmentation. |
| Apply Tag: Customer | klicktipp | Apply Customer tag in KlickTipp | Segment Matcher (SF to KT) | — | ## 7. Segmentation. |
| Apply Tag: ABC | klicktipp | Apply ABC tag in KlickTipp | Segment Matcher (SF to KT) | — | ## 7. Segmentation. |
| Daily Cleanup Trigger | scheduleTrigger | Entry point: daily deletion sync | — | Fetch Deleted SF Contacts |  |
| Fetch Deleted SF Contacts | httpRequest | Query deleted SF contacts last 24h | Daily Cleanup Trigger | Delete contact |  |
| Delete contact | klicktipp | Delete KlickTipp subscriber for deleted SF contact | Fetch Deleted SF Contacts | — |  |
| GDPR Deletion Request Trigger | klicktippTrigger | Entry point: GDPR delete request from KlickTipp | — | Is SF Link Present? |  |
| Is SF Link Present? | filter | Only proceed if Salesforce ID exists | GDPR Deletion Request Trigger | GDPR Delete Salesforce Contact |  |
| GDPR Delete Salesforce Contact | salesforce | Delete Salesforce Contact (GDPR) | Is SF Link Present? | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Add **Salesforce OAuth2** credential (connected user must have access to Contact + custom fields + TopicAssignment if used).
   2. Add **KlickTipp API** credential(s). This workflow currently uses multiple KlickTipp credential entries (ensure you standardize to the correct account).

2) **Prepare Salesforce schema**
   1. On **Contact**, create custom field `KlickTipp_ID__c` (Text).
   2. On **Contact**, create custom field `KlickTipp_marketing_status__c` (Text or Picklist).
   3. Ensure **Topics** feature is enabled and you know your **Topic IDs** for “Customer” and “ABC”.

3) **Prepare KlickTipp schema**
   1. Create/identify a custom field to store Salesforce Contact ID (workflow uses `field227883`).
   2. Ensure fields exist for first/last name, birthday, etc. (IDs used: `fieldFirstName`, `fieldLastName`, `fieldBirthday`).

---

### A) KlickTipp → Salesforce branch

4) Add node **KlickTipp Trigger** named **“Contact tagged in KlickTipp”**
   - Configure webhook path (any unique path).
   - Connect KlickTipp credential.

5) Add node **IF** named **“Salesforce Lookup Switch”**
   - Condition: Boolean **true** when Salesforce ID is missing:
     - Left value: `={{ !$json.field227883 }}`
   - Connect: Trigger → IF.

6) Add node **Salesforce** named **“Create Salesforce Contact”** (create)
   - Resource: Contact, Operation: Create
   - Map:
     - LastName, FirstName, Email, address/phone
     - Birthdate from KlickTipp birthday UNIX seconds (Luxon expression from workflow)
     - Custom fields:
       - `KlickTipp_ID__c` = KlickTipp `id`
       - `KlickTipp_marketing_status__c` = KlickTipp `status`
   - Connect: IF (true) → Create.

7) Add node **Salesforce** named **“Update Salesforce Contact”** (update)
   - Resource: Contact, Operation: Update
   - Contact ID: `={{ $('Contact tagged in KlickTipp').item.json.field227883 }}`
   - Update same fields as needed + `KlickTipp_marketing_status__c`.
   - Connect: IF (false) → Update.

8) Add node **KlickTipp** named **“Sync SF ID to KlickTipp”** (subscriber update)
   - Update data field `field227883` to the Salesforce created contact `id`: `={{ $json.id }}`
   - subscriberId: `={{ $('Contact tagged in KlickTipp').item.json.id }}`
   - Connect: Create Salesforce Contact → Sync SF ID to KlickTipp.

9) Add node **KlickTipp** named **“Get KlickTipp Tags”** (subscriber get)
   - subscriberId: `={{ $('Contact tagged in KlickTipp').item.json.id }}`
   - Connect: Sync SF ID to KlickTipp → Get KlickTipp Tags
   - Also connect: Update Salesforce Contact → Get KlickTipp Tags

10) Add node **Switch** named **“Segment Matcher (KT to SF)”**
   - allMatchingOutputs: true
   - Rule 1: `tags contains "14061012"` → output “Contact has Tag Customer”
   - Rule 2: `tags contains "14061060"` → output “Contact has Tag ABC”
   - Connect: Get KlickTipp Tags → Segment Matcher (KT to SF)

11) Add node **HTTP Request** named **“Assign SF Topic (Customer)”**
   - Method POST
   - URL: `https://<yourInstance>.my.salesforce.com/services/data/v60.0/sobjects/TopicAssignment`
   - Auth: predefined Salesforce OAuth2 credential
   - JSON body:
     - TopicId = your “Customer” TopicId
     - EntityId = Salesforce Contact Id
   - **Important:** Set EntityId to the correct node reference (e.g. created/updated contact ID). Do **not** leave it referencing a non-existent node.
   - Connect: Switch output Customer → this node.

12) Add node **HTTP Request** named **“Assign SF Topic (ABC)”**
   - Same as above with ABC TopicId
   - Connect: Switch output ABC → this node.

---

### B) Salesforce → KlickTipp branch

13) Add **Salesforce Trigger** named **“New Salesforce Contact”**
   - Trigger on: contactCreated
   - Poll: every minute
   - Connect Salesforce credential.

14) Add **Salesforce Trigger** named **“Updated Salesforce Contact”**
   - Trigger on: contactUpdated
   - Poll: every minute

15) Add **Set** node named **“Data Normalization & Mapping”**
   - Create fields: KlickTipp_ID, Email, First_name, Last_name, Birthday (UNIX), Salesforce_ID, KlickTipp_ID_exists (boolean)
   - Connect: both Salesforce triggers → Data Normalization & Mapping

16) Add **Filter** node named **“Ignore API Auto-Updates”**
   - Condition: LastModifiedById != your integration user Id
   - Prefer expression based on current item: `={{ $json.LastModifiedById }}`
   - Connect: Data Normalization & Mapping → Ignore API Auto-Updates

17) Add **IF** node named **“KlickTipp Lookup Switch”**
   - Condition: `KlickTipp_ID_exists` is false → create, else update
   - Connect: Ignore API Auto-Updates → KlickTipp Lookup Switch

18) Add **KlickTipp** node named **“Create KlickTipp Subscriber”**
   - Operation: subscriber subscribe
   - listId/tagId: your list + initial tag
   - Fields: first/last/birthday + Salesforce ID field (e.g., `field227883`)
   - Connect: IF (create branch) → Create KlickTipp Subscriber

19) Add **Salesforce** node named **“Sync KlickTipp ID to SF”**
   - Operation: update Contact
   - Contact ID: Salesforce_ID from mapping
   - Set `KlickTipp_ID__c` to KlickTipp subscriber id, and marketing status
   - Connect: Create KlickTipp Subscriber → Sync KlickTipp ID to SF

20) Add **KlickTipp** node named **“Update KlickTipp Subscriber”**
   - Operation: subscriber update
   - subscriberId: KlickTipp_ID
   - Update fields from mapping (ensure Salesforce ID field is consistent)
   - Connect: IF (update branch) → Update KlickTipp Subscriber

21) Add **KlickTipp** node named **“Check Subscription Status”**
   - Operation: subscriber get (by KlickTipp_ID)
   - Connect: Update KlickTipp Subscriber → Check Subscription Status

22) Add **Salesforce** node named **“Sync Marketing Status to SF”**
   - Update Contact: set `KlickTipp_marketing_status__c` = KlickTipp `status`
   - Connect: Check Subscription Status → Sync Marketing Status to SF

23) Add **HTTP Request** node named **“Get Salesforce Topics”**
   - GET query endpoint: `https://<yourInstance>.my.salesforce.com/services/data/v60.0/query`
   - SOQL query TopicAssignment by EntityId = Salesforce_ID
   - Connect: Sync KlickTipp ID to SF → Get Salesforce Topics
   - Connect: Sync Marketing Status to SF → Get Salesforce Topics

24) Add **Switch** node named **“Segment Matcher (SF to KT)”**
   - allMatchingOutputs: true
   - Conditions: topic name list contains “Customer” and/or “ABC”
   - Connect: Get Salesforce Topics → Segment Matcher (SF to KT)

25) Add **KlickTipp** node **“Apply Default Tag”**
   - Resource: contact-tagging, tagId = base tag
   - Connect: Sync KlickTipp ID to SF → Apply Default Tag

26) Add **KlickTipp** nodes **“Apply Tag: Customer”** and **“Apply Tag: ABC”**
   - Resource: contact-tagging
   - Connect: Segment Matcher (SF to KT) outputs to each tag node.

---

### C) Daily deletion sync

27) Add **Schedule Trigger** named **“Daily Cleanup Trigger”**
   - Set preferred daily time.

28) Add **HTTP Request** node **“Fetch Deleted SF Contacts”**
   - Use Salesforce OAuth2 predefined credential
   - QueryAll for deleted contacts last day
   - Connect: Daily Cleanup Trigger → Fetch Deleted SF Contacts
   - Recommended improvement: add a node to **split `records` into items** before deletion.

29) Add **KlickTipp** node **“Delete contact”**
   - Operation: subscriber delete by id (`KlickTipp_ID__c`)
   - Connect: Fetch Deleted SF Contacts → Delete contact

---

### D) GDPR deletion

30) Add **KlickTipp Trigger** named **“GDPR Deletion Request Trigger”**
   - Configure per KlickTipp “deletion link” trigger type.

31) Add **Filter** node **“Is SF Link Present?”**
   - Require existence of Salesforce ID field (e.g., `field227883`)
   - Connect: Trigger → Filter

32) Add **Salesforce** node **“GDPR Delete Salesforce Contact”**
   - Operation: delete Contact by ID from KlickTipp payload
   - Connect: Filter → Delete node

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow creates a two-way synchronization between KlickTipp and Salesforce Contacts, including GDPR cleanup and segmentation mapping. | From “How it works” sticky note in the canvas. |
| Setup requirements: Salesforce fields `KlickTipp_ID__c`; KlickTipp field mapped to `field227883`; connect credentials; update Salesforce instance URLs in HTTP nodes; map Topic IDs. | From “Setup steps” sticky note in the canvas. |
| **Known correction:** “Assign SF Topic (Customer/ABC)” references `$('Create a contact')` but the node is named **Create Salesforce Contact**. Update the expression or rename the node to match. | Prevents expression resolution failures. |
| **Known correction:** HTTP URLs include double slashes in `/services/data//vXX.X/...`—normalize to `/services/data/vXX.X/...` and replace domain with your Salesforce instance. | Prevents request failures. |
| **Known limitation:** Daily deletion branch deletes only `records[0]`. Add a split/loop to process all deleted records. | Ensures all deletions are synced. |
| **Field consistency check:** Salesforce ID stored in KlickTipp uses `field227883` in most nodes but `Update KlickTipp Subscriber` writes to `field227685`. Align these field IDs. | Prevents broken linkage and GDPR deletion skips. |