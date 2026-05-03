Categorize Airtable invoices with OpenAI and TOON token optimization

https://n8nworkflows.xyz/workflows/categorize-airtable-invoices-with-openai-and-toon-token-optimization-11761


# Categorize Airtable invoices with OpenAI and TOON token optimization

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Categorize Airtable invoices with OpenAI and TOON token optimization  
**Purpose:** Fetch “Ready” invoices from Airtable, gather each invoice’s client + line items, compress the context into **TOON** (token-optimized format), ask OpenAI to categorize the invoice items, convert the model output back to JSON, and **write the category back to Airtable**.

### Logical blocks
**1.1 Execution & invoice selection (Airtable fetch + loop)**  
Manual run → fetch invoices → process invoices one by one.

**1.2 Data enrichment (client + invoice items aggregation)**  
For each invoice: load the related client and all invoice items; normalize item fields and aggregate them.

**1.3 Context packaging + token optimization (JSON → TOON)**  
Build a single context object (client + invoice) and encode it into TOON to reduce tokens.

**1.4 AI categorization (OpenAI) + decoding (TOON → JSON)**  
Send TOON to OpenAI with strict output constraints (TOON only) → decode response into JSON.

**1.5 Persistence (Airtable update)**  
Update the invoice record’s `Category` field using the AI result.

---

## 2. Block-by-Block Analysis

### 2.1 Execution & invoice selection (Airtable fetch + loop)

**Overview:** Starts the workflow manually, fetches invoice records from Airtable, and iterates through them in batches (default batch settings).

**Nodes involved:**
- When clicking ‘Execute workflow’
- Get Ready Invoices
- Loop Over Items

#### Node: **When clicking ‘Execute workflow’**
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) – entry point.
- **Configuration:** No parameters.
- **Outputs:** One trigger item → to **Get Ready Invoices**.
- **Edge cases:** None (only manual start).

#### Node: **Get Ready Invoices**
- **Type / role:** Airtable Search (`n8n-nodes-base.airtable`, operation: `search`) – loads invoice records.
- **Configuration choices:**
  - Base: **Custom JS - Invoicing Template** (`apphyDa3uYAq0VOMW`)
  - Table: **Invoices** (`tblW46vfkwOFQJLMs`)
  - Search operation with **no explicit filter configured** in this JSON.
- **Input:** Manual Trigger.
- **Output:** List of invoice records → **Loop Over Items**.
- **Credentials:** Airtable token credential “Airtable”.
- **Edge cases / failures:**
  - Airtable auth errors (invalid token, revoked token).
  - Returning *all* invoices if the “Ready” filter is not configured in Airtable node UI (despite node name implying it).
  - Pagination/limits depending on Airtable settings and n8n node defaults.

#### Node: **Loop Over Items**
- **Type / role:** Split in Batches (`n8n-nodes-base.splitInBatches`) – iterates invoices.
- **Configuration choices:**
  - Uses default options (batch size not explicitly set in JSON; n8n typically defaults to 1 or node-defined default).
- **Input:** Items from **Get Ready Invoices**.
- **Outputs:**
  - Output 1 → **Get Clients**
  - Output 1 → **Get Invoice Items**
- **Edge cases / failures:**
  - If batch size > 1, downstream nodes must handle multiple items; expressions referencing `$('Get Ready Invoices').item` can become ambiguous if execution context differs.
  - If there are zero invoices, downstream nodes never run.

---

### 2.2 Data enrichment (client + invoice items aggregation)

**Overview:** For each invoice, retrieves the related client and all invoice items tied to the invoice, maps key fields, then aggregates the invoice items into a single array.

**Nodes involved:**
- Get Clients
- Get Invoice Items
- Map Fields
- Aggregate

#### Node: **Get Clients**
- **Type / role:** Airtable Get (`n8n-nodes-base.airtable`) – loads a single client record by ID.
- **Configuration choices:**
  - Base: `apphyDa3uYAq0VOMW`
  - Table: **Clients** (`tblQdiFVsZ9w3sahJ`)
  - Record ID expression:
    - `={{ $('Get Ready Invoices').item.json['Client ID'][0] }}`
- **Input:** Current invoice item from **Loop Over Items**.
- **Output:** Client record → **GroupClientAndInvoice**.
- **Credentials:** Airtable token credential “Airtable”.
- **Key dependencies / assumptions:**
  - Invoice record has a field `Client ID` that is an array, and index `[0]` contains the Airtable record ID.
