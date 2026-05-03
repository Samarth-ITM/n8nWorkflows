Generate Invoices and Send AI-Summarized Reminders with Jotform and QuickBooks

https://n8nworkflows.xyz/workflows/generate-invoices-and-send-ai-summarized-reminders-with-jotform-and-quickbooks-9752


# Generate Invoices and Send AI-Summarized Reminders with Jotform and QuickBooks

### 1. Workflow Overview

This workflow automates the end-to-end process of generating invoices and sending AI-summarized reminders for customers using data submitted from Jotform forms and integrating with QuickBooks Online (QBO). It is designed for service providers, freelancers, small businesses, and e-commerce sellers who need to streamline invoicing and follow-up reminders.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception and Data Formatting:** Receives form submissions from Jotform, extracts and formats customer, address, and product data.
- **1.2 Customer Verification and Management:** Checks if the customer exists in QuickBooks, updates existing customer details or creates a new customer accordingly.
- **1.3 Invoice Generation and Dispatch:** Retrieves the product/item details, creates an invoice for the customer, and sends the invoice via email.
- **1.4 Invoice Tracking and Storage:** Inserts invoice details into a database table for tracking reminders and payment status.
- **1.5 Scheduled Reminder Management:** Daily scheduled trigger to fetch invoices from the database, decide whether to send reminders, update reminder counts, or delete records.
- **1.6 AI-Powered Reminder Summarization:** Uses an AI agent to generate a professional summary of reminders sent during the day and emails the summary to the sales/finance team.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Formatting

- **Overview:** This block receives the Jotform submission via webhook and formats the raw data into structured objects that can be consumed by downstream nodes.
- **Nodes Involved:**  
  - Receive form submission (Webhook)  
  - Format data (Code)  
  - Sticky Note: "Receive Submission", "Format Data"

- **Node Details:**

  - **Receive form submission**  
    - Type: Webhook  
    - Role: Entry point triggered by Jotform form submission via HTTP POST at path `/requests`.  
    - Configuration: Accepts POST requests; no additional options configured.  
    - Inputs: External HTTP request.  
    - Outputs: Raw JSON payload from Jotform containing customer info, item name, and billing address.  
    - Edge Cases: Network failures, invalid payloads, malformed JSON.  
    - Version: 2.1

  - **Format data**  
    - Type: Code (JavaScript)  
    - Role: Parses billing address from HTML-formatted string and extracts customer name, email, phone, and item name into structured JSON.  
    - Configuration: Custom JS regex to extract address fields line1, line2, city, state/province, postal code, country.  
    - Inputs: Raw JSON from webhook.  
    - Outputs: Structured JSON with `address`, `customer`, and `item` objects.  
    - Edge Cases: Missing or malformed address field may result in null values.  
    - Version: 2

---

#### 2.2 Customer Verification and Management

- **Overview:** Checks if the customer exists in QuickBooks by email, updates billing details if found, or creates a new customer otherwise.
- **Nodes Involved:**  
  - Check if the customer exists (QuickBooks)  
  - If (Conditional)  
  - Update the customer (QuickBooks)  
  - Create the customer (QuickBooks)  
  - Add customer id (Code)  
  - Sticky Notes: "Check If Customer exists", "Customer Exists", "Customer Doesn't Exist", "Add Customer Id"

