Generate PDF invoices with line item support from Airtable using Google Sheets

https://n8nworkflows.xyz/workflows/generate-pdf-invoices-with-line-item-support-from-airtable-using-google-sheets-13071


# Generate PDF invoices with line item support from Airtable using Google Sheets

## 1. Workflow Overview

**Workflow name:** Invoice Generation System  
**Purpose:** Generate PDF invoices with **multiple line items** from a JSON payload, store the invoice + services in **Airtable**, fill a **Google Sheets** invoice template, then attach a **public PDF export link** back to the Airtable invoice record.

**Primary use case:** A frontend or another system sends a POST request containing a `clientId`, `date`, and an array of `services`. The workflow turns this into:
- Individual Service records in Airtable
- A new Invoice number (year-based sequence)
- A copied Google Sheets template populated with client + services + totals
- A shareable PDF link stored on the Airtable Invoice record

### 1.1 Input Reception
Receives JSON input via webhook and passes it into line-item processing.

### 1.2 Line Item Persistence & Formatting
Splits services, creates Airtable “Service” records, and formats them for Google Sheets row insertion.

### 1.3 Invoice Numbering & Template Creation
Looks up previous invoices to generate the next invoice number, then copies the Google Sheets template.

### 1.4 Template Filling (Services + Client)
Appends formatted line items into the sheet and replaces placeholders with client/invoice metadata.

### 1.5 Finalize, Share, and Sync Back to Airtable
Creates/updates the Airtable Invoice record, fills totals placeholders, enables public access, and attaches the PDF export URL.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Accepts a POST request that contains invoice header info and a list of service line items.  
**Nodes involved:** Webhook

#### Node: Webhook
- **Type / role:** `Webhook` trigger; entry point.
- **Config choices:**
  - **Method:** POST
  - **Path:** `your-webhook-path` (must be unique in your n8n instance)
- **Key data used downstream:**
  - `$('Webhook').first().json.body.date`
  - `$('Webhook').first().json.body.clientId`
  - `body.services` (array of line items)
- **Connections:**
  - **Output →** Split Services
- **Edge cases / failures:**
  - Missing/invalid JSON body (e.g., `services` not an array) causes later nodes (Split Out / Code) to fail.
  - If `date` isn’t in a format Airtable accepts for a date/time field, Airtable create may fail.

---

### Block 2 — Line Item Persistence & Formatting
**Overview:** Iterates through incoming services, stores each as a Service record in Airtable, then builds a 2D array payload suitable for Google Sheets “append values”.  
**Nodes involved:** Split Services, Add Service to DB, Formatting Data

#### Node: Split Services
- **Type / role:** `Split Out`; converts an array into one item per element.
- **Config choices:**
  - **Field to split out:** `body.services`
- **Connections:**
  - **Input ←** Webhook
  - **Output →** Add Service to DB
- **Edge cases / failures:**
  - If `body.services` is missing, null, or not an array, the node errors or outputs nothing (depending on n8n behavior/version and input).

#### Node: Add Service to DB
- **Type / role:** `Airtable` node (Create); persists each service line item.
- **Config choices:**
  - **Base:** `YOUR_AIRTABLE_BASE_ID`
  - **Table:** `YOUR_SERVICE_TABLE_ID` (cached name: “Servicio”)
  - **Operation:** Create
  - **Fields mapped from each split service item:**
    - `VAT` ← `$json.vat`
    - `Units` ← `$json.units`
    - `Value` ← `$json.price`
    - `Retention` ← `$json.retention`
    - `Description` ← `$json.description`
- **Connections:**
  - **Input ←** Split Services
  - **Output →** Formatting Data
- **Edge cases / failures:**
  - Airtable auth/base/table mismatch.
  - Type mismatches (e.g., `units` string not numeric).
  - Missing required fields (if your Airtable schema enforces them).
  - Note: totals are later assumed to exist (`fields.Total` in Formatting Data). If Airtable computes `Total` via formula, ensure the field exists and is populated on create.

#### Node: Formatting Data
- **Type / role:** `Code` node; aggregates all created Service records and formats them for the invoice sheet.
- **Config choices:**
  - Collects `$input.all()` (expects multiple Airtable “create” outputs).
  - Builds:
    - `id`: array of Airtable record IDs (service record ids)
    - `values.values`: 2D array to append in Google Sheets
  - Formatting rules:
    - `Value` → `toFixed(2) + '€'`
    - `VAT` → `'%'`
    - `Retention` → `'0%'` if 0, else `'-{retention}%'`
    - `Total` → `toFixed(2) + '€'`
