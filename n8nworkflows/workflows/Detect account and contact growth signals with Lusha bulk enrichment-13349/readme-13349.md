Detect account and contact growth signals with Lusha bulk enrichment

https://n8nworkflows.xyz/workflows/detect-account-and-contact-growth-signals-with-lusha-bulk-enrichment-13349


# Detect account and contact growth signals with Lusha bulk enrichment

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Company & Contact Signals Detection + Enrichment with Lusha  
**Purpose:** Run a daily monitoring loop over CRM target accounts, bulk-enrich companies via Lusha, detect “growth signals” (headcount, revenue, funding), then—only for signaled accounts—search/enrich key contacts, alert Sales in Slack, and update the CRM company record.

### 1.1 Daily Input & Target Account Retrieval
Runs every 24 hours and pulls a batch of companies from HubSpot for monitoring.

### 1.2 Company Preparation & Lusha Bulk Enrichment
Formats the HubSpot company list into a Lusha bulk-enrichment payload (by domain), preserving a CRM lookup to compare “old vs new”.

### 1.3 Signal Detection, Conditional Routing, Contact Enrichment, Alerting & CRM Update
Compares Lusha data vs CRM values to compute signals. If signals exist, searches and enriches contacts, posts an alert to Slack, and updates the HubSpot company fields.

---

## 2. Block-by-Block Analysis

### Block 1 — Daily Input & Target Account Retrieval
**Overview:** Triggers daily and pulls a limited set of companies from HubSpot to evaluate for changes.  
**Nodes involved:** `Daily Signal Check`, `Get Target Accounts from CRM`

#### Node: Daily Signal Check
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — workflow entrypoint.
- **Configuration:** Runs every **24 hours** (`hoursInterval: 24`).
- **Inputs/Outputs:** No inputs; outputs to **Get Target Accounts from CRM**.
- **Edge cases / failures:**
  - Timezone and scheduling drift depending on n8n instance settings.
  - Missed runs if n8n is down; consider “catch up” strategy if required (not implemented).

#### Node: Get Target Accounts from CRM
- **Type / role:** HubSpot node (`n8n-nodes-base.hubspot`) — fetch companies.
- **Configuration choices:**
  - **Resource:** company
  - **Operation:** getAll
  - **Return all:** false
  - **Limit:** 50 (pulls up to 50 companies per run)
- **Credentials:** HubSpot OAuth2 (`HubSpot OAuth2`)
- **Inputs/Outputs:** Input from Schedule; output to **Format Companies for Bulk**.
- **Edge cases / failures:**
  - OAuth token expiry/revocation.
  - Rate limiting by HubSpot.
  - Default `getAll` without filters/list selection may not align with “target accounts” intent; you likely need filters, a list, or a search operation to define ICP/targets.

---

### Block 2 — Company Preparation & Lusha Bulk Enrichment
**Overview:** Builds a single bulk payload for Lusha using company domains and stores a domain→CRM mapping for later comparisons.  
**Nodes involved:** `Format Companies for Bulk`, `Enrich All Companies in Bulk`

#### Node: Format Companies for Bulk
- **Type / role:** Code (`n8n-nodes-base.code`) — transforms HubSpot output into:
  1) a Lusha bulk JSON payload (`companiesPayload`)  
  2) a CRM lookup table (`crmLookup`) keyed by domain
- **Key configuration / logic:**
  - Filters out companies that do **not** have `properties.domain`.
  - Builds an internal list with:
    - `domain`
    - `crmId` = HubSpot company `id`
    - `crmName` = `properties.name`
    - `crmEmployees` = `properties.numberofemployees` (default `'0'`)
    - `crmRevenue` = `properties.annualrevenue` (default `'0'`)
  - Creates payload shaped as:
    - `{ companies: [{ id, domain }, ...] }`
  - Stores:
    - `companiesPayload` as **stringified JSON**
    - `crmLookup` as an object: `{ [domain]: {crmId, name, employees, revenue} }`