- **Edge cases / failures:**
  - `Client ID` missing/empty → expression evaluates to `undefined` → Airtable “record not found” or request error.
  - If `Client ID` contains multiple linked records, only the first is used.

#### Node: **Get Invoice Items**
- **Type / role:** Airtable Search (`n8n-nodes-base.airtable`, operation: `search`) – finds invoice line items for the invoice.
- **Configuration choices:**
  - Base: `apphyDa3uYAq0VOMW`
  - Table: **Invoice-Items** (`tblASYLVpsnrUoKt5`)
  - Filter formula:
    - `=FIND("{{ $json.ID }}", ARRAYJOIN({Invoice}))`
  - This expects the incoming item (from the invoice loop) contains `ID`.
- **Input:** Current invoice item from **Loop Over Items**.
- **Output:** Matching invoice-item records → **Map Fields**.
- **Credentials:** Airtable token credential “Airtable”.
- **Edge cases / failures:**
  - If `$json.ID` contains characters that break the formula or is empty, results may be wrong or empty.
  - `FIND` + `ARRAYJOIN` can produce false positives if IDs are substrings of other IDs.
  - If the field `{Invoice}` is not in the expected format (linked record array), the formula may not match.

#### Node: **Map Fields**
- **Type / role:** Set (`n8n-nodes-base.set`) – normalizes each invoice-item into a consistent schema.
- **Configuration choices (assignments):**
  - `description` = `{{$json.Description}}`
  - `quantity` = `{{$json.Hours}}` (stored as **string**)
  - `unitPrice` = `{{$json['Custom Hourly Rate'] || $json['Default Hourly Rate'][0]}}`
  - `invoiceId` = `{{$json.ID}}`
- **Input:** Invoice-item records from **Get Invoice Items**.
- **Output:** Mapped line items → **Aggregate**.
- **Edge cases / failures:**
  - `Default Hourly Rate` assumed to be an array; `[0]` can fail if empty.
  - If `Hours` is numeric and later consumers expect numeric, storing as string may cause downstream type issues.
  - Missing fields lead to null/undefined values in the context sent to OpenAI.

#### Node: **Aggregate**
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) – groups all mapped items into a single array field.
- **Configuration choices:**
  - Mode: “aggregateAllItemData”
  - Destination field: `items`
- **Input:** Mapped invoice items from **Map Fields**.
- **Output:** One aggregated item containing `items` array → **Loop Over Items** (back into the loop flow).
- **Edge cases / failures:**
  - If no invoice items are returned, the aggregate result may be empty or may output an item with an empty `items` array (behavior depends on node settings/version).
  - Large numbers of invoice items can create large payloads for OpenAI (even with TOON).

---

### 2.3 Context packaging + token optimization (JSON → TOON)

**Overview:** Combines client + invoice data into one object and converts it into TOON to reduce token usage before sending to OpenAI.

**Nodes involved:**
- GroupClientAndInvoice
- JSON to TOON

#### Node: **GroupClientAndInvoice**
- **Type / role:** Set (`n8n-nodes-base.set`) – constructs a compound object containing client and invoice data.
- **Configuration choices:**
  - Creates `clientAndInvoiceData` (object) with:
    - `client`: `{{ JSON.stringify($json) }}`
    - `invoice`: `{{ JSON.stringify($('Get Ready Invoices').item.json) }}`
- **Input:** Client record from **Get Clients**.
- **Output:** Item with `clientAndInvoiceData` → **JSON to TOON**.
- **Key expressions / variables:**
  - `$('Get Ready Invoices').item.json` references the invoice currently being processed.
- **Important design note:** It **stringifies** `client` and `invoice` into JSON strings, inside an object. This means downstream will see strings, not native objects.
- **Edge cases / failures:**
  - If invoice context resolution is wrong (batching/multiple items), it may attach the wrong invoice.
  - If either JSON contains characters or large fields, TOON conversion may inflate rather than compress.

#### Node: **JSON to TOON**
- **Type / role:** CustomJS JSON→TOON converter (`@custom-js/n8n-nodes-pdf-toolkit.jsonToToon`) – token optimization.
- **Configuration choices:**
  - `jsonData` = `={{ $json }}` (entire current item)