- **Node Details:**

  - **Check if the customer exists**  
    - Type: QuickBooks API node  
    - Role: Search QBO customers by primary email address from formatted data.  
    - Configuration: `getAll` with filter `Where PrimaryEmailAddr = '{{ $json.customer.email }}'`, limit 1.  
    - Inputs: Structured data from Format data node.  
    - Outputs: Customer record if exists, empty otherwise.  
    - Edge Cases: API rate limits, auth errors, no customer found.  
    - Version: 1

  - **If**  
    - Type: Conditional node  
    - Role: Branch workflow based on whether customer exists.  
    - Configuration: Checks if returned JSON object exists (`$json`) and if `$json.Id` exists.  
    - Inputs: Customer search result.  
    - Outputs: True branch (customer exists), False branch (customer does not exist).  
    - Edge Cases: Unexpected data structure, missing Id.  
    - Version: 2.2

  - **Update the customer**  
    - Type: QuickBooks API node  
    - Role: Update existing customer's billing address, name, and phone.  
    - Configuration: `update` operation using customer ID from QBO; billing address and contact details sourced from formatted data.  
    - Inputs: True branch from If node.  
    - Outputs: Updated customer record.  
    - Edge Cases: Conflicts on update, auth errors.  
    - Version: 1

  - **Create the customer**  
    - Type: QuickBooks API node  
    - Role: Create new customer with billing address and contact details.  
    - Configuration: `create` operation with details from formatted data.  
    - Inputs: False branch from If node.  
    - Outputs: New customer record with assigned ID.  
    - Edge Cases: Duplicate customer creation, invalid data.  
    - Version: 1

  - **Add customer id**  
    - Type: Code (JavaScript)  
    - Role: Enhances data to include the customer ID from QBO for further processing.  
    - Configuration: Merges customer ID into existing data JSON.  
    - Inputs: Output from Update or Create customer.  
    - Outputs: JSON including customer ID for next steps.  
    - Edge Cases: Missing Id property.  
    - Version: 2

---

#### 2.3 Invoice Generation and Dispatch

- **Overview:** Retrieves the product/item details from QuickBooks, creates the invoice for the customer, and sends the invoice via email.
- **Nodes Involved:**  
  - Get the product (QuickBooks)  
  - Add item id (Code)  
  - Create the invoice (QuickBooks)  
  - Send the invoice (QuickBooks)  
  - Sticky Notes: "Get The Item", "Add Item Id", "Create The Invoice", "Send The Invoice"

- **Node Details:**

  - **Get the product**  
    - Type: QuickBooks API node  
    - Role: Fetch product/item by name from QBO to get item ID for invoice line.  
    - Configuration: `getAll` with filter `WHERE name = '{{ $json.item.name }}'`, limit 1.  
    - Inputs: Data including item name and customer ID.  
    - Outputs: Product/item record with ID.  
    - Edge Cases: No matching product, multiple matches, API errors.  
    - Version: 1

  - **Add item id**  
    - Type: Code (JavaScript)  
    - Role: Adds item ID to the data object for invoice creation.  
    - Configuration: Merges item ID from product query into JSON.  
    - Inputs: Output from Get the product.  
    - Outputs: JSON enriched with item ID.  
    - Edge Cases: Missing Id property.  
    - Version: 2

  - **Create the invoice**  
    - Type: QuickBooks API node  
    - Role: Creates invoice for customer with the selected item.  
    - Configuration: `create` operation with line item referencing item ID, quantity 1, description "Jotform submission". Customer reference set from customer ID.  
    - Inputs: Data with customer ID and item ID.  
    - Outputs: Newly created invoice record including invoice ID.  
    - Edge Cases: Invalid item or customer ID, QBO API errors.  
    - Version: 1

  - **Send the invoice**  
    - Type: QuickBooks API node  
    - Role: Sends the created invoice via email to the customer's email address.  
    - Configuration: Uses email from added item id node's customer data. Invoice ID from create invoice node.  
    - Inputs: Invoice record including invoice ID and customer email.  
    - Outputs: Confirmation of sent invoice.  
    - Edge Cases: Invalid email, mail server issues, QBO send operation errors.  
    - Version: 1

---

#### 2.4 Invoice Tracking and Storage

- **Overview:** Inserts invoice details including ID, remaining balance, currency, and reminders sent count into a data table for later reminder processing.
- **Nodes Involved:**  
  - Add reminders config (Set)  
  - Insert invoice id to DB (DataTable)  
  - Sticky Notes: "Insert Invoice To DB", "Add Reminders Config"

