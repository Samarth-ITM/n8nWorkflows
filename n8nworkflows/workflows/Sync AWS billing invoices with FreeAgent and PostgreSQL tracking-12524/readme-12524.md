Sync AWS billing invoices with FreeAgent and PostgreSQL tracking

https://n8nworkflows.xyz/workflows/sync-aws-billing-invoices-with-freeagent-and-postgresql-tracking-12524


# Sync AWS billing invoices with FreeAgent and PostgreSQL tracking

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sync AWS billing invoices with FreeAgent and PostgreSQL tracking  
**Workflow name (JSON):** Sync AWS Billing with Free Agent Invoice  
**Purpose:** Each month (on the 3rd and 4th), fetch AWS invoice summaries for the previous billing month using AWS Signature V4, filter to last month’s invoices, deduplicate against a PostgreSQL table, create corresponding supplier bills in FreeAgent, mark them as paid, and log the processed invoice IDs to prevent duplicates.

### 1.1 Scheduling & Billing Period Calculation
Determines the prior month’s billing period and triggers the workflow on a schedule.

### 1.2 AWS Invoicing API Retrieval (SigV4)
Builds a signed request to AWS Invoicing API and retrieves invoice summaries.

### 1.3 Parsing & Date Filtering
Extracts invoice summaries from the AWS response and filters by a cutoff date (first day of the target billing month).

### 1.4 Deduplication Loop (PostgreSQL)
Iterates invoices and checks whether each invoice was already processed.

### 1.5 FreeAgent Bill Creation, Payment, and Tracking
Creates a FreeAgent bill for each new AWS invoice, marks it paid, then records the mapping in PostgreSQL.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Billing Period Calculation

**Overview:** Runs on a monthly schedule and prepares parameters needed for AWS query and filtering (account ID, billing month/year, cutoff date).

**Nodes involved:**
- Trigger Monthly
- Calculate Last Month Date Range

#### Node: Trigger Monthly
- **Type / role:** `Schedule Trigger` — starts workflow automatically.
- **Config choices:** Cron expression `30 0 3,4 * *` (00:30 on the 3rd and 4th of each month).
- **Outputs:** Connects to **Calculate Last Month Date Range**.
- **Edge cases / failures:**
  - Timezone impacts: workflow timezone is **Europe/London**; ensure the schedule matches your desired local time.
  - Double-run: runs two days (3rd and 4th) to increase likelihood invoices are available; deduplication prevents duplicates.

#### Node: Calculate Last Month Date Range
- **Type / role:** `Code` — computes previous month parameters.
- **Config choices (interpreted):**
  - Calculates `lastMonth = first day of previous month`
  - Sets:
    - `billing_year`
    - `billing_month` (1–12)
    - `cutoff_date` as `YYYY-MM-01`
    - `account_id` placeholder `[YOUR_AWS_ACCOUNT_ID]`
- **Key variables/expressions:** Produces JSON used downstream by signature generation and filtering.
- **Outputs:** Connects to **Generate AWS Signature**.
- **Edge cases / failures:**
  - Placeholder account ID must be replaced.
  - Date math around January works correctly (JS Date handles year decrement).
- **Version notes:** Code node v2.

---

### Block 2 — AWS Invoicing API Retrieval (SigV4)

**Overview:** Builds an AWS Signature V4 signed request for `Invoicing.ListInvoiceSummaries` and calls the AWS endpoint.

**Nodes involved:**
- Generate AWS Signature
- Call AWS For Bills

#### Node: Generate AWS Signature
- **Type / role:** `Code` — constructs SigV4 headers and body.
- **Config choices (interpreted):**
  - Uses:
    - `region = us-east-1`
    - `service = invoicing`
    - `host = invoicing.us-east-1.api.aws`
    - `x-amz-target = Invoicing.ListInvoiceSummaries`
    - `content-type = application/x-amz-json-1.0`
  - Request body includes:
    - Selector: `{ ResourceType: 'ACCOUNT_ID', Value: accountId }`
    - Filter: `BillingPeriod { Month, Year }`
    - `MaxResults: 10`
  - Creates `Authorization` header using `AWS4-HMAC-SHA256`.