- **Expressions/variables used:** Uses `$input.all()` and writes `companiesPayload` and `crmLookup` into the single returned item.
- **Inputs/Outputs:** Input from HubSpot list; output to **Enrich All Companies in Bulk**.
- **Edge cases / failures:**
  - If most companies lack `domain`, they are filtered out and Lusha receives fewer/no companies.
  - Domain normalization is not performed (case, www-prefix, trailing dots). Mismatches can cause later lookup failures.
  - Revenue/employees fields may be non-numeric strings; parsing later may produce `NaN` if not clean.

#### Node: Enrich All Companies in Bulk
- **Type / role:** Lusha community node (`@lusha-org/n8n-nodes-lusha.lusha`) — bulk company enrichment.
- **Configuration choices:**
  - **Resource:** company
  - **Operation:** enrichBulk
  - **Bulk type:** JSON
  - **Payload:** `companiesPayloadJson = {{ $json.companiesPayload }}`
- **Credentials:** Lusha API (`Lusha API`)
- **Inputs/Outputs:** Input from formatting node; output to **Detect Signals per Company**.
- **Version-specific requirements:**
  - Requires installing the **Lusha community node** and configuring credentials.
- **Edge cases / failures:**
  - API key invalid/expired.
  - Payload size limits or per-request limits (not handled here).
  - Lusha response structure differences (e.g., `employees`, `revenueRange`, `funding`) can break downstream parsing if fields differ.

---

### Block 3 — Signal Detection, Conditional Routing, Contact Enrichment, Alerting & CRM Update
**Overview:** Converts bulk enrichment results into per-company evaluations, flags those with signals, then enriches contacts and notifies Sales, while updating HubSpot company attributes.  
**Nodes involved:** `Detect Signals per Company`, `Has Signals?`, `Search for Key Contacts`, `Enrich Contacts from Search`, `Signal Alert to Sales`, `Update Account in CRM`

#### Node: Detect Signals per Company
- **Type / role:** Code (`n8n-nodes-base.code`) — computes signals per company and outputs one item per company.
- **Key configuration / logic:**
  - Reads all Lusha bulk result items: `const bulkResults = $input.all();`
  - Reads CRM lookup from earlier node:  
    `const crmLookup = $('Format Companies for Bulk').first().json.crmLookup;`
  - For each result:
    - `domain = lusha.domain || ''`
    - `crm = crmLookup[domain] || {}`
    - Computes:
      - **Headcount growth**: if both >0 and `(lushaEmployees - crmEmpCount)/crmEmpCount * 100 >= 10`
        - strength: `strong` if >=25 else `moderate`
      - **Revenue growth**: uses `lushaRevMax = lusha.revenueRange[1]` if array; compares to `crmRevenue`; triggers if >=15 (always `strong` in this code)
      - **Funding signal**: if `lusha.funding` is a non-empty object
    - Emits fields including:
      - `companyId` (HubSpot ID), `companyName`, `domain`
      - current vs previous employees
      - `industry`, `revenueRange`
      - `signals` array
      - `hasSignals` boolean
      - `signalStrength` (strong/moderate/none)
- **Inputs/Outputs:** Input from Lusha bulk enrichment; output to **Has Signals?**
- **Edge cases / failures:**
  - **Lookup mismatch** if Lusha returns a domain formatted differently from CRM (no normalization).
  - `parseInt/parseFloat` can yield `NaN`; comparisons will fail silently (no signals).
  - Assumes `lusha.revenueRange` is an array `[min,max]`; otherwise uses 0.
  - Funding structure may not be an object in some API variants.

#### Node: Has Signals?
- **Type / role:** IF (`n8n-nodes-base.if`) — routes only signaled accounts down the enrichment/alert paths.
- **Condition:** `{{ $json.hasSignals }}` is `true`.
- **Outputs:**
  - **True path:** to **Search for Key Contacts** and **Update Account in CRM** (parallel fan-out from the same IF output).
  - **False path:** not connected (no action).