- **Node Details:**

  - **Add reminders config**  
    - Type: Set  
    - Role: Sets configuration values for reminder intervals and data table ID.  
    - Configuration: JSON output with empty `dataTableId` (to be set by user), reminder intervals `[2,3,5]`, and a boolean `isInvoiceTrigger` to detect trigger source.  
    - Inputs: Output from Send the invoice node.  
    - Outputs: Config object for subsequent nodes.  
    - Edge Cases: Missing or incorrect data table ID.  
    - Version: 3.4

  - **Insert invoice id to DB**  
    - Type: Data Table  
    - Role: Inserts invoice details into a configured data table for tracking.  
    - Configuration: Columns `invoiceId`, `remainingAmount`, `currency`, `remindersSent` initialized to 0. Data mapped from invoice send node output.  
    - Inputs: Invoice send success output, Add reminders config node for data table ID.  
    - Outputs: Confirmation of insert.  
    - Edge Cases: Data table access errors, duplicate keys.  
    - Version: 1

---

#### 2.5 Scheduled Reminder Management

- **Overview:** Daily at 8 AM, triggers a process to fetch invoices from the database, check their payment status and reminder intervals, and decide whether to send reminders, skip, or delete invoices from tracking.
- **Nodes Involved:**  
  - Schedule reminders trigger (Schedule Trigger)  
  - Get Invoices (DataTable)  
  - Loop over invoices (SplitInBatches)  
  - Get the invoice (QuickBooks)  
  - Switch (Conditional)  
  - Send reminder email (Email Send)  
  - Increase sent reminders (DataTable)  
  - If3 (Conditional)  
  - Delete invoice (DataTable)  
  - Sticky Notes: "Schedule Trigger", "Get All Invoices", "Loop Over Invoices", "Get Invoice Details", "Send Reminders Logic"

- **Node Details:**

  - **Schedule reminders trigger**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow daily at 8 AM to send reminders.  
    - Configuration: Interval trigger set to hour 8 daily.  
    - Inputs: None (time-based trigger).  
    - Outputs: Trigger signal to start reminder process.  
    - Edge Cases: Trigger misfires, time zone issues.  
    - Version: 1.2

  - **Get Invoices**  
    - Type: Data Table  
    - Role: Fetch all invoices from the configured data table for processing.  
    - Configuration: Get all entries in configured `dataTableId`.  
    - Inputs: Trigger from schedule node.  
    - Outputs: List of invoice records with reminder statuses.  
    - Edge Cases: Empty table, DB access issues.  
    - Version: 1

  - **Loop over invoices**  
    - Type: SplitInBatches  
    - Role: Processes invoices one by one to avoid rate limits and handle each individually.  
    - Configuration: Default batch size (1).  
    - Inputs: List of invoices.  
    - Outputs: Single invoice per iteration for downstream nodes.  
    - Edge Cases: Large data sets may increase processing time.  
    - Version: 3

  - **Get the invoice**  
    - Type: QuickBooks API  
    - Role: Retrieves the current invoice details from QBO to check payment and status.  
    - Configuration: `get` operation by invoice ID.  
    - Inputs: Single invoice record from Data Table.  
    - Outputs: Full invoice details including balance and dates.  
    - Edge Cases: Invoice deleted or inaccessible, API errors.  
    - Version: 1

  - **Switch**  
    - Type: Conditional node  
    - Role: Determines the action for each invoice based on balance and reminder schedule.  
    - Configuration:  
      - `send now`: If balance > 0 and the current date matches next reminder date calculated by adding reminder intervals to last sent date.  
      - `already paid`: If balance == 0.  
      - `send later`: Default fallback.  
    - Inputs: Output from Get the invoice.  
    - Outputs: Branches for send now, delete (already paid), or loop again (send later).  
    - Edge Cases: Date calculation errors, missing fields.  
    - Version: 3.3

  - **Send reminder email**  
    - Type: Email Send  
    - Role: Sends a professionally formatted reminder email to the customer about due invoice.  
    - Configuration: Uses SMTP credentials with HTML email template that includes invoice number, due date, and payment link.  
    - Inputs: Invoice details from Switch branch "send now".  
    - Outputs: Confirmation of email sent.  
    - Edge Cases: SMTP failures, invalid email addresses.  
    - Version: 2.1

  - **Increase sent reminders**  
    - Type: Data Table  
    - Role: Updates the reminders sent count and last sent timestamp in the DB after sending reminder.  
    - Configuration: Updates fields `remindersSent` (incremented by 1) and `lastSentAt` (current timestamp).  
    - Inputs: After sending email.  
    - Outputs: Confirmation of update.  
    - Edge Cases: Concurrent updates, DB errors.  
    - Version: 1

  - **If3**  
    - Type: Conditional node  
    - Role: Checks if the maximum number of reminders has been sent (based on configured intervals length).  
    - Configuration: Compares `remindersSent` with length of reminder intervals.  
    - Inputs: Updated reminder count.  
    - Outputs: True branch deletes invoice from DB, False continues looping.  
    - Edge Cases: Off-by-one errors.  
    - Version: 2.2

  - **Delete invoice**  
    - Type: Data Table  
    - Role: Deletes invoice record from DB when fully paid or all reminders sent.  
    - Configuration: Delete rows matching invoice ID.  
    - Inputs: From If3 true branch or Switch branch "already paid".  
    - Outputs: Confirmation of deletion.  
    - Edge Cases: Delete failures, missing keys.  
    - Version: 1

