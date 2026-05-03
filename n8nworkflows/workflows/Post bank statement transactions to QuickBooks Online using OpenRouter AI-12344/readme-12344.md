Post bank statement transactions to QuickBooks Online using OpenRouter AI

https://n8nworkflows.xyz/workflows/post-bank-statement-transactions-to-quickbooks-online-using-openrouter-ai-12344


# Post bank statement transactions to QuickBooks Online using OpenRouter AI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Post bank statement transactions to QuickBooks Online using OpenRouter AI

**Purpose:**  
This workflow ingests a bank statement PDF uploaded through an n8n Form, extracts its text (supports password-protected PDFs), uses an OpenRouter-hosted chat model to extract and classify transactions (credit vs debit) into a strict JSON schema, then posts each transaction into **QuickBooks Online (QBO)**:
- **Credits (income)** → create **SalesReceipt** records (and auto-create missing **Items** and **Customers**).
- **Debits (expenses)** → create **Purchase** records (and auto-create missing **Vendors** and map to an **Expense Account** category from Chart of Accounts).

### 1.1 Input Reception & PDF Text Extraction
Form upload → PDF text extraction.

### 1.2 AI Transaction Extraction (OpenRouter + Structured Parser)
Extracted statement text → AI agent → validated structured JSON array output.

### 1.3 Transaction Iteration & Routing
Split array into individual items → loop → route by `transactionType` (credit/debit).

### 1.4 Credit Path (Income → SalesReceipt)
Resolve/create Item → resolve/create Customer → build payload → HTTP POST SalesReceipt.

### 1.5 Debit Path (Expense → Purchase)
Load expense accounts → resolve/create Vendor → auto-pick expense category → build payload → HTTP POST Purchase.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & PDF Extraction
**Overview:** Receives a bank statement PDF via a hosted form and extracts raw text from the PDF binary for downstream AI parsing.  
**Nodes involved:** `Workflow Overview` (sticky), `Bank Statement Form`, `Extract PDF Text`, `PDF Extraction` (sticky)

#### Node: Bank Statement Form
- **Type / role:** `Form Trigger` — entry point; provides a public form to upload a PDF.
- **Key configuration:**
  - Title: “QuickBooks Bank Statement Upload”
  - Description: “Upload your bank statement PDF to automatically create entries in QuickBooks”
  - Field: required `file` “Bank Statement PDF”, accepts `.pdf`
  - Attribution disabled (`appendAttribution: false`)
- **I/O connections:**
  - Output → `Extract PDF Text`
- **Potential failures / edge cases:**
  - Large PDFs can exceed n8n limits depending on hosting configuration.
  - Users may upload non-PDF files with `.pdf` extension; extraction may fail.
  - Multi-statement PDFs may produce very large text, impacting LLM token limits.

#### Node: Extract PDF Text
- **Type / role:** `Extract From File` — converts PDF binary into text.
- **Key configuration:**
  - Operation: `pdf`
  - Binary property: `Bank_Statement_PDF` (must match the form upload binary key)
  - Options include `password: "YOUR_CREDENTIAL_HERE"` (for protected PDFs)
- **I/O connections:**
  - Input ← `Bank Statement Form`
  - Output → `Transaction Extractor AI`
- **Version notes:** typeVersion `1.1`
- **Potential failures / edge cases:**
  - Wrong password → extraction failure or empty/partial text.
  - Scanned PDFs (images) won’t yield meaningful text unless OCR is involved (this node does not OCR).
  - Output text may include line breaks/spacing that confuses extraction.

#### Sticky Note: Workflow Overview (covers overall workflow)
Content (applies broadly, repeated later per-node in the table where relevant):
- Describes end-to-end logic, setup steps, required replacements (`company_id`, account IDs, credentials).

#### Sticky Note: PDF Extraction
- Documents that form trigger + extraction + AI parsing yields structured transactions.

---

### Block 2 — AI Transaction Extraction (OpenRouter + Structured Output)
**Overview:** Sends extracted statement text to an AI agent, forces strict JSON-array output, and validates/structures it with a schema parser.  
**Nodes involved:** `Transaction Extractor AI`, `OpenRouter Chat Model`, `JSON Output Parser`

#### Node: OpenRouter Chat Model
- **Type / role:** `lmChatOpenRouter` — provides the chat LLM via OpenRouter.
- **Key configuration:**
  - Model: `openai/gpt-oss-20b:free`
- **I/O connections:**
  - AI language model output → `Transaction Extractor AI` (via `ai_languageModel`)
