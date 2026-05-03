Automated Invoice Generation & Payment Reminders with QuickBooks, Jotform & GPT-4o

https://n8nworkflows.xyz/workflows/automated-invoice-generation---payment-reminders-with-quickbooks--jotform---gpt-4o-9760


# Automated Invoice Generation & Payment Reminders with QuickBooks, Jotform & GPT-4o

### 1. Workflow Overview

This workflow automates the entire process of invoice generation and payment reminders by integrating Jotform (for order submissions), QuickBooks Online (QBO) (for customer and invoice management), and Gmail (for sending emails). It is designed for small businesses, freelancers, and service providers who want to streamline billing and follow-up communications.

The workflow is logically divided into two main parts:

- **1.1 Order Processing & Invoice Creation:**  
  Starting from receiving a Jotform submission, it processes the order by verifying or creating customer records in QuickBooks, retrieving product details, creating invoices, sending them via email, and storing invoice metadata in a database for reminder tracking.

- **1.2 Scheduled Payment Reminder System:**  
  A daily scheduled trigger initiates the retrieval of invoices from the DB, checks payment statuses via QuickBooks, decides whether to send reminders, skip, or delete records, sends reminder emails to customers, updates reminder counts, and summarizes reminders sent via an AI-generated email to the internal team.

---

### 2. Block-by-Block Analysis

#### 2.1 Order Processing & Invoice Creation

**Overview:**  
This block handles incoming form submissions from Jotform, formats and extracts necessary data, verifies if the customer exists in QuickBooks, creates or updates customer records, retrieves the purchased product, creates an invoice, sends the invoice email, and finally stores invoice details in a database.

**Nodes Involved:**  
- Receive form submission  
- Format data  
- Check if the customer exists  
- If (decision node)  
- Update the customer  
- Create the customer  
- Add customer id  
- Get the product  
- Add item id  
- Create the invoice  
- Send the invoice  
- Add reminders config  
- Insert invoice id to DB  

**Node Details:**

- **Receive form submission**  
  - *Type:* Webhook  
  - *Role:* Entry point triggered by Jotform POST submission  
  - *Config:* POST method with unique webhook path  
  - *Input:* Incoming HTTP request from Jotform  
  - *Output:* Raw JSON containing customer and order data  
  - *Failures:* Missing or malformed webhook data, network errors  

- **Format data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses and extracts structured customer and address data from raw form submission fields  
  - *Key expressions:* Regex to parse multi-line billing address string into object fields (line1, city, postal code, etc.)  
  - *Input:* Raw JSON from webhook  
  - *Output:* Structured JSON with `address`, `customer`, and `item` objects  
  - *Failures:* Regex mismatch if form address format changes  

- **Check if the customer exists**  
  - *Type:* QuickBooks - GetAll Items  
  - *Role:* Queries QuickBooks to find if a customer exists by matching email address  
  - *Config:* Filter query `Where PrimaryEmailAddr = '{{ $json.customer.email }}'`  
  - *Input:* Formatted data with customer email  
  - *Output:* Customer record if found, empty if not  
  - *Failures:* API auth errors, rate limits, empty results  

- **If (Customer Exists?)**  
  - *Type:* If Node  
  - *Role:* Branching based on whether customer record was found  
  - *Condition:* Checks if customer data exists and has an Id field  
  - *Output:*  
    - True branch: Update customer  
    - False branch: Create customer  

- **Update the customer**  
  - *Type:* QuickBooks - Update  
  - *Role:* Updates existing customer’s billing address, name, and contact details  
  - *Input:* Customer Id and updated fields from formatted data  
  - *Failures:* Invalid customer Id, API errors  

- **Create the customer**  
  - *Type:* QuickBooks - Create  
  - *Role:* Creates a new customer record with billing address and contact info  
  - *Input:* Fields populated from formatted data  
  - *Failures:* Validation errors, duplicate customers, API issues  

- **Add customer id**  
  - *Type:* Code (JavaScript)  
  - *Role:* Adds the QuickBooks customer Id to the existing JSON data for further processing  
  - *Input:* Customer record from previous node  
  - *Output:* JSON with nested customer id included  

