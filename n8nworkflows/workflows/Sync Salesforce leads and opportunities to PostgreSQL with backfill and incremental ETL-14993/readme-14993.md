Sync Salesforce leads and opportunities to PostgreSQL with backfill and incremental ETL

https://n8nworkflows.xyz/workflows/sync-salesforce-leads-and-opportunities-to-postgresql-with-backfill-and-incremental-etl-14993


# Sync Salesforce leads and opportunities to PostgreSQL with backfill and incremental ETL

Good, I've verified everything. Let me now write the comprehensive document.## 1. Workflow Overview

This workflow implements a full ETL pipeline that synchronises **Salesforce Lead and Opportunity** records into a **PostgreSQL** table for reporting and analytics. It supports two operating modes from a single workflow definition:

- **Historical Backfill** – triggered manually, it processes a user-defined date range in weekly batches to avoid API overload.
- **Incremental Sync** – triggered on a schedule (e.g. daily), it pulls only recently created records using dynamically computed date boundaries.

### Logical Block Structure

| Block | Purpose |
|-------|---------|
| **1 – Entry & Date Range Setup** | Two entry points produce date-period metadata for downstream fetch nodes. |
| **2 – Batch Loop Control** | A SplitInBatches node iterates over weekly periods; after each batch is loaded to Postgres, control returns here. |
| **3 – Salesforce Data Extraction** | Two Salesforce nodes retrieve Lead and Opportunity records for the current date period. |
| **4 – Phone-Based Routing** | IF nodes split each stream into "phone present" and "phone empty" branches so that records without a phone still flow to the output. |
| **5 – Phone Normalization** | Set nodes clean and reformat phone numbers into Indonesia's +62 international format. |
| **6 – Opportunity De-duplication** | Removes duplicate Opportunity rows on the normalized phone key before merging. |
| **7 – Lead–Opportunity Enrichment Merge** | Enriches Lead records (input 1) with Opportunity data (input 2) using the normalized phone as the join key. |
| **8 – Stream Aggregation** | Merges three streams (no-phone leads, no-phone opportunities, phone-enriched merged records) into one unified dataset. |
| **9 – Output Normalization & Database Loading** | A Code node reshapes all records into a canonical schema, then a Postgres node upserts them using `sf_id` as the matching column. |

---

## 2. Block-by-Block Analysis

### Block 1 – Entry & Date Range Setup

**Overview:** This block contains two independent entry points. The manual trigger path lets a user specify a fixed historical range, which is then split into 7-day weekly periods. The scheduled trigger path computes yesterday-to-today date boundaries dynamically. Both paths emit objects containing `since_datetime` and `until_datetime` consumed by the batch loop.

**Nodes Involved:**
- Manual Trigger (Historical Backfill)
- Set Historical Date Range
- Generate Weekly Periods
- Schedule Trigger (Incremental)
- Set Incremental Dates

---

**Manual Trigger (Historical Backfill)**
- **Type:** `n8n-nodes-base.manualTrigger` (v1)
- **Role:** Provides a manual entry point; clicking "Execute Node" starts the backfill run.
- **Configuration:** No parameters.
- **Input:** None (entry point).
- **Output:** Single empty item → Set Historical Date Range.
- **Edge Cases:** None; this is a simple trigger.

---

**Set Historical Date Range**
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Role:** Hard-codes the backfill window. The user must edit the JSON below before running.
- **Configuration:** Raw JSON mode; output:
  ```json
  { "start_date": "2025-12-07", "end_date": "2026-04-11" }
  ```
- **Key Expressions:** None; static values.
- **Input:** Manual Trigger (Historical Backfill).
- **Output:** Generate Weekly Periods.
- **Edge Cases:** If dates are invalid or `start_date` > `end_date`, the weekly-period generator will produce zero items and the loop will exit immediately.

---

**Generate Weekly Periods**
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Converts the fixed start/end dates into an array of weekly `{ since, until, since_datetime, until_datetime }` objects.
- **Configuration:** Inline JS reads `$input.first().json.start_date` and `end_date`, then iterates in 7-day steps. `since_datetime` appends `T00:00:00Z`; `until_datetime` appends `T23:59:59Z`.
- **Key Expressions:** Uses `formatDate()` helper; output is an array of `{ json: { since, until, since_datetime, until_datetime } }`.
- **Input:** Set Historical Date Range.
- **Output:** Loop Over Date Periods.
- **Edge Cases:** Very large date ranges (years) will generate many weekly items; each triggers a full Salesforce API cycle, so total runtime can be long. No explicit upper-bound protection.

---

**Schedule Trigger (Incremental)**
- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)
- **Role:** Fires automatically on a configurable interval (default: every 1 minute if not customised; users should set to e.g. daily).
- **Configuration:** `interval: [{}]` — interval parameters left at default; user must configure the actual cron/interval.
- **Input:** None (entry point).
- **Output:** Set Incremental Dates.
- **Edge Cases:** If the schedule runs faster than the workflow completes, concurrent executions may overlap. No locking mechanism is built in.