---

#### 2.6 AI-Powered Reminder Summarization

- **Overview:** This block collects all reminders sent today, uses an AI agent to generate an HTML summary email, and sends that summary to a designated team email.
- **Nodes Involved:**  
  - Get today's sent reminders (Data Table)  
  - AI Agent (Langchain Agent)  
  - OpenAI Chat Model (LM Chat OpenAI)  
  - Send reminders sent summary (Email Send)  
  - Sticky Notes: "Get Sent Reminders", "Summarize Sent Reminders & Send An Email"

- **Node Details:**

  - **Get today's sent reminders**  
    - Type: Data Table  
    - Role: Fetches all reminders sent today by filtering `lastSentAt` field to today’s date.  
    - Configuration: Filter condition on `lastSentAt` >= start of current day (UTC).  
    - Inputs: Loop over invoices or schedule trigger.  
    - Outputs: List of invoice reminder records sent today.  
    - Edge Cases: Timezone inconsistencies, empty results.  
    - Version: 1

  - **OpenAI Chat Model**  
    - Type: Language Model (OpenAI GPT)  
    - Role: Processes the reminder data and generates textual summary content.  
    - Configuration: Uses GPT-4o-mini model with no special options.  
    - Inputs: JSON array of reminders from "Get today's sent reminders".  
    - Outputs: Text summary (HTML).  
    - Credentials: Requires valid OpenAI API credentials.  
    - Edge Cases: API rate limits, network issues, malformed data.  
    - Version: 1.2

  - **AI Agent**  
    - Type: Langchain Agent (n8n built-in)  
    - Role: Uses the output of the OpenAI model and formats it with system prompt for a professional HTML email summarizing reminders sent.  
    - Configuration: System message defines input data fields, goal, formatting guidelines, and example output.  
    - Inputs: Output of OpenAI Chat Model.  
    - Outputs: Final HTML email content in `output` property.  
    - Edge Cases: AI generating incomplete or malformed HTML.  
    - Version: 2.2

  - **Send reminders sent summary**  
    - Type: Email Send  
    - Role: Sends the AI-generated daily reminder summary email to the sales or finance team.  
    - Configuration: SMTP credentials, subject line "Summary of today's reminders sent", HTML body from AI Agent output, recipient email `sales@example.com`.  
    - Inputs: Output of AI Agent.  
    - Outputs: Email send confirmation.  
    - Edge Cases: SMTP failures, wrong recipient email.  
    - Version: 2.1

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                                  | Input Node(s)                        | Output Node(s)                       | Sticky Note                                                                                     |
|---------------------------|-------------------------------|-------------------------------------------------|------------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| Receive form submission    | Webhook                      | Receives Jotform form submission                 | -                                  | Format data                        | ## Receive Submission<br>Receives the product/service form submission from Jotform              |
| Format data               | Code                         | Parses and structures address and customer data | Receive form submission             | Check if the customer exists       | ## Format Data<br>Formats the data thus making it easier to be used in other nodes              |
| Check if the customer exists | QuickBooks                  | Checks if customer exists in QBO                  | Format data                        | If                               | ## Check If Customer exists <br>Checks if the customer exists in QBO or not                    |
| If                        | If                           | Branches logic based on customer existence       | Check if the customer exists        | Update the customer, Create the customer |                                                                                              |
| Update the customer        | QuickBooks                   | Updates existing customer details                 | If (true)                         | Add customer id                   | ## Customer Exists<br>Now since the customer exists we will update the customer details        |
| Create the customer        | QuickBooks                   | Creates a new customer in QBO                      | If (false)                        | Add customer id                   | ## Customer Doesn't Exist<br>Now since the customer doesn't exist we will create new customer  |
| Add customer id            | Code                         | Adds customer ID to data                           | Update the customer, Create the customer | Get the product                  | ## Add Customer Id<br>Adds customer id to the data                                             |
| Get the product            | QuickBooks                   | Retrieves product/item details from QBO           | Add customer id                   | Add item id                      | ## Get The Item<br>Gets the selected product/service from QBO                                  |
| Add item id                | Code                         | Adds item ID to data                              | Get the product                   | Create the invoice               | ## Add Item Id<br>Adds item (service/product) id to the data                                   |
| Create the invoice         | QuickBooks                   | Creates invoice for customer                        | Add item id                      | Send the invoice                | ## Create The Invoice<br>Creates a new invoice for that customer                              |
| Send the invoice           | QuickBooks                   | Emails the created invoice to customer             | Create the invoice               | Add reminders config            | ## Send The Invoice<br>Sends the newly created invoice for that customer(via email)           |
| Add reminders config       | Set                          | Sets reminder intervals and data table ID         | Send the invoice                | If2                            | ## Add Reminders Config<br>Adds reminders config details like intervals                        |
| If2                       | If                           | Checks trigger source to branch logic              | Add reminders config             | Insert invoice id to DB, Get Invoices |                                                                                              |
| Insert invoice id to DB    | DataTable                    | Inserts invoice details for reminder tracking      | If2 (true)                      | -                              | ## Insert Invoice To DB<br>Inserts newly created invoice needed details to DB                  |
| Get Invoices              | DataTable                    | Retrieves all invoices from DB                      | If2 (false)                     | Loop over invoices              | ## Get All Invoices<br>Gets all the invoices from DB                                          |
| Loop over invoices         | SplitInBatches               | Processes invoices one by one                        | Get Invoices                    | Get today's sent reminders, Get the invoice | ## Loop Over Invoices<br>Loops over invoices one by one                                        |
| Get today's sent reminders | DataTable                   | Fetches reminders sent today                         | Loop over invoices              | AI Agent                       | ## Get Sent Reminders<br>Gets today's sent reminders from DB                                  |
| Get the invoice            | QuickBooks                   | Retrieves invoice details for current invoice       | Loop over invoices              | Switch                        | ## Get Invoice Details<br>Gets the invoice details from QBO                                  |
| Switch                    | Switch                       | Decides whether to send reminder, skip, or delete  | Get the invoice                | Send reminder email, Delete invoice, Loop over invoices | ## Send Reminders Logic<br>The logic that decides whether or not to send a reminder email now |
| Send reminder email        | Email Send                   | Sends reminder email to customer                     | Switch ("send now")             | Increase sent reminders         |                                                                                                |
| Increase sent reminders    | DataTable                    | Updates reminders sent count and timestamp          | Send reminder email             | If3                           |                                                                                                |
| If3                       | If                           | Checks if max reminders sent to delete or continue  | Increase sent reminders         | Delete invoice, Loop over invoices |                                                                                                |
| Delete invoice             | DataTable                    | Deletes invoice record from DB                        | If3 (true), Switch ("already paid") | Loop over invoices              |                                                                                                |
| AI Agent                  | Langchain Agent              | Generates HTML summary email of reminders sent      | Get today's sent reminders      | Send reminders sent summary     | ## Summarize Sent Reminders & Send An Email<br>Summarizes today's sent reminders using AI      |
| OpenAI Chat Model          | LM Chat OpenAI               | Language model for reminder summary generation       | AI Agent                      | AI Agent                      |                                                                                                |
| Send reminders sent summary | Email Send                 | Sends AI-generated reminder summary to team          | AI Agent                      | -                              |                                                                                                |
| Schedule reminders trigger | Schedule Trigger             | Triggers reminder sending process daily at 8 AM     | -                            | Add reminders config            | ## Schedule Trigger<br>Schedules reminders trigger daily at 8 AM                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Receive form submission`  
   - Type: Webhook (HTTP POST)  
   - Path: `requests`  
   - Purpose: Receive Jotform form submissions.  

