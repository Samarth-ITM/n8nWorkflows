Generate Professional Construction Quotes from JotForm to Email with Supabase CRM

https://n8nworkflows.xyz/workflows/generate-professional-construction-quotes-from-jotform-to-email-with-supabase-crm-9608


# Generate Professional Construction Quotes from JotForm to Email with Supabase CRM

### 1. Workflow Overview

This workflow automates the generation of professional construction quotes from JotForm submissions, integrating data normalization, CRM record management, pricing rule application, and email delivery. It is designed for construction or plastering businesses to streamline quote creation, reduce manual errors, and accelerate customer response time.

The workflow is logically divided into four main stages:

- **1.1 Input Reception and Form Submission Processing:** Receives JotForm POST requests, extracts and normalizes submission data, and saves raw form data to Supabase.

- **1.2 CRM Insert:** Inserts or updates customer and deal records in the Supabase CRM database, linking submissions to customers and deals.

- **1.3 Quote Calculation Engine:** Fetches pricing rules, applies business logic to calculate line items and totals, and saves estimate headers and line items to the database.

- **1.4 Professional Quote Email Generation and Delivery:** Retrieves complete quote details, generates a styled HTML email with quote information, and sends it to the customer via Gmail.

The workflow includes robust error handling at critical points to stop execution with context when failures occur, ensuring traceability.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Stage 1: Input Reception and Form Submission Processing

**Overview:**  
This block handles incoming JotForm submissions via a webhook, parses the raw data, fetches mapping rules for normalization, transforms the data into structured fields, and saves the submission to the Supabase database.

**Nodes Involved:**  
- Webhook  
- Parser (Set node)  
- Fetch Mapping Rules (Supabase)  
- Prepare AI Context (Set node)  
- Normalize Form Data (Code node)  
- Save Form Submission (Supabase)  
- upsert form submission -error (Stop and Error node)  
- Sticky Note: "Stage 1: Webhook Processing"

**Node Details:**

- **Webhook**  
  - Type: Webhook Trigger  
  - Configuration: Listens for POST requests on path `b7ad1332-8112-4ba1-jotform`.  
  - Inputs: External HTTP POST from JotForm.  
  - Outputs: Raw JSON form submission payload.  
  - Failure Modes: Network issues, invalid POST data.

- **Parser (Set node)**  
  - Role: Extracts nested properties for easier access.  
  - Assigns: `body.rawRequest`, `ai.content` (pretty text), and `submission_id` from webhook payload.  
  - Inputs: Webhook output.  
  - Outputs: Simplified JSON with extracted fields.  
  - Failure Modes: Missing expected fields, malformed JSON.

- **Fetch Mapping Rules (Supabase)**  
  - Role: Retrieves active normalization rules from `form_value_mappings` table.  
  - Config: Filter `is_active = true`.  
  - Inputs: Output from Parser.  
  - Outputs: List of active mapping rules for data normalization.  
  - Failure Modes: Database connection issues, empty results.

- **Prepare AI Context (Set node)**  
  - Role: Constructs arrays of distinct normalized values by field for AI or logic context, and prepares cleaned raw request data with sensitive fields removed.  
  - Inputs: Output from Fetch Mapping Rules.  
  - Outputs: Context object for later normalization.  
  - Failure Modes: Expression errors if expected fields missing.

- **Normalize Form Data (Code node)**  
  - Role: Transforms raw JotForm answers into normalized structured fields, including nested customer info, numeric conversions, and selection mappings.  
  - Inputs: Output from Prepare AI Context.  
  - Outputs: JSON object with `normalized` property containing structured form data.  
  - Failure Modes: Missing fields, unexpected data types.

- **Save Form Submission (Supabase)**  
  - Role: Inserts raw submission data with metadata into `form_submissions` table for audit trail.  
  - Configuration: Fields include submission ID, timestamps, customer email/name, raw payload, processed flag, etc.  
  - Inputs: Output from Parser.  
  - Outputs: Supabase confirmation or error.  
  - Error Handling: On failure, routes to `upsert form submission -error` node.  
  - Failure Modes: DB write failure, duplicate primary key.

- **upsert form submission -error (Stop and Error node)**  
  - Role: Stops workflow on failure to save submission, providing contextual error including submission ID, form ID, timestamp, and execution ID.  
  - Inputs: From Save Form Submission failure.  
  - Outputs: Stops execution.  
  - Failure Modes: N/A (designed for error propagation).