- **Input:** Output of **GroupClientAndInvoice**.
- **Output:** TOON-encoded representation (format defined by CustomJS node) → **OpenAI Enhancement**.
- **Credentials / requirements:**
  - CustomJS API credential “CustomJS account”.
  - Sticky note indicates **self-hosted n8n** is required for CustomJS usage + API key.
- **Edge cases / failures:**
  - CustomJS API unreachable/timeouts.
  - Conversion errors if input contains unsupported types/structures.

---

### 2.4 AI categorization (OpenAI) + decoding (TOON → JSON)

**Overview:** Sends TOON context to OpenAI with strict formatting requirements; receives TOON-only output; converts it back to JSON for use in Airtable update.

**Nodes involved:**
- OpenAI Enhancement
- TOON to JSON

#### Node: **OpenAI Enhancement**
- **Type / role:** OpenAI (LangChain) chat node (`@n8n/n8n-nodes-langchain.openAi`) – runs the categorization prompt.
- **Configuration choices:**
  - Model: `chatgpt-4o-latest`
  - Prompt content (core logic):
    - Injects TOON data via: `{{ $json.toon }}`
    - Requires:
      1) Create JSON array like `[{ "category": "..." }, ...]`
      2) Convert that JSON array to TOON notation
      3) Return **only** TOON notation
      4) Exactly one category per invoice item
- **Input:** TOON from **JSON to TOON**.
- **Output:** A structured OpenAI response object → **TOON to JSON**.
- **Credentials:** OpenAI API credential “OpenAi account”.
- **Edge cases / failures:**
  - Model may not follow formatting constraints (returns prose, JSON, or malformed TOON).
  - If the TOON payload does not include the invoice items in the expected place/name, the model may hallucinate categories.
  - Token limits / context truncation for large invoices.
  - API errors: rate limits, invalid key, model deprecation.

#### Node: **TOON to JSON**
- **Type / role:** CustomJS TOON→JSON converter (`@custom-js/n8n-nodes-pdf-toolkit.toonToJson`) – decodes AI output.
- **Configuration choices:**
  - `toonData` = `={{ $json.output[0].content[0].text }}`
    - Assumes the OpenAI node returns `output[0].content[0].text` containing TOON.
- **Input:** OpenAI response from **OpenAI Enhancement**.
- **Output:** Parsed JSON (the decoded array/object) → **Update record**.
- **Credentials / requirements:** CustomJS API credential “CustomJS account”.
- **Edge cases / failures:**
  - If OpenAI returns non-TOON or the node output path differs, expression fails (undefined) and conversion fails.
  - If TOON is malformed, conversion throws an error.

---

### 2.5 Persistence (Airtable update)

**Overview:** Writes the AI-derived category back into the invoice record in Airtable.

**Nodes involved:**
- Update record

#### Node: **Update record**
- **Type / role:** Airtable Update (`n8n-nodes-base.airtable`, operation: `update`) – persists category.
- **Configuration choices:**
  - Base: `apphyDa3uYAq0VOMW`
  - Table: **Invoices** (`tblW46vfkwOFQJLMs`)
  - Matching column: `id`
  - Fields mapped:
    - `id` = `={{ $('Get Ready Invoices').item.json.id }}`
    - `Category` = `={{ $json.json.category }}`
- **Input:** Decoded JSON from **TOON to JSON**.
- **Output:** Updated Airtable record response (end of workflow).
- **Key assumptions:**
  - The TOON→JSON output contains `json.category` at the top level (i.e., a single category), not an array.
  - The workflow seems intended to assign one category per invoice (not per line item) *or* it assumes a single-item array and reads only one element. The prompt, however, describes “each invoice item”.
- **Edge cases / failures:**
  - If the decoded result is an array (as prompted), `{{$json.json.category}}` may be undefined; you may need something like `{{$json.json[0].category}}` or iteration.
  - If Airtable field `Category` is not a text field or has validations, update may fail.
  - If `$('Get Ready Invoices').item.json.id` does not resolve correctly within loop/batch context, it can update the wrong record.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get Ready Invoices | ## Collect invoice data from Airtable |