---

**Set Incremental Dates**
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Computes dynamic date boundaries: from yesterday 00:00:00 UTC to today 23:59:59 UTC.
- **Configuration:** JS creates `yesterday` and `now`, formats dates as `YYYY-MM-DD` strings and ISO datetimes. Output:
  ```json
  { "since": "2025-06-14", "until": "2025-06-15", "since_datetime": "2025-06-14T00:00:00.000Z", "until_datetime": "2025-06-15T23:59:59.999Z" }
  ```
- **Key Expressions:** Uses `setUTCHours`, `toISOString`, `padStart`.
- **Input:** Schedule Trigger (Incremental).
- **Output:** Loop Over Date Periods.
- **Edge Cases:** The incremental path emits a single date-period item (not a weekly batch), so SplitInBatches will process one batch and then exit the loop.

---

### Block 2 – Batch Loop Control

**Overview:** A SplitInBatches node iterates over the array of weekly periods (backfill) or the single incremental period. For each batch it emits the period metadata to both Salesforce fetch nodes. After the Postgres upsert completes, the data loops back here to process the next period.

**Nodes Involved:**
- Loop Over Date Periods

---

**Loop Over Date Periods**
- **Type:** `n8n-nodes-base.splitInBatches` (v3)
- **Role:** Batch iterator; output 0 fires when all batches are done; output 1 fires for each batch.
- **Configuration:** `retryOnFail: true` (retries on error); default batch size (all items in one batch per iteration since it splits one period at a time).
- **Input:** Generate Weekly Periods **or** Set Incremental Dates.
- **Output 1 (each batch):** Fetch Lead Records, Fetch Opportunity Records (simultaneously).
- **Loop-back Input:** Upsert Rows into Postgres → Loop Over Date Periods (input 0), which signals the node to advance to the next batch.
- **Edge Cases:** If the loop-back connection from Postgres is missing or broken, only the first batch will execute. If retryOnFail exhausts retries, the batch is skipped and the loop continues.

---

### Block 3 – Salesforce Data Extraction

**Overview:** Two parallel Salesforce nodes query Lead and Opportunity records created within the current date period. Both use `returnAll: true` (pagination handled internally) and pass date filters from the current batch's `since_datetime`/`until_datetime`.

**Nodes Involved:**
- Fetch Lead Records
- Fetch Opportunity Records

---

**Fetch Lead Records**
- **Type:** `n8n-nodes-base.salesforce` (v1)
- **Role:** SOQL-like retrieval of all Lead records within the current period.
- **Configuration:**
  - Operation: `getAll`
  - Resource: Lead (implied by `operation`)
  - Fields: `CreatedDate, CreatedById, Name, Phone, Email, LeadSource, Status, OwnerId`
  - Conditions: `CreatedDate >= $json.since_datetime` AND `CreatedDate <= $json.until_datetime`
  - `returnAll: true`
- **Credentials:** Salesforce OAuth2 (`Salesforce account (Developer org)`, credential ID `Rii3T2keWp8niw2M`).
- **Key Expressions:** `={{ $json.since_datetime }}`, `={{ $json.until_datetime }}` — reference fields from the current batch item.
- **Input:** Loop Over Date Periods (output 1).
- **Output:** Phone empty? (Lead).
- **Edge Cases:** If any field name does not exist in the user's Salesforce Lead object, the node will error. The Lead Field Reference sticky note warns users to verify API field names in Object Manager. Large result sets may cause memory pressure since `returnAll` fetches everything into one execution.

---

**Fetch Opportunity Records**
- **Type:** `n8n-nodes-base.salesforce` (v1)
- **Role:** SOQL-like retrieval of all Opportunity records within the current period.
- **Configuration:**
  - Operation: `getAll`
  - Resource: `opportunity`
  - Fields: `CreatedDate, CreatedById, Name, Phone__c, LeadSource, OwnerId, StageName, Amount, AccountId`
  - Conditions: `CreatedDate >= $json.since_datetime` AND `CreatedDate <= $json.until_datetime`
  - `returnAll: true`
- **Credentials:** Same Salesforce OAuth2 credential (inherited from workflow level in n8n; explicit credential reference not in JSON but same as Lead node).
- **Key Expressions:** Same date expressions as Lead node.
- **Input:** Loop Over Date Periods (output 1).
- **Output:** Phone empty? (Opportunity).
- **Edge Cases:** `Phone__c` is a custom field; if it does not exist in the target org, the fetch will fail. The Opportunity Field Reference sticky note calls this out. Same memory concerns as Lead node.

---

### Block 4 – Phone-Based Routing

**Overview:** Two IF nodes examine whether the phone field is empty for each record. Records **without** a phone bypass normalization and are sent directly to the final merge (preserving them in the output). Records **with** a phone proceed to normalization and the enrichment merge path.

**Nodes Involved:**
- Phone empty? (Lead)
- Phone empty? (Opportunity)

---