- **Connections:**
  - **Input ←** Add Service to DB
  - **Output →** Get Previous Invoices
- **Edge cases / failures:**
  - If `fields.Value` / `fields.Total` are not numbers, `toFixed()` throws.
  - If Airtable didn’t return computed fields (depending on API/settings), `fields.Total` may be missing.
  - Currency/locale formatting is hardcoded (`€`, dot decimal).

---

### Block 3 — Invoice Numbering & Template Creation
**Overview:** Retrieves existing invoices to compute the next invoice number for the current year, then duplicates a Google Sheets template into a target folder.  
**Nodes involved:** Get Previous Invoices, Generate Invoice #, Copy Invoice Template

#### Node: Get Previous Invoices
- **Type / role:** `Airtable` node (Search); fetches invoices to determine the highest invoice number for the current year.
- **Config choices:**
  - **Base:** `YOUR_AIRTABLE_BASE_ID`
  - **Table:** `YOUR_INVOICE_TABLE_ID` (cached name: “Factura”)
  - **Operation:** Search (no explicit filter configured)
  - **Always Output Data:** enabled (continues even if no results)
- **Connections:**
  - **Input ←** Formatting Data
  - **Output →** Generate Invoice #
- **Edge cases / failures:**
  - Without filtering, large invoice tables can slow down executions or hit Airtable pagination limits/timeouts.
  - If returned items don’t include `Invoice #` in the expected shape, numbering may fail or skip.

#### Node: Generate Invoice #
- **Type / role:** `Code` node; parses invoice numbers and generates next sequence for the current year.
- **Config choices:**
  - Expects invoice numbers like: `YYYY-0001`
  - Logic:
    - Find max sequence where year matches current year
    - Next = max + 1, padded to 4 digits
  - Outputs:
    - `nextInvoiceNumber` (e.g., `2026-0007`)
    - `year`, `sequenceNumber`, `lastInvoiceNumber`
- **Connections:**
  - **Input ←** Get Previous Invoices
  - **Output →** Copy Invoice Template
- **Edge cases / failures:**
  - If some invoices have non-standard formats, they’re ignored (safe), but may cause gaps.
  - Concurrency risk: simultaneous executions can generate the same next number (no locking).

#### Node: Copy Invoice Template
- **Type / role:** `Google Drive` node (Copy); duplicates the template spreadsheet.
- **Config choices:**
  - **Template file ID:** `YOUR_GOOGLE_SHEETS_TEMPLATE_FILE_ID`
  - **New file name:** `{{$json.nextInvoiceNumber}}_Factura`
  - **Destination folder:** `YOUR_GOOGLE_DRIVE_OUTPUT_FOLDER_ID`
  - **Drive:** My Drive
- **Connections:**
  - **Input ←** Generate Invoice #
  - **Output →** Add Services to Invoice
- **Edge cases / failures:**
  - Google auth/permission errors (template not accessible, cannot write to folder).
  - Wrong file type (must be a Google Sheets file for later Sheets API calls).

---

### Block 4 — Template Filling (Line Items + Client)
**Overview:** Writes service rows into the sheet via Google Sheets API, fetches client info from Airtable, then replaces placeholders in the sheet with invoice/client values.  
**Nodes involved:** Add Services to Invoice, Get Client Info, Add Client Info

#### Node: Add Services to Invoice
- **Type / role:** `HTTP Request`; calls Google Sheets v4 “append values” endpoint.
- **Config choices:**
  - **POST URL pattern:** `https://sheets.googleapis.com/v4/spreadsheets/{{ $json.id }}/values/A19:append`
    - Uses the spreadsheet ID returned by “Copy Invoice Template” (`$json.id`).
  - **Query params:**
    - `valueInputOption=USER_ENTERED`
    - `insertDataOption=INSERT_ROWS`
  - **Body:** `$('Formatting Data').first().json.values`
    - Must be of shape: `{ "values": [ [ ...row1 ], [ ...row2 ] ] }`
  - **Auth:** predefined credential type `googleSheetsOAuth2Api`
- **Connections:**
  - **Input ←** Copy Invoice Template
  - **Output →** Get Client Info
- **Edge cases / failures:**
  - If the template doesn’t have room/expected layout starting at row 19, the invoice formatting will break.
  - If `Formatting Data` produced malformed body (missing `values.values` nesting), Google API rejects it.
  - If sheet is not sheetId `0` later (in batchUpdate), placeholder replacement might not work.

