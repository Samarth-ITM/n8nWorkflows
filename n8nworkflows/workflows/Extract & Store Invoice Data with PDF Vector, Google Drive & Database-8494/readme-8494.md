Extract & Store Invoice Data with PDF Vector, Google Drive & Database

https://n8nworkflows.xyz/workflows/extract---store-invoice-data-with-pdf-vector--google-drive---database-8494


# Extract & Store Invoice Data with PDF Vector, Google Drive & Database

### 1. Workflow Overview

This workflow automates the end-to-end processing of invoices for accounts payable in an enterprise environment. It is designed to run continuously every 5 minutes, monitoring a specified Google Drive folder for new invoice PDFs, extracting detailed invoice data using AI-powered PDF parsing, validating and enriching this data, managing vendor information, routing invoices for approval based on preset thresholds, integrating approved invoices into ERP systems like QuickBooks or SAP, and updating real-time dashboards.

The workflow logically divides into the following blocks:

- **1.1 Invoice Collection:** Periodic scanning and filtering of new invoice files on Google Drive.
- **1.2 AI Data Extraction:** Downloading invoice PDFs and extracting comprehensive data fields via PDF Vector AI.
- **1.3 Vendor Management:** Looking up vendors in the database, creating new entries if necessary, and validating vendor status.
- **1.4 Validation & Approval:** Performing detailed invoice validations (calculations, duplicates, PO matching) and determining approval routing.
- **1.5 ERP Integration & Finalization:** Saving approved invoices to the database, syncing with ERP systems, marking files as processed, and updating analytics dashboards.

---

### 2. Block-by-Block Analysis

#### 2.1 Invoice Collection

**Overview:**  
This block triggers every 5 minutes to check a Google Drive folder for new invoices. It filters out already processed files to avoid duplication and downloads only new invoices for processing.

**Nodes Involved:**  
- Check Every 5 Minutes (Schedule Trigger)  
- List New Invoices (Google Drive)  
- Check Already Processed (Postgres)  
- Filter New Files (Code)  
- Download Invoice (Google Drive)  

**Node Details:**

- **Check Every 5 Minutes**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every 5 minutes and on workflow activation.  
  - Configuration: Interval set to 5 minutes.  
  - Outputs to: List New Invoices  
  - Failure: Trigger failure unlikely, but connectivity issues may delay runs.

- **List New Invoices**  
  - Type: Google Drive (List Files)  
  - Role: Lists files in a configured Google Drive folder (folder ID provided via incoming JSON).  
  - Configuration: Retrieves file ID, name, MIME type, and creation time.  
  - Inputs: Trigger output with folder ID.  
  - Outputs to: Check Already Processed  
  - Failure: API quota limits, invalid folder ID, or permission errors.

- **Check Already Processed**  
  - Type: Postgres (Execute Query)  
  - Role: Queries database for files already processed to avoid duplicates.  
  - Configuration: SQL query selects file IDs from `processed_invoices` matching listed files.  
  - Inputs: List of files from previous node.  
  - Outputs to: Filter New Files  
  - Failure: Database connection issues, malformed queries.

- **Filter New Files**  
  - Type: Code (JavaScript)  
  - Role: Filters out files that have been processed already, returns only new files.  
  - Key Expressions: Compares file IDs from Google Drive listing against DB results.  
  - Inputs: Lists of files and processed IDs.  
  - Outputs to: Download Invoice  
  - Failure: Expression errors if node names change or JSON structure differs.

- **Download Invoice**  
  - Type: Google Drive (Download)  
  - Role: Downloads the actual invoice PDF content for AI extraction.  
  - Configuration: Downloads file by ID from filtered list.  
  - Inputs: New file JSON from filter.  
  - Outputs to: Extract Invoice Data  
  - Failure: File missing, permission errors, network failures.

---

#### 2.2 AI Data Extraction

**Overview:**  
This block processes the downloaded PDF invoices using PDF Vector AI to extract detailed structured invoice data including vendor info, line items, taxes, payment terms, and bank details.

**Nodes Involved:**  
- Extract Invoice Data (PDF Vector)  

**Node Details:**