- **Get the product**  
  - *Type:* QuickBooks - Get All Items  
  - *Role:* Retrieves product/service details from QuickBooks matching the submitted item name  
  - *Input:* Item name from JSON  
  - *Output:* Product record including Id  
  - *Failures:* Product not found, API errors  

- **Add item id**  
  - *Type:* Code (JavaScript)  
  - *Role:* Adds the QuickBooks item Id to the existing JSON data  
  - *Input:* Product record  
  - *Output:* JSON including item id  

- **Create the invoice**  
  - *Type:* QuickBooks - Create Invoice  
  - *Role:* Creates an invoice in QuickBooks for the customer, with one line item using the product Id  
  - *Input:* Customer Id, item Id, amount, description  
  - *Failures:* Invalid references, API errors  

- **Send the invoice**  
  - *Type:* QuickBooks - Send Invoice  
  - *Role:* Emails the newly created invoice to the customer's email address  
  - *Input:* Invoice Id and customer email  
  - *Failures:* Email delivery errors, missing email, API issues  

- **Add reminders config**  
  - *Type:* Set  
  - *Role:* Defines parameters for reminders such as DB table ID and reminder intervals (in days)  
  - *Input:* None (static or conditional)  
  - *Output:* JSON with config fields for use in reminders block  

- **Insert invoice id to DB**  
  - *Type:* Data Table (Database)  
  - *Role:* Inserts invoice metadata into a database table to track reminders and payment balances  
  - *Input:* Invoice Id, balance, currency, reminders sent count initialized to zero  
  - *Failures:* DB connection, schema mismatch  

---

#### 2.2 Scheduled Payment Reminder System

**Overview:**  
This block runs daily at 8 AM to manage payment reminders. It retrieves invoice records from the database, iterates through each invoice, checks its current status in QuickBooks, decides whether to send a reminder email, skip sending, or delete the record if fully paid or reminders exhausted. It also increments reminder counts and sends a summary email generated by AI to the internal team.

**Nodes Involved:**  
- Schedule reminders trigger  
- Add reminders config  
- If2 (Check trigger source)  
- Insert invoice id to DB (path reuse)  
- Get Invoices (DB fetch)  
- Loop over invoices (split batch)  
- Get today's sent reminders (DB filter)  
- Get the invoice (QuickBooks)  
- Switch (Reminder decision logic)  
- Send reminder email  
- Increase sent reminders (DB update)  
- If3 (Check if reminders exhausted)  
- Delete invoice (DB delete)  
- AI Agent (LangChain)  
- OpenAI Chat Model  
- Send reminders sent summary (Gmail)  

**Node Details:**

- **Schedule reminders trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the reminder workflow daily at 8 AM server time  
  - *Failures:* Timezone issues, missed schedule  

- **Add reminders config**  
  - *Type:* Set  
  - *Role:* Provides DB table ID and reminder intervals to control timing for reminders  
  - *Input:* None (static or conditional)  
  - *Output:* JSON config  

- **If2 (Check trigger source)**  
  - *Type:* If  
  - *Role:* Determines if workflow was triggered by schedule or invoice creation path  
  - *Use:* To branch either to DB insert or to get invoices for reminders  

- **Get Invoices**  
  - *Type:* Data Table (DB)  
  - *Role:* Fetches all invoice records for reminder processing  
  - *Output:* List of invoices with metadata  

- **Loop over invoices**  
  - *Type:* Split in Batches  
  - *Role:* Processes invoices individually in batches for API calls and decision logic  

- **Get today's sent reminders**  
  - *Type:* Data Table (DB)  
  - *Role:* Retrieves reminders sent on the current day for summary and tracking  
  - *Parameters:* Filters by lastSentAt date >= start of today UTC  

- **Get the invoice**  
  - *Type:* QuickBooks - Get Invoice  
  - *Role:* Retrieves up-to-date invoice details (balance, status) from QuickBooks for each invoice ID  

- **Switch (Reminder decision logic)**  
  - *Type:* Switch  
  - *Role:* Decides among three paths:  
    - Send reminder now (balance > 0 and reminder interval matched)  
    - Delete invoice (balance = 0)  
    - Send later (default catch-all)  

- **Send reminder email**  
  - *Type:* Gmail  
  - *Role:* Sends a professionally formatted friendly reminder email to the customer with invoice details and payment link  
  - *Failures:* Email auth, invalid recipient, SMTP errors  

