Create Stripe Invoices from Airtable Orders with Google Sheets Logging

https://n8nworkflows.xyz/workflows/create-stripe-invoices-from-airtable-orders-with-google-sheets-logging-8950


# Create Stripe Invoices from Airtable Orders with Google Sheets Logging

### 1. Workflow Overview

This n8n workflow automates the creation of invoices in Stripe from B2B orders sourced in Airtable and logs the invoice details into Google Sheets for tracking. It is designed primarily for businesses that manage orders manually but want to automate invoicing and record-keeping.

**Key Use Cases:**  
- B2B order invoice automation with manual payment capture  
- Periodic polling of new paid B2B orders for invoicing  
- Integration between Airtable (order source), Stripe (invoicing), and Google Sheets (logging)  

**Logical Blocks:**  
- **1.1 Trigger & Input Reception:** Scheduled polling of Airtable for new orders  
- **1.2 Order Filtering:** Filter only paid B2B orders  
- **1.3 Stripe Customer Creation:** Create or update customer records in Stripe  
- **1.4 Line Item Processing:** Prepare individual order line items for invoicing  
- **1.5 Stripe Invoice Creation & Finalization:** Create draft invoice and then finalize it for payment  
- **1.6 Data Formatting & Logging:** Format invoice data and append to Google Sheets for tracking  
- **1.7 Documentation & Setup Notes:** Sticky notes providing configuration guidance and explanations  

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

**Overview:**  
This block triggers the workflow every hour and fetches order data from Airtable.

**Nodes Involved:**  
- Hourly Trigger  
- Fetch B2B Order

**Node Details:**

- **Hourly Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution every hour automatically  
  - Config: Interval set to 1 hour  
  - Inputs: None  
  - Outputs: Connects to "Fetch B2B Order"  
  - Edge Cases: Timezone mismatches could cause unintended execution times; no input data to validate  
  - Version: 1.1  

- **Fetch B2B Order**  
  - Type: Airtable node  
  - Role: Retrieves a specific order record from Airtable using a hardcoded record ID  
  - Config:  
    - Base ID and Table ID set to specific Airtable base and orders table  
    - Record ID hardcoded ("rec5GbGenP8Wurf4W") - must be replaced per setup  
  - Credentials: Airtable Personal Access Token  
  - Inputs: From "Hourly Trigger"  
  - Outputs: Connects to "Filter B2B Paid Orders"  
  - Edge Cases:  
    - Invalid record ID or API credentials will cause fetch failure  
    - Network or API rate limits  
  - Version: 2.1  

#### 1.2 Order Filtering

**Overview:**  
Filters orders to only continue processing if the order is marked as paid and tagged as B2B.

**Nodes Involved:**  
- Filter B2B Paid Orders

**Node Details:**  

- **Filter B2B Paid Orders**  
  - Type: IF node  
  - Role: Conditional check on order data  
  - Config:  
    - Conditions: financial_status equals "paid" AND tags contains "B2B"  
  - Inputs: From "Fetch B2B Order"  
  - Outputs: On true, proceeds to "Create Stripe Customer"; false path not connected (stops workflow)  
  - Edge Cases:  
    - Missing or unexpected field names cause filter to fail or skip processing  
  - Version: 2  

#### 1.3 Stripe Customer Creation

**Overview:**  
Creates a Stripe customer from Airtable order customer data, handling existing customers gracefully.

**Nodes Involved:**  
- Create Stripe Customer

**Node Details:**  

- **Create Stripe Customer**  
  - Type: Stripe node  
  - Role: Create customer in Stripe using name, email, and phone from the order  
  - Config:  
    - Customer name from "Customer Name" field  
    - Email and Phone Number mapped from order fields  
    - continueOnFail enabled to continue if customer already exists (e.g., Stripe error for duplicate)  
  - Credentials: Stripe account API  
  - Inputs: From "Filter B2B Paid Orders"  
  - Outputs: To "Process Line Items"  
  - Edge Cases:  
    - API rate limits or invalid credentials  
    - Handling of existing customers relies on continueOnFail; may mask other errors  
  - Version: 1  

#### 1.4 Line Item Processing

**Overview:**  
Processes the order's line items for invoice creation, including fallback test data if no real items found.

**Nodes Involved:**  
- Process Line Items

**Node Details:**  

