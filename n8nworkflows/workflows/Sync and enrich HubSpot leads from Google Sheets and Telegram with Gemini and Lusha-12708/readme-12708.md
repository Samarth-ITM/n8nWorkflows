Sync and enrich HubSpot leads from Google Sheets and Telegram with Gemini and Lusha

https://n8nworkflows.xyz/workflows/sync-and-enrich-hubspot-leads-from-google-sheets-and-telegram-with-gemini-and-lusha-12708


# Sync and enrich HubSpot leads from Google Sheets and Telegram with Gemini and Lusha

## 1. Workflow Overview

**Workflow name:** Sync and Enrich HubSpot Leads via AI and Lusha  
**Stated title:** Sync and enrich HubSpot leads from Google Sheets and Telegram with Gemini and Lusha

**Purpose:**  
This workflow ingests leads from **Google Sheets** (polling trigger) and **Telegram** (webhook trigger), normalizes them into a shared structure (`updates_payload`), resolves/infers company domains with **Gemini**, prevents duplicate companies in **HubSpot** using a **custom fuzzy matching engine**, enriches contact + firmographics with **Lusha**, updates/creates HubSpot **company + contact**, and finally writes back status/notifications (Google Sheet update and/or Telegram confirmation) and logs a HubSpot note.

**Typical use cases**
- SDR/BD teams capturing leads from ad-hoc Telegram messages and structured Sheets.
- Reducing CRM duplication by domain/name matching before writing.
- Auto-enriching missing emails + company firmographics while controlling enrichment usage.

### 1.1 Logical Blocks (functional grouping)
1. **Data Intake & Parsing**
   - Two entry points: Google Sheets row updates and Telegram messages.
   - Telegram free text is parsed into a structured JSON lead payload via Gemini.
2. **Contextual Lookup**
   - Fetch existing HubSpot companies (reference data).
   - Resolve missing domains with Gemini (“Domain Resolver”).
3. **Normalization & Matching Engine**
   - Merge lead + reference data.
   - Custom code matches lead to existing company (domain hard match; name fuzzy/substring).
4. **CRM Routing & Core Writes**
   - Switch on similarity score to update existing HubSpot company or create a new one.
5. **Enrichment & Relationship Writes**
   - If lead email missing → Lusha person enrichment to find email.
   - Resolve company ID reliably; optional retry.
   - Add HubSpot note, create/associate HubSpot contact, enrich company firmographics via Lusha and patch only changed properties.
6. **Feedback Loop**
   - If Telegram-triggered → send Telegram success message.
   - If Sheets-triggered → update row status (“Processed” / acknowledgement).

---

## 2. Block-by-Block Analysis

### Block 1 — Data Intake & Parsing
**Overview:** Captures leads from Google Sheets and Telegram. Telegram messages are converted from unstructured text into a consistent JSON payload.

**Nodes involved:**
- **New or Updated row** (Google Sheets Trigger)
- **Filter processed lines** (Filter)
- **Telegram Trigger** (Telegram Trigger)
- **Parse Telegram Message** (Gemini)
- **Format Telegram Lead Data** (Code)

#### Node: New or Updated row
- **Type / role:** Google Sheets Trigger (`googleSheetsTrigger`) — polls for new/updated rows.
- **Key configuration:**
  - Polling: **every minute**
  - `documentId` and `sheetName`: currently blank placeholders (must be set in your instance).
  - `retryOnFail: true`, `waitBetweenTries: 2000ms`
- **Outputs:** Sends row JSON into **Filter processed lines**.
- **Failure modes / edge cases:**
  - OAuth token expiry / permission errors.
  - Misconfigured sheet ID/name → no events.
  - High-frequency updates can re-trigger if status logic is not stable.

#### Node: Filter processed lines
- **Type / role:** Filter — drops rows already marked as processed.
- **Logic:** Pass only items where `status` is **empty** (`operation: empty`).
- **Outputs:** Three parallel outputs:
  - to **Get HubSpot data**
  - to **Domain Resolver**
  - to **Append Resolved Domain to Lead** (lead stream)
- **Edge cases:**
  - If sheet uses a different column name than `status`, everything will pass (or nothing will).
  - Rows with whitespace status (not empty) may be skipped unintentionally.

#### Node: Telegram Trigger
- **Type / role:** Telegram Trigger — webhook on incoming messages.
- **Configuration:** listens to `updates: ["message"]`
- **Outputs:** to **Parse Telegram Message**
- **Failure modes:**
  - Telegram webhook misconfigured or token revoked.
  - Non-text messages (photos, etc.) will not have `message.text` and can break downstream prompt.

#### Node: Parse Telegram Message
- **Type / role:** Google Gemini (LangChain node) — parses message text into structured JSON.
- **Model:** `models/gemini-2.5-flash`
- **Prompt behavior:** Instructs Gemini to output **ONLY** a JSON object with fields:
  - `company`, `company_domain`, `first_name`, `last_name`, `email`, `city`, `note`, `source_type = "Telegram Bot"`
  - Also requests domain inference if not explicit.
- **`jsonOutput: true`**
- **Outputs:** to **Format Telegram Lead Data**
- **Failure modes / edge cases:**
  - Model may still wrap JSON in markdown fences; downstream node attempts to strip them.
  - Hallucinated domains are possible; matching logic later may still mitigate duplicates.
  - Missing `message.text` → prompt contains `undefined`.