---

#### 2.2 Stage 2: CRM Insert

**Overview:**  
This block upserts the customer record based on normalized form data and creates a new deal record linked to the customer and current form submission.

**Nodes Involved:**  
- Normalize Form Data (from previous block)  
- Upsert Customer (Supabase)  
- Create Deal (Supabase)  
- upsert form customer -error (Stop and Error node)  
- Sticky Note: "Stage 2: CRM Insert"

**Node Details:**

- **Upsert Customer (Supabase)**  
  - Role: Inserts or updates customer record based on email (unique key).  
  - Fields: email, first/last name, billing address JSONB, phone, notes (project description).  
  - Inputs: Normalized form data.  
  - Outputs: Customer record with `customer_id`.  
  - Error Handling: `continueErrorOutput` enabled; failures routed to error handler.  
  - Failure Modes: DB constraint violation, network errors.

- **Create Deal (Supabase)**  
  - Role: Creates a new deal record per submission, linked to customer and form submission.  
  - Fields: customer_id, form_submission_id, deal_name (constructed from last name and phone), status default 'lead'.  
  - Inputs: Output from Upsert Customer.  
  - Outputs: Deal record with `deal_id`.  
  - Failure Modes: DB errors, missing customer_id.

- **upsert form customer -error (Stop and Error node)**  
  - Role: Stops workflow on failure in Upsert Customer node with detailed error context.  
  - Inputs: Failures from Upsert Customer.  
  - Outputs: Stops execution.

---

#### 2.3 Stage 3: Quote Calculation Engine

**Overview:**  
This block fetches pricing rules, applies complex business logic to generate quote line items with calculations including VAT, groups duplicates, calculates totals, and stores estimate header and line items in Supabase.

**Nodes Involved:**  
- Create Deal (from previous block)  
- Fetch Pricing Rules (Supabase)  
- Calculate Quote Line Items (Code node)  
- Save Estimate Header (Supabase)  
- Prepare Line Item Data (Set node)  
- Split Out (SplitOut node)  
- Insert Line Items (Supabase)  
- Sticky Note: "Stage 3: Quote Calculation Engine"

**Node Details:**

- **Fetch Pricing Rules (Supabase)**  
  - Role: Retrieves all active pricing rules from `service_rules_enriched` view.  
  - Inputs: Output from Create Deal.  
  - Outputs: List of pricing rules including trigger conditions and prices.  
  - Failure Modes: DB errors, empty result set.

- **Calculate Quote Line Items (Code node)**  
  - Role: Applies normalization fallback, filters active rules, validates triggers and additional conditions, calculates line totals including VAT, groups identical items, sorts by priority, and sums totals.  
  - Inputs: Pricing rules and normalized form data.  
  - Outputs: Object containing `deal_id`, calculated `lines` array, and summary totals.  
  - Failure Modes: No rules matched, missing data, calculation errors.

- **Save Estimate Header (Supabase)**  
  - Role: Inserts new estimate record linked to deal with subtotal, VAT, currency, and unique estimate number (execution ID).  
  - Inputs: Output from Calculate Quote Line Items.  
  - Outputs: Estimate record with `estimate_id`.  
  - Failure Modes: DB errors, duplicate keys.

- **Prepare Line Item Data (Set node)**  
  - Role: Prepares each line item with estimate_id for batch insertion.  
  - Inputs: Estimate ID from Save Estimate Header and line items from calculation.  
  - Outputs: JSON array with estimate_id and line item details.  
  - Failure Modes: Missing input data.

- **Split Out (SplitOut node)**  
  - Role: Splits array of line items into individual items for separate insert operations.  
  - Inputs: Output from Prepare Line Item Data.  
  - Outputs: Single line items streamed one-by-one.  
  - Failure Modes: Empty array.

- **Insert Line Items (Supabase)**  
  - Role: Inserts each line item record into `estimate_line_items` table.  
  - Inputs: Output from Split Out node.  
  - Outputs: Confirmation of insert.  
  - Error Handling: Retries on failure, waits between tries.  
  - Failure Modes: DB write failures.

---

#### 2.4 Stage 4: Professional Quote Email Generation and Delivery

