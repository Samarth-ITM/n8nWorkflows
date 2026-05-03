Enrich new HubSpot contacts with Lusha and alert reps in Slack

https://n8nworkflows.xyz/workflows/enrich-new-hubspot-contacts-with-lusha-and-alert-reps-in-slack-13344


# Enrich new HubSpot contacts with Lusha and alert reps in Slack

## 1. Workflow Overview

**Purpose:** Automatically enrich newly created HubSpot contacts with Lusha data (phone, title, seniority, firmographics), write the enriched fields back to HubSpot, and notify sales reps in Slack when the contact appears “high seniority”.

**Primary use cases:**
- RevOps/Sales teams who want real-time CRM enrichment on contact creation
- Improving data completeness (phone/title/company info) while avoiding writing low-quality/empty enrichment back to HubSpot
- Prioritizing outreach via Slack alerts for higher-value contacts (VP/Director/C-level/etc.)

### 1.1 Input Reception (HubSpot)
A HubSpot trigger fires on new contact creation and immediately checks whether the contact has an email (required for Lusha lookup).

### 1.2 Enrichment + Quality Gate (Lusha + validation)
The workflow calls Lusha to enrich by email, then runs a quality check to ensure at least 2 meaningful fields (phone/title/company) were found before updating HubSpot.

### 1.3 Writeback + Sales Alerting (HubSpot update + Slack)
If enrichment passes the quality gate, the contact is updated in HubSpot. A separate check determines if the seniority is “high,” and posts a Slack message for those contacts.

---

## 2. Block-by-Block Analysis

### Block 1 — HubSpot Trigger + Email Presence Check
**Overview:** Listens for new HubSpot contacts and ensures the contact has an email before attempting enrichment (Lusha is queried by email).  
**Nodes involved:**  
- `New Contact Created in HubSpot`  
- `Has Email?`  

#### Node: New Contact Created in HubSpot
- **Type / role:** `n8n-nodes-base.hubspotTrigger` — event trigger (webhook-style) for HubSpot CRM events.
- **Configuration (interpreted):**
  - Event: `contact.creation` (fires when a new contact is created)
  - Uses HubSpot OAuth2 credentials.
- **Key data produced:** HubSpot payload typically includes:
  - `objectId` (used later as `contactId`)
  - `properties` (including `email`, `firstname`, `lastname` if present)
- **Connections:**
  - Output → `Has Email?`
- **Failure/edge cases:**
  - **OAuth/auth issues**: expired or missing HubSpot OAuth2 token.
  - **Permissions**: token must have rights to read contact events and later update contacts.
  - **Event payload variance**: if HubSpot returns a minimal payload, some properties might be missing or nested differently; the workflow assumes `json.properties.email` exists when present.
- **Version notes:** Node `typeVersion: 1`.

#### Node: Has Email?
- **Type / role:** `n8n-nodes-base.if` — gate to prevent Lusha calls without email.
- **Configuration (interpreted):**
  - Condition: `{{$json.properties?.email}}` **is not empty**
  - Uses optional chaining (`properties?.email`) to avoid expression errors if `properties` is undefined.
- **Connections:**
  - **True** → `Enrich Contact with Lusha`
  - **False** → (no connection; workflow ends for that item)
- **Failure/edge cases:**
  - If HubSpot payload structure changes and `properties.email` isn’t populated at creation time, enrichment will be skipped.
- **Version notes:** Node `typeVersion: 2` (IF node v2 condition model).

---

### Block 2 — Lusha Enrichment + Data Quality Check
**Overview:** Enriches the contact via Lusha using email, then normalizes/merges Lusha + HubSpot data into a single payload and decides whether enrichment quality is sufficient.  
**Nodes involved:**  
- `Enrich Contact with Lusha`  
- `Data Quality Check`  
- `Was Enriched?`

#### Node: Enrich Contact with Lusha
- **Type / role:** `@lusha-org/n8n-nodes-lusha.lusha` — Lusha community node to enrich contact data.
- **Configuration (interpreted):**
  - Operation: `enrichSingle`
  - Search by: `email`
  - Email expression:  
    `{{ $('New Contact Created in HubSpot').first().json.properties.email }}`
    - Notably pulls from the trigger node directly (first item), not from the incoming item.
- **Connections:**
  - Input ← `Has Email?` (true branch)
  - Output → `Data Quality Check`