2. **Create Code Node**  
   - Name: `Format data`  
   - Type: Code (JavaScript)  
   - Purpose: Parse the billing address and extract customer and item info.  
   - Code Snippet: Extract address fields using regex from the `billingAddress` HTML string; map name, email, phone, and item name from incoming JSON.  

3. **Create QuickBooks Node**  
   - Name: `Check if the customer exists`  
   - Type: QuickBooks (getAll)  
   - Operation: Search customers by `PrimaryEmailAddr` equal to formatted customer email.  
   - Limit: 1  

4. **Create Conditional Node**  
   - Name: `If`  
   - Type: If  
   - Condition: Check if customer record exists (`$json` exists and `$json.Id` exists).  

5. **Create QuickBooks Node**  
   - Name: `Update the customer`  
   - Type: QuickBooks (update)  
   - Customer ID: From `$json.Id`  
   - Fields: Update billing address, name, phone from formatted data.  

6. **Create QuickBooks Node**  
   - Name: `Create the customer`  
   - Type: QuickBooks (create)  
   - Fields: Set billing address, name, email, phone from formatted data.  

7. **Create Code Node**  
   - Name: `Add customer id`  
   - Type: Code  
   - Purpose: Merge customer ID from output of Create or Update customer into JSON for further use.  

8. **Create QuickBooks Node**  
   - Name: `Get the product`  
   - Type: QuickBooks (getAll)  
   - Operation: Search items by name equal to formatted item name.  
   - Limit: 1  