- **Potential failures / edge cases:**
  - OpenRouter API key missing/invalid, quota exceeded, model unavailable.
  - “free” models may be rate-limited or less reliable for strict JSON formatting.

#### Node: JSON Output Parser
- **Type / role:** `outputParserStructured` — validates LLM output against a manual JSON Schema.
- **Key configuration:**
  - Schema: **array of objects** requiring:
    - `transactionType`: `"credit"` or `"debit"`
    - `date`: string matching `YYYY-MM-DD`
    - `description`: string
    - `amount`: number
    - `referenceNumber`: string (empty if missing)
    - `payee`: string (empty if missing)
- **I/O connections:**
  - Output parser → `Transaction Extractor AI` (via `ai_outputParser`)
- **Potential failures / edge cases:**
  - If the model returns anything except a pure JSON array, parsing fails.
  - Date formatting errors (not matching regex) will fail parsing.
  - Amount returned as string instead of number will fail schema validation.

#### Node: Transaction Extractor AI
- **Type / role:** `LangChain Agent` — orchestrates prompt + model + parser to extract structured transactions.
- **Key configuration:**
  - Prompt instructs:
    - Extract all transactions from `$json.text` (from PDF extraction)
    - Classify as credit/debit
    - Output **only** JSON array, no wrapper object, no explanations
    - Missing fields must be `""` not `null`
  - System message enforces precision and strict schema output.
  - `hasOutputParser: true` (wired to `JSON Output Parser`)
  - **Error handling:** `onError: continueRegularOutput` (workflow continues even if node errors; may produce unexpected downstream state)
- **I/O connections:**
  - Input ← `Extract PDF Text`
  - Model ← `OpenRouter Chat Model`
  - Parser ← `JSON Output Parser`
  - Output → `Split Transactions`
- **Version notes:** typeVersion `1.7`
- **Potential failures / edge cases:**
  - Token limits: large statements may truncate.
  - Ambiguous credit/debit signs: statement formats differ (CR/DR, +/-); model may misclassify.
  - Duplicate transactions in statement could be extracted twice; workflow has no explicit dedupe step before posting to QBO.

---

### Block 3 — Transaction Iteration & Routing
**Overview:** Converts the AI-produced array into individual transaction items, iterates them in batches, and routes each to credit or debit processing.  
**Nodes involved:** `Split Transactions`, `Loop Over Items`, `Credit or Debit?`, `Transaction Processing` (sticky)

#### Node: Split Transactions
- **Type / role:** `Split Out` — expands an array field into separate n8n items.
- **Key configuration:**
  - Field to split out: `output`
    - Assumes the agent result is stored under `$json.output` (typical for these nodes).
- **I/O connections:**
  - Input ← `Transaction Extractor AI`
  - Output → `Loop Over Items`
- **Potential failures / edge cases:**
  - If the agent output key changes (e.g., not `output`), this node produces no items.
  - Empty array → nothing to process.

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — controls looping/iteration over items.
- **Key configuration:** defaults (batch size not explicitly set; n8n default is typically 1 unless configured).
- **I/O connections:**
  - Input ← `Split Transactions`
  - Output (index 1 in connections) → `Credit or Debit?`
  - It also receives “continue” signals back from posting nodes to proceed to next batch.
- **Potential failures / edge cases:**
  - If downstream nodes don’t connect back to this node correctly, the loop may stop early.
  - If any branch errors without continuing, the batch loop can stall.

#### Node: Credit or Debit?
- **Type / role:** `Switch` — routes each transaction item by `transactionType`.
- **Key configuration:**
  - Rule “Credit”: `$json.transactionType === "credit"`
  - Rule “Debit”: `$json.transactionType === "debit"`
  - Outputs are renamed “Credit” and “Debit”.
- **I/O connections:**
  - Credit output → `Get All QB Items` (credit path)
  - Debit output → `Search Categories` (debit path)
- **Potential failures / edge cases:**
  - If `transactionType` is missing/empty, item won’t match any route (dropped).
  - Case sensitivity: schema enforces lowercase; if LLM violates, routing fails.

#### Sticky Note: Transaction Processing
- Documents split + loop + routing behavior.

---