#### Node: Format Telegram Lead Data
- **Type / role:** Code — cleans Gemini output and converts to lead object.
- **Key logic:**
  - Reads `items[0].json.content.parts[0].text`
  - Removes ```json fences, parses JSON
  - Adds:
    - `is_live_trigger = true`
    - `chat_id = Telegram chat id`
- **Outputs:** two parallel connections:
  - to **Get HubSpot data**
  - to **Combine Lead with CRM Reference Data**
- **Failure modes:**
  - If Gemini output structure differs (no `content.parts[0].text`), parsing fails.
  - If JSON invalid → throws error: “Failed to parse JSON…”.

---

### Block 2 — Contextual Lookup (HubSpot reference + domain resolution)
**Overview:** Retrieves HubSpot company list as matching reference data, and resolves the most likely official domain using Gemini when missing/uncertain.

**Nodes involved:**
- **Get HubSpot data** (HTTP Request)
- **Split HubSpot Search Results** (Split Out)
- **Mark as HubSpot Reference Data** (Code)
- **Domain Resolver** (Gemini)
- **Append Resolved Domain to Lead** (Merge)

#### Node: Get HubSpot data
- **Type / role:** HTTP Request — pulls companies from HubSpot CRM v3.
- **Request:**
  - `GET https://api.hubapi.com/crm/v3/objects/companies`
  - Query:
    - `limit=100`
    - `properties=name,city,domain`
- **Auth:** predefined credential type `hubspotAppToken` (HubSpot Private App token).
- **Outputs:** to **Split HubSpot Search Results**
- **Failure modes / edge cases:**
  - Limit 100 may be insufficient for large CRMs → matching misses older companies.
  - Rate limits (HubSpot) and pagination not handled.
  - Token scope missing `crm.objects.companies.read`.

#### Node: Split HubSpot Search Results
- **Type / role:** Split Out — splits `results[]` into one item per company.
- **Field:** `results`
- **Outputs:** to **Mark as HubSpot Reference Data**
- **Failure modes:** If HubSpot response shape changes or errors return without `results`.

#### Node: Mark as HubSpot Reference Data
- **Type / role:** Code — tags items so matching engine can separate reference data.
- **Logic:** adds `item.json.is_reference_data = true`
- **Outputs:** to **Combine Lead with CRM Reference Data** (reference stream)

#### Node: Domain Resolver
- **Type / role:** Gemini — resolves official corporate domain.
- **Model:** `models/gemini-2.0-flash`
- **Error handling:** `onError: continueRegularOutput` and `alwaysOutputData: true`
- **Prompt rules:**
  - If a valid domain exists in “Existing Domain” or from email → return it.
  - Else infer official corporate domain.
  - Output must be ONLY domain string or `null`.
- **`jsonOutput: true`** (note: prompt asks plain string; this mismatch can cause structure variability)
- **Outputs:** to **Append Resolved Domain to Lead** (index 1)
- **Failure modes / edge cases:**
  - Output type ambiguity: node may return string-like text inside structured fields.
  - AI may output quotes or formatting; matching engine later strips quotes/brackets/newlines.

#### Node: Append Resolved Domain to Lead
- **Type / role:** Merge — combines lead item with the domain resolver output.
- **Mode:** `combineByPosition` (expects both inputs to align item-by-item)
- **Inputs:**
  - Input 0: from **Filter processed lines** (lead row)
  - Input 1: from **Domain Resolver** (AI domain)
- **Outputs:** to **Combine Lead with CRM Reference Data**
- **Edge cases:**
  - If HubSpot fetch/Domain Resolver produce different item counts than lead stream (or parallel timing produces mismatch), combine-by-position can mis-pair results.

---

### Block 3 — Normalization & Matching Engine
**Overview:** Merges the incoming lead stream with HubSpot reference data, then runs a custom match algorithm to compute `similarity_score` and determine whether an existing HubSpot company should be updated or a new one created.

**Nodes involved:**
- **Combine Lead with CRM Reference Data** (Merge)
- **Match Lead to Existing Company** (Code)
- **Switch Logic** (Switch)

#### Node: Combine Lead with CRM Reference Data
- **Type / role:** Merge — appends reference data stream to lead stream.
- **Mode:** `append` (node name indicates append; parameter object empty)
- **Inputs:**
  - Lead stream: from **Append Resolved Domain to Lead** (Sheets) or **Format Telegram Lead Data** (Telegram)
  - Reference stream: from **Mark as HubSpot Reference Data**
- **Outputs:** to **Match Lead to Existing Company**
- **Edge cases:**
  - Append yields a mixed list; downstream code must correctly separate by flags/fields (it does).

#### Node: Match Lead to Existing Company
- **Type / role:** Code — “secret sauce” matching and normalization into `updates_payload`.
- **Key behaviors:**
  - Splits `$input.all()` into:
    - `hubspotCompanies`: items with `hs_object_id`/`properties.hs_object_id` OR `is_reference_data`
    - `leads`: items with `company` and not HubSpot ID fields
  - Domain resolution priority for the lead:
    1. `cleanedGemini` extracted from `lead.json.text` or `lead.json.content.parts[0].text` (strips quotes/brackets/newlines; converts `"null"` to null)
    2. `lead.json.company_domain`
    3. domain part of `lead.json.email`
    4. `lead.json.domain`
  - Matching engine:
    - **Hard match** on domain equality → score 1.0
    - else **name match**:
      - exact match → 1.0
      - substring containment → 0.95
  - Outputs unified object:
    - `updates_payload: { ...lead.json, domain: leadDomain || null }`
    - `similarity_score: 0..100`
    - `existing_company_id` from best matching HubSpot record
- **Outputs:** to **Switch Logic**
- **Failure modes / edge cases:**
  - If HubSpot properties structure differs (e.g., domain under `properties.domain`), code partially accounts for it.
  - False positives possible with substring matches (e.g., “Meta” vs “MetaX”).
  - If Domain Resolver returns unexpected structure, extraction may fail and domain remains null.