- **Edge cases / failures:**
  - If upstream sets `hasSignals` as non-boolean (e.g., `"true"` string), condition may not match depending on coercion; here it’s treated as boolean check.

#### Node: Search for Key Contacts
- **Type / role:** Lusha node (`@lusha-org/n8n-nodes-lusha.lusha`) — finds contacts at the company.
- **Configuration choices:**
  - **Operation:** searchContacts
  - `contactSearchFilters`: `{}` (empty)
  - `searchAdditionalOptions`: `{}` (empty)
- **Credentials:** Lusha API
- **Inputs/Outputs:** Input from **Has Signals?** (true); output to **Enrich Contacts from Search**.
- **Edge cases / failures (important):**
  - With empty filters, search may return broad/irrelevant results or none, depending on node defaults.
  - The node does not receive explicit domain/company identifiers in parameters; if the Lusha node does not implicitly use incoming item context, it may not target the intended company. Typically you should pass domain/company name as filters.

#### Node: Enrich Contacts from Search
- **Type / role:** Lusha node (`@lusha-org/n8n-nodes-lusha.lusha`) — bulk enrich contacts returned by the search.
- **Configuration choices:**
  - **Operation:** enrichFromSearch
  - **Selection:** all
- **Credentials:** Lusha API
- **Inputs/Outputs:** Input from contact search; output to **Signal Alert to Sales**.
- **Edge cases / failures:**
  - Can return many contacts; may hit quotas/limits.
  - If search returns none, enrichment may output zero items and downstream Slack alert behavior depends on n8n execution semantics (may not run).

#### Node: Signal Alert to Sales
- **Type / role:** Slack (`n8n-nodes-base.slack`) — notifies a channel with signal details.
- **Configuration choices:**
  - **Channel:** `#account-signals`
  - **Message text:** uses expressions referencing `Detect Signals per Company`:
    - Company name/domain
    - Lists signals: `signals.map(...).join('\n')`
- **Credentials:** Slack OAuth2 (`Slack OAuth2`)
- **Inputs/Outputs:** Input from **Enrich Contacts from Search**; no further outputs.
- **Edge cases / failures:**
  - If multiple enriched contacts are output items, this node may post **multiple Slack messages** (one per item), while referencing a single “Detect Signals per Company” item—this can cause duplicates/spam.
  - Slack API rate limits, missing channel permissions, or invalid OAuth scopes.

#### Node: Update Account in CRM
- **Type / role:** HubSpot (`n8n-nodes-base.hubspot`) — updates company record with enriched attributes.
- **Configuration choices:**
  - **Resource:** company
  - **Operation:** update
  - **companyId:** `{{ $('Detect Signals per Company').item.json.companyId }}`
  - **Fields updated:**
    - `industry` = `{{ $('Detect Signals per Company').item.json.industry }}`
    - `numberofemployees` = `{{ $('Detect Signals per Company').item.json.currentEmployeeCount }}`
- **Credentials:** HubSpot OAuth2
- **Inputs/Outputs:** Input from **Has Signals?** (true). No further outputs.
- **Edge cases / failures:**
  - If `companyId` is empty (e.g., domain lookup failed), update will fail.
  - Field type mismatches (industry enumerations, numeric types) can reject updates.
  - HubSpot rate limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Company & Contact Signals Detection | Sticky Note | Documentation / canvas annotation | — | — | ## Company & Contact Signals Detection + Enrichment\n\n**Who it's for:** Sales teams who want to identify accounts showing buying intent\n\n**What it does:** Detects company signals (growth, hiring, funding) and enriches both account and contact records. Uses Lusha bulk enrichment for efficiency.\n\n### How it works\n1. Runs daily to check target accounts from your CRM\n2. Bulk-enriches all companies in one API call with Lusha\n3. Detects growth signals (headcount changes, revenue growth, funding)\n4. Searches key contacts at signal-showing accounts\n5. Alerts sales team with signal summaries via Slack\n6. Updates CRM with all enriched data\n\n### Setup\n1. Install the Lusha community node\n2. Configure CRM credentials\n3. Add Lusha API credentials\n4. Define your target account list or ICP filters\n5. Configure Slack channel for signal alerts |