**Overview:**  
This final stage fetches the complete quote with all line items and customer details, generates a rich HTML email with professional styling and actionable buttons, and sends the email through Gmail OAuth2 credentials.

**Nodes Involved:**  
- Insert Line Items (from previous block)  
- Fetch Complete Quote (Supabase)  
- Generate Email HTML (Code node)  
- Send Email node (Gmail)  
- Sticky Note: "Stage 4: Professional Quote Email"

**Node Details:**

- **Fetch Complete Quote (Supabase)**  
  - Role: Retrieves full quote data from `v_estimate_proforma` view by estimate_id, including customer info and all line items as JSONB.  
  - Inputs: Output from Insert Line Items.  
  - Outputs: Detailed quote JSON.  
  - Error Handling: Retries on fail.  
  - Failure Modes: Missing estimate_id, DB errors.

- **Generate Email HTML (Code node)**  
  - Role: Maps quote data to template properties, builds a dark-themed, mobile-friendly HTML email with quote details, VAT breakdown, CTA buttons (Accept Quote, Schedule Discussion), validity notice, legal disclaimers, and company footer.  
  - Inputs: Fetch Complete Quote JSON.  
  - Outputs: JSON with HTML email content, recipient email, subject, estimate and customer info for tracking.  
  - Failure Modes: Template errors, missing data fields.

- **Send Email node (Gmail)**  
  - Role: Sends the generated HTML email to the customer's email address using OAuth2 Gmail credentials.  
  - Inputs: Email metadata and HTML content from Generate Email HTML.  
  - Configuration: Sender name "Stucco Planet", attribution disabled.  
  - Failure Modes: Gmail API errors, auth failures, quota limits.

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                                  | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                  |
|------------------------------|-------------------------|-------------------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook                      | Webhook Trigger         | Receives JotForm POST submissions                |                            | Parser                     |                                                                                                              |
| Parser                       | Set                     | Extracts raw request, pretty text, submissionID | Webhook                    | Fetch Mapping Rules, Save Form Submission |                                                                                                              |
| Fetch Mapping Rules          | Supabase                | Retrieves active form value mappings              | Parser                     | Prepare AI Context          |                                                                                                              |
| Prepare AI Context           | Set                     | Constructs normalization context                  | Fetch Mapping Rules         | Normalize Form Data         |                                                                                                              |
| Normalize Form Data          | Code                    | Normalizes and structures raw form data           | Prepare AI Context          | Upsert Customer             |                                                                                                              |
| Save Form Submission         | Supabase                | Stores raw submission with metadata               | Parser                     | (Error branch) upsert form submission -error |                                                                                                              |
| upsert form submission -error | Stop and Error         | Stops workflow on form submission save failure   | Save Form Submission (fail) |                            |                                                                                                              |
| Upsert Customer             | Supabase                | Inserts/updates customer record                    | Normalize Form Data         | Create Deal                |                                                                                                              |
| Create Deal                 | Supabase                | Creates new deal linked to customer and submission | Upsert Customer             | Fetch Pricing Rules         |                                                                                                              |
| upsert form customer -error | Stop and Error          | Stops workflow on customer upsert failure         | Upsert Customer (fail)      |                            |                                                                                                              |
| Fetch Pricing Rules         | Supabase                | Retrieves active pricing rules                     | Create Deal                | Calculate Quote Line Items  |                                                                                                              |
| Calculate Quote Line Items  | Code                    | Applies business logic, calculates line items     | Fetch Pricing Rules         | Save Estimate Header        |                                                                                                              |
| Save Estimate Header        | Supabase                | Saves estimate summary header                      | Calculate Quote Line Items  | Prepare Line Item Data      |                                                                                                              |
| Prepare Line Item Data      | Set                     | Prepares line item data for batch insert          | Save Estimate Header        | Split Out                  |                                                                                                              |
| Split Out                  | SplitOut                | Splits array of line items into individual items  | Prepare Line Item Data      | Insert Line Items           |                                                                                                              |
| Insert Line Items          | Supabase                | Inserts each line item into DB                      | Split Out                  | Fetch Complete Quote        |                                                                                                              |
| Fetch Complete Quote       | Supabase                | Retrieves full quote details from view             | Insert Line Items           | Generate Email HTML         |                                                                                                              |
| Generate Email HTML        | Code                    | Builds professional HTML quote email               | Fetch Complete Quote        | Send Email node             |                                                                                                              |
| Send Email node            | Gmail                   | Sends quote email to customer                       | Generate Email HTML         |                            |                                                                                                              |
| When clicking ‘Test workflow’ | Manual Trigger         | Trigger for testing SQL Schema Generator            |                            | SQL Schema Generator        |                                                                                                              |
| SQL Schema Generator        | Code (disabled)         | Generates SQL schema for Supabase                   | When clicking ‘Test workflow’ |                            |                                                                                                              |
| Stage 1: Webhook Processing | Sticky Note             | Describes Stage 1 flow and error handling          |                            |                            |                                                                                                              |
| Stage 2: CRM Insert         | Sticky Note             | Describes Stage 2 flow and error handling          |                            |                            |                                                                                                              |
| Stage 3: Quote Calculation Engine | Sticky Note         | Describes Stage 3 flow and validation               |                            |                            |                                                                                                              |
| Stage 4: Professional Quote Email | Sticky Note         | Describes Stage 4 flow and email features           |                            |                            |                                                                                                              |
| Sticky Note1               | Sticky Note             | Demo form URL and quote system description          |                            |                            | https://form.jotform.com/252844786304060                                                                     |
| Sticky Note                | Sticky Note             | Overall workflow description and business impact   |                            |                            |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `b7ad1332-8112-4ba1-jotform`  
   - Purpose: Receive JotForm submission POST requests.