### Block 4 — Credit Path (Income → SalesReceipt)
**Overview:** For each credit transaction, the workflow ensures a relevant QBO Item exists (create if missing), ensures a Customer exists (create if missing), then posts a SalesReceipt via HTTP.  
**Nodes involved:** `Get All QB Items`, `Check Which Items to Create`, `Need to Create Items?`, `Create Items`, `Collect All Item Mappings`, `Get many customers`, `Customers Exists?1`, `Create a Customer`, `Build Salesreceipt Payload`, `Create QuickBooks SalesReceipt`, `Credit Path` (sticky)

#### Node: Get All QB Items
- **Type / role:** `QuickBooks` — retrieves all existing QBO Items.
- **Key configuration:**
  - Resource: `item`
  - Operation: `getAll`
  - Return all: `true`
- **I/O connections:**
  - Input ← `Credit or Debit?` (Credit)
  - Output → `Check Which Items to Create`
- **Potential failures / edge cases:**
  - OAuth credential invalid/expired.
  - Large item catalogs may be slow; pagination handled by node, but may time out in large orgs.

#### Node: Check Which Items to Create
- **Type / role:** `Code` — decides whether to reuse an existing QBO Item or create a new Service item.
- **Key configuration (logic):**
  - Reads current transaction from `$('Loop Over Items').first().json` (important: uses **first** item of the batch context).
  - Normalizes names: lowercase, non-alphanumerics replaced with spaces.
  - Shortens payee name by removing terms like “services”, “limited/ltd/pvt/private”.
  - Creates:
    - `desiredItemName` (normalized)
    - `existingItemMapping` if match found in QBO items
    - `itemsToCreate` if not found (Service item) with fixed refs:
      - `incomeAccountRef: '79'`
      - `expenseAccountRef: '80'`
    - `needsCreation` boolean
- **I/O connections:**
  - Input ← `Get All QB Items` (many items)
  - Output → `Need to Create Items?`
- **Potential failures / edge cases:**
  - If transaction payee/description is empty, desired name may normalize to empty → mismatches and odd item names.
  - Hard-coded account IDs (79/80) must exist in the target QBO company.
  - Uses `.first()` which is safe if batch size is 1; if batch size > 1, this can mis-associate transactions.

#### Node: Need to Create Items?
- **Type / role:** `IF` — branches based on `needsCreation === true`.
- **Key configuration:**
  - Condition: boolean equals `true` on `$json.needsCreation`
- **I/O connections:**
  - True → `Create Items`
  - False → `Collect All Item Mappings`
- **Potential failures / edge cases:**
  - If `needsCreation` missing (code failure), branch may default false and later mapping may fail.

#### Node: Create Items
- **Type / role:** `HTTP Request` — creates a QBO Item via REST API.
- **Key configuration:**
  - POST `https://sandbox-quickbooks.api.intuit.com/v3/company/company_id/item?minorversion=75`
  - Body uses the first element: `$json.itemsToCreate[0]`
  - Auth: `QuickBooks OAuth2` predefined credential
  - Content-Type JSON
- **I/O connections:**
  - Input ← `Need to Create Items?` (true)
  - Output → `Collect All Item Mappings`
- **Potential failures / edge cases:**
  - `company_id` placeholder must be replaced.
  - Name collisions: QBO item names must be unique; creation may fail.
  - income/expense account refs invalid → 400 error.

#### Node: Collect All Item Mappings
- **Type / role:** `Code` — produces a unified `itemRef` either from the created item or from existing mapping.
- **Key configuration (logic):**
  - Reads `decision` from `Check Which Items to Create`.
  - If `needsCreation`: reads created item from `Create Items` response (`created.Item || created`).
  - Else: uses `existingItemMapping[desiredItemName]`.
  - Throws hard error if mapping missing.
- **I/O connections:**
  - Input ← from either `Create Items` or `Need to Create Items?` (false path)
  - Output → `Get many customers`
- **Potential failures / edge cases:**
  - If QBO API response structure differs, `created.Item` may not exist.
  - If `desiredItemName` normalization mismatches keys, mapping lookup fails.

#### Node: Get many customers
- **Type / role:** `QuickBooks` — searches customers.
- **Key configuration:**
  - Operation: `getAll` (Customer is the default resource for this node instance as configured)
  - Filter query: `WHERE DisplayName = '{{ $json.itemRef.name }}'`
    - Note: this uses the **Item name** as the Customer DisplayName.
- **I/O connections:**
  - Input ← `Collect All Item Mappings`
  - Output → `Customers Exists?1`
- **Potential failures / edge cases:**
  - Using Item name as Customer name may be undesirable (customers ≠ products/services).
  - If multiple customers share the same display name, QBO may return multiple results; downstream uses `.first()` patterns elsewhere.