**Phone empty? (Lead)**
- **Type:** `n8n-nodes-base.if` (v2.2)
- **Role:** Routes Lead records based on whether `Phone` is blank.
- **Configuration:**
  - Condition: `String($json.Phone || '').trim()` is **empty** (string operation `empty`).
  - Combinator: `and` (single condition).
  - Case-sensitive: true; type validation: strict.
- **Key Expressions:** `={{ String($json.Phone || '').trim() }}`
- **Input:** Fetch Lead Records.
- **Output 0 (true – phone empty):** Merge All Streams (input 0).
- **Output 1 (false – phone present):** Normalize Lead Phone (+62).
- **Edge Cases:** Null, undefined, whitespace-only, and empty-string values all evaluate to "empty" and take the true branch. A phone value of "0" is NOT empty and will flow to normalization.

---

**Phone empty? (Opportunity)**
- **Type:** `n8n-nodes-base.if` (v2.2)
- **Role:** Routes Opportunity records based on whether `Phone__c` is blank.
- **Configuration:**
  - Condition: `String($json.Phone__c || '').trim()` is **empty** (string operation `empty`).
  - Same strict/case settings as Lead IF node.
- **Key Expressions:** `={{ String($json.Phone__c || '').trim() }}`
- **Input:** Fetch Opportunity Records.
- **Output 0 (true – phone empty):** Merge All Streams (input 1).
- **Output 1 (false – phone present):** Normalize Opportunity Phone (+62).
- **Edge Cases:** Same empty-value handling as the Lead IF node.

---

### Block 5 – Phone Normalization

**Overview:** Two Set nodes apply identical cleaning logic to raw phone strings: strip non-digit characters, convert leading `0` to `+62`, and prevent double country-code prefix. The output field name differs between the two nodes to serve as the merge key.

**Nodes Involved:**
- Normalize Lead Phone (+62)
- Normalize Opportunity Phone (+62)

---

**Normalize Lead Phone (+62)**
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Role:** Cleans the Lead's `Phone` field and stores the result in `body.nomorlead`.
- **Configuration:**
  - Assignment: `body.nomorlead` (string) = expression chaining `.trim()`, `.replace(/\s+/g, "")`, `.replace(/[^0-9]/g, "")`, `.replace(/^0/, "+62")`, `.replace(/^62+/, "+62")`.
  - `includeOtherFields: true` — preserves all original fields alongside the new one.
- **Key Expressions:**
  ```
  ($json.Phone || "").toString().trim().replace(/\s+/g, "").replace(/[^0-9]/g, "").replace(/^0/, "+62").replace(/^62+/, "+62")
  ```