- **Increase sent reminders**  
  - *Type:* Data Table (DB update)  
  - *Role:* Updates the remindersSent count and lastSentAt timestamp in the DB after sending a reminder  

- **If3 (Check if reminders exhausted)**  
  - *Type:* If  
  - *Role:* Checks if the number of reminders sent has reached or exceeded the configured reminder intervals length (e.g., after 3 reminders)  
  - *Output:*  
    - True: Delete invoice from DB (stop reminders)  
    - False: Continue processing  

- **Delete invoice**  
  - *Type:* Data Table (DB delete)  
  - *Role:* Removes invoice record from DB when fully paid or reminders are all sent  

- **AI Agent & OpenAI Chat Model**  
  - *Type:* Langchain Agent & OpenAI Chat Model  
  - *Role:* Summarizes all reminders sent today into a professional HTML email summary for internal teams  
  - *Input:* List of today's sent reminders with invoice details  
  - *Failures:* API key issues, rate limits, malformed input  

- **Send reminders sent summary**  
  - *Type:* Gmail  
  - *Role:* Sends the AI-generated summary email to a configured internal recipient (e.g., sales or finance team)  

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                                   | Input Node(s)                | Output Node(s)               | Sticky Note                                                     |
|----------------------------|-------------------------------|-------------------------------------------------|------------------------------|------------------------------|----------------------------------------------------------------|
| Receive form submission     | Webhook                      | Receives form submission from Jotform           | -                            | Format data                  | ## Receive Submission Receives the product/service form submission from Jotform |
| Format data                | Code                         | Formats and extracts structured data             | Receive form submission       | Check if the customer exists | ## Format Data Formats the data thus making it easier to be used in other nodes |
| Check if the customer exists | QuickBooks (getAll)          | Checks if customer exists in QuickBooks          | Format data                  | If                          | ## Check If Customer exists Checks if the customer exists in QBO or not |
| If                        | If                           | Branches based on customer existence              | Check if the customer exists  | Update the customer, Create the customer |                                                                |
| Update the customer         | QuickBooks (update)           | Updates existing customer details                 | If (true)                    | Add customer id              | ## Customer Exists Now since the customer exists we will update the customer details like updating the billing details with the new one |
| Create the customer         | QuickBooks (create)           | Creates new customer in QuickBooks                | If (false)                   | Add customer id              | ## Customer Doesn't Exist Now since the customer doesn't exist we will create new customer |
| Add customer id             | Code                         | Adds customer Id to JSON data                      | Update the customer, Create the customer | Get the product            | ## Add Customer Id Adds customer id to the data                  |
| Get the product             | QuickBooks (getAll)           | Retrieves product/service from QuickBooks         | Add customer id              | Add item id                 | ## Get The Item Gets the selected product/service from QBO       |
| Add item id                 | Code                         | Adds product Id to JSON data                       | Get the product              | Create the invoice          | ## Add Item Id Adds item (service/product) id to the data        |
| Create the invoice          | QuickBooks (create)           | Creates invoice for customer                       | Add item id                 | Send the invoice            | ## Create The Invoice Creates a new invoice for that customer     |
| Send the invoice            | QuickBooks (send)             | Sends invoice email to customer                    | Create the invoice           | Add reminders config        | ## Send The Invoice Sends the newly created invoice for that customer(via email) |
| Add reminders config        | Set                          | Defines reminder intervals and DB table id        | Send the invoice             | If2                        | ## Add Reminders Config Adds reminders config details like `intervals in days` so first reminder will be sent after 2 days, second one after 3 days and finally after 5 days |
| Insert invoice id to DB     | Data Table                   | Inserts invoice metadata into DB                   | If2 (true branch)            | -                          | ## Insert Invoice To DB Inserts newly created invoice needed details to DB so customer will be notified later on about the invoice |
| Schedule reminders trigger  | Schedule Trigger             | Triggers reminders workflow daily at 8 AM         | -                            | Add reminders config        | ## Schedule Trigger Schedules reminders trigger daily at 8 AM    |
| If2                        | If                           | Checks if triggered by invoice creation or schedule | Add reminders config         | Insert invoice id to DB, Get Invoices | ## Check Trigger Checks if the previous node has been executed by the above workflow or by the schedule trigger |
| Get Invoices               | Data Table                   | Retrieves all invoices from DB                      | If2 (false branch)           | Loop over invoices          | ## Get All Invoices Gets all the invoices from DB                 |
| Loop over invoices          | SplitInBatches               | Processes invoices one by one                       | Get Invoices                 | Get today's sent reminders, Get the invoice | ## Loop Over Invoices Loops over invoices one by one              |
| Get today's sent reminders  | Data Table                   | Retrieves reminders sent today from DB             | Loop over invoices           | AI Agent                   | ## Get Sent Reminders Gets today's sent reminders from DB         |
| Get the invoice             | QuickBooks (get)             | Gets invoice details from QuickBooks                | Loop over invoices           | Switch                     | ## Get Invoice Details Gets the invoice details from QBO so we know whether or not any changes have been made or not |
| Switch                     | Switch                       | Decides reminder action: send now, delete, or send later | Get the invoice              | Send reminder email, Delete invoice, Loop over invoices | ## Send Reminders Logic The logic that decides whether or not to send a reminder email now, skip it and send it later or delete the invoice/s from DB (because all the reminders have been sent or the invoice has been paid) |
| Send reminder email         | Gmail                        | Sends payment reminder email to customer           | Switch (send now)            | Increase sent reminders     |                                                                |
| Increase sent reminders     | Data Table                   | Increments remindersSent count and updates timestamp | Send reminder email          | If3                        |                                                                |
| If3                        | If                           | Checks if max reminders sent                         | Increase sent reminders      | Delete invoice, Loop over invoices |                                                                |
| Delete invoice             | Data Table                   | Deletes invoice record from DB if paid or reminders exhausted | If3 (true), Switch (delete)  | Loop over invoices          |                                                                |
| AI Agent                   | LangChain Agent              | Summarizes reminders sent today into HTML email    | Get today's sent reminders   | Send reminders sent summary | ## Summarize Sent Reminders & Send An Email Summarizes today's sent reminders using AI and send a summery email to the team like sales team or finance team |
| OpenAI Chat Model           | LangChain Language Model     | Supports AI Agent with GPT-4o-mini model             | AI Agent                    | AI Agent                   |                                                                |
| Send reminders sent summary | Gmail                        | Sends AI-generated summary email to internal team  | AI Agent                    | -                          |                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Method: POST  
   - Path: Unique (e.g., `ebee0263-fc61-414f-a9cc-faf3269ce30d`)  
   - Purpose: Receive Jotform submissions  