| Get Ready Invoices | Airtable (Search) | Fetch invoice records | When clicking ‘Execute workflow’ | Loop Over Items | ## Collect invoice data from Airtable |
| Loop Over Items | Split in Batches | Iterate over invoices | Get Ready Invoices; Aggregate | Get Clients; Get Invoice Items | ## Collect invoice data from Airtable |
| Get Invoice Items | Airtable (Search) | Fetch invoice line items for an invoice | Loop Over Items | Map Fields | ## Collect invoice data from Airtable |
| Map Fields | Set | Normalize invoice-item fields | Get Invoice Items | Aggregate | ## Collect invoice data from Airtable |
| Aggregate | Aggregate | Combine line items into an array | Map Fields | Loop Over Items | ## Collect invoice data from Airtable |
| Get Clients | Airtable (Get) | Load client record linked to invoice | Loop Over Items | GroupClientAndInvoice | ## Collect invoice data from Airtable |
| GroupClientAndInvoice | Set | Build combined context payload | Get Clients | JSON to TOON | ## Categorizing Invoices |
| JSON to TOON | CustomJS jsonToToon | Encode payload in TOON | GroupClientAndInvoice | OpenAI Enhancement | ## Categorizing Invoices |
| OpenAI Enhancement | OpenAI (LangChain) | Categorize using TOON prompt | JSON to TOON | TOON to JSON | ## Categorizing Invoices |
| TOON to JSON | CustomJS toonToJson | Decode TOON back to JSON | OpenAI Enhancement | Update record | ## Categorizing Invoices |
| Update record | Airtable (Update) | Save category to invoice | TOON to JSON | — | ## Save categories back to Airtable |
| Sticky Note | Sticky Note | Documentation block | — | — | # Categorizing invoices with TOON optimization  \n\nHere is an example of how you can categorize all \nyour invoices in Airtable, from our [invoice creation example](https://n8n.io/workflows/9772-automatic-invoice-generation-and-email-with-airtable-and-customjs-pdf-generator/):\n\nTo save OpenAI tokens, we convert them into [TOON](https://github.com/toon-format/toon) and back again.\n\n### Setup \n-  This workflow uses the CustomJS JSON to TOON node from [CustomJS](https://www.customjs.space), which requires a self-hosted n8n instance and a CustomJS API key.\n- You will also need an API key for an Airtable table, which you can clone if you like. [Public Airtable Example](https://airtable.com/apphyDa3uYAq0VOMW/shrSe39NZYrqm4gtE)\n![Airtable Screenshot](https://www.beta.customjs.space/images/integration/n8n/InvoiceGeneratorWorkflow.png)\n\n### How it works \n1. **Manual Trigger**  \n   Run the workflow on demand\n\n2. **Load Data from Airtable**  \n   - Fetch ready invoices  \n   - Resolve related clients  \n   - Fetch and aggregate invoice items  \n\n3. **Prepare Structured Context**  \n   Group client, invoice, and invoice items into a single object\n\n4. **JSON → TOON**  \n   Convert structured data into token-efficient TOON\n\n5. **AI Categorization**  \n   - OpenAI receives TOON data  \n   - Assigns exactly one category per invoice  \n   - Returns only TOON (no JSON, no prose)\n\n6. **TOON → JSON**  \n   Convert AI output back into valid JSON\n\n7. **Persist Result**  \n   Update the invoice category field in Airtable\n\n@[youtube](QX6ffaf-uvA) |  |
| Sticky Note1 | Sticky Note | Section label | — | — | ## Collect invoice data from Airtable |
| Sticky Note2 | Sticky Note | Section label | — | — | ## Categorizing Invoices |
| Sticky Note3 | Sticky Note | Section label | — | — | ## Save categories back to Airtable |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- New workflow name: *Categorize Airtable invoices with OpenAI and TOON token optimization* (or your preferred name).

2) **Add Manual Trigger**
- Node: **Manual Trigger**
- Connect to next node.

3) **Add Airtable node: Get Ready Invoices**
- Node type: **Airtable**
- Credentials: create/select **Airtable Personal Access Token** (or Airtable Token) credential.
- Operation: **Search**
- Base: `apphyDa3uYAq0VOMW` (or your base)
- Table: `Invoices`
- (Optional but recommended) Add a filter to truly fetch only “Ready” invoices, e.g. `Status = "Ready"` depending on your schema.
- Connect: Manual Trigger → Get Ready Invoices.

4) **Add Split in Batches: Loop Over Items**
- Node type: **Split in Batches**
- Set batch size (recommended **1** to keep cross-node item references predictable).
- Connect: Get Ready Invoices → Loop Over Items.