- **Key variables:**
  - Inputs from previous node: `account_id`, `billing_month`, `billing_year`
  - Output fields: `url`, `method`, `headers`, `body`
- **Outputs:** Connects to **Call AWS For Bills**.
- **Edge cases / failures:**
  - **Hardcoded credentials**: access key/secret are inline placeholders; should be moved to n8n credentials/environment variables for security.
  - Permission failure: IAM principal must allow `invoicing:ListInvoiceSummaries`.
  - Clock skew: SigV4 can fail if system time is off.
  - Endpoint/service specifics: AWS Invoicing API availability/host may vary by AWS partition/account; errors will surface in HTTP response.
- **Version notes:** Code node v2; uses Node.js `crypto`.

#### Node: Call AWS For Bills
- **Type / role:** `HTTP Request` — sends signed POST to AWS.
- **Config choices (interpreted):**
  - URL: `{{$json.url}}`
  - Method: POST
  - Sends headers: Content-Type, X-Amz-Date, X-Amz-Target, Authorization (from signature node output)
  - Body: `{{$json.body}}` sent as JSON
  - Response: **Full response enabled** (captures status/headers/body), stored under `json` with a `data` field used later.
- **Inputs:** From **Generate AWS Signature**.
- **Outputs:** To **Parse AWS Response**.
- **Edge cases / failures:**
  - HTTP non-2xx (auth error, invalid signature, throttling).
  - If AWS returns non-JSON or unexpected structure, parsing will fail later.
- **Version notes:** HTTP Request v4.2.

---

### Block 3 — Parsing & Date Filtering

**Overview:** Converts AWS response into items of invoice summaries, then filters invoices to those issued on/after the cutoff date.

**Nodes involved:**
- Parse AWS Response
- Filter By Date

#### Node: Parse AWS Response
- **Type / role:** `Code` — parses returned payload and emits one item per invoice.
- **Config choices (interpreted):**
  - Reads `$input.first().json`
  - Parses `JSON.parse(response.data)`
  - Extracts `data.InvoiceSummaries || []`
  - Returns `[{json: invoice}, ...]`
- **Inputs:** From **Call AWS For Bills** (expects `response.data` to contain JSON string).
- **Outputs:** To **Filter By Date**.
- **Edge cases / failures:**
  - If `response.data` is already an object (not a string), `JSON.parse` will throw.
  - If AWS response has error format, `InvoiceSummaries` may be missing.
- **Version notes:** Code node v2.

#### Node: Filter By Date
- **Type / role:** `Code` — filters invoices based on issued date and cutoff date.
- **Config choices (interpreted):**
  - Cutoff date sourced from: `$node['Calculate Last Month Date Range'].json.cutoff_date`
  - Interprets AWS `IssuedDate` as Unix seconds; converts via `new Date(item.json.IssuedDate * 1000)`
  - Keeps invoices where `issuedDate >= cutoffDate`
  - If none match, returns a single item: `{ message: 'No invoices after cutoff date' }`
- **Inputs:** From **Parse AWS Response**.
- **Outputs:** To **Loop Over Invoices**.
- **Edge cases / failures:**
  - If an invoice item lacks `IssuedDate`, date conversion yields invalid date and filter may behave unexpectedly (invalid date comparisons become `false`).
  - The “no invoices” message item will still be passed to the loop; downstream nodes expect fields like `InvoiceId` and may fail unless the loop is guarded (it is not).
  - Cutoff is built as UTC midnight; ensure alignment with AWS timestamps.
- **Version notes:** Code node v2.

---

### Block 4 — Deduplication Loop (PostgreSQL)

**Overview:** Iterates through invoices and checks a tracking table to skip invoices already processed.

**Nodes involved:**
- Loop Over Invoices
- Check If Already Processed
- If New Invoice

#### Node: Loop Over Invoices
- **Type / role:** `Split In Batches` — iterates items, enabling per-invoice processing.
- **Config choices:** Default batch settings (no explicit batch size shown; n8n default typically 1).
- **Connections (important):**
  - **Output 1 (main, index 0):** unused
  - **Output 2 (main, index 1):** to **Check If Already Processed**
  - After recording, **Record In PostgreSQL** connects back to **Loop Over Invoices** to continue iteration.