- **Extract Invoice Data**  
  - Type: PDF Vector (Extract)  
  - Role: Uses AI to parse invoice PDF content into structured JSON fields.  
  - Configuration:  
    - Prompt includes detailed extraction instructions covering 30+ fields.  
    - Schema strictly defines expected output structure (vendor, customer, line items, taxes, bank details, etc.).  
    - Input is the downloaded PDF binary.  
  - Inputs: PDF binary from Download Invoice.  
  - Outputs to: Lookup Vendor  
  - Failure: API key misconfiguration, extraction errors on unreadable PDFs, schema validation failures.

---

#### 2.3 Vendor Management

**Overview:**  
This block checks if the extracted vendor exists in the database. If not, it creates a new vendor record flagged for review. This ensures vendor master data integrity and compliance.

**Nodes Involved:**  
- Lookup Vendor (Postgres)  
- Vendor Exists? (If)  
- Create New Vendor (Postgres)  

**Node Details:**

- **Lookup Vendor**  
  - Type: Postgres (Execute Query)  
  - Role: Searches `vendor_master` table by vendor name (case-insensitive) or tax ID.  
  - Inputs: Extracted vendor data JSON.  
  - Outputs to: Vendor Exists?  
  - Failure: DB connectivity, query errors.

- **Vendor Exists?**  
  - Type: If  
  - Role: Checks if vendor was found (JSON length > 0).  
  - Inputs: Lookup Vendor output.  
  - Outputs:  
    - True branch → Validate & Enrich Invoice  
    - False branch → Create New Vendor  
  - Failure: Expression evaluation errors if JSON structure changes.

- **Create New Vendor**  
  - Type: Postgres (Execute Query)  
  - Role: Inserts new vendor record with status `pending_review`.  
  - Inputs: Extracted vendor details from AI extraction node.  
  - Outputs to: Validate & Enrich Invoice  
  - Failure: Data validation errors, DB write issues.

---

#### 2.4 Validation & Approval

**Overview:**  
This block validates invoice calculations, checks for duplicates, verifies PO matching, and determines if approval is required based on invoice amount and vendor status. If approval is needed, it generates a secure link and notifies approvers via Slack.

**Nodes Involved:**  
- Validate & Enrich Invoice (Code)  
- Check Duplicate (Postgres)  
- Check PO (Postgres)  
- Needs Approval? (If)  
- Generate Approval Link (Code)  
- Send Approval Request (Slack)  

**Node Details:**

- **Validate & Enrich Invoice**  
  - Type: Code (JavaScript)  
  - Role: Performs multiple validations and enrichments:  
    - Validates line item subtotals vs. reported subtotal  
    - Validates tax totals vs. invoice total  
    - Checks for duplicate invoice via DB node output  
    - Checks for PO existence and remaining balance  
    - Sets approval flags and level based on amount and vendor status  
  - Inputs: Vendor info, extracted invoice data, duplicate check results, PO check results.  
  - Outputs to: Check Duplicate, Check PO, Needs Approval?  
  - Failure: Async await usage requires n8n v0.190.0+; errors if called nodes missing or data malformed.

- **Check Duplicate**  
  - Type: Postgres (Execute Query)  
  - Role: Searches invoices DB for existing invoice by vendor ID and invoice number to prevent double payments.  
  - Inputs: Validation node JSON with vendorId and invoiceNumber.  
  - Outputs: To validation code node as awaited JSON.  
  - Failure: Query or connection errors.

- **Check PO**  
  - Type: Postgres (Execute Query)  
  - Role: Retrieves active PO details matching extracted PO number, to verify 3-way match.  
  - Inputs: PO number from extracted invoice.  
  - Outputs: To validation node as awaited JSON.  
  - Failure: Query errors, missing PO, or DB connection.

- **Needs Approval?**  
  - Type: If  
  - Role: Branches workflow based on whether approval is required (`requiresApproval` boolean).  
  - Inputs: Validation output.  
  - Outputs:  
    - True branch → Generate Approval Link → Send Approval Request → Save Invoice  
    - False branch → Save Invoice directly  
  - Failure: Expression evaluation errors.