5) **Add Airtable node: Get Clients**
- Node type: **Airtable**
- Operation: **Get** (by Record ID)
- Base: same base
- Table: `Clients`
- Record ID expression: `{{$('Get Ready Invoices').item.json['Client ID'][0]}}`
- Connect: Loop Over Items → Get Clients.

6) **Add Airtable node: Get Invoice Items**
- Node type: **Airtable**
- Operation: **Search**
- Base: same base
- Table: `Invoice-Items`
- FilterByFormula:
  - `FIND("{{$json.ID}}", ARRAYJOIN({Invoice}))`
- Connect: Loop Over Items → Get Invoice Items.

7) **Add Set node: Map Fields**
- Node type: **Set**
- Add fields:
  - `description` (String) = `{{$json.Description}}`
  - `quantity` (String) = `{{$json.Hours}}`
  - `unitPrice` (String) = `{{$json['Custom Hourly Rate'] || $json['Default Hourly Rate'][0]}}`
  - `invoiceId` (String) = `{{$json.ID}}`
- Connect: Get Invoice Items → Map Fields.

8) **Add Aggregate node**
- Node type: **Aggregate**
- Mode: aggregate all incoming item data into one item
- Destination field name: `items`
- Connect: Map Fields → Aggregate.

9) **Connect Aggregate back into the loop**
- Connect: Aggregate → Loop Over Items  
  (This matches the provided workflow’s structure; it effectively ensures aggregation completes per invoice loop.)

10) **Add Set node: GroupClientAndInvoice**
- Node type: **Set**
- Add field:
  - `clientAndInvoiceData` (Object) with value:
    - `{
        "client": {{ JSON.stringify($json) }},
        "invoice": {{ JSON.stringify($('Get Ready Invoices').item.json) }}
      }`
- Connect: Get Clients → GroupClientAndInvoice.

11) **Install/enable CustomJS TOON nodes (required)**
- Requirement: **self-hosted n8n** + CustomJS API key (per sticky note).
- Configure **CustomJS API** credentials in n8n.

12) **Add CustomJS node: JSON to TOON**
- Node type: `@custom-js/n8n-nodes-pdf-toolkit.jsonToToon`
- Parameter: `jsonData` = `{{$json}}`
- Credentials: CustomJS API credential.
- Connect: GroupClientAndInvoice → JSON to TOON.

13) **Add OpenAI node: OpenAI Enhancement**
- Node type: `@n8n/n8n-nodes-langchain.openAi`
- Credentials: OpenAI API key credential.
- Model: `chatgpt-4o-latest`
- Prompt/message content: include TOON injection `{{$json.toon}}` and strict output requirements (TOON only), matching the workflow text.
- Connect: JSON to TOON → OpenAI Enhancement.

14) **Add CustomJS node: TOON to JSON**
- Node type: `@custom-js/n8n-nodes-pdf-toolkit.toonToJson`
- Parameter: `toonData` = `{{$json.output[0].content[0].text}}`
- Credentials: CustomJS API credential.
- Connect: OpenAI Enhancement → TOON to JSON.

15) **Add Airtable node: Update record**
- Node type: **Airtable**
- Operation: **Update**
- Base: same base
- Table: `Invoices`
- Matching column: `id`
- Fields:
  - `id` = `{{$('Get Ready Invoices').item.json.id}}`
  - `Category` = `{{$json.json.category}}`
- Connect: TOON to JSON → Update record.

16) **Validate data shape**
- Run with a single invoice first.
- If OpenAI returns an array (as the prompt suggests), adjust mapping:
  - Example: `Category = {{$json.json[0].category}}` (or implement iteration to set categories per item).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Example workflow referenced for invoice creation | https://n8n.io/workflows/9772-automatic-invoice-generation-and-email-with-airtable-and-customjs-pdf-generator/ |
| TOON format reference | https://github.com/toon-format/toon |
| CustomJS service (TOON nodes provider) | https://www.customjs.space |
| Public Airtable example base | https://airtable.com/apphyDa3uYAq0VOMW/shrSe39NZYrqm4gtE |
| Airtable screenshot | https://www.beta.customjs.space/images/integration/n8n/InvoiceGeneratorWorkflow.png |
| Embedded video reference | `@[youtube](QX6ffaf-uvA)` |

