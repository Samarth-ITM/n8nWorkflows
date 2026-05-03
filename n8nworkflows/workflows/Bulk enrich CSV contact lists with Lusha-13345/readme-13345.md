Bulk enrich CSV contact lists with Lusha

https://n8nworkflows.xyz/workflows/bulk-enrich-csv-contact-lists-with-lusha-13345


# Bulk enrich CSV contact lists with Lusha

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Bulk Data Enrichment with Lusha  
**Purpose:** Bulk-enrich a CSV contact list using the Lusha Bulk Enrichment API and export an enriched CSV including emails/phones and company/person attributes.

**Target use cases:** Marketing Ops / RevOps teams preparing campaign or outbound lists who want to enrich records at scale (up to 100 contacts per API call).

### Logical blocks
**1.1 Input Reception & Batching**
- Manual start → read CSV → split items into batches of 100 for API efficiency.

**1.2 Bulk Enrichment via Lusha**
- Convert each batch into Lusha’s expected JSON payload → send to Lusha community node bulk enrichment operation.

**1.3 Output Flattening & CSV Export**
- Flatten Lusha’s enriched objects into tabular rows → export to `enriched_contacts.csv`.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Batching

**Overview:** Starts the workflow manually, loads contacts from a CSV, then batches them into groups of 100 items to reduce API calls and respect typical bulk limits.

**Nodes involved:**
- Start Manually
- Read CSV File
- Batch into Groups of 100

#### Node: Start Manually
- **Type / Role:** `Manual Trigger` — entry point for ad-hoc runs.
- **Configuration:** No parameters (default manual trigger).
- **Input/Output:** No input; outputs a single trigger event to **Read CSV File**.
- **Failure/edge cases:** None (unless workflow execution is restricted by instance permissions).

#### Node: Read CSV File
- **Type / Role:** `Spreadsheet File` — reads a CSV and converts rows to n8n items.
- **Configuration choices:**
  - `fileFormat: csv`
  - Operation not explicitly shown in JSON (defaults to “Read from file/binary” behavior in n8n’s Spreadsheet File node). In practice, this node requires a **binary file input** or a configured file source depending on how you set it up in the UI.
- **Key expectations:**
  - Sticky note indicates the CSV must contain an `email` column.
- **Input/Output:**
  - Input from **Start Manually**
  - Output items (one per row) to **Batch into Groups of 100**
- **Potential failures / edge cases:**
  - Missing file input or incorrect file path/source configuration in the node UI.
  - CSV parsing issues (delimiter/quote issues, empty file).
  - Missing `email` column: later code node filters out rows without `email`, which can result in empty enrichment calls.

#### Node: Batch into Groups of 100
- **Type / Role:** `Split In Batches` — iterates through items in fixed-size chunks.
- **Configuration choices:**
  - `batchSize: 100`
- **Input/Output:**
  - Input from **Read CSV File**
  - Output to **Format Batch for Lusha**
- **Version-specific notes:** v3 node supports the current batching behavior; if reproducing on older n8n versions, batching UI/options can differ.
- **Potential failures / edge cases:**
  - If total items < 100, only one batch runs (expected).
  - This workflow does **not** loop back to request the next batch (no “Continue” connection). As provided, it effectively processes only the first emitted batch depending on execution semantics. To reliably process all batches, you typically connect the node chain back into `Split In Batches` (see reproduction section).

---

### 2.2 Bulk Enrichment via Lusha

**Overview:** Converts the current batch into the JSON payload Lusha expects, then calls the Lusha Bulk Enrichment API through the community node.

**Nodes involved:**
- Format Batch for Lusha
- Enrich contacts in bulk

#### Node: Format Batch for Lusha
- **Type / Role:** `Code` — transforms items into Lusha’s `{ contacts: [...], metadata: {} }` payload.
- **Configuration choices (interpreted):**
  - Reads all incoming items: `const items = $input.all();`
  - Filters out rows missing `item.json.email`
  - Builds contacts with incrementing `contactId` (string) and `email`
  - Wraps into `payload = { contacts, metadata: {} }`
  - Outputs **one single item** containing `contactsPayload` as a JSON string:
    - `return [{ json: { contactsPayload: JSON.stringify(payload) } }];`
- **Key variables/expressions:**
  - Uses `$input.all()` (so it expects a batch worth of items)
  - Uses `item.json.email` as the only identifier for enrichment.
- **Input/Output:**
  - Input from **Batch into Groups of 100**
  - Output to **Enrich contacts in bulk**
- **Potential failures / edge cases:**
  - If all rows lack `email`, `contacts` becomes empty → Lusha request may fail or return empty results.
  - Duplicated emails aren’t deduplicated; may waste credits.
  - `contactIdCounter` resets per batch; if you need global uniqueness across batches, you’d need a different strategy.

#### Node: Enrich contacts in bulk
- **Type / Role:** `Lusha (community node)` — performs the bulk enrichment call.
- **Configuration choices:**
  - `operation: enrichBulk`
  - `bulkType: json`
  - `contactsPayloadJson: ={{ $json.contactsPayload }}`
  - Additional options empty: `contactBulkAdditionalOptions: {}`