2. **Create Code Node "Format data"**  
   - Parses billing address string into structured JSON fields  
   - Extracts customer name, email, phone, and item name  
   - Input: Webhook output  
   - Output: Structured JSON for downstream use  

3. **Create QuickBooks Node "Check if the customer exists"**  
   - Operation: GetAll (resource: customer)  
   - Filter: Query `Where PrimaryEmailAddr = '{{ $json.customer.email }}'`  
   - Credentials: QuickBooks OAuth2  

4. **Create If Node "If"**  
   - Condition: Customer record exists & has Id  
   - True branch: Update customer  
   - False branch: Create customer  

5. **Create QuickBooks Node "Update the customer"**  
   - Operation: Update  
   - CustomerId: from If node JSON  
   - Fields: Billing address, name, phone from formatted data  
   - Credential: QuickBooks OAuth2  

6. **Create QuickBooks Node "Create the customer"**  
   - Operation: Create  
   - Fields: Billing address, name, phone, email from formatted data  
   - Credential: QuickBooks OAuth2  

7. **Create Code Node "Add customer id"**  
   - Adds customer Id from previous node to JSON data for further steps  

8. **Create QuickBooks Node "Get the product"**  
   - Operation: GetAll (resource: item)  
   - Filter: Query `WHERE name = '{{ $json.item.name }}'`  
   - Credential: QuickBooks OAuth2  

9. **Create Code Node "Add item id"**  
   - Adds item Id from product node to JSON data  

10. **Create QuickBooks Node "Create the invoice"**  
    - Operation: Create invoice  
    - CustomerRef: customer Id from JSON  
    - Line item: uses item id, amount 1, description "Jotform submission"  