2. **Create Parser Node (Set):**  
   - Input: Webhook output  
   - Assign fields:  
     - `body.rawRequest` = `{{$json.body.rawRequest}}`  
     - `ai.content` = `{{$json.ai.content}}`  
     - `submission_id` = `{{$json.submission_id}}`

3. **Create Fetch Mapping Rules Node (Supabase):**  
   - Operation: `getAll`  
   - Table: `form_value_mappings`  
   - Filter: `is_active = true`  
   - Credentials: Configure Supabase API with project URL and service role key.

4. **Create Prepare AI Context Node (Set):**  
   - Extract distinct normalized values for `core_service_type`, `property_status`, `ceiling_height` from previous node output.  
   - Remove sensitive/unneeded fields from `body.rawRequest`.  
   - Assign `ai.content` and `submission_id` from previous nodes.

5. **Create Normalize Form Data Node (Code):**  
   - Input: Output from Prepare AI Context  
   - Implement JS code to:  
     - Extract and normalize form fields (mapping Dutch labels to internal codes).  
     - Convert numeric fields.  
     - Flatten nested customer and address info.

6. **Create Save Form Submission Node (Supabase):**  
   - Table: `form_submissions`  
   - Fields: submission ID, timestamp, form ID (hardcoded), raw payload, customer email/name, suspicious flag (false), time to submit, processed flag (true).  
   - On error: `continueErrorOutput` false.

7. **Create upsert form submission -error Node (Stop and Error):**  
   - Configure to stop workflow with detailed error object capturing node name, submission ID, timestamp, error message, execution ID.

8. **Connect Save Form Submission failure output to upsert form submission -error.**

9. **Connect Parser output to Fetch Mapping Rules and Save Form Submission nodes.**

10. **Connect Fetch Mapping Rules output to Prepare AI Context.**

11. **Connect Prepare AI Context output to Normalize Form Data.**

12. **Create Upsert Customer Node (Supabase):**  
    - Table: `customers`  
    - Fields: email, first_name, last_name, billing_address (JSONB), phone, notes.  
    - Use normalized data from Normalize Form Data.  
    - Enable `continueErrorOutput` true for error handling.

13. **Create Create Deal Node (Supabase):**  
    - Table: `deals`  
    - Fields: customer_id (from Upsert Customer), form_submission_id (from prepared AI context), deal_name (compose from last name and phone).  
    - Status defaults to 'lead'.  
    - Enable retry on fail and wait between tries.

14. **Create upsert form customer -error Node (Stop and Error):**  
    - Stops on Upsert Customer failure with error context.

15. **Connect Normalize Form Data output to Upsert Customer.**

16. **Connect Upsert Customer output to Create Deal and upsert form customer -error.**