9. **Create Code Node**  
   - Name: `Add item id`  
   - Type: Code  
   - Purpose: Merge item ID from Get the product node output into JSON.  

10. **Create QuickBooks Node**  
    - Name: `Create the invoice`  
    - Type: QuickBooks (create)  
    - Resource: Invoice  
    - Fields: Line item includes the item ID, amount 1, description "Jotform submission", customer reference from customer ID.  

11. **Create QuickBooks Node**  
    - Name: `Send the invoice`  
    - Type: QuickBooks (send)  
    - Invoice ID: From Create the invoice output  
    - Email: Customer email from previous nodes  

12. **Create Set Node**  
    - Name: `Add reminders config`  
    - Type: Set  
    - JSON Output:  
      ```json
      {
        "dataTableId": "<YOUR_DATA_TABLE_ID>",
        "reminderIntervalsInDays": [2, 3, 5],
        "isInvoiceTrigger": true
      }
      ```  
    - Note: Replace `<YOUR_DATA_TABLE_ID>` with the actual data table ID.  

13. **Create Conditional Node**  
    - Name: `If2`  
    - Type: If  
    - Condition: Check if `isInvoiceTrigger` is true to branch logic.  

14. **Create Data Table Node**  
    - Name: `Insert invoice id to DB`  
    - Type: Data Table (insert)  
    - Columns: `invoiceId`, `remainingAmount`, `currency`, `remindersSent` (0)  
    - DataTable ID: From reminders config node  

15. **Create Data Table Node**  
    - Name: `Get Invoices`  
    - Type: Data Table (get all)  
    - DataTable ID: From reminders config node  

16. **Create SplitInBatches Node**  
    - Name: `Loop over invoices`  
    - Type: SplitInBatches  
    - Purpose: Process invoices one by one  

17. **Create Data Table Node**  
    - Name: `Get today's sent reminders`  
    - Type: Data Table (get)  
    - Filter: `lastSentAt` >= start of today (UTC)  
    - DataTable ID: From reminders config node  

18. **Create QuickBooks Node**  
    - Name: `Get the invoice`  
    - Type: QuickBooks (get)  
    - Invoice ID: From current invoice in loop  