- **Generate Approval Link**  
  - Type: Code (JavaScript)  
  - Role: Creates a secure, expiring token-based URL for invoice approval.  
  - Configuration: Generates random 32-byte hex token, builds URL, stores token data (DB storage not shown).  
  - Inputs: Invoice number, vendor ID, amount from validation node.  
  - Outputs: Approval data and URL JSON.  
  - Outputs to: Send Approval Request  
  - Failure: Node requires Node.js crypto module availability, secure token storage not implemented in workflow.

- **Send Approval Request**  
  - Type: Slack  
  - Role: Posts formatted notification with invoice details and approval link to a designated Slack channel.  
  - Configuration: Uses template strings to insert invoice and vendor data, includes attachments with due date and payment terms.  
  - Inputs: Approval link JSON.  
  - Failure: Slack API token/configuration issues, rate limits.

---

#### 2.5 ERP Integration & Finalization

**Overview:**  
This final block saves the invoice data into a database, synchronizes approved invoices with ERP software (QuickBooks), marks the invoice file as processed to prevent reprocessing, and updates real-time analytics dashboards.

**Nodes Involved:**  
- Save Invoice (Postgres)  
- Create in QuickBooks (QuickBooks)  
- Mark as Processed (Postgres)  
- Update Analytics Dashboard (Webhook)  

**Node Details:**

- **Save Invoice**  
  - Type: Postgres (Execute Query)  
  - Role: Inserts invoice data into the `invoices` table with status based on approval requirement.  
  - Inputs: Invoice data, vendor ID, approval flags.  
  - Outputs to: Create in QuickBooks and Mark as Processed  
  - Failure: DB write errors, JSON serialization issues.

- **Create in QuickBooks**  
  - Type: QuickBooks (Create Invoice)  
  - Role: Syncs invoice data to QuickBooks ERP using OAuth2 authentication.  
  - Configuration: Maps invoice fields to QuickBooks API, includes line items, due date, vendor reference, memo, and invoice number.  
  - Inputs: Saved invoice JSON.  
  - Outputs to: Mark as Processed (via Save Invoice output)  
  - Failure: OAuth token expiry, API quota, data mapping errors.

- **Mark as Processed**  
  - Type: Postgres (Execute Query)  
  - Role: Inserts record in `processed_invoices` to track processed file and invoice IDs.  
  - Inputs: File ID from Download Invoice, Invoice ID from Save Invoice.  
  - Outputs to: Update Analytics Dashboard  
  - Failure: DB insertion errors.

- **Update Analytics Dashboard**  
  - Type: Webhook  
  - Role: Sends update to external real-time dashboard system to refresh metrics.  
  - Configuration: Calls external dashboard URL with real-time update frequency.  
  - Inputs: Triggered after marking invoice as processed.  
  - Failure: Network errors, endpoint unavailability.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                    | Input Node(s)              | Output Node(s)                      | Sticky Note                                                                                                   |