- **Credentials:** Lusha API credential required.
- **Failure/edge cases:**
  - **Rate limits / quota** on Lusha API.
  - **No match / partial match**: Lusha may return empty fields; downstream quality check handles this.
  - **Expression data coupling:** using `$('New Contact Created in HubSpot').first()` can be risky if the workflow ever processes multiple items concurrently; it will always reference the first trigger item rather than the current item. (Today it likely runs 1 event = 1 item, so it works.)
- **Version notes:** Node `typeVersion: 1`.
- **Dependency note:** Requires installing the community node: https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha

#### Node: Data Quality Check
- **Type / role:** `n8n-nodes-base.code` — transforms and validates enrichment output.
- **Configuration (interpreted):**
  - Reads:
    - Lusha result from `$json`
    - HubSpot trigger payload from `$('New Contact Created in HubSpot').first().json`
  - Quality scoring:
    - `hasPhone = !!lusha.primaryPhone`
    - `hasTitle = !!lusha.jobTitle`
    - `hasCompany = !!lusha.companyName`
    - `fieldsFound = count(true)` among these three
    - `wasEnriched = fieldsFound >= 2`
  - Produces a normalized output object with:
    - Identifiers: `contactId`, `email`
    - Person: `firstName`, `lastName`, `phone`, `jobTitle`, `seniority`, `linkedinUrl`
    - Company: `company`, `companyDomain`, `industry`, `companySize`, `location`
    - Metadata: `wasEnriched`, `fieldsFound`, `enrichedAt`
  - Special parsing:
    - `companySize`: `Array.isArray(lusha.companySize) ? lusha.companySize[1] : ''`
- **Connections:**
  - Output → `Was Enriched?`
- **Failure/edge cases:**
  - **Unexpected Lusha schema**: if fields differ (e.g., `primaryPhone` absent or nested), quality gate may undercount.
  - **companySize indexing**: assumes array and uses `[1]`; if Lusha returns `[min,max]` this chooses max; if different shape, may be wrong/empty.
  - **Multi-item caveat**: also references `$('New Contact Created in HubSpot').first()`.
- **Version notes:** Node `typeVersion: 2` (Code node v2 behavior).

#### Node: Was Enriched?
- **Type / role:** `n8n-nodes-base.if` — gates CRM writeback and alerting.
- **Configuration (interpreted):**
  - شرط: `{{$json.wasEnriched}}` is `true`
- **Connections:**
  - **True** → `Update HubSpot Contact` **and** `Check Seniority Level` (fan-out)
  - **False** → (no connection; workflow ends)
- **Failure/edge cases:**
  - If enrichment returns only 0–1 of (phone/title/company), nothing will be updated in HubSpot (by design).
- **Version notes:** Node `typeVersion: 2`.

---

### Block 3 — Update HubSpot + Seniority Filter + Slack Alert
**Overview:** Updates the HubSpot contact with enriched fields. In parallel, it determines whether the contact is high-seniority and sends a Slack alert if so.  
**Nodes involved:**  
- `Update HubSpot Contact`  
- `Check Seniority Level`  
- `Is High Seniority?`  
- `Alert Rep on Slack`

#### Node: Update HubSpot Contact
- **Type / role:** `n8n-nodes-base.hubspot` — updates the HubSpot contact record.
- **Configuration (interpreted):**
  - Resource: `contact`
  - Operation: `update`
  - Contact ID: `{{$json.contactId}}` (from Data Quality Check)
  - Fields mapped back to HubSpot:
    - `phone` ← `{{$json.phone}}`
    - `company` ← `{{$json.company}}`
    - `website` ← `{{$json.companyDomain}}`
    - `industry` ← `{{$json.industry}}`
    - `jobtitle` ← `{{$json.jobTitle}}`
    - `numberofemployees` ← `{{$json.companySize}}`
- **Connections:**
  - Input ← `Was Enriched?` (true branch)
  - Output → none
- **Credentials:** HubSpot OAuth2 credential.
- **Failure/edge cases:**
  - **Property availability**: HubSpot standard properties usually include these, but portals may have different schemas; `numberofemployees` sometimes expects numeric or specific formats.
  - **Data type mismatch**: `companySize` may be a string; HubSpot may reject invalid formats.
  - **Rate limits**: HubSpot API limits could throttle during bursts.