#### Node: Customers Exists?1
- **Type / role:** `IF` — checks whether a customer was found.
- **Key configuration:**
  - Condition checks if `$json.Id` is empty.
  - True branch (Id empty) → create customer.
  - False branch → proceed to build receipt.
- **I/O connections:**
  - True → `Create a Customer`
  - False → `Build Salesreceipt Payload`
- **Potential failures / edge cases:**
  - If the “Get many customers” node returns an array/list behavior, `$json.Id` may not exist in this shape. (The node configuration suggests it outputs items per customer, but results vary with node/operation.)

#### Node: Create a Customer
- **Type / role:** `QuickBooks` — creates a new Customer.
- **Key configuration:**
  - Operation: `create`
  - DisplayName and CompanyName set to `Collect All Item Mappings.item.json.itemRef.name`
- **I/O connections:**
  - Input ← `Customers Exists?1` (true)
  - Output → `Build Salesreceipt Payload`
- **Potential failures / edge cases:**
  - Customer name uniqueness constraints may apply in QBO.
  - If itemRef name contains invalid characters, QBO may reject.

#### Node: Build Salesreceipt Payload
- **Type / role:** `Code` — builds the SalesReceipt payload and resolves CustomerRef.
- **Key configuration (logic):**
  - Transaction: `$('Loop Over Items').first().json`
  - ItemRef: `$('Collect All Item Mappings').first().json.itemRef`
  - Customer resolution:
    - Try `$('Get many customers').first().json`
    - Else fallback to `$('Create a Customer').first().json`
    - Throws error if no customer Id.
  - Validates amount is numeric and non-zero; validates itemRef exists.
  - Sets:
    - `DepositToAccountRef.value = '35'`
    - `TaxCodeRef.value = 'NON'`
    - `TotalAmt = amount`
    - `PrivateNote = description + ref`
- **I/O connections:**
  - Input ← from either `Create a Customer` or `Customers Exists?1` false branch
  - Output → `Create QuickBooks SalesReceipt`
- **Potential failures / edge cases:**
  - Hard-coded account `35` must exist and be the intended deposit account.
  - If amount is very small or 0, it throws.
  - Uses `.first()`; if loop batches > 1, may mismatch again.

#### Node: Create QuickBooks SalesReceipt
- **Type / role:** `HTTP Request` — posts SalesReceipt to QBO REST API (sandbox).
- **Key configuration:**
  - POST `https://sandbox-quickbooks.api.intuit.com/v3/company/company_id/salesreceipt?minorversion=75`
  - Body: `={{ JSON.stringify($json) }}`
  - Auth: `quickBooksOAuth2Api` credential
- **I/O connections:**
  - Input ← `Build Salesreceipt Payload`
  - Output → `Loop Over Items` (to continue the batch loop)
- **Potential failures / edge cases:**
  - `company_id` placeholder must be replaced.
  - QBO validation errors: missing refs, invalid account IDs, tax code, etc.
  - If HTTP node returns an error and isn’t configured to continue, loop may stop.

#### Sticky Note: Credit Path (Income)
- Documents item/customer creation and SalesReceipt posting.

---

### Block 5 — Debit Path (Expenses → Purchase)
**Overview:** For each debit transaction, the workflow pulls the chart of accounts, ensures a Vendor exists for the payee, auto-selects an Expense category by keyword matching, builds a Purchase payload, and posts it to QBO.  
**Nodes involved:** `Search Categories`, `Find Vendor`, `Vendor Exists?`, `Create Vendor`, `Build Expense Payload`, `Create QuickBooks Expense`, `Debit Path` (sticky)

#### Node: Search Categories
- **Type / role:** `HTTP Request` — queries QBO chart of accounts.
- **Key configuration:**
  - GET/Query request to: `https://sandbox-quickbooks.api.intuit.com/v3/company/company_id/query?minorversion=75`
  - Query param:
    - `SELECT Id, Name, AccountType, AccountSubType FROM Account`
  - Auth: `quickBooksOAuth2Api`
- **I/O connections:**
  - Input ← `Credit or Debit?` (Debit)
  - Output → `Find Vendor`
- **Potential failures / edge cases:**
  - `company_id` placeholder must be replaced.
  - Large COA may be slow; response parsing assumes `QueryResponse.Account`.

#### Node: Find Vendor
- **Type / role:** `QuickBooks` — searches vendors by display name.
- **Key configuration:**
  - Resource: `vendor`
  - Operation: `getAll`
  - Filter: `WHERE DisplayName = '{{ payee }}'` where payee comes from `$('Loop Over Items').item.json.payee`