- **Edge cases / failures:**
  - If the input item is the “No invoices…” message, it will still be iterated and likely break the SQL query (missing `InvoiceId`).

#### Node: Check If Already Processed
- **Type / role:** `Postgres` — executes a SELECT count query to detect duplicates.
- **Config choices (interpreted):**
  - Operation: Execute Query
  - SQL:
    ```sql
    SELECT COUNT(*) as count
    FROM aws_invoices_processed
    WHERE aws_invoice_id = '{{ $json.InvoiceId }}';
    ```
  - Uses Postgres credentials “Postgres account”.
- **Inputs:** From **Loop Over Invoices** (each invoice item).
- **Outputs:** To **If New Invoice**.
- **Edge cases / failures:**
  - SQL injection is low-risk here but still string interpolation; prefer parameterized queries if possible.
  - `COUNT(*)` returned type may be string depending on driver; workflow uses `.toNumber()` later to normalize.
  - Table missing or wrong schema breaks the workflow.

#### Node: If New Invoice
- **Type / role:** `IF` — branches only if invoice has not been processed.
- **Config choices (interpreted):**
  - Condition: `{{$json.count.toNumber()}} == 0`
- **Inputs:** From **Check If Already Processed**.
- **Outputs:**
  - **True branch:** to **Prepare FreeAgent Bill**
  - (False branch not connected): invoices already processed are effectively dropped.
- **Edge cases / failures:**
  - If `count` is missing/non-numeric, `.toNumber()` can error.

---

### Block 5 — FreeAgent Bill Creation, Payment, and Tracking

**Overview:** For each new AWS invoice, creates a supplier bill in FreeAgent, marks it as paid, then records it in PostgreSQL.

**Nodes involved:**
- Prepare FreeAgent Bill
- Create FreeAgent Bill
- Prepare Payment
- Mark Bill as Paid
- Record In PostgreSQL

#### Node: Prepare FreeAgent Bill
- **Type / role:** `Code` — maps AWS invoice to FreeAgent bill payload.
- **Config choices (interpreted):**
  - Reads current invoice from: `$('Loop Over Invoices').item.json` (paired item access)
  - Builds `bill` object with:
    - `contact`: `https://api.freeagent.com/v2/contacts/[YOUR_FREEAGENT_CONTACT_ID]`
    - `reference`: `InvoiceId`
    - Uses **DueDate as dated_on** and **IssuedDate as due_on** (comment: “AWS has the dates backwards”)
    - `total_value`: `PaymentCurrencyAmount.TotalAmount`
    - `bill_items[0]`:
      - `category`: `https://api.freeagent.com/v2/categories/[YOUR_FREEAGENT_CATEGORY_ID]`
      - description includes billing period `YYYY-MM`
      - `sales_tax_rate: '20.0'`
- **Key logic:** `formatDate(timestamp)` validates timestamps and formats `YYYY-MM-DD`.
- **Outputs:** `{ json: { bill } }` to **Create FreeAgent Bill**.
- **Edge cases / failures:**
  - If AWS timestamps are missing/invalid, this node throws and stops execution for that item.
  - VAT/tax handling is hardcoded to 20.0; may be incorrect for your tax setup.
  - FreeAgent expects correct contact/category URLs; placeholders must be replaced.
- **Version notes:** Code node v2.

#### Node: Create FreeAgent Bill
- **Type / role:** `HTTP Request` — creates bill in FreeAgent.
- **Config choices (interpreted):**
  - POST `https://api.freeagent.com/v2/bills`
  - Auth: OAuth2 (generic credential type) “FreeAgent OAuth2”
  - Body: JSON stringify of the incoming object (contains `{ bill: ... }`)
- **Inputs:** From **Prepare FreeAgent Bill**.
- **Outputs:** To **Prepare Payment** (expects response to contain `bill.url`, `bill.total_value`, `bill.dated_on`).
- **Edge cases / failures:**
  - OAuth token expiry/refresh issues.
  - FreeAgent validation errors (wrong fields, tax rate, category/contact URLs).
  - Response shape mismatch: if FreeAgent returns a different structure on error, downstream code will fail.
- **Version notes:** HTTP Request v4.2.