- **Input:** Phone empty? (Lead) — false branch.
- **Output:** Merge Lead with Opportunity (input 0).
- **Edge Cases:** If `Phone` is null/undefined, the `|| ""` fallback ensures the expression still runs, producing an empty string (which after `.replace(/^0/, "+62")` won't match, so `body.nomorlead` will be `+62` if the string was empty — **this is a subtle bug**: an empty string passes the regex chain and results in `+62`). However, this branch is only reached when phone is NOT empty, so the risk is minimal in practice.

---

**Normalize Opportunity Phone (+62)**
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Role:** Cleans the Opportunity's `Phone__c` field and stores the result in `body.nomoroppty`.
- **Configuration:**
  - Same chained replacements as the Lead node, but source field is `Phone__c` and output field is `body.nomoroppty`.
  - `includeOtherFields: true`.
- **Key Expressions:**
  ```
  ($json.Phone__c || "").toString().trim().replace(/\s+/g, "").replace(/[^0-9]/g, "").replace(/^0/, "+62").replace(/^62+/, "+62")
  ```
- **Input:** Phone empty? (Opportunity) — false branch.
- **Output:** Remove Duplicate Opportunities.
- **Edge Cases:** Same empty-string edge case as Lead node (but only reached when phone is present).

---

### Block 6 – Opportunity De-duplication

**Overview:** Before merging with Leads, duplicate Opportunity records sharing the same normalized phone are collapsed to a single entry. This prevents one-to-many inflation during the enrichment join.

**Nodes Involved:**
- Remove Duplicate Opportunities

---

**Remove Duplicate Opportunities**
- **Type:** `n8n-nodes-base.removeDuplicates` (v2)
- **Role:** Deduplicates Opportunities by their normalized phone key.
- **Configuration:**
  - Compare mode: `selectedFields`
  - Fields to compare: `body.nomoroppty`
- **Input:** Normalize Opportunity Phone (+62).
- **Output:** Merge Lead with Opportunity (input 1).
- **Edge Cases:** Only the first occurrence is kept; subsequent duplicates are silently removed. If `body.nomoroppty` is `+62` (from the edge-case empty-string bug above), all such records would incorrectly collapse into one.

---

### Block 7 – Lead–Opportunity Enrichment Merge

**Overview:** A Merge node in `enrichInput1` mode joins Leads (input 1) with deduplicated Opportunities (input 2) on the normalized phone key. This enriches each Lead with matching Opportunity fields (StageName, Amount, AccountId, etc.) while preserving all Lead records even if no matching Opportunity exists.

**Nodes Involved:**
- Merge Lead with Opportunity

---

**Merge Lead with Opportunity**
- **Type:** `n8n-nodes-base.merge` (v3.2)
- **Role:** Left-enrichment join of Leads with Opportunities on phone.
- **Configuration:**
  - Mode: `combine`
  - Advanced: true
  - Join mode: `enrichInput1` (left join — all Leads kept; matching Opportunities append their fields)
  - Merge by fields: `body.nomorlead` (input 1) = `body.nomoroppty` (input 2)
- **Input 0:** Normalize Lead Phone (+62) — leads with phone.
- **Input 1:** Remove Duplicate Opportunities — deduplicated opportunities with phone.
- **Output:** Merge All Streams (input 2).
- **Edge Cases:** If multiple Opportunities share the same phone, only the first (after dedup) will join. Leads without a matching Opportunity still appear in the output with null Opportunity fields.

---

### Block 8 – Stream Aggregation

**Overview:** A three-input Merge node consolidates all records into a single stream: (1) leads without phone, (2) opportunities without phone, (3) phone-enriched merged records. No join logic is applied — all items are simply concatenated.

**Nodes Involved:**
- Merge All Streams

---

**Merge All Streams**
- **Type:** `n8n-nodes-base.merge` (v3.2)
- **Role:** Appends three streams into one unified list.
- **Configuration:**
  - Number of inputs: 3
  - Mode: default (append / combine all items).
- **Input 0:** Phone empty? (Lead) — true branch (leads without phone).
- **Input 1:** Phone empty? (Opportunity) — true branch (opportunities without phone).
- **Input 2:** Merge Lead with Opportunity — enriched records with phone.
- **Output:** Normalize Output Fields.
- **Edge Cases:** If any input has zero items, the node still processes the others. The node waits until all connected inputs have received data for the current execution before emitting output.

---

### Block 9 – Output Normalization & Database Loading

**Overview:** A Code node maps every record — regardless of origin — into a flat, unified schema with consistent field names. The Postgres node then upserts each row into the `n8n_salesforce_data` table, using `sf_id` as the deduplication key.

**Nodes Involved:**
- Normalize Output Fields
- Upsert Rows into Postgres

---

**Normalize Output Fields**
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Transforms heterogeneous Salesforce records into a canonical output schema.
- **Configuration:** Inline JS iterates over `items` and for each:
  1. Extracts `Source_Object` from `attributes.type` (e.g. "Lead" or "Opportunity").
  2. Extracts `SF_Id` from the last segment of `attributes.url` or falls back to `Id`.
  3. Reads `Phone` from `j.Phone` (Lead) or `j.body.nomorlead`; reads Opportunity phone from `j.Phone__c` or `j.body.nomoroppty`.
  4. Selects `rawPhone` preferring Lead phone then Opportunity phone.
  5. Re-normalizes phone to +62 format (second pass, ensuring consistency even for no-phone-branch records that bypass the earlier Set nodes).
  6. Outputs a flat object with fields: `Source_Object`, `SF_Id`, `CreatedById`, `CreatedDate`, `Name`, `Phone`, `Clean_Phone`, `Email`, `LeadSource`, `Status`, `StageName`, `OwnerId`, `AccountId`, `Amount`.
- **Key Expressions:** `attrs.type`, `attrs.url.split('/').pop()`, `p.replace(/\D/g, "")`, `p.startsWith("0")`, `p.startsWith("62")`.
- **Input:** Merge All Streams.
- **Output:** Upsert Rows into Postgres.
- **Edge Cases:** Records from the "no phone" branches will have `rawPhone` and `cleanPhone` as null (since neither `j.Phone` nor `j.body.nomorlead` will exist for Opportunities without phone, and vice versa). The second normalization pass correctly handles nulls. If `attributes` is missing (e.g. from a manually injected item), `sourceObject` and `recordId` become null.

---

**Upsert Rows into Postgres**
- **Type:** `n8n-nodes-base.postgres` (v2.6)
- **Role:** Writes unified records to PostgreSQL, inserting new rows or updating existing ones based on `sf_id`.
- **Configuration:**
  - Operation: `upsert`
  - Schema: `public`
  - Table: `n8n_salesforce_data`
  - Matching column: `sf_id` (used to identify existing rows)
  - Column mappings (expression-based):
    - `Name` ← `$json.Name`
    - `Email` ← `$json.Email`
    - `Phone` ← `$json.Clean_Phone`
    - `sf_id` ← `$json.SF_Id`
    - `Amount` ← `$json.Amount`
    - `Status` ← `$json.Status`
    - `OwnerId` ← `$json.OwnerId`
    - `AccountId` ← `$json.AccountId`
    - `StageName` ← `$json.StageName`
    - `synced_at` ← `$now`
    - `LeadSource` ← `$json.LeadSource`
    - `CreatedById` ← `$json.CreatedById`
    - `CreatedDate` ← `$json.CreatedDate`
  - `attemptToConvertTypes: false`, `convertFieldsToString: false`
- **Credentials:** PostgreSQL (`pg_[n8n] crm_automation_demo`, credential ID `pamrjOYaEuCMOgNf`).
- **Input:** Normalize Output Fields.
- **Output:** Loop Over Date Periods (loop-back to advance to next batch).
- **Edge Cases:**
  - The target table must exist with a matching schema before the workflow runs. A `synced_at` column of type `timestamp`/`timestamptz` is required.
  - If `sf_id` is null for any record, the upsert will fail because the matching column cannot be null.
  - Network timeout or credential expiration will cause the batch to fail; `retryOnFail` on the SplitInBatches node may retry the loop iteration.
  - Very large batches (thousands of rows) may hit Postgres connection limits or memory issues; no explicit chunking is done at the Postgres node level.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Lead Field Reference | stickyNote | Documentation: lists Lead API fields to verify | — | — | ## Lead Field Reference — Use API field names from **Salesforce Object Manager → Lead → Fields & Relationships**. Update the Lead fetch node only with fields that exist in your Salesforce Lead object. |
| Opportunity Field Reference | stickyNote | Documentation: lists Opportunity API fields to verify | — | — | ## Opportunity Field Reference — Use API field names from **Salesforce Object Manager → Opportunity → Fields & Relationships**. Update the Opportunity fetch node only with fields that exist in your Salesforce Opportunity object. |
| Batch Processing Note | stickyNote | Documentation: describes two modes, core flow, phone normalization, batch processing, and setup instructions | — | — | ## Salesforce Data Bank ETL (Backfill & Incremental Sync) — This workflow extracts Lead and Opportunity data from Salesforce and loads it into PostgreSQL for reporting and analytics. It supports two operating modes in a single workflow. ### Two Input Modes — Historical Backfill (Manual Trigger): Run once to populate past data. Set start_date and end_date in the Set Date Range node. The workflow splits the range into 7-day batches to reduce load and prevent API issues. — Incremental Sync (Schedule Trigger): Runs automatically and pulls recent data (daily). The date range is calculated dynamically using ISO datetime. ### Core Flow — Fetch Lead & Opportunity data from Salesforce — Split records based on phone availability — Normalize phone numbers into +62 format (Indonesia IDD) — Merge datasets using phone as a bridge — Standardize fields into a unified schema — Upsert into PostgreSQL (no duplicate records) ### Phone Normalization — Phone numbers are cleaned and converted into +62 format, representing Indonesia's international dialing code (IDD). This ensures consistent formatting, reliable merging, and compatibility with downstream systems. ### Batch Processing — Date periods are processed in batches (weekly chunks) to reduce load on Salesforce and downstream systems. This helps prevent oversized requests, timeouts, and temporary API errors during backfills. ### Setup — a) Salesforce credential — Configure Salesforce OAuth2 with access to Lead and Opportunity objects — b) PostgreSQL — Prepare target table (e.g. n8n_salesforce_data) with matching schema — c) Historical range — Set start_date and end_date in Set Date Range node before running backfill — d) Schedule trigger — Configure interval for incremental sync (e.g. daily) |
| Phone-Based Routing Overview | stickyNote | Documentation: explains the phone-present vs phone-empty branching logic | — | — | ## Phone-Based Routing Overview — This workflow intentionally splits data into two paths: Records **without a phone value** bypass phone normalization and still flow into the final output. Records **with a phone value** are normalized and used for lead-to-opportunity enrichment. This ensures records with empty phone fields are still preserved in the final dataset. |
| Lead and Opportunity Merge Note | stickyNote | Documentation: explains the merge-on-phone enrichment step | — | — | ## Lead and Opportunity Merge — This merge step enriches lead records with opportunity data using the normalized phone-based merge key. Records that do not have a usable phone value still remain in the final combined dataset. |
| Opportunity De-duplication Note | stickyNote | Documentation: explains why duplicate opportunities are removed | — | — | ## Opportunity De-duplication — This branch removes duplicate opportunity rows before the merge step. The current design intentionally keeps a single opportunity record per normalized opportunity phone key. |
| Manual Trigger (Historical Backfill) | manualTrigger (v1) | Manual entry point for backfill mode | — | Set Historical Date Range | |
| Set Historical Date Range | set (v3.4) | Hard-codes start/end dates for backfill | Manual Trigger (Historical Backfill) | Generate Weekly Periods | |
| Generate Weekly Periods | code (v2) | Splits date range into weekly batch objects | Set Historical Date Range | Loop Over Date Periods | |
| Schedule Trigger (Incremental) | scheduleTrigger (v1.2) | Scheduled entry point for incremental sync | — | Set Incremental Dates | |
| Set Incremental Dates | code (v2) | Computes yesterday-to-today date boundaries dynamically | Schedule Trigger (Incremental) | Loop Over Date Periods | |
| Loop Over Date Periods | splitInBatches (v3) | Iterates over weekly date periods; loop-back from Postgres | Generate Weekly Periods / Set Incremental Dates / Upsert Rows into Postgres | Fetch Lead Records, Fetch Opportunity Records (output 1) | |
| Fetch Lead Records | salesforce (v1) | Retrieves Lead records from Salesforce for current period | Loop Over Date Periods | Phone empty? (Lead) | ## Lead Field Reference — Use API field names from **Salesforce Object Manager → Lead → Fields & Relationships**. Update the Lead fetch node only with fields that exist in your Salesforce Lead object. |
| Fetch Opportunity Records | salesforce (v1) | Retrieves Opportunity records from Salesforce for current period | Loop Over Date Periods | Phone empty? (Opportunity) | ## Opportunity Field Reference — Use API field names from **Salesforce Object Manager → Opportunity → Fields & Relationships**. Update the Opportunity fetch node only with fields that exist in your Salesforce Opportunity object. |
| Phone empty? (Lead) | if (v2.2) | Routes Leads: empty phone → merge directly; present phone → normalize | Fetch Lead Records | Merge All Streams (input 0), Normalize Lead Phone (+62) | ## Phone-Based Routing Overview — This workflow intentionally splits data into two paths: Records **without a phone value** bypass phone normalization and still flow into the final output. Records **with a phone value** are normalized and used for lead-to-opportunity enrichment. This ensures records with empty phone fields are still preserved in the final dataset. |
| Phone empty? (Opportunity) | if (v2.2) | Routes Opportunities: empty phone → merge directly; present phone → normalize | Fetch Opportunity Records | Merge All Streams (input 1), Normalize Opportunity Phone (+62) | ## Phone-Based Routing Overview — This workflow intentionally splits data into two paths: Records **without a phone value** bypass phone normalization and still flow into the final output. Records **with a phone value** are normalized and used for lead-to-opportunity enrichment. This ensures records with empty phone fields are still preserved in the final dataset. |
| Normalize Lead Phone (+62) | set (v3.4) | Cleans and formats Lead phone to +62 Indonesia format; writes `body.nomorlead` | Phone empty? (Lead) false branch | Merge Lead with Opportunity (input 0) | |
| Normalize Opportunity Phone (+62) | set (v3.4) | Cleans and formats Opportunity phone to +62 Indonesia format; writes `body.nomoroppty` | Phone empty? (Opportunity) false branch | Remove Duplicate Opportunities | |
| Remove Duplicate Opportunities | removeDuplicates (v2) | Deduplicates Opportunities by `body.nomoroppty` before merge | Normalize Opportunity Phone (+62) | Merge Lead with Opportunity (input 1) | ## Opportunity De-duplication — This branch removes duplicate opportunity rows before the merge step. The current design intentionally keeps a single opportunity record per normalized opportunity phone key. |
| Merge Lead with Opportunity | merge (v3.2) | Left-enrichment join: Leads + Opportunities on normalized phone | Normalize Lead Phone (+62) (input 0), Remove Duplicate Opportunities (input 1) | Merge All Streams (input 2) | ## Lead and Opportunity Merge — This merge step enriches lead records with opportunity data using the normalized phone-based merge key. Records that do not have a usable phone value still remain in the final combined dataset. |
| Merge All Streams | merge (v3.2) | Appends three streams (no-phone leads, no-phone opps, enriched records) | Phone empty? (Lead) true, Phone empty? (Opportunity) true, Merge Lead with Opportunity | Normalize Output Fields | |
| Normalize Output Fields | code (v2) | Maps all records to a unified flat schema with second-pass phone normalization | Merge All Streams | Upsert Rows into Postgres | |
| Upsert Rows into Postgres | postgres (v2.6) | Upserts records into `n8n_salesforce_data` table on `sf_id` match | Normalize Output Fields | Loop Over Date Periods (loop-back) | |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named "Salesforce Leads & Opportunities to PostgreSQL (Backfill & Incremental Sync ETL)".

2. **Add Sticky Notes** for documentation (optional but recommended):
   - Place one at the top-left with the "Batch Processing Note" content (full ETL description, two modes, setup steps).
   - Place one near the Lead fetch area with "Lead Field Reference" content.
   - Place one near the Opportunity fetch area with "Opportunity Field Reference" content.
   - Place one spanning the phone routing area with "Phone-Based Routing Overview" content.
   - Place one near the merge area with "Lead and Opportunity Merge Note" content.
   - Place one near the dedup area with "Opportunity De-duplication Note" content.

3. **Add Manual Trigger (Historical Backfill):**
   - Type: Manual Trigger
   - No configuration needed.

4. **Add Set Historical Date Range:**
   - Type: Set (v3.4)
   - Mode: Raw JSON
   - JSON output: `{"start_date": "2025-12-07", "end_date": "2026-04-11"}` (adjust to your actual range)
   - Connect: Manual Trigger → Set Historical Date Range

5. **Add Generate Weekly Periods:**
   - Type: Code (v2)
   - JS Code: Parses `start_date` and `end_date`, loops in 7-day increments, outputs array of `{ since, until, since_datetime, until_datetime }`.
   - Connect: Set Historical Date Range → Generate Weekly Periods

6. **Add Schedule Trigger (Incremental):**
   - Type: Schedule Trigger (v1.2)
   - Configure interval (e.g. daily at a specific time)

7. **Add Set Incremental Dates:**
   - Type: Code (v2)
   - JS Code: Computes yesterday and today as `since`/`until` with `T00:00:00Z` and `T23:59:59Z` suffixes.
   - Connect: Schedule Trigger → Set Incremental Dates

8. **Add Loop Over Date Periods:**
   - Type: SplitInBatches (v3)
   - Enable `retryOnFail`
   - Connect: Generate Weekly Periods → Loop Over Date Periods (input 0)
   - Connect: Set Incremental Dates → Loop Over Date Periods (input 0)
   - **Do NOT connect the "done" output (output 0) to anything.**

9. **Add Fetch Lead Records:**
   - Type: Salesforce (v1)
   - Operation: `getAll`
   - Fields: `CreatedDate, CreatedById, Name, Phone, Email, LeadSource, Status, OwnerId`
   - Conditions: `CreatedDate >= {{ $json.since_datetime }}` AND `CreatedDate <= {{ $json.until_datetime }}`
   - `returnAll: true`
   - Credential: Salesforce OAuth2 (configure with your org's OAuth2 credentials, ensuring API access to Lead object)
   - Connect: Loop Over Date Periods (output 1) → Fetch Lead Records

10. **Add Fetch Opportunity Records:**
    - Type: Salesforce (v1)
    - Operation: `getAll`
    - Resource: `opportunity`
    - Fields: `CreatedDate, CreatedById, Name, Phone__c, LeadSource, OwnerId, StageName, Amount, AccountId`
    - Conditions: same date filter expressions as Lead node
    - `returnAll: true`
    - Credential: same Salesforce OAuth2
    - Connect: Loop Over Date Periods (output 1) → Fetch Opportunity Records

11. **Add Phone empty? (Lead):**
    - Type: IF (v2.2)
    - Condition: `String($json.Phone || '').trim()` → operation `empty`
    - Connect: Fetch Lead Records → Phone empty? (Lead)
    - True branch (output 0) will connect to Merge All Streams later.
    - False branch (output 1) will connect to Normalize Lead Phone.

12. **Add Phone empty? (Opportunity):**
    - Type: IF (v2.2)
    - Condition: `String($json.Phone__c || '').trim()` → operation `empty`
    - Connect: Fetch Opportunity Records → Phone empty? (Opportunity)
    - True branch (output 0) will connect to Merge All Streams later.
    - False branch (output 1) will connect to Normalize Opportunity Phone.

13. **Add Normalize Lead Phone (+62):**
    - Type: Set (v3.4)
    - `includeOtherFields: true`
    - Add assignment: `body.nomorlead` (string) = `{{ ($json.Phone || "").toString().trim().replace(/\s+/g, "").replace(/[^0-9]/g, "").replace(/^0/, "+62").replace(/^62+/, "+62") }}`
    - Connect: Phone empty? (Lead) false branch → Normalize Lead Phone (+62)

14. **Add Normalize Opportunity Phone (+62):**
    - Type: Set (v3.4)
    - `includeOtherFields: true`
    - Add assignment: `body.nomoroppty` (string) = `{{ ($json.Phone__c || "").toString().trim().replace(/\s+/g, "").replace(/[^0-9]/g, "").replace(/^0/, "+62").replace(/^62+/, "+62") }}`
    - Connect: Phone empty? (Opportunity) false branch → Normalize Opportunity Phone (+62)

15. **Add Remove Duplicate Opportunities:**
    - Type: Remove Duplicates (v2)
    - Compare: `selectedFields`
    - Fields to compare: `body.nomoroppty`
    - Connect: Normalize Opportunity Phone (+62) → Remove Duplicate Opportunities

16. **Add Merge Lead with Opportunity:**
    - Type: Merge (v3.2)
    - Mode: Combine, Advanced: enabled
    - Join mode: `enrichInput1`
    - Merge by fields: Input 1 field = `body.nomorlead`, Input 2 field = `body.nomoroppty`
    - Connect: Normalize Lead Phone (+62) → Merge Lead with Opportunity (input 0)
    - Connect: Remove Duplicate Opportunities → Merge Lead with Opportunity (input 1)

17. **Add Merge All Streams:**
    - Type: Merge (v3.2)
    - Number of inputs: 3
    - Connect: Phone empty? (Lead) true branch → Merge All Streams (input 0)
    - Connect: Phone empty? (Opportunity) true branch → Merge All Streams (input 1)
    - Connect: Merge Lead with Opportunity → Merge All Streams (input 2)

18. **Add Normalize Output Fields:**
    - Type: Code (v2)
    - JS Code: Full script that extracts `attributes.type`, `attributes.url`, reads phone from `Phone`/`body.nomorlead`/`Phone__c`/`body.nomoroppty`, performs second-pass +62 normalization, and outputs the unified schema (`Source_Object`, `SF_Id`, `CreatedById`, `CreatedDate`, `Name`, `Phone`, `Clean_Phone`, `Email`, `LeadSource`, `Status`, `StageName`, `OwnerId`, `AccountId`, `Amount`).
    - Connect: Merge All Streams → Normalize Output Fields

19. **Add Upsert Rows into Postgres:**
    - Type: Postgres (v2.6)
    - Operation: `upsert`
    - Schema: `public`
    - Table: `n8n_salesforce_data`
    - Mapping mode: `defineBelow`
    - Matching columns: `sf_id`
    - Column value expressions (all referencing `$json`):
      - `Name` ← `{{ $json.Name }}`
      - `Email` ← `{{ $json.Email }}`
      - `Phone` ← `{{ $json.Clean_Phone }}`
      - `sf_id` ← `{{ $json.SF_Id }}`
      - `Amount` ← `{{ $json.Amount }}`
      - `Status` ← `{{ $json.Status }}`
      - `OwnerId` ← `{{ $json.OwnerId }}`
      - `AccountId` ← `{{ $json.AccountId }}`
      - `StageName` ← `{{ $json.StageName }}`
      - `synced_at` ← `{{ $now }}`
      - `LeadSource` ← `{{ $json.LeadSource }}`
      - `CreatedById` ← `{{ $json.CreatedById }}`
      - `CreatedDate` ← `{{ $json.CreatedDate }}`
    - Credential: PostgreSQL (configure with host, database, user, password for your Postgres instance)
    - Connect: Normalize Output Fields → Upsert Rows into Postgres
    - Connect: Upsert Rows into Postgres → Loop Over Date Periods (loop-back to input 0)

20. **Prepare the PostgreSQL target table** (before first run):
    ```sql
    CREATE TABLE IF NOT EXISTS public.n8n_salesforce_data (
      id            TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
      "CreatedDate"  TIMESTAMPTZ,
      "CreatedById"  TEXT,
      "Name"         TEXT,
      "Phone"        TEXT,
      "Email"        TEXT,
      "LeadSource"   TEXT,
      "Status"       TEXT,
      "StageName"    TEXT,
      "OwnerId"      TEXT,
      "Amount"       NUMERIC,
      "AccountId"    TEXT,
      synced_at      TIMESTAMPTZ,
      sf_id          TEXT UNIQUE NOT NULL
    );
    ```

21. **Configure Salesforce OAuth2 credential:**
    - In n8n credential settings, create a Salesforce OAuth2 credential.
    - Provide Consumer Key and Consumer Secret from your Salesforce Connected App.
    - Set the callback URL to n8n's OAuth callback endpoint.
    - Grant access to Lead and Opportunity objects (API scope).
    - Authorize the credential by completing the OAuth flow.

22. **Verify field names:**
    - Open Salesforce → Setup → Object Manager → Lead → Fields & Relationships.
    - Confirm that `Phone`, `Email`, `LeadSource`, `Status`, `OwnerId`, `CreatedById`, `CreatedDate` exist as API names.
    - Repeat for Opportunity: confirm `Phone__c`, `StageName`, `Amount`, `AccountId`, `LeadSource`, `OwnerId`, `CreatedById`, `CreatedDate`.
    - If any field name differs in your org, update the corresponding Fetch node's `fields` parameter.

23. **Set the backfill date range** in the Set Historical Date Range node's JSON (step 4) to your desired start and end dates before running the manual trigger.

24. **Configure the schedule** in the Schedule Trigger node (step 6) to your preferred incremental sync frequency (e.g. daily at 02:00 UTC).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Phone normalization targets Indonesia's international dialing code (+62). If your Salesforce data uses a different country's format, modify the `.replace(/^0/, "+XX")` and `.replace(/^XX+/, "+XX")` patterns in both Normalize Set nodes and the Code node accordingly. | Phone normalization logic |
| The `Phone__c` field on Opportunity is a custom field (`__c` suffix). It must exist in your Salesforce org; otherwise the Fetch Opportunity Records node will fail. | Salesforce custom field |
| The Postgres upsert relies on `sf_id` being non-null and unique. Records where `sf_id` is null will cause upsert errors. | Database constraint |
| The `synced_at` column is set to `$now` at insertion time, providing a timestamp for the last successful sync of each row. | Postgres column |
| The SplitInBatches `retryOnFail` setting will retry the entire batch on any failure (including Postgres upsert failures). Consider adding error-handling branches for production use. | Retry behavior |
| For very large Salesforce orgs, `returnAll: true` may cause memory issues. Consider switching to pagination with batch-size limits if needed. | Salesforce API limits |
| The workflow is deactivated by default (`active: false`). Activate it after validating configuration to start the incremental schedule. | Workflow activation |
| The `body.nomorlead` and `body.nomoroppty` field names are used as internal merge keys. They are consumed by the Merge node and the final Code node but are not written to Postgres. | Internal field naming convention |
| No sticky note IDs are referenced in this document per requirements. | Documentation constraint |