|-------------------------|------------------------|----------------------------------|----------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow Overview       | Sticky Note            | Overview & purpose                | -                          | -                                 | ## 📋 Invoice Processing Pipeline ...                                                                         |
| Setup Guide            | Sticky Note            | Initial setup instructions       | -                          | -                                 | ## ⚙️ Initial Setup Required ...                                                                                |
| Step 1: Collection      | Sticky Note            | Invoice collection description   | -                          | -                                 | ## 1️⃣ Invoice Collection ...                                                                                   |
| Step 2: Extraction      | Sticky Note            | AI extraction description        | -                          | -                                 | ## 2️⃣ AI Data Extraction ...                                                                                   |
| Step 3: Vendors         | Sticky Note            | Vendor management description    | -                          | -                                 | ## 3️⃣ Vendor Management ...                                                                                     |
| Step 4: Validation      | Sticky Note            | Validation & approval description| -                          | -                                 | ## 4️⃣ Validation & Approval ...                                                                                 |
| Step 5: Integration     | Sticky Note            | ERP integration description      | -                          | -                                 | ## 5️⃣ ERP Integration ...                                                                                        |
| Check Every 5 Minutes   | Schedule Trigger       | Trigger workflow every 5 minutes | -                          | List New Invoices                 | Monitor for new invoices                                                                                        |
| List New Invoices       | Google Drive           | List files in Drive folder       | Check Every 5 Minutes       | Check Already Processed           | Get unprocessed invoices                                                                                        |
| Check Already Processed | Postgres               | Check DB for processed files     | List New Invoices           | Filter New Files                 | Avoid reprocessing                                                                                              |
| Filter New Files        | Code                   | Filter new files only            | Check Already Processed     | Download Invoice                 |                                                                                                               |
| Download Invoice        | Google Drive           | Download invoice PDF             | Filter New Files            | Extract Invoice Data             | Get file content                                                                                               |
| Extract Invoice Data    | PDF Vector             | AI extract invoice data          | Download Invoice            | Lookup Vendor                   | AI extraction                                                                                                  |
| Lookup Vendor           | Postgres               | Lookup vendor in DB              | Extract Invoice Data        | Vendor Exists?                  | Check vendor database                                                                                          |
| Vendor Exists?          | If                     | Conditional branch on vendor     | Lookup Vendor               | Validate & Enrich Invoice, Create New Vendor |                                                                                                     |
| Create New Vendor       | Postgres               | Insert new vendor record         | Vendor Exists? (false path) | Validate & Enrich Invoice       | Add to vendor master                                                                                           |
| Validate & Enrich Invoice | Code                 | Validate invoice & enrich data  | Vendor Exists?, Create New Vendor | Check Duplicate, Check PO, Needs Approval? | Complex validation logic                                                       |
| Check Duplicate         | Postgres               | Check for duplicate invoice      | Validate & Enrich Invoice   | Validate & Enrich Invoice (awaited) | Prevent double payment                                                                                         |
| Check PO                | Postgres               | Check purchase order             | Validate & Enrich Invoice   | Validate & Enrich Invoice (awaited) | 3-way matching                                                                                                 |
| Needs Approval?         | If                     | Conditional branch on approval   | Validate & Enrich Invoice   | Generate Approval Link, Save Invoice |                                                                                                               |
| Generate Approval Link  | Code                   | Generate secure approval URL     | Needs Approval? (true path) | Send Approval Request           | Create secure link                                                                                             |
| Send Approval Request   | Slack                  | Notify approvers                 | Generate Approval Link      | -                              | Notify approvers                                                                                               |
| Save Invoice            | Postgres               | Save invoice in DB               | Needs Approval?, Generate Approval Link | Create in QuickBooks, Mark as Processed | Store in database                                                                                              |
| Create in QuickBooks    | QuickBooks             | Sync invoice with ERP            | Save Invoice                | Mark as Processed               | ERP integration                                                                                                |
| Mark as Processed       | Postgres               | Track processed invoice files   | Save Invoice, Create in QuickBooks | Update Analytics Dashboard   | Track processed files                                                                                          |
| Update Analytics Dashboard | Webhook              | Update real-time dashboards      | Mark as Processed            | -                              | Real-time metrics                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run every 5 minutes.  
   - Name it "Check Every 5 Minutes".

2. **Add a Google Drive node to list files**  
   - Type: Google Drive (List Files)  
   - Set operation to "list" files in a folder.  
   - Configure the folder ID dynamically or statically (as `invoiceFolderId` input).  
   - Select fields: id, name, mimeType, createdTime.  
   - Connect "Check Every 5 Minutes" output to this node.

3. **Add a Postgres node to check processed files**  
   - Type: Postgres (Execute Query)  
   - Query: `SELECT file_id FROM processed_invoices WHERE file_id IN ({{ $json.files.map(f => `'${f.id}'`).join(',') }})`  
   - Connect output of Google Drive list node here.

4. **Add a Code node to filter new files**  
   - Type: Code (JavaScript)  
   - Write code to compare files listed with processed file IDs and filter only new ones.  
   - Connect Postgres check node output here.

5. **Add a Google Drive node to download invoice PDFs**  
   - Type: Google Drive (Download)  
   - Use file ID from filtered files.  
   - Connect Code node output here.