17. **Create Fetch Pricing Rules Node (Supabase):**  
    - Table: `service_rules_enriched` (view)  
    - Operation: getAll, returnAll true.

18. **Connect Create Deal output to Fetch Pricing Rules.**

19. **Create Calculate Quote Line Items Node (Code):**  
    - Input: Pricing rules and normalized form data.  
    - Implement provided JS code for filtering rules, calculating quantities, applying VAT, grouping lines, and calculating totals.  
    - Outputs lines, deal_id, and summary.

20. **Connect Fetch Pricing Rules output to Calculate Quote Line Items.**

21. **Create Save Estimate Header Node (Supabase):**  
    - Table: `estimates`  
    - Fields: deal_id, subtotal, total_vat, currency (hardcoded 'USD' or 'EUR'), estimate_number (use execution.id).  
    - Enable retry on fail.

22. **Connect Calculate Quote Line Items output to Save Estimate Header.**

23. **Create Prepare Line Item Data Node (Set):**  
    - Assign: estimate_id (from Save Estimate Header), deal_id, lines (from Calculate Quote Line Items).

24. **Connect Save Estimate Header output to Prepare Line Item Data.**

25. **Create Split Out Node:**  
    - Field to split: `lines`  
    - Include additional fields as needed.

26. **Connect Prepare Line Item Data output to Split Out.**

27. **Create Insert Line Items Node (Supabase):**  
    - Table: `estimate_line_items`  
    - Fields: estimate_id, description, quantity, unit_price, vat_rate, sort_order, catalog_id.  
    - Execute once: false (batch insert)  
    - Enable retry on fail.

28. **Connect Split Out output to Insert Line Items.**

29. **Create Fetch Complete Quote Node (Supabase):**  
    - Table/View: `v_estimate_proforma`  
    - Filter: estimate_id = from Save Estimate Header output  
    - Return all results.

30. **Connect Insert Line Items output to Fetch Complete Quote.**

31. **Create Generate Email HTML Node (Code):**  
    - Input: Quote data from Fetch Complete Quote.  
    - Implement provided JS code that constructs a professional HTML email with company branding, detailed line items, financial summary, CTA buttons, validity, and legal disclaimers.

32. **Connect Fetch Complete Quote output to Generate Email HTML.**

33. **Create Send Email Node (Gmail):**  
    - Recipient: `email_to` from Generate Email HTML output  
    - Message body: `html` field from previous node  
    - Subject: `email_subject` from previous node  
    - Credentials: OAuth2 Gmail, configured with valid client ID and secret.

34. **Connect Generate Email HTML output to Send Email node.**

35. **Optional:** Create Manual Trigger and SQL Schema Generator (Code node) for testing and database setup purposes, but disabled by default.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Demo form URL for testing quote generation: https://form.jotform.com/252844786304060. Submit sample data to see the workflow in action.                                                                                                                                                                                                                                                                                                                                                                             | Demo form link from Sticky Note1                    |
| The workflow relies on Supabase for database storage, including tables for customers, deals, form submissions, service rules, estimates, and line items. A comprehensive SQL schema generator node is included (disabled by default) to set up or update the database schema in Supabase.                                                                                                                                                                                                                                 | Use SQL Schema Generator node for DB setup          |
| Pricing rules and form value mappings are configured in Supabase tables to enable dynamic, code-free updates to normalization and pricing logic. Adding new service types or price adjustments requires only database changes, not workflow edits.                                                                                                                                                                                                                                                                      | Key feature of rules-based pricing and normalization |
| Error handling nodes stop workflow execution with rich context to aid troubleshooting and maintain data integrity. Each critical write operation has failover paths implemented.                                                                                                                                                                                                                                                                                                                                       | Best practice implemented throughout the workflow   |
| The professional quote email uses a dark-themed HTML template with responsive design and includes CTA buttons for quote acceptance and scheduling discussions, enhancing customer experience and improving response rates.                                                                                                                                                                                                                                                                                             | Email template in Generate Email HTML node           |
| The workflow uses multiple retry strategies and wait times for Supabase writes to increase reliability in case of transient failures.                                                                                                                                                                                                                                                                                                                                                                              | Reliability enhancement                              |

---

**Disclaimer:** The text provided is derived solely from an automated workflow created using n8n, an integration and automation tool. The processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.