#### Node: Switch Logic
- **Type / role:** Switch — routes to update vs create based on match confidence.
- **Rule:** if `similarity_score >= 80` → output 0 (matched)
- **Fallback output:** `extra` → treated as “create new”
- **Outputs:**
  - Matched (index 0) → **HubSpot: Update Company**
  - Fallback → **HubSpot: Create Company**
- **Edge cases:**
  - Threshold tuning is critical; too low causes overwrites, too high causes duplicates.

---

### Block 4 — CRM Routing & Core Writes (HubSpot company upsert)
**Overview:** Updates an existing company or creates a new one in HubSpot, using normalized data from `updates_payload`.

**Nodes involved:**
- **HubSpot: Update Company** (HubSpot)
- **HubSpot: Create Company** (HubSpot)

#### Node: HubSpot: Update Company
- **Type / role:** HubSpot node — updates an existing company.
- **Operation:** `company.update`
- **Target ID:** `existing_company_id`
- **Fields updated:**
  - `city = updates_payload.city`
  - `companyDomainName` extracted via regex from `updates_payload.domain`:
    - `domain.match(/[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/)?.[0] || ""`
- **Auth:** HubSpot App Token
- **Outputs:** to **Switch on email field**
- **Failure modes:**
  - Missing/invalid `existing_company_id` → HubSpot error.
  - Property names depend on HubSpot portal configuration; `companyDomainName` must be valid.

#### Node: HubSpot: Create Company
- **Type / role:** HubSpot node — creates new company.
- **Operation:** `company.create`
- **Name:** `updates_payload.company`
- **Additional fields:**
  - `city`
  - `lifecycleStatus = lead`
  - `companyDomainName` from domain regex
  - `description` composed with lead source metadata + timestamp + original contact fields
- **Outputs:** to **Switch on email field**
- **Failure modes:**
  - Duplicate domain constraint issues in HubSpot (if enforced by your process).
  - Missing company name can error or create low-quality record.

---

### Block 5 — Contact & Firmographic Enrichment (Lusha + HubSpot relationships)
**Overview:** If email is missing, fetch it from Lusha. Resolve the final HubSpot company ID robustly, then add a note, create/associate contact if email exists, and enrich company firmographics with delta updates only.

**Nodes involved:**
- **Switch on email field** (Switch)
- **Lusha Enrichment - email** (HTTP Request)
- **Map Contact Enrichment Data** (Code)
- **Resolve Company ID** (Code)
- **Validate Company ID Status** (Switch)
- **Retry ID resolution** (Wait)
- **Add a Note to Hubspot company** (HTTP Request)
- **Check for Existing Contact Email** (Switch)
- **Create or update a contact** (HubSpot)
- **Lusha Company Enrichment** (HTTP Request)
- **Prepare Firmographic Updates** (Code)
- **Filter Leads with Data Changes** (Filter)
- **HubSpot: Update Enriched Company** (HTTP Request)

#### Node: Switch on email field
- **Type / role:** Switch — decides whether to call Lusha for email enrichment.
- **Rule:** if `!updates_payload.email` is true → output 0 → **Lusha Enrichment - email**
- **Fallback:** `extra` → **Resolve Company ID**
- **Inputs:** from both **HubSpot: Update Company** and **HubSpot: Create Company**
- **Edge cases:**
  - If email is present but invalid (e.g., “-”), it will skip Lusha.

#### Node: Lusha Enrichment - email
- **Type / role:** HTTP Request — Lusha person endpoint.
- **Request:** `GET https://api.lusha.com/v2/person` with query:
  - `firstName`, `lastName`, `companyName`, `companyDomain` from `Match Lead to Existing Company`.  
- **Auth:** Header auth “Lusha API Key”
- **Response handling:** `neverError: true` (won’t throw on non-2xx); JSON response expected.
- **Outputs:** to **Map Contact Enrichment Data**
- **Failure modes / edge cases:**
  - If rate-limited or no credits, response may indicate error but node won’t fail → downstream must handle missing `contact.data...`.
  - Names not trimmed or missing → weaker match; code trims first/last name.

#### Node: Map Contact Enrichment Data
- **Type / role:** Code — injects Lusha-discovered email into the unified lead object.
- **Logic:**
  - Takes base object from `Match Lead to Existing Company`
  - Reads `lushaResponse.contact.data.emailAddresses[0].email`
  - If present → sets `item.updates_payload.email`
- **Outputs:** to **Resolve Company ID**
- **Edge cases:**
  - Lusha response shape changes → email extraction returns undefined.

#### Node: Resolve Company ID
- **Type / role:** Code — ensures the workflow has a usable HubSpot company ID for downstream steps.
- **Logic summary:**
  - `finalId = $json.id || $json.companyId`
  - If missing, tries to read:
    - `$('HubSpot: Create Company').item.json.companyId || id`
    - `$('HubSpot: Update Company').item.json.companyId || id`
  - If still missing → `retry_needed = true`
  - Always rebuilds output as:
    - `{ id: finalId, retry_needed, updates_payload: {...leadData, email: enrichedEmail || leadData.email} }`
- **Outputs:** to **Validate Company ID Status**
- **Failure modes / edge cases:**
  - If run in bulk/multiple items, cross-item `.item` access can reference wrong item if not aligned.
  - HubSpot nodes may return `id` vs `companyId` depending on node version/operation.