- **I/O connections:**
  - Input ← `Search Categories`
  - Output → `Vendor Exists?`
- **Potential failures / edge cases:**
  - Payee empty → query becomes `DisplayName = ''`, likely no result.
  - Vendors with same name or partial matches aren’t handled (exact match only).

#### Node: Vendor Exists?
- **Type / role:** `IF` — checks if vendor was found.
- **Key configuration:**
  - Condition: `$json.Id` is empty
  - True branch → create vendor
  - False branch → proceed with existing vendor
- **I/O connections:**
  - True → `Create Vendor`
  - False → `Build Expense Payload`
- **Potential failures / edge cases:**
  - If `Find Vendor` returns multiple vendor items, the condition may behave unexpectedly depending on which item is passed.

#### Node: Create Vendor
- **Type / role:** `QuickBooks` — creates a vendor using the transaction payee.
- **Key configuration:**
  - Resource: `vendor`
  - Operation: `create`
  - DisplayName and CompanyName: `$('Loop Over Items').item.json.payee`
- **I/O connections:**
  - Input ← `Vendor Exists?` (true)
  - Output → `Build Expense Payload`
- **Potential failures / edge cases:**
  - Payee missing/empty → QBO may reject vendor creation.
  - Vendor name uniqueness constraints may apply.

#### Node: Build Expense Payload
- **Type / role:** `Code` — builds a QBO Purchase payload and auto-maps to an expense category.
- **Key configuration (logic):**
  - Transaction: `$('Loop Over Items').item.json`
  - Vendor: uses `$input.first().json` (expects either the found or created vendor object)
  - Loads categories from `Search Categories`:
    - `QueryResponse.Account` filtered to `AccountType === 'Expense'`
  - Keyword mapping (based on lowercased description):
    - if contains `upi`/`bank`/`charge` → “bank charges”
    - contains `insurance` → “insurance expense”
    - contains `ads`/`advertising` → “advertising”
    - contains `vendor`/`service` → “professional fees”
    - else fallback → “services”
  - Throws if no categories or no matching category found.
  - Builds payload fields:
    - `EntityRef` vendor
    - `PaymentType: 'Cash'`
    - `Line[0].AccountBasedExpenseLineDetail.AccountRef` = selected expense account Id/Name
    - `TaxCodeRef: 'NON'`
- **I/O connections:**
  - Input ← from either `Create Vendor` or `Vendor Exists?` false branch
  - Output → `Create QuickBooks Expense`
- **Potential failures / edge cases:**
  - Category names must exactly match (`bank charges`, `insurance expense`, etc.) in QBO; otherwise mapping fails.
  - `PaymentType: Cash` may not reflect real payment method; adjust as needed.
  - Relies on `Search Categories` node data being present in execution (`$items('Search Categories')...`); if node didn’t run (e.g., rerun partial execution), it fails.

#### Node: Create QuickBooks Expense
- **Type / role:** `HTTP Request` — posts a Purchase (expense) to QBO.
- **Key configuration:**
  - POST `https://sandbox-quickbooks.api.intuit.com/v3/company/company_id/purchase?minorversion=75`
  - Body is templated from the built payload (TxnDate, PrivateNote, EntityRef, PaymentType, Line, etc.)
  - Bank AccountRef is hard-coded to:
    - `value: "35"`, `name: "Checking"`
- **I/O connections:**
  - Input ← `Build Expense Payload`
  - Output → `Loop Over Items` (continue loop)
- **Potential failures / edge cases:**
  - `company_id` placeholder must be replaced.
  - AccountRef `35` must exist and correspond to the correct bank account.
  - QBO validation errors if vendor/entity refs invalid.