#### Node: Get Client Info
- **Type / role:** `Airtable` node (Get); retrieves client record fields for template replacement.
- **Config choices:**
  - **Record ID:** `$('Webhook').first().json.body.clientId`
  - **Table:** `YOUR_CLIENT_TABLE_ID` (cached name: “Client”)
- **Connections:**
  - **Input ←** Add Services to Invoice
  - **Output →** Add Client Info
- **Edge cases / failures:**
  - If `clientId` is not an Airtable record ID (e.g., a custom field), this will fail.
  - Permissions/base mismatch.

#### Node: Add Client Info
- **Type / role:** `HTTP Request`; Google Sheets batchUpdate with find/replace placeholders.
- **Config choices:**
  - **POST URL:** `https://sheets.googleapis.com/v4/spreadsheets/{{ $('Add Services to Invoice').item.json.spreadsheetId }}:batchUpdate`
  - **Requests:** find/replace on `sheetId: "0"` for:
    - `invoice_date` → webhook `date`
    - `invoice_number` → generated invoice number
    - `billingName` → `$json.Name`
    - `billingId` → `$json.ID`
    - `billingAddress` → `$json.Address`
    - `billingCity` → `$json.City`
    - `billingCountry` → `$json.Country`
  - **Auth:** `googleSheetsOAuth2Api`
- **Connections:**
  - **Input ←** Get Client Info
  - **Output →** Create a record1 (template)
- **Edge cases / failures:**
  - If the client table uses different field names than `Name`, `ID`, etc., replacements become blank.
  - If placeholders in the template don’t exactly match the “find” strings, nothing is replaced.
  - If the target sheet is not `sheetId=0`, replacements won’t apply.

---

### Block 5 — Finalize, Share, and Sync Back to Airtable
**Overview:** Creates the Airtable invoice record (linked to client + services), fills totals placeholders in the sheet, shares the sheet publicly, then updates the Airtable invoice record with a PDF export link.  
**Nodes involved:** Create a record1 (template), Add Invoice Summary Info, Share file, Update record  
**Also present but not connected:** Create a record1 (duplicate node)

#### Node: Create a record1 (template)
- **Type / role:** `Airtable` node (Create); creates an Invoice record in Airtable and links it to Client + Service records.
- **Config choices:**
  - **Table:** `YOUR_INVOICE_TABLE_ID` (“Factura”)
  - **Fields:**
    - `Date` ← webhook `date`
    - `Client` ← webhook `clientId` (linked record field)
    - `Service` ← `$('Formatting Data').first().json.id` (array of service record IDs)
    - `Invoice #` ← generated invoice number
  - **Typecast:** enabled (lets Airtable coerce types where possible)
- **Connections:**
  - **Input ←** Add Client Info
  - **Output →** Add Invoice Summary Info
- **Edge cases / failures:**
  - Linking fields must be configured as linked-record fields in Airtable; otherwise it will error.
  - If `Formatting Data` returns `id` array empty (e.g., no services), invoice is created without line items.

#### Node: Add Invoice Summary Info
- **Type / role:** `HTTP Request`; Google Sheets batchUpdate to insert totals into placeholders.
- **Config choices:**
  - **POST URL:** `https://sheets.googleapis.com/v4/spreadsheets/{{ $('Add Services to Invoice').item.json.spreadsheetId }}:batchUpdate`
  - **find/replace placeholders (sheetId "0"):**
    - `total_taxable_base` → `Number($json.fields.Total_Taxable_Base).toFixed(2) + '€'`
    - `total_vat` → `Number($json.fields['Total VAT']).toFixed(2) + '€'`
    - `total_retention` → `'-' + Number($json.fields['Total Retention']).toFixed(2) + '€'`
    - `total_total` → `Number($json.fields.Total).toFixed(2) + '€'`
  - **Important:** This node expects `$json.fields...` from the Airtable “Create invoice” output.
- **Connections:**
  - **Input ←** Create a record1 (template)
  - **Output →** Share file
- **Edge cases / failures:**
  - In the provided Airtable schema, some total fields appear “removed”/readOnly; if `Total` is not present in Airtable response, `Number(undefined)` becomes `NaN` and the sheet will get `NaN€`.
  - If totals are computed via rollups/formulas, Airtable may not have calculated them at creation time; you may need a “Get record” after create to fetch final computed totals.