#### Node: Validate Company ID Status
- **Type / role:** Switch — if ID missing, wait and retry resolution.
- **Rule:** if `retry_needed == true` → output 0 → **Retry ID resolution**
- **Fallback:** proceeds to:
  - **Add a Note to Hubspot company**
  - **Check for Existing Contact Email**
  - **Lusha Company Enrichment**
- **Edge cases:**
  - If ID never resolves, the workflow can loop (wait → resolve → still missing → wait…).

#### Node: Retry ID resolution
- **Type / role:** Wait — delays before attempting ID resolution again.
- **Configuration:** `amount: 2` (unit depends on node defaults; typically seconds in n8n Wait)
- **Outputs:** back to **Resolve Company ID**
- **Failure modes:** Wait node requires n8n queue/execution persistence properly configured for long waits (though 2 units is short).

#### Node: Add a Note to Hubspot company
- **Type / role:** HTTP Request — creates a HubSpot Note and associates it to the company.
- **Request:** `POST https://api.hubapi.com/crm/v3/objects/notes`
- **Body (raw JSON):**
  - `hs_note_body = updates_payload.note`
  - `hs_timestamp = now ISO`
  - Association to company ID using `associationTypeId: 190` (HubSpot-defined note-to-company)
- **Auth:** HTTP Header Auth “HubSpot Bearer Token”
- **Outputs:** (no further connections shown)
- **Failure modes / edge cases:**
  - Association type ID can vary across object types; `190` is commonly company-note but should be verified.
  - Missing `updates_payload.note` may create empty notes (or fail if required in your portal rules).

#### Node: Check for Existing Contact Email
- **Type / role:** Switch — only create/associate contact if email exists.
- **Rule (named output “Has email”):** `updates_payload.email` is not empty → output 0 → **Create or update a contact**
- **Fallback:** directly goes to the feedback routing nodes (**Identify Telegram triggered flow** and **Identify updated row**)
- **Edge cases:** Email present but malformed still passes and may create bad contact records.

#### Node: Create or update a contact
- **Type / role:** HubSpot node — upserts contact by email and associates to the company.
- **Operation:** “Create or update”
- **Key fields:**
  - Email: `updates_payload.email`
  - First/Last name: from payload
  - `associatedCompanyId = {{ $json.id }}`
- **Options:** `resolveData: false` (does not expand referenced data)
- **Outputs:** to both feedback filters:
  - **Identify Telegram triggered flow**
  - **Identify updated row**
- **Failure modes:**
  - HubSpot scope missing for contacts write.
  - If company association fails (invalid ID), contact may still be created but unlinked.

#### Node: Lusha Company Enrichment
- **Type / role:** HTTP Request — Lusha company endpoint for firmographics.
- **Request:** `GET https://api.lusha.com/v2/company` with query:
  - `domain`, `company`
- **Auth:** Header auth “Lusha API Key”
- **Response handling:** `neverError: true`
- **Outputs:** to **Prepare Firmographic Updates**
- **Failure modes / edge cases:**
  - Lusha may return partial/empty `data`.
  - Credits/rate limits return error payload but won’t fail the node.

#### Node: Prepare Firmographic Updates
- **Type / role:** Code — compute a delta of HubSpot properties to update.
- **Logic:**
  - Takes base lead object from `Match Lead to Existing Company`
  - Uses company ID from `Resolve Company ID` (`companyId = resolveNode.id`)
  - Reads Lusha company `data` and prepares `delta_update` only if changed:
    - `description`
    - `annualrevenue` (uses upper bound of `revenueRange`)
    - `founded_year`
    - `website`
  - Sets:
    - `hubspot_id`
    - `has_changes = Object.keys(delta).length > 0`
- **Outputs:** to **Filter Leads with Data Changes**
- **Edge cases / likely issue:**
  - The node attempts to compare against `resolveNode.properties`, but **Resolve Company ID output does not include `properties`**, so comparisons default to empty and may cause unnecessary updates every time.

#### Node: Filter Leads with Data Changes
- **Type / role:** Filter — only patch HubSpot if there are actual changes.
- **Condition:** `has_changes == true`
- **Outputs:** to **HubSpot: Update Enriched Company**
- **Edge cases:** If `has_changes` incorrectly set true (see previous note), this may call HubSpot unnecessarily.

#### Node: HubSpot: Update Enriched Company
- **Type / role:** HTTP Request — patches company properties in HubSpot.
- **Request:** `PATCH https://api.hubapi.com/crm/v3/objects/companies/{{hubspot_id}}`
- **Body:** `{ "properties": <delta_update> }`
- **Auth:** HTTP Header Auth “HubSpot Bearer Token”
- **Outputs:** none further
- **Failure modes:**
  - Invalid property names (e.g., `founded_year` must exist as a company property).
  - Rate limits or auth issues.

---

### Block 6 — Feedback Loop (Sheets + Telegram)
**Overview:** Routes to either Google Sheets acknowledgement or Telegram confirmation depending on whether the lead originated from Telegram (`is_live_trigger`).

**Nodes involved:**
- **Identify Telegram triggered flow** (Filter)
- **Update Telegram user** (Telegram)
- **Identify updated row** (Filter)
- **Acknowledge in sheet** (Google Sheets)

#### Node: Identify Telegram triggered flow
- **Type / role:** Filter — passes only Telegram-triggered leads.
- **Condition:** `Match Lead to Existing Company -> updates_payload.is_live_trigger` is true
- **Outputs:** to **Update Telegram user**
- **Edge cases:** If `is_live_trigger` missing, the lead will not be acknowledged in Telegram.

#### Node: Update Telegram user
- **Type / role:** Telegram — sends formatted success message.
- **Message content:** includes company, contact name, action taken (based on `Switch Logic` branch index), note.
- **chatId:** `updates_payload.chat_id`
- **Parse mode:** Markdown
- **Failure modes:**
  - Chat ID missing/incorrect.
  - Markdown formatting issues if note contains special characters.