#### Sticky Note: Debit Path (Expenses)
- Documents COA loading, vendor creation, keyword mapping, and posting purchases.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / overview |  |  | ## How it works… (includes setup steps: connect QBO, OpenRouter creds, PDF password, replace `company_id`, verify account IDs 35/79/80, test sample) |
| Bank Statement Form | Form Trigger | Entry point (PDF upload form) |  | Extract PDF Text | ## PDF Extraction: Form trigger receives bank statement PDF… |
| Extract PDF Text | Extract From File | Extract text from uploaded PDF | Bank Statement Form | Transaction Extractor AI | ## PDF Extraction: Form trigger receives bank statement PDF… |
| Transaction Extractor AI | LangChain Agent | Extract/normalize transactions into strict JSON array | Extract PDF Text | Split Transactions | ## PDF Extraction: …sent to AI agent… identify all transactions… |
| OpenRouter Chat Model | OpenRouter Chat Model | LLM provider for extraction |  | Transaction Extractor AI | ## PDF Extraction: …sent to AI agent… |
| JSON Output Parser | Structured Output Parser | Enforce JSON schema for extracted transactions |  | Transaction Extractor AI | ## PDF Extraction: …structured output parser… |
| Split Transactions | Split Out | Split AI array into individual transaction items | Transaction Extractor AI | Loop Over Items | ## Transaction Processing: Split extracted transactions… |
| Loop Over Items | Split In Batches | Iterate transactions and control looping | Split Transactions; Create QuickBooks SalesReceipt; Create QuickBooks Expense | Credit or Debit? | ## Transaction Processing: Split… Loop processes each one… |
| Credit or Debit? | Switch | Route by transaction type | Loop Over Items | Get All QB Items; Search Categories | ## Transaction Processing: …routing credits and debits… |
| Transaction Processing | Sticky Note | Documentation |  |  | ## Transaction Processing… |
| Credit Path | Sticky Note | Documentation |  |  | ## Credit Path (Income)… |
| Get All QB Items | QuickBooks | Fetch all Items for matching/creation | Credit or Debit? | Check Which Items to Create | ## Credit Path (Income)… |
| Check Which Items to Create | Code | Decide reuse/create Item; build mapping | Get All QB Items | Need to Create Items? | ## Credit Path (Income)… |
| Need to Create Items? | IF | Branch on item creation necessity | Check Which Items to Create | Create Items; Collect All Item Mappings | ## Credit Path (Income)… |
| Create Items | HTTP Request | Create QBO Item via REST | Need to Create Items? | Collect All Item Mappings | ## Credit Path (Income)… |
| Collect All Item Mappings | Code | Produce final `itemRef` for receipt | Create Items; Need to Create Items? | Get many customers | ## Credit Path (Income)… |
| Get many customers | QuickBooks | Find customer by DisplayName | Collect All Item Mappings | Customers Exists?1 | ## Credit Path (Income)… |
| Customers Exists?1 | IF | Branch on customer found | Get many customers | Create a Customer; Build Salesreceipt Payload | ## Credit Path (Income)… |
| Create a Customer | QuickBooks | Create missing customer | Customers Exists?1 | Build Salesreceipt Payload | ## Credit Path (Income)… |
| Build Salesreceipt Payload | Code | Build SalesReceipt payload; resolve CustomerRef | Customers Exists?1; Create a Customer | Create QuickBooks SalesReceipt | ## Credit Path (Income)… |
| Create QuickBooks SalesReceipt | HTTP Request | Post SalesReceipt to QBO | Build Salesreceipt Payload | Loop Over Items | ## Credit Path (Income)… |
| Debit Path | Sticky Note | Documentation |  |  | ## Debit Path (Expenses)… |
| Search Categories | HTTP Request | Query Chart of Accounts | Credit or Debit? | Find Vendor | ## Debit Path (Expenses)… |
| Find Vendor | QuickBooks | Lookup vendor by payee | Search Categories | Vendor Exists? | ## Debit Path (Expenses)… |
| Vendor Exists? | IF | Branch on vendor found | Find Vendor | Create Vendor; Build Expense Payload | ## Debit Path (Expenses)… |
| Create Vendor | QuickBooks | Create missing vendor | Vendor Exists? | Build Expense Payload | ## Debit Path (Expenses)… |
| Build Expense Payload | Code | Choose expense category + build Purchase payload | Vendor Exists?; Create Vendor | Create QuickBooks Expense | ## Debit Path (Expenses)… |
| Create QuickBooks Expense | HTTP Request | Post Purchase to QBO | Build Expense Payload | Loop Over Items | ## Debit Path (Expenses)… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Form Trigger** node:
   - Name: `Bank Statement Form`
   - Form title: “QuickBooks Bank Statement Upload”
   - Description: “Upload your bank statement PDF to automatically create entries in QuickBooks”
   - Add a **File** field:
     - Label: “Bank Statement PDF”
     - Required: true
     - Accept: `.pdf`
   - Disable attribution if desired.

3. **Add “Extract From File”** node:
   - Name: `Extract PDF Text`
   - Operation: `PDF`
   - Binary property name: `Bank_Statement_PDF` (must match the form upload output)
   - If PDFs are password protected, set **Options → Password** to your PDF password.
   - Connect: `Bank Statement Form` → `Extract PDF Text`