- **Version notes:** Node `typeVersion: 2`.

#### Node: Check Seniority Level
- **Type / role:** `n8n-nodes-base.code` — classifies contacts as “high seniority”.
- **Configuration (interpreted):**
  - Normalizes: `seniority = ($json.seniority || '').toLowerCase()`
  - Checks inclusion of any of:
    - `vp`, `vice president`, `director`, `c-suite`, `cxo`, `founder`, `owner`, `head`
  - Outputs original JSON plus `isHighSeniority: boolean`
- **Connections:**
  - Input ← `Was Enriched?` (true branch)
  - Output → `Is High Seniority?`
- **Failure/edge cases:**
  - Seniority taxonomy mismatch: Lusha might return values like “SVP”, “Managing Director”, “Partner”, etc. (not captured unless you extend the list).
- **Version notes:** Node `typeVersion: 2`.

#### Node: Is High Seniority?
- **Type / role:** `n8n-nodes-base.if` — conditional route for Slack notification.
- **Configuration (interpreted):**
  - Condition: `{{$json.isHighSeniority}}` is `true`
- **Connections:**
  - **True** → `Alert Rep on Slack`
  - **False** → none
- **Failure/edge cases:** None beyond incorrect classification input.
- **Version notes:** Node `typeVersion: 2`.

#### Node: Alert Rep on Slack
- **Type / role:** `n8n-nodes-base.slack` — posts a formatted message to a channel.
- **Configuration (interpreted):**
  - Channel: `#new-contacts`
  - Message text uses Slack markdown and n8n expressions:
    - Includes name, title/seniority, company/industry, email, phone (or N/A), LinkedIn (or N/A), and fieldsFound.
- **Connections:**
  - Input ← `Is High Seniority?` (true branch)
  - Output → none
- **Credentials:** Slack OAuth2 credential.
- **Failure/edge cases:**
  - Channel not found / bot not invited to channel.
  - Missing Slack scopes (e.g., `chat:write`).
  - Message formatting: relies on fields existing; uses fallbacks for `phone` and `linkedinUrl`.