#### Node: Identify updated row
- **Type / role:** Filter — passes only Sheets-triggered leads.
- **Condition:** `is_live_trigger` is false
- **Outputs:** to **Acknowledge in sheet**
- **Edge cases:** Sheets leads must have `is_live_trigger` explicitly false or absent; here it checks boolean false, so absent may not pass depending on n8n boolean coercion rules.

#### Node: Acknowledge in sheet
- **Type / role:** Google Sheets — updates the originating row.
- **Operation:** update
- **documentId / sheetName:** blank placeholders (must be configured).
- **Failure modes:**
  - If row identification fields aren’t mapped (not visible in JSON), update may fail silently or update wrong row.
  - OAuth scope/permission issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New or Updated row | Google Sheets Trigger | Poll Sheet for new/updated rows | — | Filter processed lines | ## 1. Data Intake & Parsing  / These nodes capture new leads from Google Sheets (polling) or Telegram (webhooks). / Gemini AI parses unstructured messages into a standard JSON payload, ensuring data consistency regardless of the source. |
| Filter processed lines | Filter | Skip already-processed sheet rows | New or Updated row | Get HubSpot data; Domain Resolver; Append Resolved Domain to Lead | ## 1. Data Intake & Parsing  / These nodes capture new leads from Google Sheets (polling) or Telegram (webhooks). / Gemini AI parses unstructured messages into a standard JSON payload, ensuring data consistency regardless of the source. |
| Telegram Trigger | Telegram Trigger | Receive Telegram message webhook | — | Parse Telegram Message | ## 1. Data Intake & Parsing  / These nodes capture new leads from Google Sheets (polling) or Telegram (webhooks). / Gemini AI parses unstructured messages into a standard JSON payload, ensuring data consistency regardless of the source. |
| Parse Telegram Message | Google Gemini (LangChain) | Extract structured lead JSON from Telegram text | Telegram Trigger | Format Telegram Lead Data | ## 1. Data Intake & Parsing  / These nodes capture new leads from Google Sheets (polling) or Telegram (webhooks). / Gemini AI parses unstructured messages into a standard JSON payload, ensuring data consistency regardless of the source. |
| Format Telegram Lead Data | Code | Parse Gemini output JSON, add chat_id and is_live_trigger | Parse Telegram Message | Get HubSpot data; Combine Lead with CRM Reference Data | ## 1. Data Intake & Parsing  / These nodes capture new leads from Google Sheets (polling) or Telegram (webhooks). / Gemini AI parses unstructured messages into a standard JSON payload, ensuring data consistency regardless of the source. |
| Get HubSpot data | HTTP Request | Fetch existing companies for reference matching | Filter processed lines; Format Telegram Lead Data | Split HubSpot Search Results | ## 2. Contextual Lookup / Uses Gemini AI to resolve missing corporate domains and fetches existing company data from HubSpot. / This preparation is key for accurate matching and prevents CRM fragmentation. |
| Split HubSpot Search Results | Split Out | Split HubSpot company search results into items | Get HubSpot data | Mark as HubSpot Reference Data | ## 2. Contextual Lookup / Uses Gemini AI to resolve missing corporate domains and fetches existing company data from HubSpot. / This preparation is key for accurate matching and prevents CRM fragmentation. |
| Mark as HubSpot Reference Data | Code | Tag HubSpot items as reference data | Split HubSpot Search Results | Combine Lead with CRM Reference Data | ## 2. Contextual Lookup / Uses Gemini AI to resolve missing corporate domains and fetches existing company data from HubSpot. / This preparation is key for accurate matching and prevents CRM fragmentation. |
| Domain Resolver | Google Gemini (LangChain) | Resolve official corporate domain string | Filter processed lines | Append Resolved Domain to Lead | ## 2. Contextual Lookup / Uses Gemini AI to resolve missing corporate domains and fetches existing company data from HubSpot. / This preparation is key for accurate matching and prevents CRM fragmentation. |
| Append Resolved Domain to Lead | Merge | Combine sheet lead with AI-resolved domain (by position) | Filter processed lines; Domain Resolver | Combine Lead with CRM Reference Data | ## 2. Contextual Lookup / Uses Gemini AI to resolve missing corporate domains and fetches existing company data from HubSpot. / This preparation is key for accurate matching and prevents CRM fragmentation. |
| Combine Lead with CRM Reference Data | Merge | Append reference data + lead data into one stream | Append Resolved Domain to Lead / Format Telegram Lead Data; Mark as HubSpot Reference Data | Match Lead to Existing Company | ## 3. The Matching Engine / Custom JS logic performs fuzzy matching against HubSpot data. / It calculates a similarity score to determine if a lead belongs to an existing record, optimizing enrichment credit usage, and prevents HubSpot record duplication. |
| Match Lead to Existing Company | Code | Normalize to updates_payload and compute similarity + existing_company_id | Combine Lead with CRM Reference Data | Switch Logic | ## 3. The Matching Engine / Custom JS logic performs fuzzy matching against HubSpot data. / It calculates a similarity score to determine if a lead belongs to an existing record, optimizing enrichment credit usage, and prevents HubSpot record duplication. |
| Switch Logic | Switch | Route to update vs create based on similarity_score threshold | Match Lead to Existing Company | HubSpot: Update Company; HubSpot: Create Company | ## 4. CRM Routing & Core Writes / This stage directs the lead based on its score: high scores trigger the HubSpot: Update Company node to refresh existing records, while low scores route to HubSpot: Create Company to generate a new entity with proper source metadata. |
| HubSpot: Update Company | HubSpot | Update matched company core fields | Switch Logic | Switch on email field | ## 4. CRM Routing & Core Writes / This stage directs the lead based on its score: high scores trigger the HubSpot: Update Company node to refresh existing records, while low scores route to HubSpot: Create Company to generate a new entity with proper source metadata. |
| HubSpot: Create Company | HubSpot | Create new company with lead-source metadata | Switch Logic | Switch on email field | ## 4. CRM Routing & Core Writes / This stage directs the lead based on its score: high scores trigger the HubSpot: Update Company node to refresh existing records, while low scores route to HubSpot: Create Company to generate a new entity with proper source metadata. |
| Switch on email field | Switch | Decide whether to enrich email via Lusha | HubSpot: Update Company; HubSpot: Create Company | Lusha Enrichment - email; Resolve Company ID | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Lusha Enrichment - email | HTTP Request | Find contact email via Lusha person API | Switch on email field | Map Contact Enrichment Data | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Map Contact Enrichment Data | Code | Copy Lusha email into updates_payload | Lusha Enrichment - email | Resolve Company ID | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Resolve Company ID | Code | Ensure final HubSpot company ID exists; set retry flag | Switch on email field / Map Contact Enrichment Data / Retry ID resolution | Validate Company ID Status | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Validate Company ID Status | Switch | If missing ID, wait and retry; else proceed | Resolve Company ID | Retry ID resolution; Add a Note to Hubspot company; Check for Existing Contact Email; Lusha Company Enrichment | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Retry ID resolution | Wait | Delay and retry resolving HubSpot company ID | Validate Company ID Status | Resolve Company ID | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Add a Note to Hubspot company | HTTP Request | Create HubSpot note associated to company | Validate Company ID Status | — | ## 6. Feedback Loop / This final stage of the workflow closes the execution by / - Logging the interaction as a HubSpot Note / - Updating the original Google Sheet status to 'Processed' / - Notify the Telegram user with a formatted success summary. |
| Check for Existing Contact Email | Switch | If email exists, upsert contact; else skip | Validate Company ID Status | Create or update a contact; Identify Telegram triggered flow; Identify updated row | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Create or update a contact | HubSpot | Upsert contact by email and associate to company | Check for Existing Contact Email | Identify Telegram triggered flow; Identify updated row | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Lusha Company Enrichment | HTTP Request | Fetch company firmographics from Lusha | Validate Company ID Status | Prepare Firmographic Updates | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Prepare Firmographic Updates | Code | Compute delta_update for HubSpot company properties | Lusha Company Enrichment | Filter Leads with Data Changes | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Filter Leads with Data Changes | Filter | Only update HubSpot if firmographics changed | Prepare Firmographic Updates | HubSpot: Update Enriched Company | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| HubSpot: Update Enriched Company | HTTP Request | PATCH HubSpot company with delta properties | Filter Leads with Data Changes | — | ## 5. Contact & Firmographic Enrichment / This stage uses Lusha API to find missing emails and firmographic data (revenue, year founded). / It automatically updates the HubSpot company and associate/creates the contact person, ensuring your CRM has 360-degree data coverage. |
| Identify Telegram triggered flow | Filter | Route Telegram-origin leads to Telegram notification | Create or update a contact; Check for Existing Contact Email | Update Telegram user | ## 6. Feedback Loop / This final stage of the workflow closes the execution by / - Logging the interaction as a HubSpot Note / - Updating the original Google Sheet status to 'Processed' / - Notify the Telegram user with a formatted success summary. |
| Update Telegram user | Telegram | Send success confirmation to Telegram chat | Identify Telegram triggered flow | — | ## 6. Feedback Loop / This final stage of the workflow closes the execution by / - Logging the interaction as a HubSpot Note / - Updating the original Google Sheet status to 'Processed' / - Notify the Telegram user with a formatted success summary. |
| Identify updated row | Filter | Route Sheets-origin leads to sheet acknowledgement | Create or update a contact; Check for Existing Contact Email | Acknowledge in sheet | ## 6. Feedback Loop / This final stage of the workflow closes the execution by / - Logging the interaction as a HubSpot Note / - Updating the original Google Sheet status to 'Processed' / - Notify the Telegram user with a formatted success summary. |
| Acknowledge in sheet | Google Sheets | Update the processed row status in the sheet | Identify updated row | — | ## 6. Feedback Loop / This final stage of the workflow closes the execution by / - Logging the interaction as a HubSpot Note / - Updating the original Google Sheet status to 'Processed' / - Notify the Telegram user with a formatted success summary. |
| Sticky Note3 | Sticky Note | Global architecture + setup guide | — | — | # HubSpot Smart Lead Enrichment and Matching via Google Sheets, Telegram, Lusha, and Gemini AI / Core Architecture… / Template Setup Guide… |
| Sticky Note4 | Sticky Note | Block comment (Intake) | — | — | ## 1. Data Intake & Parsing / … |
| Sticky Note5 | Sticky Note | Block comment (Lookup) | — | — | ## 2. Contextual Lookup / … |
| Sticky Note6 | Sticky Note | Block comment (Matching) | — | — | ## 3. The Matching Engine / … |
| Sticky Note | Sticky Note | Block comment (Routing) | — | — | ## 4. CRM Routing & Core Writes / … |
| Sticky Note7 | Sticky Note | Block comment (Enrichment) | — | — | ## 5. Contact & Firmographic Enrichment / … |
| Sticky Note8 | Sticky Note | Block comment (Feedback) | — | — | ## 6. Feedback Loop / … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: **“Sync and Enrich HubSpot Leads via AI and Lusha”** (or your preferred name).
   - Set workflow settings as needed (optional): an error workflow is referenced in the JSON, but not required to rebuild core logic.