4. **Add OpenRouter model node**:
   - Name: `OpenRouter Chat Model`
   - Choose model: `openai/gpt-oss-20b:free` (or another OpenRouter-supported model)
   - Configure **OpenRouter credentials** (API key) in n8n Credentials.
   - No direct “main” connection required; it will connect via AI port.

5. **Add Structured Output Parser** node:
   - Name: `JSON Output Parser`
   - Schema type: Manual
   - Use a JSON schema defining an **array** of objects with fields:
     - `transactionType` enum: `credit|debit`
     - `date` pattern `^\d{4}-\d{2}-\d{2}$`
     - `description` string
     - `amount` number
     - `referenceNumber` string
     - `payee` string
   - This node connects via the agent’s output parser port.

6. **Add LangChain Agent** node:
   - Name: `Transaction Extractor AI`
   - Prompt type: “Define”
   - Paste instructions to:
     - Extract transactions from the provided statement text
     - Classify `credit` vs `debit`
     - Output ONLY a JSON array matching the schema
     - Use empty strings for missing fields
   - Add a system message enforcing strict schema output.
   - Enable/attach output parser.
   - Set error handling to continue if you want partial runs (`On Error: Continue`)—optional.
   - Connect:
     - `Extract PDF Text` → `Transaction Extractor AI` (main)
     - `OpenRouter Chat Model` → `Transaction Extractor AI` (AI language model connection)
     - `JSON Output Parser` → `Transaction Extractor AI` (AI output parser connection)

7. **Add Split Out** node:
   - Name: `Split Transactions`
   - Field to split out: `output`
   - Connect: `Transaction Extractor AI` → `Split Transactions`

8. **Add Split In Batches** node:
   - Name: `Loop Over Items`
   - Batch size: typically 1
   - Connect: `Split Transactions` → `Loop Over Items`

9. **Add Switch** node:
   - Name: `Credit or Debit?`
   - Create two rules:
     - Credit: `={{ $json.transactionType }}` equals `credit`
     - Debit: `={{ $json.transactionType }}` equals `debit`
   - Connect: `Loop Over Items` → `Credit or Debit?`

---

### Credit path build-out (SalesReceipt)

10. **Add QuickBooks node** for items:
   - Name: `Get All QB Items`
   - Resource: `Item`
   - Operation: `Get All`
   - Return all: true
   - Configure **QuickBooks OAuth2 credentials** (sandbox or production).
   - Connect: `Credit or Debit? (Credit)` → `Get All QB Items`

11. **Add Code node**:
   - Name: `Check Which Items to Create`
   - Implement logic:
     - Normalize and shorten transaction payee/description
     - Compare with QBO Items list
     - Output `{ desiredItemName, itemsToCreate[], existingItemMapping, needsCreation }`
     - Hardcode or parameterize income/expense account refs (79/80 in the provided workflow)
   - Connect: `Get All QB Items` → `Check Which Items to Create`

12. **Add IF node**:
   - Name: `Need to Create Items?`
   - Condition: `={{ $json.needsCreation }}` equals `true`
   - Connect: `Check Which Items to Create` → `Need to Create Items?`

13. **Add HTTP Request node** to create items:
   - Name: `Create Items`
   - Method: POST
   - URL: `https://sandbox-quickbooks.api.intuit.com/v3/company/<YOUR_COMPANY_ID>/item?minorversion=75`
   - Auth: QuickBooks OAuth2 credential
   - JSON body: create Item from `$json.itemsToCreate[0]`
   - Connect: `Need to Create Items? (true)` → `Create Items`

14. **Add Code node**:
   - Name: `Collect All Item Mappings`
   - If item created, return `itemRef` from response; else return `itemRef` from mapping.
   - Connect:
     - `Create Items` → `Collect All Item Mappings`
     - `Need to Create Items? (false)` → `Collect All Item Mappings`

15. **Add QuickBooks node** for customer lookup:
   - Name: `Get many customers`
   - Operation: `Get All` (Customer)
   - Filter query: `WHERE DisplayName = '{{ $json.itemRef.name }}'`
   - Connect: `Collect All Item Mappings` → `Get many customers`

16. **Add IF node**:
   - Name: `Customers Exists?1`
   - Condition: treat empty `$json.Id` as “not found”
   - Connect: `Get many customers` → `Customers Exists?1`