#### Node: Share file
- **Type / role:** `Google Drive` node (Share); sets file permissions so anyone with link can view.
- **Config choices:**
  - **File ID:** `{{$json.spreadsheetId}}` (from previous HTTP response)
  - **Permissions:** `type=anyone`, `role=reader`
- **Connections:**
  - **Input ←** Add Invoice Summary Info
  - **Output →** Update record
- **Edge cases / failures:**
  - Domain/admin restrictions may prevent “anyone” sharing.
  - If `$json.spreadsheetId` is missing (API response shape differs), the node fails.

#### Node: Update record
- **Type / role:** `Airtable` node (Update); attaches the invoice PDF URL to the created Airtable invoice record.
- **Config choices:**
  - **Record ID:** `$('Create a record1').item.json.id`
    - Note: the connected create node is **Create a record1 (template)**, but this expression references **Create a record1** (different node name).
  - **Invoice attachment field value:** an array with one object:
    - `url` = `https://docs.google.com/spreadsheets/d/{{ spreadsheetId }}/export?format=pdf`
- **Connections:**
  - **Input ←** Share file
  - **Output →** none (end)
- **Edge cases / failures (important):**
  - **Likely misreference bug:** Since **Create a record1** is not connected in the execution path, `$('Create a record1')...` may resolve to no data and fail. It should typically reference `$('Create a record1 (template)')...`.
  - Airtable attachment fields accept URL attachments, but Airtable may require the URL to be publicly accessible; sharing step helps, but PDF export sometimes needs additional parameters (and some Google configurations may still require auth).

#### Node: Create a record1 (not connected)
- **Type / role:** `Airtable` Create invoice record (duplicate of the template node, but with additional schema fields listed).
- **Configuration:** Similar field mapping to the connected “(template)” node.
- **Connections:** None (not used in current workflow graph).
- **Risk:** Confusion and expression bugs (as seen in Update record referencing the unconnected node).
- **Recommendation:** Either delete it or connect it and update references consistently.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Entry point; receives invoice payload | — | Split Services | # Automated Invoice Generator (Airtable + Google Sheets) / How it works / Setup steps / Links: https://airtable.com/appUL5P7KN2Kqsgsf/shr5vCMwg2o4Z5wZI ; https://docs.google.com/spreadsheets/d/1XF9vcbsgYqDDgQwBn4RNtnzCkBjbk3jmKU7MBgIXtbo/edit?usp=sharing ; https://www.linkedin.com/in/sergiomedinah/ ; https://sergio-medina.com/ |
| Split Services | Split Out | Split `body.services` into items | Webhook | Add Service to DB | ## Process Line Items — Iterates through the incoming list of services, logging each one into the Airtable "Services" table and preparing the data for the invoice. |
| Add Service to DB | Airtable | Create Service record per line item | Split Services | Formatting Data | ## Process Line Items — Iterates through the incoming list of services, logging each one into the Airtable "Services" table and preparing the data for the invoice. |
| Formatting Data | Code | Build array payload for Sheets append + collect service IDs | Add Service to DB | Get Previous Invoices | ## Process Line Items — Iterates through the incoming list of services, logging each one into the Airtable "Services" table and preparing the data for the invoice. |
| Get Previous Invoices | Airtable | Search invoices for numbering | Formatting Data | Generate Invoice # | ## Fill Template — Calculates the new invoice number, duplicates the Google Sheet template, and maps the service rows and client details into the file. |
| Generate Invoice # | Code | Compute next `YYYY-####` invoice number | Get Previous Invoices | Copy Invoice Template | ## Fill Template — Calculates the new invoice number, duplicates the Google Sheet template, and maps the service rows and client details into the file. |
| Copy Invoice Template | Google Drive | Copy Google Sheets template into output folder | Generate Invoice # | Add Services to Invoice | ## Fill Template — Calculates the new invoice number, duplicates the Google Sheet template, and maps the service rows and client details into the file. |
| Add Services to Invoice | HTTP Request | Append line items to Sheet via Sheets API | Copy Invoice Template | Get Client Info | ## Fill Template — Calculates the new invoice number, duplicates the Google Sheet template, and maps the service rows and client details into the file. |
| Get Client Info | Airtable | Fetch client record fields | Add Services to Invoice | Add Client Info | ## Fill Template — Calculates the new invoice number, duplicates the Google Sheet template, and maps the service rows and client details into the file. |
| Add Client Info | HTTP Request | Replace placeholders with client/invoice data | Get Client Info | Create a record1 (template) | ## Fill Template — Calculates the new invoice number, duplicates the Google Sheet template, and maps the service rows and client details into the file. |
| Create a record1 (template) | Airtable | Create Invoice record linked to client + services | Add Client Info | Add Invoice Summary Info | ## Finalize & Sync — Creates the master Invoice record in Airtable, generates a public PDF link from Google Drive, and attaches the final document to the CRM record. |
| Add Invoice Summary Info | HTTP Request | Replace placeholders with totals | Create a record1 (template) | Share file | ## Finalize & Sync — Creates the master Invoice record in Airtable, generates a public PDF link from Google Drive, and attaches the final document to the CRM record. |
| Share file | Google Drive | Set file permission to anyone with link (reader) | Add Invoice Summary Info | Update record | ## Finalize & Sync — Creates the master Invoice record in Airtable, generates a public PDF link from Google Drive, and attaches the final document to the CRM record. |
| Update record | Airtable | Update Invoice record with PDF export URL attachment | Share file | — | ## Finalize & Sync — Creates the master Invoice record in Airtable, generates a public PDF link from Google Drive, and attaches the final document to the CRM record. |
| Create a record1 | Airtable | Unused duplicate “create invoice” node | — | — | ## Finalize & Sync — Creates the master Invoice record in Airtable, generates a public PDF link from Google Drive, and attaches the final document to the CRM record. |
| Sticky Note | Sticky Note | Documentation / setup notes | — | — | # Automated Invoice Generator (Airtable + Google Sheets) / How it works / Setup steps / Links: https://airtable.com/appUL5P7KN2Kqsgsf/shr5vCMwg2o4Z5wZI ; https://docs.google.com/spreadsheets/d/1XF9vcbsgYqDDgQwBn4RNtnzCkBjbk3jmKU7MBgIXtbo/edit?usp=sharing ; https://www.linkedin.com/in/sergiomedinah/ ; https://sergio-medina.com/ |
| Sticky Note1 | Sticky Note | Block label (Process Line Items) | — | — | ## Process Line Items — Iterates through the incoming list of services, logging each one into the Airtable "Services" table and preparing the data for the invoice. |
| Sticky Note2 | Sticky Note | Block label (Fill Template) | — | — | ## Fill Template — Calculates the new invoice number, duplicates the Google Sheet template, and maps the service rows and client details into the file. |
| Sticky Note3 | Sticky Note | Block label (Finalize & Sync) | — | — | ## Finalize & Sync — Creates the master Invoice record in Airtable, generates a public PDF link from Google Drive, and attaches the final document to the CRM record. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Webhook trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Path: `your-webhook-path`
   - Expected JSON body shape (example):
     - `date` (string/date)
     - `clientId` (Airtable record ID)
     - `services` (array of objects with `description`, `price`, `units`, `vat`, `retention`)