- **Process Line Items**  
  - Type: Code node (JavaScript)  
  - Role: Extracts line items from order data or generates test data if missing  
  - Logic:  
    - Attempts to access Shopify order data from execution context or input items  
    - If missing or empty, returns a single test product line item  
    - Maps each line item into structured objects with price, quantity, title, and customer ID for Stripe  
  - Inputs: From "Create Stripe Customer" (Stripe customer object)  
  - Outputs: To "Create Stripe Invoice"  
  - Edge Cases:  
    - Failure to access execution data or items (logged but does not stop workflow)  
    - Missing or malformed order line items triggers test data fallback  
  - Version: 2  

#### 1.5 Stripe Invoice Creation & Finalization

**Overview:**  
Creates a draft invoice in Stripe with 30-day payment terms, then finalizes it for payment and emailing.

**Nodes Involved:**  
- Create Stripe Invoice  
- Finalize Invoice

**Node Details:**  

- **Create Stripe Invoice**  
  - Type: HTTP Request node  
  - Role: Calls Stripe API to create an invoice for the customer  
  - Config:  
    - POST to https://api.stripe.com/v1/invoices  
    - Parameters: customer ID, currency, auto_advance=false, collection_method=send_invoice, days_until_due=30, pending_invoice_items_behavior=include  
    - Headers: Content-Type application/x-www-form-urlencoded  
  - Credentials: Stripe API  
  - Inputs: From "Process Line Items"  
  - Outputs: To "Finalize Invoice"  
  - Edge Cases:  
    - API errors (authentication, invalid customer, network)  
    - Currency mismatches or missing parameters  
  - Version: 4.1  

- **Finalize Invoice**  
  - Type: HTTP Request node  
  - Role: Finalizes the draft invoice to make it payable and trigger emails  
  - Config:  
    - POST to https://api.stripe.com/v1/invoices/{invoice_id}/finalize  
    - Parameter: auto_advance=true  
    - Headers: Content-Type application/x-www-form-urlencoded  
  - Credentials: Stripe API  
  - Inputs: From "Create Stripe Invoice"  
  - Outputs: To "Format Data for Sheets"  
  - Edge Cases:  
    - Invoice ID must be valid and present  
    - API errors or network issues  
  - Version: 4.1  

#### 1.6 Data Formatting & Logging

**Overview:**  
Formats the finalized invoice data into a clean structure and logs it into a Google Sheet for record keeping.

**Nodes Involved:**  
- Format Data for Sheets  
- Log to Google Sheets

**Node Details:**  

- **Format Data for Sheets**  
  - Type: Code node (JavaScript)  
  - Role: Converts raw Stripe invoice JSON into a sheet-friendly flat object  
  - Data includes invoice identifiers, customer info, monetary amounts converted from cents, URLs, timestamps, and Shopify order reference if available  
  - Inputs: From "Finalize Invoice"  
  - Outputs: To "Log to Google Sheets"  
  - Edge Cases:  
    - Missing optional fields handled gracefully (e.g., due_date, finalized_at)  
    - Shopify order ID populated only if available from previous nodes  
  - Version: 2  

- **Log to Google Sheets**  
  - Type: Google Sheets node  
  - Role: Appends or updates a row in a specified Google Sheet with formatted invoice data  
  - Config:  
    - Document ID and Sheet name set (must be updated per user)  
    - Columns mapped for invoice and customer details, amounts, URLs, dates, and timestamps  
  - Credentials: Google Sheets OAuth2 (authorized account)  
  - Inputs: From "Format Data for Sheets"  
  - Outputs: None (workflow ends here)  
  - Edge Cases:  
    - Authorization errors or insufficient access to sheet  
    - Incorrect document ID or sheet name causes failure  
  - Version: 4  

#### 1.7 Documentation & Setup Notes

**Overview:**  
Sticky notes distributed throughout the workflow provide detailed instructions, setup guidance, and explanations for each major step.

**Nodes Involved:**  
- Workflow Description  
- Schedule Setup Guide  
- Airtable Setup Guide  
- Order Filter Guide  
- Stripe Customer Guide  
- Line Items Guide  
- Invoice Creation Guide  
- Finalization Guide  
- Data Formatting Guide  
- Sheets Logging Guide

**Node Details:**  