17. **Add QuickBooks node** for customer creation:
   - Name: `Create a Customer`
   - Operation: `Create`
   - DisplayName/CompanyName: `Collect All Item Mappings.itemRef.name`
   - Connect: `Customers Exists?1 (true)` → `Create a Customer`

18. **Add Code node**:
   - Name: `Build Salesreceipt Payload`
   - Build SalesReceipt payload using:
     - Transaction from current loop item
     - ItemRef from `Collect All Item Mappings`
     - CustomerRef from either found or created customer
     - DepositToAccountRef = `35` (or your bank account id)
   - Connect:
     - `Customers Exists?1 (false)` → `Build Salesreceipt Payload`
     - `Create a Customer` → `Build Salesreceipt Payload`

19. **Add HTTP Request node** to post SalesReceipt:
   - Name: `Create QuickBooks SalesReceipt`
   - POST `https://sandbox-quickbooks.api.intuit.com/v3/company/<YOUR_COMPANY_ID>/salesreceipt?minorversion=75`
   - Auth: QuickBooks OAuth2 credential
   - Body: the JSON from `Build Salesreceipt Payload`
   - Connect: `Build Salesreceipt Payload` → `Create QuickBooks SalesReceipt`

20. **Close the loop**:
   - Connect: `Create QuickBooks SalesReceipt` → `Loop Over Items` (so it processes the next transaction)

---

### Debit path build-out (Purchase)

21. **Add HTTP Request** to query accounts:
   - Name: `Search Categories`
   - GET (or request with query parameter)
   - URL: `https://sandbox-quickbooks.api.intuit.com/v3/company/<YOUR_COMPANY_ID>/query?minorversion=75`
   - Query parameter: `query = SELECT Id, Name, AccountType, AccountSubType FROM Account`
   - Auth: QuickBooks OAuth2 credential
   - Connect: `Credit or Debit? (Debit)` → `Search Categories`

22. **Add QuickBooks node** to find vendor:
   - Name: `Find Vendor`
   - Resource: `Vendor`
   - Operation: `Get All`
   - Filter: `WHERE DisplayName = '{{ payee }}'` using the current loop item payee.
   - Connect: `Search Categories` → `Find Vendor`

23. **Add IF node**:
   - Name: `Vendor Exists?`
   - Condition: `$json.Id` is empty → vendor missing
   - Connect: `Find Vendor` → `Vendor Exists?`

24. **Add QuickBooks node** to create vendor:
   - Name: `Create Vendor`
   - Resource: Vendor
   - Operation: Create
   - DisplayName/CompanyName from transaction payee
   - Connect: `Vendor Exists? (true)` → `Create Vendor`

25. **Add Code node**:
   - Name: `Build Expense Payload`
   - Inputs:
     - Vendor object from either found or created path (use `$input.first()`)
     - Categories list from `Search Categories` execution data
     - Transaction from `Loop Over Items`
   - Implement keyword→category mapping and fallback.
   - Connect:
     - `Vendor Exists? (false)` → `Build Expense Payload`
     - `Create Vendor` → `Build Expense Payload`

26. **Add HTTP Request** to create Purchase:
   - Name: `Create QuickBooks Expense`
   - POST `https://sandbox-quickbooks.api.intuit.com/v3/company/<YOUR_COMPANY_ID>/purchase?minorversion=75`
   - Auth: QuickBooks OAuth2 credential
   - Use JSON body from `Build Expense Payload` (or template it as in the workflow)
   - Ensure bank AccountRef is correct (`35` used in the workflow).
   - Connect: `Build Expense Payload` → `Create QuickBooks Expense`

27. **Close the loop**:
   - Connect: `Create QuickBooks Expense` → `Loop Over Items`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace `company_id` in all QuickBooks HTTP Request URLs with your real QuickBooks Company ID. | Applies to: `Create Items`, `Create QuickBooks SalesReceipt`, `Search Categories`, `Create QuickBooks Expense` |
| Verify hard-coded account IDs: `35` (Checking/bank), `79` (Income), `80` (Expense). | These must match your QBO chart of accounts. |
| PDF password must be set in `Extract PDF Text` if statements are protected. | Otherwise extraction fails or returns empty text. |
| AI output must remain strict JSON array; schema validation is enforced. | Parser will fail if model returns commentary/non-JSON. |
| Workflow auto-creates missing entities (Items, Customers, Vendors) which can clutter QBO if payee naming is noisy. | Consider adding dedupe rules, name sanitation, or approval steps. |