- **Version notes:** Node `typeVersion: 2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Auto-Enrich New HubSpot Contacts | Sticky Note | Documentation / canvas annotation | — | — | ## Auto-Enrich New HubSpot Contacts with Lusha<br><br>**Who it's for:** RevOps & Sales teams using HubSpot<br><br>**What it does:** The moment a new contact is created in HubSpot, Lusha enriches it with verified phone, job title, seniority, and company firmographics. A data quality check ensures only enriched contacts are updated, and reps get a Slack alert for high-value new contacts.<br><br>### How it works<br>1. HubSpot trigger fires when a new contact is created<br>2. Checks if the contact has an email address<br>3. Lusha enriches the contact by email<br>4. A data quality check validates enrichment results<br>5. Enriched data is written back to HubSpot<br>6. High-seniority contacts trigger a Slack alert for reps<br><br>### Setup<br>1. Install the [Lusha community node](https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha)<br>2. Add your Lusha API, HubSpot OAuth2, and Slack credentials<br>3. Customize the seniority filter and Slack channel<br>4. Activate the workflow — every new HubSpot contact will be enriched in real time |
| 🔔 1. HubSpot Trigger | Sticky Note | Documentation / block label | — | — | Fires instantly when a new contact is created in HubSpot. A check ensures the contact has an email before calling Lusha.<br><br>**Nodes:** HubSpot Trigger → Has Email? (IF)<br><br>💡 Replace with a Salesforce trigger if you use Salesforce. |
| 🔍 2. Lusha Enrichment + Quality Check | Sticky Note | Documentation / block label | — | — | Lusha enriches the contact by email, returning verified phone, title, seniority, and company data. A code node checks if Lusha returned meaningful data before updating CRM.<br><br>**Nodes:** Lusha Enrich → Data Quality Check → Was Enriched? (IF)<br><br>📖 [Lusha API docs](https://www.lusha.com/docs/) |
| 💾 3. Update CRM + Notify | Sticky Note | Documentation / block label | — | — | Enriched data is written back to the HubSpot contact record. Contacts with VP/Director/C-Suite seniority also trigger a Slack alert.<br><br>**Nodes:** Update HubSpot → Is High Seniority? → Slack Alert<br><br>💡 Customize which seniority levels trigger alerts. |
| New Contact Created in HubSpot | HubSpot Trigger | Entry point: detect new HubSpot contact creation | — | Has Email? | Fires instantly when a new contact is created in HubSpot. A check ensures the contact has an email before calling Lusha.<br><br>**Nodes:** HubSpot Trigger → Has Email? (IF)<br><br>💡 Replace with a Salesforce trigger if you use Salesforce. |
| Has Email? | IF | Gate: only proceed if email exists | New Contact Created in HubSpot | Enrich Contact with Lusha (true) | Fires instantly when a new contact is created in HubSpot. A check ensures the contact has an email before calling Lusha.<br><br>**Nodes:** HubSpot Trigger → Has Email? (IF)<br><br>💡 Replace with a Salesforce trigger if you use Salesforce. |
| Enrich Contact with Lusha | Lusha (community) | Enrich contact data using email | Has Email? (true) | Data Quality Check | Lusha enriches the contact by email, returning verified phone, title, seniority, and company data. A code node checks if Lusha returned meaningful data before updating CRM.<br><br>**Nodes:** Lusha Enrich → Data Quality Check → Was Enriched? (IF)<br><br>📖 [Lusha API docs](https://www.lusha.com/docs/) |
| Data Quality Check | Code | Normalize + quality-score enrichment | Enrich Contact with Lusha | Was Enriched? | Lusha enriches the contact by email, returning verified phone, title, seniority, and company data. A code node checks if Lusha returned meaningful data before updating CRM.<br><br>**Nodes:** Lusha Enrich → Data Quality Check → Was Enriched? (IF)<br><br>📖 [Lusha API docs](https://www.lusha.com/docs/) |
| Was Enriched? | IF | Gate: only update CRM / notify if enrichment quality passes | Data Quality Check | Update HubSpot Contact (true), Check Seniority Level (true) | Lusha enriches the contact by email, returning verified phone, title, seniority, and company data. A code node checks if Lusha returned meaningful data before updating CRM.<br><br>**Nodes:** Lusha Enrich → Data Quality Check → Was Enriched? (IF)<br><br>📖 [Lusha API docs](https://www.lusha.com/docs/) |
| Update HubSpot Contact | HubSpot | Write enriched fields back to HubSpot | Was Enriched? (true) | — | Enriched data is written back to the HubSpot contact record. Contacts with VP/Director/C-Suite seniority also trigger a Slack alert.<br><br>**Nodes:** Update HubSpot → Is High Seniority? → Slack Alert<br><br>💡 Customize which seniority levels trigger alerts. |
| Check Seniority Level | Code | Compute isHighSeniority flag from seniority text | Was Enriched? (true) | Is High Seniority? | Enriched data is written back to the HubSpot contact record. Contacts with VP/Director/C-Suite seniority also trigger a Slack alert.<br><br>**Nodes:** Update HubSpot → Is High Seniority? → Slack Alert<br><br>💡 Customize which seniority levels trigger alerts. |
| Is High Seniority? | IF | Gate: only Slack-alert high seniority contacts | Check Seniority Level | Alert Rep on Slack (true) | Enriched data is written back to the HubSpot contact record. Contacts with VP/Director/C-Suite seniority also trigger a Slack alert.<br><br>**Nodes:** Update HubSpot → Is High Seniority? → Slack Alert<br><br>💡 Customize which seniority levels trigger alerts. |
| Alert Rep on Slack | Slack | Notify reps/channel about high-value contact | Is High Seniority? (true) | — | Enriched data is written back to the HubSpot contact record. Contacts with VP/Director/C-Suite seniority also trigger a Slack alert.<br><br>**Nodes:** Update HubSpot → Is High Seniority? → Slack Alert<br><br>💡 Customize which seniority levels trigger alerts. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: `Instant CRM Enrichment with Lusha` (or your preferred name)

2. **Install the Lusha community node**
   - In your n8n instance, install: `@lusha-org/n8n-nodes-lusha`  
     Link: https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha
   - Restart n8n if required for community nodes to load.

3. **Add node: HubSpot Trigger**
   - Node type: **HubSpot Trigger**
   - Event: **Contact → creation** (event key `contact.creation`)
   - Credentials: **HubSpot OAuth2**
     - Ensure scopes/permissions allow reading contact events and later updating contacts.
   - This is your workflow entry node.

4. **Add node: IF (Has Email?)**
   - Node type: **IF**
   - Condition (String):
     - Value 1 (expression): `{{$json.properties?.email}}`
     - Operation: **not empty**
   - Connect: `HubSpot Trigger` → `Has Email?`
   - Use only the **true** output for enrichment; leave false unconnected (or connect to logging if desired).

5. **Add node: Lusha (Enrich Contact with Lusha)**
   - Node type: **Lusha** (community node)
   - Operation: **Enrich Single**
   - Search by: **email**
   - Email (expression): `{{ $('New Contact Created in HubSpot').first().json.properties.email }}`
     - (Optionally improve robustness by using the incoming item’s email instead.)
   - Credentials: **Lusha API** (API key/token as required by the node)
   - Connect: `Has Email?` (true) → `Enrich Contact with Lusha`

6. **Add node: Code (Data Quality Check)**
   - Node type: **Code**
   - Paste logic to:
     - Read Lusha data from `$json`
     - Read HubSpot trigger payload from `$('New Contact Created in HubSpot').first().json`
     - Compute `fieldsFound` across `primaryPhone`, `jobTitle`, `companyName`
     - Set `wasEnriched = fieldsFound >= 2`
     - Output normalized fields including `contactId`, `email`, `phone`, `jobTitle`, `seniority`, `companyDomain`, etc.
   - Connect: `Enrich Contact with Lusha` → `Data Quality Check`

7. **Add node: IF (Was Enriched?)**
   - Node type: **IF**
   - Condition (Boolean):
     - Value 1 (expression): `{{$json.wasEnriched}}`
     - Operation: **is true**
   - Connect: `Data Quality Check` → `Was Enriched?`

8. **Add node: HubSpot (Update HubSpot Contact)**
   - Node type: **HubSpot**
   - Resource: **Contact**
   - Operation: **Update**
   - Contact ID (expression): `{{$json.contactId}}`
   - Update fields mapping (expressions):
     - `phone` → `{{$json.phone}}`
     - `company` → `{{$json.company}}`
     - `website` → `{{$json.companyDomain}}`
     - `industry` → `{{$json.industry}}`
     - `jobtitle` → `{{$json.jobTitle}}`
     - `numberofemployees` → `{{$json.companySize}}`
   - Credentials: **HubSpot OAuth2**
   - Connect: `Was Enriched?` (true) → `Update HubSpot Contact`

9. **Add node: Code (Check Seniority Level)**
   - Node type: **Code**
   - Implement:
     - `seniority = ($json.seniority || '').toLowerCase()`
     - `isHighSeniority = [...]some(s => seniority.includes(s))`
     - Return `{ ...$json, isHighSeniority }`
   - Connect: `Was Enriched?` (true) → `Check Seniority Level`
   - (This is parallel to the HubSpot update.)

10. **Add node: IF (Is High Seniority?)**
   - Node type: **IF**
   - Condition (Boolean):
     - Value 1 (expression): `{{$json.isHighSeniority}}`
     - Operation: **is true**
   - Connect: `Check Seniority Level` → `Is High Seniority?`

11. **Add node: Slack (Alert Rep on Slack)**
   - Node type: **Slack**
   - Operation: **Post message** (send text to channel)
   - Channel: `#new-contacts` (customize)
   - Text: include expressions for name/title/seniority/company/email/phone/linkedin + enrichment metadata.
   - Credentials: **Slack OAuth2**
     - Ensure the app/bot has `chat:write` and is allowed in the channel.
   - Connect: `Is High Seniority?` (true) → `Alert Rep on Slack`

12. **(Optional) Add sticky notes**
   - Add canvas notes to describe each block and include links:
     - Lusha node install link
     - Lusha API docs: https://www.lusha.com/docs/

13. **Activate workflow**
   - Create a test contact in HubSpot with an email and verify:
     - Lusha is called
     - Data Quality gating behaves as expected
     - HubSpot fields update
     - Slack message posts only for high-seniority matches

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Lusha community node: `@lusha-org/n8n-nodes-lusha` | https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha |
| Lusha API documentation | https://www.lusha.com/docs/ |
| Workflow behavior summary and setup checklist (as written in the canvas note) | “Auto-Enrich New HubSpot Contacts with Lusha” sticky note content (included above) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.