- **Credentials:**
  - Uses `Lusha API` credential (API key/token), referenced as “Lusha API”.
- **Input/Output:**
  - Input from **Format Batch for Lusha** (single payload item)
  - Output to **Format Enriched Results**
- **Potential failures / edge cases:**
  - Authentication/authorization failure (invalid API key, revoked credentials).
  - Rate limits / quota exhaustion / credit limits (common for enrichment providers).
  - Payload schema mismatch (if Lusha expects different fields or structure).
  - Partial successes: some contacts may return incomplete fields; downstream formatting handles missing fields with `|| ''`.
- **Version-specific requirements:**
  - Requires the `@lusha-org/n8n-nodes-lusha` community node installed and compatible with your n8n version.

---

### 2.3 Output Flattening & CSV Export

**Overview:** Normalizes Lusha’s enriched response objects into flat columns suitable for CSV export, then writes a new CSV file.

**Nodes involved:**
- Format Enriched Results
- Export Enriched CSV

#### Node: Format Enriched Results
- **Type / Role:** `Code` — maps Lusha output objects to a consistent row schema.
- **Configuration choices (interpreted):**
  - Iterates through all input items
  - For each item, reads `item.json` into `d`
  - Produces a new item with fields such as:
    - `firstName`, `lastName`
    - `email` from `primaryEmail`
    - `phone` from `primaryPhone`
    - job/company attributes: `jobTitle`, `seniority`, `companyName`, domain, industry, size, revenue, location
    - adds timestamp `enrichedAt` (ISO string at runtime)
  - Uses fallback empty strings for missing data: `d.someField || ''`
- **Input/Output:**
  - Input from **Enrich contacts in bulk**
  - Output to **Export Enriched CSV**
- **Potential failures / edge cases:**
  - If Lusha node returns nested arrays/structures rather than one contact per item, the mapping may not match (you may need to inspect the actual output and adjust).
  - Field names (e.g., `primaryEmail`, `companyMainIndustry`) are assumed; if the node returns different naming, output will be blank.

#### Node: Export Enriched CSV
- **Type / Role:** `Spreadsheet File` — converts items into a CSV file.
- **Configuration choices:**
  - `operation: toFile`
  - `fileFormat: csv`
  - Output filename: `enriched_contacts.csv`
- **Input/Output:**
  - Input from **Format Enriched Results**
  - Output is a binary file (CSV) returned by the node execution.
- **Potential failures / edge cases:**
  - Large datasets can create large in-memory binaries.
  - If running on n8n cloud or restricted filesystem environments, “writing” behavior depends on node settings; typically this node outputs a binary rather than saving to disk unless paired with a Write Binary File node (depending on your configuration).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Bulk Data Enrichment | Sticky Note | Documentation / context |  |  | ## Bulk Data Enrichment with Lusha / **Who it's for:** Marketing Ops & RevOps teams preparing campaign lists / **What it does:** Upload campaign lists or CSV files, enrich them in batch with Lusha, and export records with emails, phones, and confidence scores. / ### How it works / 1. Trigger manually or on a schedule / 2. Read contacts from a spreadsheet/CSV / 3. Batch contacts into groups of up to 100 / 4. Send each batch to Lusha Bulk Enrichment API / 5. Merge results and write enriched CSV / ### Setup / 1. Install the Lusha community node / 2. Add your Lusha API credentials / 3. Place your CSV/XLSX file in the configured path / 4. Run manually or set a schedule |