#### Node: Prepare Payment
- **Type / role:** `Code` — prepares a bank transaction explanation to mark the bill as paid.
- **Config choices (interpreted):**
  - Reads from prior response: `$json.bill.url`, `$json.bill.total_value`, `$json.bill.dated_on`
  - Creates payload:
    - `bank_account`: `https://api.freeagent.com/v2/bank_accounts/[YOUR_FREEAGENT_BANK_ACCOUNT_ID]`
    - `paid_bill`: bill URL
    - `dated_on`: bill dated_on
    - `gross_value`: negative amount string `-${billAmount}`
- **Outputs:** `{ json: payment }` to **Mark Bill as Paid**.
- **Edge cases / failures:**
  - If bill response doesn’t include `.bill.*`, payment creation will fail.
  - Gross value formatting: sent as a string; ensure FreeAgent accepts string numeric.
  - Placeholder bank account ID must be replaced.

#### Node: Mark Bill as Paid
- **Type / role:** `HTTP Request` — creates bank transaction explanation in FreeAgent.
- **Config choices (interpreted):**
  - POST `https://api.freeagent.com/v2/bank_transaction_explanations`
  - OAuth2 “FreeAgent OAuth2”
  - Body: prepared explanation payload
- **Outputs:** To **Record In PostgreSQL**.
- **Edge cases / failures:**
  - FreeAgent may reject if bank account/currency mismatch, or if bill is already paid.
  - If this step fails after bill creation, you may have an unpaid bill but no DB record (partial completion).

#### Node: Record In PostgreSQL
- **Type / role:** `Postgres` — inserts processed invoice record.
- **Config choices (interpreted):**
  - Inserts into `aws_invoices_processed`:
    - `aws_invoice_id`: `$('Loop Over Invoices').item.json.InvoiceId`
    - `freeagent_invoice_url`: `{{$json.bill.url}}` (note: assumes current item still contains `bill.url`)
    - `billing_period`: `YYYY-MM` from invoice billing period
    - `amount`, `currency` from `PaymentCurrencyAmount`
  - Uses Postgres credentials “Postgres account”.
- **Connections:**
  - Input from **Mark Bill as Paid**
  - Output back to **Loop Over Invoices** to continue the batch loop.
- **Edge cases / failures:**
  - Data dependency risk: after “Mark Bill as Paid”, `$json` may not contain `bill.url` unless FreeAgent returns it; this can break the insert. A safer approach is to keep bill info in item data (merge) or reference it from the “Create FreeAgent Bill” node explicitly.
  - Unique constraint violations if invoice already recorded (race conditions, reruns without proper deduplication).
  - Amount type formatting in SQL: inserted without quotes; ensure it’s numeric and uses dot decimal.