2. **Create credentials**
   1. **HubSpot (Private App token)**
      - Credential type: *HubSpot App Token* (for HubSpot nodes + one HTTP request node that uses predefined type).
      - Ensure scopes include: companies read/write, contacts write, notes write (as needed).
   2. **HubSpot Bearer Token (HTTP Header Auth)**
      - Credential type: *HTTP Header Auth*
      - Header: `Authorization: Bearer <token>`
      - Used by:
        - “Add a Note to Hubspot company”
        - “HubSpot: Update Enriched Company”
   3. **Google Sheets Trigger OAuth2** and **Google Sheets OAuth2**
      - One for trigger, one for update node (can be same credential depending on your setup).
   4. **Telegram API**
      - Telegram bot token credential.
   5. **Google Gemini (PaLM / Google AI)**
      - Credential used by the two Gemini nodes.
   6. **Lusha API Key**
      - Credential type: *HTTP Header Auth*
      - Provide header required by Lusha (commonly `api_key: ...` or `Authorization: Bearer ...` depending on your Lusha plan; match what your credential already uses).

3. **Build Block 1: Google Sheets intake**
   1. Add node **Google Sheets Trigger** → name: **New or Updated row**
      - Poll every minute.
      - Set **Spreadsheet (documentId)** and **Sheet name**.
   2. Add node **Filter** → name: **Filter processed lines**
      - Condition: field `status` **is empty**.
   3. Connect: `New or Updated row` → `Filter processed lines`.