| 📅 1. Daily Account Pull | Sticky Note | Documentation / block label | — | — | A daily schedule pulls your target accounts from HubSpot for signal monitoring.\n\n**Nodes:** Schedule Trigger → HubSpot Get Companies\n\n💡 Define your target account list in HubSpot using filters or a dedicated list. |
| 🔄 2. Company Bulk Enrichment | Sticky Note | Documentation / block label | — | — | All target accounts are formatted and enriched via the Lusha Company Bulk API in a single call. Returns employee count, revenue, funding, and industry data.\n\n**Nodes:** Format Companies → Lusha Company Bulk Enrich\n\n📖 [Lusha API docs](https://www.lusha.com/docs/) |
| 📤 3. Detect Signals & Alert Sales | Sticky Note | Documentation / block label | — | — | A code node compares fresh Lusha data vs CRM records to detect growth signals: headcount increase (10%+), revenue growth (15%+), and funding activity. For accounts showing signals, Lusha searches for key contacts and alerts your sales team via Slack.\n\n**Nodes:** Detect Signals → Has Signals? → Search Contacts + Update CRM → Enrich Contacts → Slack Alert\n\n💡 Adjust signal thresholds in the Code node to match your definition of growth. |
| Daily Signal Check | Schedule Trigger | Time-based workflow trigger | — | Get Target Accounts from CRM | |
| Get Target Accounts from CRM | HubSpot | Fetch target companies | Daily Signal Check | Format Companies for Bulk | |
| Format Companies for Bulk | Code | Build Lusha bulk payload + CRM lookup | Get Target Accounts from CRM | Enrich All Companies in Bulk | |
| Enrich All Companies in Bulk | Lusha (community) | Bulk company enrichment by domain | Format Companies for Bulk | Detect Signals per Company | |
| Detect Signals per Company | Code | Compute signals and per-company output items | Enrich All Companies in Bulk | Has Signals? | |
| Has Signals? | IF | Route only companies with signals | Detect Signals per Company | Search for Key Contacts; Update Account in CRM | |
| Search for Key Contacts | Lusha (community) | Contact search for signaled accounts | Has Signals? (true) | Enrich Contacts from Search | |
| Enrich Contacts from Search | Lusha (community) | Enrich contacts returned by search | Search for Key Contacts | Signal Alert to Sales | |
| Signal Alert to Sales | Slack | Post signal summary to Sales channel | Enrich Contacts from Search | — | |
| Update Account in CRM | HubSpot | Update HubSpot company fields | Has Signals? (true) | — | |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Company & Contact Signals Detection + Enrichment with Lusha**

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Name: `Daily Signal Check`
   - Configure interval: every **24 hours**

3. **Add HubSpot node to fetch companies**
   - Node: **HubSpot**
   - Name: `Get Target Accounts from CRM`
   - Credentials: create/select **HubSpot OAuth2**
   - Operation:
     - Resource: **Company**
     - Operation: **Get All**
     - Return All: **false**
     - Limit: **50**
   - Connect: `Daily Signal Check` → `Get Target Accounts from CRM`
   - (Optional but recommended) adjust this node to pull only your target list/ICP (filters/search/list), since the provided workflow uses a generic getAll.

4. **Add Code node to format companies for Lusha bulk enrichment**
   - Node: **Code**
   - Name: `Format Companies for Bulk`
   - Paste logic that:
     - filters companies with a `domain`
     - constructs `companiesPayload` as a JSON string `{ companies: [{id,domain}...] }`
     - constructs `crmLookup` keyed by domain storing CRM id/name/employees/revenue
   - Connect: `Get Target Accounts from CRM` → `Format Companies for Bulk`

5. **Install and configure the Lusha community node**
   - Install `@lusha-org/n8n-nodes-lusha` in your n8n environment (community nodes).
   - Create **Lusha API** credentials (API key / required auth per node).

6. **Add Lusha bulk company enrichment node**
   - Node: **Lusha**
   - Name: `Enrich All Companies in Bulk`
   - Credentials: **Lusha API**
   - Set:
     - Resource: **company**
     - Operation: **enrichBulk**
     - Bulk type: **json**
     - Companies payload JSON: `{{ $json.companiesPayload }}`
   - Connect: `Format Companies for Bulk` → `Enrich All Companies in Bulk`

7. **Add Code node to detect signals**
   - Node: **Code**
   - Name: `Detect Signals per Company`
   - Implement logic to:
     - read Lusha results
     - read `crmLookup` from `Format Companies for Bulk`
     - compute signals:
       - headcount growth threshold **10%+** (strong if **25%+**)
       - revenue growth threshold **15%+**
       - funding presence
     - output one item per company including `hasSignals`
   - Connect: `Enrich All Companies in Bulk` → `Detect Signals per Company`

8. **Add IF node to filter only companies with signals**
   - Node: **IF**
   - Name: `Has Signals?`
   - Condition: Boolean `{{ $json.hasSignals }}` is **true**
   - Connect: `Detect Signals per Company` → `Has Signals?`

9. **Add HubSpot update node (true branch)**
   - Node: **HubSpot**
   - Name: `Update Account in CRM`
   - Credentials: **HubSpot OAuth2**
   - Resource: **Company**
   - Operation: **Update**
   - Company ID: `{{ $('Detect Signals per Company').item.json.companyId }}`
   - Update fields:
     - `industry` = `{{ $('Detect Signals per Company').item.json.industry }}`
     - `numberofemployees` = `{{ $('Detect Signals per Company').item.json.currentEmployeeCount }}`
   - Connect: `Has Signals?` (true output) → `Update Account in CRM`

10. **Add Lusha contact search node (true branch)**
    - Node: **Lusha**
    - Name: `Search for Key Contacts`
    - Credentials: **Lusha API**
    - Operation: **searchContacts**
    - Leave filters/options as in the workflow (empty), or preferably add filters (domain/company) to target the signaled account.
    - Connect: `Has Signals?` (true output) → `Search for Key Contacts`

11. **Add Lusha enrich-from-search node**
    - Node: **Lusha**
    - Name: `Enrich Contacts from Search`
    - Credentials: **Lusha API**
    - Operation: **enrichFromSearch**
    - Selection: **all**
    - Connect: `Search for Key Contacts` → `Enrich Contacts from Search`

12. **Add Slack alert node**
    - Node: **Slack**
    - Name: `Signal Alert to Sales`
    - Credentials: create/select **Slack OAuth2** with permission to post to the chosen channel
    - Channel: `#account-signals`
    - Text: build a message that references `Detect Signals per Company` fields and formats the `signals` array.
    - Connect: `Enrich Contacts from Search` → `Signal Alert to Sales`

13. **(Optional) Add sticky notes**
    - Add sticky notes for the three blocks and the overall description, including the link: https://www.lusha.com/docs/

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Lusha API docs | https://www.lusha.com/docs/ |
| Setup reminders: install Lusha community node; configure CRM credentials; add Lusha API credentials; define target account list/ICP filters; configure Slack channel | From the main sticky note content |
| Thresholds for signals are defined in the “Detect Signals per Company” code (headcount 10%+, revenue 15%, strong headcount at 25%+) | Block note: “Adjust signal thresholds in the Code node…” |