11. **Create QuickBooks Node "Send the invoice"**  
    - Operation: Send invoice email  
    - InvoiceId: from created invoice  
    - Email: customer email from JSON  

12. **Create Set Node "Add reminders config"**  
    - Define:  
      - `dataTableId`: [Your DB table ID]  
      - `reminderIntervalsInDays`: [2, 3, 5] (default)  

13. **Create Data Table Node "Insert invoice id to DB"**  
    - Insert new row with invoiceId, remainingAmount (Balance), currency, remindersSent=0  
    - Use `dataTableId` from config node  

14. **Create Schedule Trigger Node "Schedule reminders trigger"**  
    - Set daily trigger at 8 AM  

15. **Reuse "Add reminders config" Node** (connect schedule trigger)  

16. **Create If Node "If2"**  
    - Checks trigger source: if invoice creation path (true) or schedule path (false)  
    - True branch: Insert invoice id to DB  
    - False branch: Get invoices from DB  

17. **Create Data Table Node "Get Invoices"**  
    - Operation: Get all rows from invoice reminders DB  

18. **Create Split In Batches Node "Loop over invoices"**  
    - Process invoices one by one  

19. **Create Data Table Node "Get today's sent reminders"**  
    - Filter: `lastSentAt >= start of today` (UTC)  

20. **Create QuickBooks Node "Get the invoice"**  
    - Get invoice details from QuickBooks by invoiceId  

21. **Create Switch Node "Switch"**  
    - Rules:  
      - "send now": balance > 0 and reminder interval matches today  
      - "already paid": balance == 0  
      - "send later": default  

22. **Create Gmail Node "Send reminder email"**  
    - To: invoice customer email  
    - HTML email with invoice reminder template  
    - Subject: Friendly reminder with invoice number  

23. **Create Data Table Node "Increase sent reminders"**  
    - Update remindersSent +1 and lastSentAt with current timestamp  

24. **Create If Node "If3"**  
    - Check if remindersSent >= number of reminder intervals  
    - True: delete invoice from DB  
    - False: continue looping  

25. **Create Data Table Node "Delete invoice"**  
    - Delete invoice record from DB by Id  

26. **Create LangChain Node "AI Agent"**  
    - Summarizes today's sent reminders data into professional HTML email  

27. **Create LangChain Node "OpenAI Chat Model"**  
    - Model: gpt-4o-mini  
    - Used as language model for AI Agent  

28. **Create Gmail Node "Send reminders sent summary"**  
    - Send AI-generated summary email to internal team (e.g., sales@example.com)  

29. **Connect nodes as per described logic and data flow**  
    - Ensure credentials configured: QuickBooks OAuth2, Gmail OAuth2, OpenAI API key  
    - Confirm database table schema with columns: invoiceId (string), remainingAmount (number), currency (string), remindersSent (number), lastSentAt (datetime)  
    - Test webhook with Jotform configured to POST to webhook URL  

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| The workflow requires a Jotform webhook setup to trigger on form submission.                                                      | https://www.jotform.com/help/245-how-to-setup-a-webhook-with-jotform/                                             |
| QuickBooks Online OAuth2 credentials must be configured for QuickBooks API nodes.                                                 | https://developer.intuit.com/app/developer/qbo/docs/get-started/get-client-id-and-client-secret                    |
| Gmail OAuth2 credentials must be set up for sending emails via Gmail nodes.                                                       | https://docs.n8n.io/integrations/builtin/credentials/google                                                       |
| The database table must have columns: invoiceId (string), remainingAmount (number), currency (string), remindersSent (number), lastSentAt (datetime). | Used for tracking invoice reminder state                                                                         |
| AI agent uses OpenAI GPT-4o-mini model to generate professional HTML summaries of reminders sent each day.                        | Requires OpenAI API key and LangChain integration setup                                                           |
| Reminder intervals by default are configured as 2, 3, and 5 days after the invoice creation/sending.                             | Configurable in the "Add reminders config" node                                                                   |
| The workflow handles error cases like missing customer or product, QuickBooks API failures, and email sending issues.             | Recommended to add error handling or notifications for production use                                             |

---

*Disclaimer: The text above is an expert analysis and detailed documentation of an n8n workflow automating invoice generation and payment reminders with Jotform, QuickBooks, and Gmail. It complies fully with content policies and uses only legal, public data.*