4. **Build Block 1: Telegram intake**
   1. Add node **Telegram Trigger** → name: **Telegram Trigger**
      - Updates: `message`
   2. Add node **Google Gemini (LangChain)** → name: **Parse Telegram Message**
      - Model: `models/gemini-2.5-flash`
      - Enable **JSON output**
      - Prompt: instruct to return only JSON with fields (company, company_domain, etc.) and `source_type="Telegram Bot"` (as in the workflow).
   3. Add node **Code** → name: **Format Telegram Lead Data**
      - Implement logic:
        - read Gemini output text from `content.parts[0].text`
        - strip ```json fences
        - `JSON.parse`
        - set `is_live_trigger=true` and `chat_id` from Telegram trigger.
   4. Connect: `Telegram Trigger` → `Parse Telegram Message` → `Format Telegram Lead Data`.

5. **Build Block 2: HubSpot company reference lookup**
   1. Add node **HTTP Request** → name: **Get HubSpot data**
      - Method: GET
      - URL: `https://api.hubapi.com/crm/v3/objects/companies`
      - Query parameters:
        - `limit=100`
        - `properties=name,city,domain`
      - Auth: use **HubSpot App Token** (predefined credential type).
   2. Add node **Split Out** → name: **Split HubSpot Search Results**
      - Split field: `results`
   3. Add node **Code** → name: **Mark as HubSpot Reference Data**
      - Add `is_reference_data=true` to each item.
   4. Connect: `Get HubSpot data` → `Split HubSpot Search Results` → `Mark as HubSpot Reference Data`.

6. **Build Block 2: Domain resolution for Sheets leads**
   1. Add node **Google Gemini (LangChain)** → name: **Domain Resolver**
      - Model: `models/gemini-2.0-flash`
      - Set **Continue on fail** (onError continue)
      - Prompt: return only a domain string or `null` (no protocol, no formatting).
   2. Add node **Merge** → name: **Append Resolved Domain to Lead**
      - Mode: **Combine by position**
   3. Connect:
      - `Filter processed lines` → `Domain Resolver`
      - `Domain Resolver` → `Append Resolved Domain to Lead` (input 1)
      - `Filter processed lines` → `Append Resolved Domain to Lead` (input 0)

7. **Build Block 3: Merge lead stream with reference data**
   1. Add node **Merge** → name: **Combine Lead with CRM Reference Data**
      - Mode: **Append**
   2. Connect reference stream:
      - `Mark as HubSpot Reference Data` → `Combine Lead with CRM Reference Data` (input 1)
   3. Connect lead stream(s):
      - Sheets: `Append Resolved Domain to Lead` → `Combine Lead with CRM Reference Data` (input 0)
      - Telegram: `Format Telegram Lead Data` → `Combine Lead with CRM Reference Data` (input 0)
   4. Also connect `Format Telegram Lead Data` → `Get HubSpot data` (so Telegram runs also fetch reference data).

8. **Build Block 3: Matching engine**
   1. Add node **Code** → name: **Match Lead to Existing Company**
      - Implement:
        - separate reference vs lead items
        - resolve `domain` using Gemini output / company_domain / email domain / existing domain
        - compute best match score
        - output `{updates_payload, similarity_score, existing_company_id}`
   2. Add node **Switch** → name: **Switch Logic**
      - Rule: `similarity_score >= 80`
      - Output 0: matched
      - Fallback: create
   3. Connect: `Combine Lead with CRM Reference Data` → `Match Lead to Existing Company` → `Switch Logic`.

9. **Build Block 4: HubSpot company create/update**
   1. Add node **HubSpot** → name: **HubSpot: Update Company**
      - Resource: Company; Operation: Update
      - Company ID: `existing_company_id`
      - Update fields: `city`, `companyDomainName` (regex-extracted domain).
   2. Add node **HubSpot** → name: **HubSpot: Create Company**
      - Resource: Company; Operation: Create
      - Name from `updates_payload.company`
      - Set city, lifecycle stage/status, domain, and description containing source metadata.
   3. Connect:
      - `Switch Logic` matched output → `HubSpot: Update Company`
      - `Switch Logic` fallback → `HubSpot: Create Company`