- **Sticky Notes**  
  - Type: Sticky Note nodes  
  - Role: Provide human-readable explanations, setup instructions, and best practices  
  - Positioned near related functional nodes for contextual clarity  
  - Contain important warnings about replacing hardcoded IDs and credentials  
  - Include tips for testing, security, and customization  

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                            | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                          |
|-------------------------|-----------------------|--------------------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| Workflow Description    | Sticky Note           | Overview and setup instructions            | None                  | None                      | ## 🚀 B2B Invoice Automation Workflow... (full content)                                           |
| Schedule Setup Guide    | Sticky Note           | Explains schedule trigger configuration    | None                  | None                      | ## ⏰ Schedule Setup... (full content)                                                             |
| Hourly Trigger          | Schedule Trigger      | Triggers workflow hourly                    | None                  | Fetch B2B Order            | ## ⏰ Schedule Setup... (refer to above note)                                                      |
| Airtable Setup Guide    | Sticky Note           | Explains Airtable node configuration       | None                  | None                      | ## 📋 Airtable Setup... (full content)                                                             |
| Fetch B2B Order         | Airtable              | Fetches order data from Airtable            | Hourly Trigger        | Filter B2B Paid Orders     | ## 📋 Airtable Setup... (refer to above note)                                                     |
| Order Filter Guide      | Sticky Note           | Explains order filtering logic              | None                  | None                      | ## 🔍 Order Filtering... (full content)                                                            |
| Filter B2B Paid Orders  | IF                    | Filters paid B2B orders                      | Fetch B2B Order       | Create Stripe Customer     | ## 🔍 Order Filtering... (refer to above note)                                                    |
| Stripe Customer Guide   | Sticky Note           | Explains Stripe customer creation           | None                  | None                      | ## 👤 Stripe Customer Creation... (full content)                                                   |
| Create Stripe Customer  | Stripe                | Creates or updates Stripe customer          | Filter B2B Paid Orders| Process Line Items         | ## 👤 Stripe Customer Creation... (refer to above note)                                           |
| Line Items Guide        | Sticky Note           | Explains line item processing                | None                  | None                      | ## 📄 Line Item Processing... (full content)                                                       |
| Process Line Items      | Code                  | Extracts and prepares line items             | Create Stripe Customer| Create Stripe Invoice      | ## 📄 Line Item Processing... (refer to above note)                                               |
| Invoice Creation Guide  | Sticky Note           | Explains invoice creation in Stripe         | None                  | None                      | ## 🧾 Invoice Creation... (full content)                                                           |
| Create Stripe Invoice   | HTTP Request          | Creates draft invoice in Stripe              | Process Line Items    | Finalize Invoice           | ## 🧾 Invoice Creation... (refer to above note)                                                   |
| Finalization Guide      | Sticky Note           | Explains invoice finalization process        | None                  | None                      | ## ✅ Invoice Finalization... (full content)                                                       |
| Finalize Invoice        | HTTP Request          | Finalizes invoice to make it payable          | Create Stripe Invoice | Format Data for Sheets     | ## ✅ Invoice Finalization... (refer to above note)                                               |
| Data Formatting Guide   | Sticky Note           | Explains formatting invoice data for sheets | None                  | None                      | ## 📊 Data Formatting... (full content)                                                            |
| Format Data for Sheets  | Code                  | Formats invoice data for Google Sheets       | Finalize Invoice      | Log to Google Sheets       | ## 📊 Data Formatting... (refer to above note)                                                    |
| Sheets Logging Guide    | Sticky Note           | Explains Google Sheets logging setup         | None                  | None                      | ## 📋 Google Sheets Logging... (full content)                                                     |
| Log to Google Sheets    | Google Sheets         | Logs invoice data into a spreadsheet         | Format Data for Sheets| None                      | ## 📋 Google Sheets Logging... (refer to above note)                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Hourly Trigger" node:**  
   - Type: Schedule Trigger  
   - Set interval to every 1 hour  
   - Connect no inputs, output connects to next node

2. **Create "Fetch B2B Order" Airtable node:**  
   - Set operation to fetch a record by ID  
   - Configure:  
     - Base ID: your Airtable base ID  
     - Table ID: your Orders table ID  
     - Record ID: replace with actual order record ID to process  
   - Authenticate with Airtable Personal Access Token credentials  
   - Connect input from "Hourly Trigger"  
   - Output connects to filter node

3. **Create "Filter B2B Paid Orders" IF node:**  
   - Add two conditions with AND logic:  
     - `financial_status` equals "paid"  
     - `tags` contains "B2B"  
   - Input from "Fetch B2B Order"  
   - True output connects to Stripe customer creation node  
   - False output remains unconnected (stops workflow)