6. **Add PDF Vector node for AI extraction**  
   - Type: PDF Vector (Extract)  
   - Configure prompt with detailed instructions for invoice data extraction.  
   - Define strict JSON schema with all necessary invoice fields including nested objects and arrays.  
   - Input binary property: the downloaded PDF file.  
   - Connect Google Drive download node here.  
   - Credential: Add and configure PDF Vector API key.

7. **Add Postgres node to lookup vendor**  
   - Query vendor_master table by vendor name (case-insensitive) and tax ID from extracted data.  
   - Connect PDF Vector output here.

8. **Add an If node to check if vendor exists**  
   - Condition: JSON length > 0 (vendor found).  
   - Connect Lookup Vendor output here.

9. **Add Postgres node to create vendor**  
   - Insert new vendor into vendor_master with status "pending_review" and all extracted vendor details.  
   - Connect "false" branch from If node here.

10. **Add a Code node for validation & enrichment**  
    - Implement comprehensive validations for invoice math, tax, duplicates, PO matching, and approval logic.  
    - Await outputs from Check Duplicate and Check PO nodes (next steps).  
    - Connect both Vendor Exists? true branch and Create Vendor outputs here.

11. **Add Postgres node to check duplicate invoices**  
    - Query invoices table filtering by vendor ID and invoice number from validation node data.  
    - Connect validation node output here (used via await in validation code).

12. **Add Postgres node to check PO**  
    - Query purchase_orders table filtering by PO number and active status.  
    - Connect validation node output here (used via await in validation code).

13. **Add If node to evaluate if approval is required**  
    - Condition: `requiresApproval` boolean from validation output.  
    - Connect validation node output here.

14. **Add Code node to generate approval link**  
    - Generate secure random token and build approval URL with expiry.  
    - Connect true branch of Needs Approval? node here.

15. **Add Slack node to send approval request**  
    - Compose message with invoice and vendor data and approval link.  
    - Send to appropriate Slack channel (e.g., #invoice-approvals).  
    - Connect Generate Approval Link output here.  
    - Credential: Slack OAuth token.

16. **Add Postgres node to save invoice**  
    - Insert invoice data into invoices table with status depending on approval requirement (pending_approval or approved).  
    - Include JSON serialized raw invoice data.  
    - Connect both Needs Approval? true (after approval link) and false branches here.

17. **Add QuickBooks node to create invoice in ERP**  
    - Map invoice data fields to QuickBooks API invoice creation.  
    - Use OAuth2 authentication.  
    - Connect Save Invoice node output here.

18. **Add Postgres node to mark invoice file as processed**  
    - Insert into processed_invoices table with file_id and invoice_id.  
    - Connect Save Invoice and QuickBooks node outputs here.

19. **Add Webhook node or HTTP Request node to update analytics dashboard**  
    - Configure to call real-time dashboard update URL.  
    - Connect Mark as Processed output here.

20. **Add Sticky Notes** for overview, step descriptions, and setup instructions at appropriate places for documentation and team clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow requires initial setup including Google Drive folder creation, database schema setup, PDF Vector API key, Slack and email notification credentials, and ERP API. | Setup Guide sticky note                                         |
| Invoice approval routing thresholds are set at >$1k (Manager), >$5k (Department Head), and >$10k (CFO) which can be customized as needed.                                        | Step 4: Validation & Approval sticky note                      |
| AI extraction prompt and schema are comprehensive to handle multiple invoice formats and multi-page PDFs, ensuring high data quality.                                          | Step 2: Extraction sticky note                                 |
| Slack notifications include clickable approval links with secure tokens expiring after 7 days for security compliance.                                                         | Send Approval Request node and Generate Approval Link node     |
| Database tables include `processed_invoices`, `vendor_master`, `invoices`, and `purchase_orders` with assumed schemas for tracking and validation.                             | Implied from Postgres queries throughout the workflow          |
| This workflow is designed for n8n version 0.190.0 or higher due to use of async/await in code nodes.                                                                             | Validation & Enrich Invoice code node requirement              |

---

Disclaimer: The provided text is exclusively generated from an automated n8n workflow export. It complies fully with current content policies, containing no illegal, offensive, or protected elements. All data processed is legal and public.