10. **Build Block 5: Email enrichment decision**
   1. Add node **Switch** → name: **Switch on email field**
      - Rule: if email missing (`!updates_payload.email`) → go to Lusha person enrichment.
      - Fallback: go directly to company ID resolution.
   2. Connect:
      - `HubSpot: Update Company` → `Switch on email field`
      - `HubSpot: Create Company` → `Switch on email field`

11. **Build Block 5: Lusha person enrichment (optional)**
   1. Add node **HTTP Request** → name: **Lusha Enrichment - email**
      - GET `https://api.lusha.com/v2/person`
      - Query params: firstName, lastName, companyName, companyDomain
      - Set response: JSON; enable “never error” behavior if desired.
   2. Add node **Code** → name: **Map Contact Enrichment Data**
      - If Lusha returns an email, set `updates_payload.email`.
   3. Connect:
      - `Switch on email field` (email missing path) → `Lusha Enrichment - email` → `Map Contact Enrichment Data` → `Resolve Company ID`.

12. **Build Block 5: Resolve company ID + retry**
   1. Add node **Code** → name: **Resolve Company ID**
      - Extract ID from current item or from HubSpot create/update node outputs; set `retry_needed` if missing.
   2. Add node **Switch** → name: **Validate Company ID Status**
      - If `retry_needed == true` → wait and loop back.
      - Else continue.
   3. Add node **Wait** → name: **Retry ID resolution**
      - Wait `2` (units as per your node’s configuration).
   4. Connect:
      - `Switch on email field` (fallback path) → `Resolve Company ID`
      - `Resolve Company ID` → `Validate Company ID Status`
      - `Validate Company ID Status` (retry path) → `Retry ID resolution` → `Resolve Company ID`

13. **Build Block 5: HubSpot note + contact creation**
   1. Add node **HTTP Request** → name: **Add a Note to Hubspot company**
      - POST `/crm/v3/objects/notes`
      - Body includes note properties and association to `id` using associationTypeId `190`.
      - Auth: HubSpot Bearer header auth.
   2. Add node **Switch** → name: **Check for Existing Contact Email**
      - If `updates_payload.email` not empty → create contact
      - Else skip to feedback routing.
   3. Add node **HubSpot** → name: **Create or update a contact**
      - Email: `updates_payload.email`
      - First/Last: payload
      - Associate to company ID: `{{$json.id}}`
   4. Connect from `Validate Company ID Status` (success path) to:
      - `Add a Note to Hubspot company`
      - `Check for Existing Contact Email`
   5. Connect:
      - `Check for Existing Contact Email` (has email) → `Create or update a contact`

14. **Build Block 5: Lusha firmographic enrichment + delta patch**
   1. Add node **HTTP Request** → name: **Lusha Company Enrichment**
      - GET `https://api.lusha.com/v2/company`
      - Query: domain, company
   2. Add node **Code** → name: **Prepare Firmographic Updates**
      - Build `delta_update`, set `has_changes`.
   3. Add node **Filter** → name: **Filter Leads with Data Changes**
      - Pass only if `has_changes == true`
   4. Add node **HTTP Request** → name: **HubSpot: Update Enriched Company**
      - PATCH `/crm/v3/objects/companies/{{hubspot_id}}`
      - JSON body: `{ "properties": delta_update }`
      - Auth: HubSpot Bearer header auth.
   5. Connect:
      - `Validate Company ID Status` (success path) → `Lusha Company Enrichment` → `Prepare Firmographic Updates` → `Filter Leads with Data Changes` → `HubSpot: Update Enriched Company`

15. **Build Block 6: Feedback routing**
   1. Add node **Filter** → name: **Identify Telegram triggered flow**
      - Condition: `updates_payload.is_live_trigger == true`
   2. Add node **Telegram** → name: **Update Telegram user**
      - `chatId = updates_payload.chat_id`
      - Text: success summary including whether “Updated” vs “Created” (using Switch Logic branch index).
   3. Add node **Filter** → name: **Identify updated row**
      - Condition: `updates_payload.is_live_trigger == false`
   4. Add node **Google Sheets** → name: **Acknowledge in sheet**
      - Operation: update
      - Set Spreadsheet ID and Sheet name
      - Map row identifier + set `status = "Processed"` (you must configure the row key mapping based on your sheet structure).
   5. Connect:
      - From **Create or update a contact** → both filters (`Identify Telegram triggered flow`, `Identify updated row`)
      - From **Check for Existing Contact Email** fallback output → both filters as well (so workflow still closes even without email)
      - `Identify Telegram triggered flow` → `Update Telegram user`
      - `Identify updated row` → `Acknowledge in sheet`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **HubSpot Smart Lead Enrichment and Matching via Google Sheets, Telegram, Lusha, and Gemini AI** / “Core Architecture – Agnostic Data Structure” / Setup: credentials, spreadsheet IDs, similarity threshold (80), ensure Gemini “Execute Once” disabled for bulk | Sticky note “HubSpot Smart Lead Enrichment and Matching via Google Sheets, Telegram, Lusha, and Gemini AI” (global comment) |
| Similarity threshold default is **80** | “Switch Logic” + global setup note |
| HubSpot company fetch uses `limit=100` without pagination | “Get HubSpot data” node design consideration |
| Potential repeated firmographic updates: delta comparison uses `resolveNode.properties`, but Resolve Company ID output doesn’t include properties | “Prepare Firmographic Updates” implementation detail |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.