4. **Create "Create Stripe Customer" node:**  
   - Type: Stripe node  
   - Operation: Create customer  
   - Map:  
     - Name: from Airtable "Customer Name" field  
     - Email: from Airtable "Email" field  
     - Phone: from Airtable "Phone Number" field (optional)  
   - Enable "Continue On Fail" to handle existing customers gracefully  
   - Authenticate with Stripe API credentials  
   - Input from filter node  
   - Output connects to line item processing code node

5. **Create "Process Line Items" Code node:**  
   - Paste JavaScript code that:  
     - Attempts to extract Shopify order and line items from previous data  
     - If none found, returns a test product line item  
     - Maps line items to include customer ID, price, quantity, currency  
   - Input from Stripe customer creation node  
   - Output connects to Stripe invoice creation HTTP node

6. **Create "Create Stripe Invoice" HTTP Request node:**  
   - Method: POST  
   - URL: https://api.stripe.com/v1/invoices  
   - Body parameters (form encoded):  
     - customer: customer_id from previous node  
     - currency: from order data or default to USD  
     - auto_advance: false  
     - collection_method: send_invoice  
     - days_until_due: 30  
     - pending_invoice_items_behavior: include  
   - Headers: Content-Type application/x-www-form-urlencoded  
   - Authentication: Stripe API credentials  
   - Input from line item processing node  
   - Output connects to invoice finalization node

7. **Create "Finalize Invoice" HTTP Request node:**  
   - Method: POST  
   - URL: dynamically set to https://api.stripe.com/v1/invoices/{{ $json.id }}/finalize  
   - Body parameter: auto_advance=true  
   - Headers: Content-Type application/x-www-form-urlencoded  
   - Authentication: Stripe API credentials  
   - Input from "Create Stripe Invoice" node  
   - Output connects to data formatting code node

8. **Create "Format Data for Sheets" Code node:**  
   - Paste JavaScript code to convert Stripe invoice JSON into flat object with:  
     - Invoice identifiers, customer info, amounts (converted to dollars), URLs, timestamps  
     - Optional Shopify order ID if available  
   - Input from "Finalize Invoice" node  
   - Output connects to Google Sheets node

9. **Create "Log to Google Sheets" node:**  
   - Operation: Append or Update  
   - Document ID: set to your Google Sheets document ID  
   - Sheet Name / GID: set to your target sheet tab  
   - Map columns to the formatted data fields from previous node  
   - Authenticate with Google Sheets OAuth2 credentials  
   - Input from "Format Data for Sheets" node  
   - No further output needed

10. **Add Sticky Notes:**  
    - Place descriptive sticky notes near each functional block, including:  
      - Workflow Description with overview and instructions  
      - Schedule setup guide near the trigger  
      - Airtable setup guide near the Airtable node  
      - Order filtering explanation near the IF node  
      - Stripe customer creation notes  
      - Line items processing explanation  
      - Invoice creation and finalization guides  
      - Data formatting and Google Sheets logging instructions  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow triggers hourly but can be adjusted for different frequency. For testing, a manual trigger can be used first.                                            | Schedule Setup Guide Sticky Note                                                                |
| Airtable record IDs and API credentials must be securely stored and never committed to public repositories.                                                           | Airtable Setup Guide Sticky Note                                                                |
| Stripe API credentials require appropriate permissions for customer and invoice management.                                                                            | Stripe Customer Guide Sticky Note                                                               |
| The workflow gracefully handles missing line item data by creating a test product, useful for debugging.                                                             | Line Items Guide Sticky Note                                                                    |
| Invoice creation uses a 30-day payment term with manual finalization to allow review before sending.                                                                   | Invoice Creation Guide Sticky Note                                                              |
| Finalization triggers invoice number generation and makes the invoice payable, including email notifications if configured in Stripe.                                | Finalization Guide Sticky Note                                                                  |
| Google Sheets logging requires appropriate OAuth2 permissions and proper column headers matching the data fields for seamless appending or updating of records.       | Sheets Logging Guide Sticky Note                                                                |
| Original Shopify order data is optionally integrated if available, though this workflow primarily relies on Airtable as the order source.                              | Data Formatting Guide Sticky Note                                                               |
| For more information on Stripe invoice APIs, refer to https://stripe.com/docs/api/invoices                                                                            | General resource for Stripe integration                                                        |
| For Airtable API details, see https://airtable.com/api                                                                                                                | General resource for Airtable integration                                                       |
| To configure Google Sheets API access and OAuth2, see https://developers.google.com/sheets/api/guides/authorizing                                                    | General resource for Google Sheets integration                                                  |

---

**Disclaimer:**  
The provided content exclusively originates from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.