- **Version notes:** Postgres node v2.4.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger Monthly | Schedule Trigger | Monthly schedule entrypoint | — | Calculate Last Month Date Range | ## How it works … (full “Workflow Overview” sticky note content applies) |
| Calculate Last Month Date Range | Code | Compute last month billing period + cutoff date | Trigger Monthly | Generate AWS Signature | **1. Fetch AWS Invoices** Retrieves last month's invoices via AWS Invoicing API; ## How it works … (Workflow Overview) |
| Generate AWS Signature | Code | Build AWS SigV4 request (headers/body) | Calculate Last Month Date Range | Call AWS For Bills | **1. Fetch AWS Invoices** …; ## How it works … |
| Call AWS For Bills | HTTP Request | Call AWS Invoicing API | Generate AWS Signature | Parse AWS Response | **1. Fetch AWS Invoices** …; ## How it works … |
| Parse AWS Response | Code | Parse AWS response and emit invoice items | Call AWS For Bills | Filter By Date | **1. Fetch AWS Invoices** …; ## How it works … |
| Filter By Date | Code | Keep invoices after cutoff date | Parse AWS Response | Loop Over Invoices | **1. Fetch AWS Invoices** …; ## How it works … |
| Loop Over Invoices | Split In Batches | Iterate invoices one-by-one | Filter By Date; Record In PostgreSQL | Check If Already Processed (output 2) | **2. Deduplicate** Skips invoices already in PostgreSQL; ## How it works … |
| Check If Already Processed | Postgres | SELECT count for invoice ID | Loop Over Invoices | If New Invoice | **2. Deduplicate** …; ## How it works … |
| If New Invoice | IF | Branch if invoice not yet processed | Check If Already Processed | Prepare FreeAgent Bill (true) | **2. Deduplicate** …; ## How it works … |
| Prepare FreeAgent Bill | Code | Map AWS invoice to FreeAgent bill payload | If New Invoice | Create FreeAgent Bill | **3. Create & Pay Bill** Creates FreeAgent bill, marks paid, logs to database; ## How it works … |
| Create FreeAgent Bill | HTTP Request | Create bill in FreeAgent | Prepare FreeAgent Bill | Prepare Payment | **3. Create & Pay Bill** …; ## How it works … |
| Prepare Payment | Code | Prepare bank transaction explanation payload | Create FreeAgent Bill | Mark Bill as Paid | **3. Create & Pay Bill** …; ## How it works … |
| Mark Bill as Paid | HTTP Request | Mark bill paid via bank transaction explanation | Prepare Payment | Record In PostgreSQL | **3. Create & Pay Bill** …; ## How it works … |
| Record In PostgreSQL | Postgres | Insert processed invoice tracking record | Mark Bill as Paid | Loop Over Invoices | **3. Create & Pay Bill** …; ## How it works … |
| Workflow Overview | Sticky Note | Documentation / setup instructions | — | — | (content is the sticky note itself) |
| Sticky Note | Sticky Note | Block label | — | — | **1. Fetch AWS Invoices** Retrieves last month's invoices via AWS Invoicing API |
| Sticky Note1 | Sticky Note | Block label | — | — | **2. Deduplicate** Skips invoices already in PostgreSQL |
| Sticky Note2 | Sticky Note | Block label | — | — | **3. Create & Pay Bill** Creates FreeAgent bill, marks paid, logs to database |

> Sticky note “Workflow Overview” content (verbatim summary): explains 5-step flow and setup steps including required SQL table creation.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: “Sync AWS Billing with Free Agent Invoice”
   - Set timezone to **Europe/London** (Workflow settings).

2. **Add Trigger node**
   - Node: **Schedule Trigger** (name: “Trigger Monthly”)
   - Mode: Cron expression
   - Expression: `30 0 3,4 * *`

3. **Add Code node to compute last month**
   - Node: **Code** (name: “Calculate Last Month Date Range”)
   - Paste logic to compute `billing_year`, `billing_month`, `cutoff_date`, and set `account_id`.
   - Replace `account_id` with your AWS Account ID.

4. **Add Code node to generate AWS SigV4**
   - Node: **Code** (name: “Generate AWS Signature”)
   - Implement SigV4 signing for:
     - host `invoicing.us-east-1.api.aws`
     - region `us-east-1`
     - service `invoicing`
     - target `Invoicing.ListInvoiceSummaries`
     - content-type `application/x-amz-json-1.0`
   - Replace placeholders:
     - `[YOUR_AWS_ACCESS_KEY]`
     - `[YOUR_AWS_SECRET_KEY]`
   - Ensure IAM permissions include `invoicing:ListInvoiceSummaries`.

5. **Add HTTP Request node to call AWS**
   - Node: **HTTP Request** (name: “Call AWS For Bills”)
   - Method: POST
   - URL: `{{$json.url}}`
   - Send headers enabled:
     - Content-Type: `{{$json.headers['Content-Type']}}`
     - X-Amz-Date: `{{$json.headers['X-Amz-Date']}}`
     - X-Amz-Target: `{{$json.headers['X-Amz-Target']}}`
     - Authorization: `{{$json.headers['Authorization']}}`
   - Send JSON body: `{{$json.body}}`
   - Response: enable “Full response” (so downstream can access `data`).

6. **Add Code node to parse AWS response**
   - Node: **Code** (name: “Parse AWS Response”)
   - Parse `JSON.parse($input.first().json.data)` and return one item per `InvoiceSummaries[]`.

7. **Add Code node to filter by cutoff date**
   - Node: **Code** (name: “Filter By Date”)
   - Read cutoff date from the earlier node (`Calculate Last Month Date Range`).
   - Filter by `IssuedDate` (Unix seconds) >= cutoff date.