2. **Split incoming services into individual items**
   - Add node: **Split Out**
   - Field to split out: `body.services`
   - Connect: **Webhook → Split Out**

3. **Create Service records in Airtable**
   - Add node: **Airtable** (operation: **Create**)
   - Credentials: Airtable personal access token / OAuth (per your setup)
   - Base: select your base (`YOUR_AIRTABLE_BASE_ID`)
   - Table: Services (`YOUR_SERVICE_TABLE_ID`)
   - Map fields:
     - Description = `{{$json.description}}`
     - Value = `{{$json.price}}`
     - Units = `{{$json.units}}`
     - VAT = `{{$json.vat}}`
     - Retention = `{{$json.retention}}`
   - Connect: **Split Out → Airtable (Create service)**

4. **Aggregate created services + format for Google Sheets**
   - Add node: **Code** (JavaScript)
   - Implement logic to:
     - Collect all service record IDs into an array
     - Build a `values` payload with a 2D array of rows `[Description, Value€, Units, VAT%, Retention%, Total€]`
   - Connect: **Airtable (Create service) → Code (Formatting Data)**

5. **Fetch previous invoices from Airtable**
   - Add node: **Airtable** (operation: **Search**)
   - Base: `YOUR_AIRTABLE_BASE_ID`
   - Table: Invoices (`YOUR_INVOICE_TABLE_ID`)
   - (Recommended) Add a filter to only current year invoice numbers, if possible.
   - Enable “Always Output Data” to avoid breaking when no invoices exist.
   - Connect: **Formatting Data → Search invoices**