| 📥 1. Read & Batch Input | Sticky Note | Documentation for block 1 |  |  | Trigger the workflow manually, read your CSV file, and split contacts into batches of 100 for efficient API usage. / **Nodes:** Manual Trigger → Read CSV → Split In Batches / 💡 Place your CSV file with an `email` column in the configured path before running. |
| 🔄 2. Bulk Enrich with Lusha | Sticky Note | Documentation for block 2 |  |  | Each batch is formatted into the Lusha Bulk Enrichment API payload and sent as a single call. Returns phone, email, job title, seniority, and company firmographics. / **Nodes:** Format Batch → Lusha Bulk Enrich / 📖 https://www.lusha.com/docs/ |
| 📤 3. Format & Export CSV | Sticky Note | Documentation for block 3 |  |  | Enriched results are flattened into a clean tabular format and exported as a new CSV file. / **Nodes:** Format Results → Export CSV / The output file `enriched_contacts.csv` contains: name, email, phone, title, seniority, company, industry, size, revenue. |
| Start Manually | Manual Trigger | Manual entry point |  | Read CSV File | Trigger the workflow manually, read your CSV file, and split contacts into batches of 100 for efficient API usage. / **Nodes:** Manual Trigger → Read CSV → Split In Batches / 💡 Place your CSV file with an `email` column in the configured path before running. |
| Read CSV File | Spreadsheet File | Parse CSV rows into items | Start Manually | Batch into Groups of 100 | Trigger the workflow manually, read your CSV file, and split contacts into batches of 100 for efficient API usage. / **Nodes:** Manual Trigger → Read CSV → Split In Batches / 💡 Place your CSV file with an `email` column in the configured path before running. |
| Batch into Groups of 100 | Split In Batches | Batch items to 100 per API call | Read CSV File | Format Batch for Lusha | Trigger the workflow manually, read your CSV file, and split contacts into batches of 100 for efficient API usage. / **Nodes:** Manual Trigger → Read CSV → Split In Batches / 💡 Place your CSV file with an `email` column in the configured path before running. |
| Format Batch for Lusha | Code | Build Lusha bulk JSON payload | Batch into Groups of 100 | Enrich contacts in bulk | Each batch is formatted into the Lusha Bulk Enrichment API payload and sent as a single call. Returns phone, email, job title, seniority, and company firmographics. / **Nodes:** Format Batch → Lusha Bulk Enrich / 📖 https://www.lusha.com/docs/ |
| Enrich contacts in bulk | Lusha (community) | Call Lusha Bulk Enrichment | Format Batch for Lusha | Format Enriched Results | Each batch is formatted into the Lusha Bulk Enrichment API payload and sent as a single call. Returns phone, email, job title, seniority, and company firmographics. / **Nodes:** Format Batch → Lusha Bulk Enrich / 📖 https://www.lusha.com/docs/ |
| Format Enriched Results | Code | Flatten response to CSV columns | Enrich contacts in bulk | Export Enriched CSV | Enriched results are flattened into a clean tabular format and exported as a new CSV file. / **Nodes:** Format Results → Export CSV / The output file `enriched_contacts.csv` contains: name, email, phone, title, seniority, company, industry, size, revenue. |
| Export Enriched CSV | Spreadsheet File | Convert items to CSV file output | Format Enriched Results |  | Enriched results are flattened into a clean tabular format and exported as a new CSV file. / **Nodes:** Format Results → Export CSV / The output file `enriched_contacts.csv` contains: name, email, phone, title, seniority, company, industry, size, revenue. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Bulk Data Enrichment with Lusha**

2. **Add the trigger**
   - Add node: **Manual Trigger**
   - Name: **Start Manually**

3. **Add CSV reader**
   - Add node: **Spreadsheet File**
   - Name: **Read CSV File**
   - Set **File Format**: `CSV`
   - Configure the input source depending on your environment:
     - Common approach: add a preceding node (e.g., **Read Binary File**) that loads a local file, then connect it into Spreadsheet File; or configure Spreadsheet File to read from binary.
   - Ensure your CSV has an `email` column.

4. **Add batching**
   - Add node: **Split In Batches**
   - Name: **Batch into Groups of 100**
   - Set **Batch Size**: `100`

5. **Add payload formatter**
   - Add node: **Code**
   - Name: **Format Batch for Lusha**
   - Paste this logic (adapt as needed to match your input column names):
     - Filters items missing `email`
     - Builds `contacts: [{ contactId, email }]`
     - Outputs `contactsPayload` as a JSON string

6. **Install and configure Lusha community node**
   - Install community node: `@lusha-org/n8n-nodes-lusha` (via n8n community nodes).
   - Add node: **Lusha**
   - Name: **Enrich contacts in bulk**
   - Set:
     - **Operation**: `Enrich Bulk`
     - **Bulk Type**: `JSON`
     - **Contacts Payload JSON**: expression `{{ $json.contactsPayload }}`

7. **Configure Lusha credentials**
   - Create credential: **Lusha API**
   - Enter your API key/token as required by the node.
   - Select this credential in **Enrich contacts in bulk**.

8. **Add result formatter**
   - Add node: **Code**
   - Name: **Format Enriched Results**
   - Map Lusha output fields to flat CSV columns (firstName, lastName, primaryEmail/Phone, jobTitle, company fields, etc.).
   - Add `enrichedAt` timestamp if desired.

9. **Add CSV export**
   - Add node: **Spreadsheet File**
   - Name: **Export Enriched CSV**
   - Set **Operation**: `To File`
   - Set **File Format**: `CSV`
   - Set **File Name**: `enriched_contacts.csv`

10. **Connect nodes in order**
   - Start Manually → Read CSV File → Batch into Groups of 100 → Format Batch for Lusha → Enrich contacts in bulk → Format Enriched Results → Export Enriched CSV

11. **Important: make batching process all batches (recommended fix)**
   - In many n8n patterns, you must loop:
     - Connect the last node in the batch processing chain (often after API call or after formatting) back into **Split In Batches** “Next Batch” input/continuation (depending on node ports in your n8n version).
   - Without this loop, the workflow may process only the first batch.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Lusha API documentation link provided in the workflow notes | https://www.lusha.com/docs/ |
| Setup prerequisites listed in the workflow notes: install Lusha community node, add Lusha API credentials, place CSV/XLSX file in configured path, run manually or on schedule | (From sticky note “Bulk Data Enrichment with Lusha”) |