19. **Create Switch Node**  
    - Name: `Switch`  
    - Conditions:  
      - `send now`: Invoice balance > 0 and today matches reminder interval date  
      - `already paid`: Invoice balance == 0  
      - `send later`: default  

20. **Create Email Send Node**  
    - Name: `Send reminder email`  
    - Type: Email Send  
    - SMTP Credentials: Configure with your SMTP server  
    - To: Customer email from invoice data  
    - Subject: Friendly invoice reminder with invoice number  
    - HTML: Professional email template with invoice details and payment link  

21. **Create Data Table Node**  
    - Name: `Increase sent reminders`  
    - Type: Data Table (update)  
    - Increment remindersSent by 1, update lastSentAt timestamp to now  
    - Filter by invoiceId  

22. **Create Conditional Node**  
    - Name: `If3`  
    - Type: If  
    - Condition: `remindersSent` >= length of reminder intervals array  

23. **Create Data Table Node**  
    - Name: `Delete invoice`  
    - Type: Data Table (delete)  
    - Filter by invoice ID  

24. **Create Schedule Trigger Node**  
    - Name: `Schedule reminders trigger`  
    - Type: Schedule Trigger  
    - Schedule: Daily at 8 AM  

25. **Create Langchain OpenAI Node**  
    - Name: `OpenAI Chat Model`  
    - Type: Language Model Chat (OpenAI GPT)  
    - Model: `gpt-4o-mini` or similar  
    - Credentials: OpenAI API credentials  

26. **Create Langchain Agent Node**  
    - Name: `AI Agent`  
    - Type: Langchain Agent  
    - System Prompt: Defines input format and output HTML email summary structure for reminders sent today.  
    - Input: Output from `Get today's sent reminders` node.  

27. **Create Email Send Node**  
    - Name: `Send reminders sent summary`  
    - Type: Email Send  
    - SMTP Credentials: Same or different SMTP server  
    - To: Team email (e.g., sales@example.com)  
    - Subject: Summary of today's reminders sent  
    - HTML Body: Output from AI Agent node  

28. **Connect nodes as per logical flow:**  
    - From `Receive form submission` → `Format data` → `Check if the customer exists` → `If` branch → `Update the customer` or `Create the customer` → `Add customer id` → `Get the product` → `Add item id` → `Create the invoice` → `Send the invoice` → `Add reminders config` → `If2` → insert to DB or get invoices → loop → get invoice → switch → send email or delete → update reminders or delete → end.  
    - The scheduled trigger starts from `Schedule reminders trigger` → `Add reminders config` → `If2` (false branch) and follows the reminder sending logic.  
    - AI summarization occurs post fetching today's sent reminders and sends summary email.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow automates the complete invoicing and reminder process integrating Jotform and QuickBooks Online, suitable for freelancers, small businesses, and service providers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow overview and audience description                                                        |
| Requires setting up Jotform webhook with path `/requests` to send form submissions to n8n. More info: https://www.jotform.com/help/245-how-to-setup-a-webhook-with-jotform/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Jotform webhook setup                                                                              |
| QuickBooks Online OAuth2 credentials are required with appropriate API permissions. More info: https://developer.intuit.com/app/developer/qbo/docs/get-started/get-client-id-and-client-secret                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | QuickBooks Online API setup                                                                        |
| SMTP credentials must be configured for sending reminder and summary emails; example uses Mailtrap for testing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | SMTP/email setup                                                                                   |
| A data table with columns `invoiceId` (string), `remainingAmount` (number), `currency` (string), `remindersSent` (number), `lastSentAt` (date time) is required for storing invoice reminder tracking data.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Database table for invoice tracking                                                               |
| Update the `Add reminders config` node to specify the correct data table ID and reminder interval days (default: 2, 3, 5).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Reminder configuration                                                                             |
| The AI agent uses OpenAI GPT models to generate a professional summary of reminders sent daily, which can be customized in the system prompt.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | AI summarization using Langchain and OpenAI GPT                                                   |
| The workflow includes detailed sticky notes explaining each logical block and node purpose to facilitate maintenance and modification.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Internal documentation within workflow                                                           |

---

**Disclaimer:** The text analyzed and documented here comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.