6. **Generate next invoice number**
   - Add node: **Code** (JavaScript)
   - Parse invoice numbers formatted as `YYYY-####`, find max for current year, output `nextInvoiceNumber`.
   - Connect: **Search invoices → Generate Invoice #**

7. **Copy the Google Sheets invoice template**
   - Add node: **Google Drive** (operation: **Copy**)
   - Credentials: Google OAuth2 with Drive access
   - Source file: `YOUR_GOOGLE_SHEETS_TEMPLATE_FILE_ID`
   - Destination folder: `YOUR_GOOGLE_DRIVE_OUTPUT_FOLDER_ID`
   - New name: `{{$json.nextInvoiceNumber}}_Factura`
   - Connect: **Generate Invoice # → Copy template**

8. **Append services into the copied sheet**
   - Add node: **HTTP Request**
   - Auth: **Google Sheets OAuth2** credential
   - POST to Sheets API append endpoint:
     - Spreadsheet ID from the Drive copy output
     - Range `A19:append` (adjust to match your template)
   - Query params:
     - `valueInputOption = USER_ENTERED`
     - `insertDataOption = INSERT_ROWS`
   - JSON body: the object that contains `values: [[...],[...]]` from Formatting Data
   - Connect: **Copy template → Add Services to Invoice**

9. **Get client details from Airtable**
   - Add node: **Airtable** (operation: **Get**)
   - Record ID: `{{$('Webhook').first().json.body.clientId}}`
   - Table: Clients (`YOUR_CLIENT_TABLE_ID`)
   - Connect: **Add Services to Invoice → Get Client Info**

10. **Replace placeholders in the sheet with client + invoice values**
   - Add node: **HTTP Request** (Google Sheets OAuth2)
   - POST to `spreadsheets/{spreadsheetId}:batchUpdate`
   - Use find/replace requests on the correct `sheetId` (often 0) for:
     - invoice date, invoice number, billing name/id/address/city/country
   - Connect: **Get Client Info → Add Client Info**

11. **Create the Airtable Invoice record**
   - Add node: **Airtable** (operation: **Create**)
   - Table: Invoices
   - Fields:
     - Date = `{{$('Webhook').first().json.body.date}}`
     - Client (linked) = `{{$('Webhook').first().json.body.clientId}}`
     - Service (linked) = `{{$('Formatting Data').first().json.id}}` (array)
     - Invoice # = `{{$('Generate Invoice #').item.json.nextInvoiceNumber}}`
   - Connect: **Add Client Info → Create Invoice record**

12. **Insert totals into the sheet**
   - Add node: **HTTP Request** (Google Sheets OAuth2)
   - POST batchUpdate find/replace for `total_taxable_base`, `total_vat`, `total_retention`, `total_total`
   - Source totals from the created invoice record fields (`$json.fields...`)
   - Connect: **Create Invoice record → Add Invoice Summary Info**
   - If totals are computed later in Airtable, insert an intermediate **Airtable Get** to re-fetch the invoice record before replacing totals.

13. **Share the spreadsheet publicly (optional but used here)**
   - Add node: **Google Drive** (operation: **Share**)
   - File ID: spreadsheetId from the Sheets API / Drive copy output
   - Permission: anyone / reader
   - Connect: **Add Invoice Summary Info → Share file**

14. **Update Airtable invoice with a PDF export URL**
   - Add node: **Airtable** (operation: **Update**)
   - Record ID: use the ID from the *connected* “Create Invoice record” node
   - Set attachment field “Invoice” to an array with `url`:
     - `https://docs.google.com/spreadsheets/d/{spreadsheetId}/export?format=pdf`
   - Connect: **Share file → Update record**
   - **Important fix:** Ensure the expression references the connected create node (avoid referencing an unused duplicate node).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Airtable template (3 tables: Clients, Invoices, Services, linked) | https://airtable.com/appUL5P7KN2Kqsgsf/shr5vCMwg2o4Z5wZI |
| Google Sheets invoice template example | https://docs.google.com/spreadsheets/d/1XF9vcbsgYqDDgQwBn4RNtnzCkBjbk3jmKU7MBgIXtbo/edit?usp=sharing |
| Author contact | https://www.linkedin.com/in/sergiomedinah/ |
| Author website | https://sergio-medina.com/ |
| Operational note: invoice numbering can collide under concurrency | Consider storing/locking the last number in Airtable or using an atomic counter approach |
| Likely workflow bug: Update record references an unconnected node (“Create a record1”) | Change to reference “Create a record1 (template)” or remove the duplicate node |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.