8. **Add Split In Batches**
   - Node: **Split In Batches** (name: “Loop Over Invoices”)
   - Keep defaults (commonly batch size 1).
   - Connect **Filter By Date → Loop Over Invoices**.

9. **Add PostgreSQL node for dedup check**
   - Node: **Postgres** (name: “Check If Already Processed”)
   - Credentials: create/select a PostgreSQL credential (“Postgres account”).
   - Operation: Execute Query
   - Query:
     ```sql
     SELECT COUNT(*) as count
     FROM aws_invoices_processed
     WHERE aws_invoice_id = '{{ $json.InvoiceId }}';
     ```
   - Connect from **Loop Over Invoices (output 2)** → **Check If Already Processed**.

10. **Add IF node**
    - Node: **IF** (name: “If New Invoice”)
    - Condition: number equals
      - Left: `{{$json.count.toNumber()}}`
      - Right: `0`
    - Connect **Check If Already Processed → If New Invoice**.

11. **Add Code node to prepare FreeAgent bill**
    - Node: **Code** (name: “Prepare FreeAgent Bill”)
    - Build payload `{ bill: { ... } }`:
      - Replace placeholders:
        - `[YOUR_FREEAGENT_CONTACT_ID]`
        - `[YOUR_FREEAGENT_CATEGORY_ID]`
      - Keep date conversion logic; note the DueDate/IssuedDate swap as implemented.

12. **Add FreeAgent OAuth2 credential**
    - In n8n Credentials: create **OAuth2 API** credential for FreeAgent (“FreeAgent OAuth2”).
    - Ensure it can access `/v2/bills` and `/v2/bank_transaction_explanations`.

13. **Add HTTP Request node to create bill**
    - Node: **HTTP Request** (name: “Create FreeAgent Bill”)
    - Method: POST
    - URL: `https://api.freeagent.com/v2/bills`
    - Auth: OAuth2 (generic) → select “FreeAgent OAuth2”
    - Body: JSON (send the incoming `{ bill: ... }` object)

14. **Add Code node to prepare payment**
    - Node: **Code** (name: “Prepare Payment”)
    - Replace `[YOUR_FREEAGENT_BANK_ACCOUNT_ID]`
    - Construct `bank_transaction_explanation` with negative `gross_value`.

15. **Add HTTP Request node to mark as paid**
    - Node: **HTTP Request** (name: “Mark Bill as Paid”)
    - Method: POST
    - URL: `https://api.freeagent.com/v2/bank_transaction_explanations`
    - Auth: OAuth2 (generic) → “FreeAgent OAuth2”
    - Body: JSON from previous node.

16. **Add PostgreSQL node to record processed invoice**
    - Node: **Postgres** (name: “Record In PostgreSQL”)
    - Operation: Execute Query
    - Query inserts invoice ID, FreeAgent bill URL, billing period, amount, currency.
    - Connect **Mark Bill as Paid → Record In PostgreSQL**.
    - Connect **Record In PostgreSQL → Loop Over Invoices** (to continue batches).

17. **Create the PostgreSQL tracking table**
    - Run in your database:
      ```sql
      CREATE TABLE aws_invoices_processed (
        id SERIAL PRIMARY KEY,
        aws_invoice_id VARCHAR(255) UNIQUE,
        freeagent_invoice_url VARCHAR(500),
        billing_period VARCHAR(7),
        amount DECIMAL(10,2),
        currency VARCHAR(3),
        created_at TIMESTAMP DEFAULT NOW()
      );
      ```

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Runs on the 3rd and 4th to account for AWS invoice availability; dedup prevents duplicates. | Workflow Overview sticky note |
| AWS credentials are currently placed directly in the Code node; replace with secure credential/env handling. IAM needs `invoicing:ListInvoiceSummaries`. | Workflow Overview sticky note + Generate AWS Signature node |
| FreeAgent setup requires OAuth2 credential, supplier contact URL, expense category URL, and bank account URL. | Workflow Overview sticky note |
| PostgreSQL is used as a processed-invoice ledger to prevent duplicate bill creation. | Workflow Overview sticky note |
| Tag: “Pernille-AI.com”. | Workflow metadata |

