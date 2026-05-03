Automated Invoice Creation & Customer Communication with Jotform, Xero, Outlook & Telegram

https://n8nworkflows.xyz/workflows/automated-invoice-creation---customer-communication-with-jotform--xero--outlook---telegram-9798


# Automated Invoice Creation & Customer Communication with Jotform, Xero, Outlook & Telegram

### 1. Workflow Overview

This workflow automates the entire process of handling product or service orders submitted through a Jotform form. It integrates with Xero accounting software to manage customer contacts and invoices, uses Outlook to email invoices to customers, and sends Telegram notifications to the sales team for timely follow-up. The main use case is to streamline order-to-invoice processing and team communication for freelancers, service providers, consultants, and small businesses.

The workflow consists of the following logical blocks:

- **1.1 Receive Submission:** Captures form data submitted via Jotform webhook.
- **1.2 Data Formatting:** Extracts and structures customer and order data from the raw form input.
- **1.3 Customer Verification:** Checks if the customer exists in Xero; updates or creates customer accordingly.
- **1.4 Invoice Creation:** Generates an invoice in Xero for the verified or newly created customer.
- **1.5 Invoice Delivery:** Sends the invoice to the customer via Outlook email.
- **1.6 Post-Invoice Wait:** Pauses the workflow to allow time for invoice processing.
- **1.7 Invoice Status Check & Team Notification:** Retrieves invoice details, evaluates if action is needed, and notifies the sales team via Telegram if no update has occurred.

---

### 2. Block-by-Block Analysis

#### 1.1 Receive Submission

- **Overview:**  
  This block triggers the workflow upon receiving a Jotform submission via webhook.

- **Nodes Involved:**  
  - Receive form submission (Webhook)  
  - Sticky Note (Receive Submission)  

- **Node Details:**

  - **Receive form submission**  
    - *Type:* Webhook (HTTP POST)  
    - *Role:* Entry point for form submissions from Jotform  
    - *Configuration:* HTTP POST method, fixed webhook path for Jotform integration  
    - *Input:* External HTTP POST request with form data payload  
    - *Output:* JSON data containing customer and order details  
    - *Edge cases:* Missing or malformed webhook payload; network or authentication failures on receiving side  
    - *Notes:* Requires Jotform webhook set to this exact URL  

  - **Sticky Note (Receive Submission)**  
    - *Type:* Documentation note  
    - *Role:* Describes purpose of this block  

#### 1.2 Data Formatting

- **Overview:**  
  Parses and structures raw form data to extract customer info, billing address components, and selected item details for subsequent processing.

- **Nodes Involved:**  
  - Format data (Code)  
  - Sticky Note (Format Data)  

- **Node Details:**

  - **Format data**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts nested data from the billing address HTML string and other form fields  
    - *Configuration:* Uses regex to parse HTML-formatted address; constructs structured JSON with fields: line1, line2, city, stateProvince, postalZipCode, country, plus customer name, email, phone, and item name  
    - *Input:* Output from webhook node (raw form submission)  
    - *Output:* Structured JSON with separated address and customer/item details  
    - *Edge cases:* Address format changes breaking regex; missing address fields leading to null values  
    - *Notes:* Essential for matching Xero contact fields  

  - **Sticky Note (Format Data)**  
    - *Type:* Documentation note  

#### 1.3 Customer Verification

- **Overview:**  
  Searches Xero for an existing customer by email. Updates contact if exists, otherwise creates a new contact.

- **Nodes Involved:**  
  - Check if the customer exists (Xero)  
  - If (Conditional)  
  - Update contact (Xero)  
  - Create new contact (Xero)  
  - Sticky Note (Check If Contact exists)  
  - Sticky Note (Update Contact)  
  - Sticky Note (Create Contact)  

- **Node Details:**

  - **Check if the customer exists**  
    - *Type:* Xero API (Get All Contacts)  
    - *Role:* Query Xero contacts filtered by email address from formatted data  
    - *Configuration:* Limit 1, filter where EmailAddress equals customer email  
    - *Input:* Formatted customer data with email  
    - *Output:* Contact data if found, empty if not  
    - *Edge cases:* API errors, rate limits, malformed filter expression, customer email missing or invalid  
    - *Version:* Requires Xero OAuth2 credentials setup  

  - **If**  
    - *Type:* Conditional  
    - *Role:* Determines contact existence based on presence of ContactID in previous node output  
    - *Configuration:* Checks if JSON is non-empty and has ContactID  
    - *Input:* Output of Check if the customer exists  
    - *Output:* Two branches: true (contact exists) and false (contact does not exist)  
    - *Edge cases:* False negatives if ContactID missing due to API changes or data format issues  

  - **Update contact**  
    - *Type:* Xero API (Update Contact)  
    - *Role:* Updates existing contact with latest name and phone from form data  
    - *Configuration:* Uses ContactID from If node, updates name and mobile phone fields  
    - *Input:* ContactID and formatted customer data  
    - *Output:* Updated contact record  
    - *Edge cases:* API update failures, invalid ContactID, validation errors on fields  

  - **Create new contact**  
    - *Type:* Xero API (Create Contact)  
    - *Role:* Creates new contact using customer and address details from formatted data  
    - *Configuration:* Sets name, email, mobile phone, and detailed address fields (line1, line2, city, region, postal code, country)  
    - *Input:* Formatted customer and address data  
    - *Output:* Newly created contact record with ContactID  
    - *Edge cases:* Duplicate contact creation if email already exists, API errors, missing required fields  

  - **Sticky Notes**  
    - Provide descriptive summaries for each node's function in this block  

#### 1.4 Invoice Creation

- **Overview:**  
  Creates an accounts receivable invoice in Xero for the identified or newly created contact.

- **Nodes Involved:**  
  - Create the invoice (Xero)  
  - Sticky Note (Create The Invoice)  

- **Node Details:**

  - **Create the invoice**  
    - *Type:* Xero API (Create Invoice)  
    - *Role:* Generates a new ACCREC invoice linked to the contact by ContactID  
    - *Configuration:* Sets invoice type ACCREC, line items with item code from formatted item name, fixed unit amount 10, tax type INPUT, account code 200  
    - *Input:* ContactID from contact creation/update nodes, item name from formatted data  
    - *Output:* Created invoice details including InvoiceID and status  
    - *Edge cases:* Item code mismatch with Xero product codes, API errors, invoice creation validation errors  
    - *Notes:* Fixed unit amount hardcoded to 10 (could be parameterized)  

  - **Sticky Note (Create The Invoice)**  
    - Describes the purpose of this node  

#### 1.5 Invoice Delivery

- **Overview:**  
  Uses AI to generate a professional HTML email based on the invoice data, then sends the invoice to the customer via Outlook.

- **Nodes Involved:**  
  - AI Agent (LangChain agent)  
  - OpenAI Chat Model (gpt-4o-mini)  
  - Send email (Microsoft Outlook)  
  - Sticky Note (Send The Invoice)  

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain AI Agent  
    - *Role:* Processes Xero invoice JSON to create a professional HTML email content  
    - *Configuration:* Uses system prompt to instruct generation of professional invoice email content  
    - *Input:* Invoice data from Create the invoice node  
    - *Output:* HTML email content as text  
    - *Edge cases:* API rate limits, malformed input JSON, AI generation delays or errors  

  - **OpenAI Chat Model**  
    - *Type:* OpenAI GPT-4o-mini model  
    - *Role:* Provides language model backend for AI Agent  
    - *Configuration:* Model set to gpt-4o-mini, no special options  
    - *Input:* Prompt from AI Agent  
    - *Output:* Generated text response  

  - **Send email**  
    - *Type:* Microsoft Outlook node  
    - *Role:* Sends email to customer with invoice attached or in body  
    - *Configuration:* Subject "New Invoice", body set from AI Agent output, recipient email from invoice contact email  
    - *Input:* AI Agent generated HTML content and invoice contact email  
    - *Output:* Email send status  
    - *Edge cases:* Email sending failures, invalid recipient email, Outlook credential issues  

  - **Sticky Note (Send The Invoice)**  
    - Describes this block's function  

#### 1.6 Post-Invoice Wait

- **Overview:**  
  Waits 30 seconds before proceeding to invoice status check to allow for processing time.

- **Nodes Involved:**  
  - Wait (Wait node)  
  - Sticky Note (Wait For Sometime)  

- **Node Details:**

  - **Wait**  
    - *Type:* Wait  
    - *Role:* Pauses execution for 30 seconds (configurable)  
    - *Configuration:* Amount set to 30 seconds  
    - *Input:* Output from Create the invoice or AI Agent node  
    - *Output:* Passes data forward after delay  
    - *Edge cases:* Workflow timeouts, long delays if system under load  

  - **Sticky Note (Wait For Sometime)**  
    - Describes wait purpose  

#### 1.7 Invoice Status Check & Team Notification

- **Overview:**  
  Retrieves the created invoice details from Xero, compares its status to the original, and if no changes detected, uses AI to generate a Telegram notification message to alert the team for timely follow-up.

- **Nodes Involved:**  
  - Get the invoice (Xero)  
  - Switch (Conditional)  
  - AI Agent1 (LangChain agent)  
  - OpenAI Chat Model1 (gpt-4o-mini)  
  - Notify the team (Telegram)  
  - Sticky Notes (Get The Invoice, Notify The Team To Act Fast)  

- **Node Details:**

  - **Get the invoice**  
    - *Type:* Xero API (Get Invoice)  
    - *Role:* Retrieves up-to-date invoice details using InvoiceID  
    - *Configuration:* Uses InvoiceID from Create the invoice node output  
    - *Input:* InvoiceID JSON field  
    - *Output:* Current invoice status and details  
    - *Edge cases:* API errors, invalid InvoiceID, network failures  

  - **Switch**  
    - *Type:* Switch conditional  
    - *Role:* Compares current invoice status with original invoice status from creation  
    - *Configuration:* Outputs "no change" if statuses match, else no output  
    - *Input:* Output from Get the invoice and Create the invoice nodes  
    - *Output:* Branch for "no change" status only  
    - *Edge cases:* Status field missing, unexpected status values  

  - **AI Agent1**  
    - *Type:* LangChain AI Agent  
    - *Role:* Generates a Telegram message to notify the sales team of invoice inactivity  
    - *Configuration:* System prompt instructs to create a message notifying the team to act fast on unattended invoices  
    - *Input:* Invoice details from "no change" Switch output  
    - *Output:* Telegram message text  

  - **OpenAI Chat Model1**  
    - *Type:* OpenAI GPT-4o-mini model  
    - *Role:* Language model backend for AI Agent1  
    - *Configuration:* Same as AI Agent  

  - **Notify the team**  
    - *Type:* Telegram node  
    - *Role:* Sends generated notification message to configured Telegram chat ID  
    - *Configuration:* Chat ID must be set to sales team group or individual  
    - *Input:* Message text from AI Agent1  
    - *Output:* Telegram send status  
    - *Edge cases:* Telegram API failures, invalid chat ID, network issues  

  - **Sticky Notes**  
    - Describe invoice retrieval and notification purpose  

---

### 3. Summary Table

| Node Name                | Node Type                   | Functional Role                           | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                                             |
|--------------------------|-----------------------------|-----------------------------------------|----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Receive form submission  | Webhook                     | Trigger workflow on Jotform submission  | -                          | Format data                 | ## Receive Submission<br>Receives the product/service form submission from Jotform                                        |
| Sticky Note              | Sticky Note                 | Documentation                           | -                          | -                           | ## Receive Submission<br>Receives the product/service form submission from Jotform                                        |
| Format data              | Code                        | Parse and structure form data            | Receive form submission     | Check if the customer exists | ## Format Data<br>Formats the data thus making it easier to be used in other nodes                                        |
| Sticky Note              | Sticky Note                 | Documentation                           | -                          | -                           | ## Format Data<br>Formats the data thus making it easier to be used in other nodes                                        |
| Check if the customer exists | Xero (Get All Contacts)    | Search Xero for existing customer         | Format data                | If                          | ## Check If Contact exists <br>Checks if the contact exists in Xero or not                                               |
| If                       | Conditional                 | Determine customer existence             | Check if the customer exists | Update contact, Create new contact |                                                                                                                         |
| Update contact           | Xero (Update Contact)       | Update existing contact info in Xero     | If (true branch)           | Create the invoice           | ## Update Contact<br>Updates the contact                                                                                 |
| Create new contact       | Xero (Create Contact)       | Create new contact in Xero                | If (false branch)          | Create the invoice           | ## Create Contact<br>Creates a new contact                                                                                |
| Sticky Note              | Sticky Note                 | Documentation                           | -                          | -                           | ## Update Contact<br>Updates the contact<br>## Create Contact<br>Creates a new contact                                    |
| Create the invoice       | Xero (Create Invoice)       | Generate new invoice for contact         | Update contact, Create new contact | Wait, AI Agent            | ## Create The Invoice<br>Creates a new invoice for that contact                                                           |
| Sticky Note              | Sticky Note                 | Documentation                           | -                          | -                           | ## Create The Invoice<br>Creates a new invoice for that contact                                                           |
| AI Agent                 | LangChain AI Agent          | Generate professional invoice email HTML | Create the invoice         | Send email                  | ## Send The Invoice<br>Sends the newly created invoice for that customer(via email)                                       |
| OpenAI Chat Model        | OpenAI GPT-4o-mini          | AI language model backend for AI Agent   | AI Agent                   | AI Agent                   |                                                                                                                         |
| Send email               | Microsoft Outlook           | Email invoice to customer                 | AI Agent                   | Wait                       |                                                                                                                         |
| Sticky Note              | Sticky Note                 | Documentation                           | -                          | -                           | ## Send The Invoice<br>Sends the newly created invoice for that customer(via email)                                       |
| Wait                     | Wait                        | Pause workflow for processing delay      | Create the invoice         | Get the invoice             | ## Wait For Sometime<br>Now we will wait for 30 seconds                                                                  |
| Sticky Note              | Sticky Note                 | Documentation                           | -                          | -                           | ## Wait For Sometime<br>Now we will wait for 30 seconds                                                                  |
| Get the invoice          | Xero (Get Invoice)          | Retrieve invoice details                  | Wait                       | Switch                     | ## Get The Invoice<br>Gets the invoice details that has been just created in Xero                                         |
| Switch                   | Conditional                 | Check if invoice status changed           | Get the invoice            | AI Agent1                  |                                                                                                                         |
| AI Agent1                | LangChain AI Agent          | Generate Telegram notification message   | Switch (no change output)  | Notify the team             | ## Notify The Team To Act Fast<br>Notifies the team about the invoice inactivity                                         |
| OpenAI Chat Model1       | OpenAI GPT-4o-mini          | AI language model backend for AI Agent1  | AI Agent1                  | AI Agent1                  |                                                                                                                         |
| Notify the team          | Telegram                    | Send notification message to sales team  | AI Agent1                  | -                         |                                                                                                                         |
| Sticky Note              | Sticky Note                 | Documentation                           | -                          | -                           | ## Notify The Team To Act Fast<br>Notifies the team (sales team for example) about the invoice since no change has been done |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive form submission"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `1d06aca8-7d95-4394-8630-87450e7a2fbe` (or custom unique string)  
   - Purpose: Receive Jotform submission payload  
   - Credentials: None required  
   - Connect next to: Format data node  

2. **Create Code Node: "Format data"**  
   - Type: Code (JavaScript)  
   - Code: Use regex to parse billing address HTML string and extract fields: line1, line2, city, stateProvince, postalZipCode, country. Also extract customer name, email, phone, and item name from input JSON.  
   - Input: Output from webhook node  
   - Output: Structured JSON with `address`, `customer`, and `item` objects  
   - Connect next to: Check if the customer exists node  

3. **Create Xero Node: "Check if the customer exists"**  
   - Type: Xero - Get All Contacts  
   - Parameters:  
     - Limit: 1  
     - Filter: `EmailAddress="{{ $json.customer.email }}"` (use expression)  
     - Organization ID: Your Xero organization ID  
   - Credentials: Xero OAuth2 (configure with your Xero account)  
   - Connect next to: If node  

4. **Create If Node: "If"**  
   - Type: Conditional  
   - Condition: Check if output JSON is non-empty and has ContactID field  
   - True Branch: Update contact  
   - False Branch: Create new contact  

5. **Create Xero Node: "Update contact"**  
   - Type: Xero - Update Contact  
   - Parameters:  
     - Contact ID: `{{$json.ContactID}}`  
     - Update fields: name, phone (mobile) from formatted data  
     - Organization ID: Same as above  
   - Credentials: Xero OAuth2  
   - Connect next to: Create the invoice node  

6. **Create Xero Node: "Create new contact"**  
   - Type: Xero - Create Contact  
   - Parameters:  
     - Name, Email Address, Phone (mobile), Address lines, City, Region, Postal Code, Country - all from formatted data expressions  
     - Organization ID: Same as above  
   - Credentials: Xero OAuth2  
   - Connect next to: Create the invoice node  

7. **Create Xero Node: "Create the invoice"**  
   - Type: Xero - Create Invoice  
   - Parameters:  
     - Type: ACCREC  
     - Contact ID: `{{$json.ContactID}}` (from updated or created contact)  
     - Line Items: Use item code from formatted data item name, unit amount 10, tax type INPUT, account code 200  
     - Organization ID: Same as above  
   - Credentials: Xero OAuth2  
   - Connect next to: AI Agent node and Wait node (parallel)  

8. **Create LangChain AI Agent Node: "AI Agent"**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text: `{{$json}}` (invoice JSON)  
     - System Message: Instruct to generate professional HTML invoice email content based on Xero invoice response  
     - Prompt Type: Define  
   - Connect next to: Send email node  

9. **Create Microsoft Outlook Node: "Send email"**  
   - Type: Microsoft Outlook  
   - Parameters:  
     - Subject: "New Invoice"  
     - Body content: `{{$json.output}}` (output from AI Agent)  
     - To recipients: `{{ $('Create the invoice').item.json.Contact.EmailAddress }}`  
   - Credentials: Outlook OAuth2 (configure with your Microsoft account)  
   - Connect no further (end of email sending path)  

10. **Create Wait Node: "Wait"**  
    - Type: Wait  
    - Parameters:  
      - Amount: 30 seconds (default)  
    - Connect next to: Get the invoice node  

11. **Create Xero Node: "Get the invoice"**  
    - Type: Xero - Get Invoice  
    - Parameters:  
      - Invoice ID: `{{$json.InvoiceID}}` (from create invoice output)  
      - Organization ID: Same as above  
    - Credentials: Xero OAuth2  
    - Connect next to: Switch node  

12. **Create Switch Node: "Switch"**  
    - Type: Switch conditional  
    - Rules:  
      - Output "no change" if `Create the invoice.Status` equals current invoice `Status`  
    - Connect "no change" output next to: AI Agent1 node  

13. **Create LangChain AI Agent Node: "AI Agent1"**  
    - Type: LangChain Agent  
    - Parameters:  
      - Text: `{{$json}}` (invoice details)  
      - System message: Instruct to generate Telegram notification message about invoice inactivity for sales team  
      - Prompt Type: Define  
    - Connect next to: Notify the team node  

14. **Create OpenAI Chat Model Node: "OpenAI Chat Model" and "OpenAI Chat Model1"**  
    - Type: OpenAI GPT-4o-mini  
    - Credentials: OpenAI API key  
    - Connected as backend for AI Agent and AI Agent1 respectively  

15. **Create Telegram Node: "Notify the team"**  
    - Type: Telegram  
    - Parameters:  
      - Text: `{{$json.output}}` (message from AI Agent1)  
      - Chat ID: Set to your sales team Telegram chat/group ID  
    - Credentials: Telegram Bot API token  
    - Connect no further (end of notification path)  

16. **Add Sticky Notes**  
    - Add descriptive sticky notes at key blocks for documentation and clarity as per the original workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow automates from form submission to invoice creation and team notification using Jotform, Xero, Outlook, Telegram, and AI-powered message generation.                                                                  | Overview and purpose                                                                                      |
| Jotform webhook setup requires linking form to webhook URL. See: https://www.jotform.com/help/245-how-to-setup-a-webhook-with-jotform/                                                                                      | Jotform webhook integration                                                                                |
| Xero credentials require OAuth2 authentication setup in n8n. See: https://docs.n8n.io/integrations/builtin/credentials/xero                                                                                                  | Xero integration                                                                                           |
| Ensure product/service names in Jotform match exactly with Xero Item `Code` field to avoid invoice line item mismatches.                                                                                                      | Important for invoice accuracy                                                                             |
| Outlook email node requires proper OAuth2 credentials. See: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook                                                                               | Outlook integration                                                                                        |
| Telegram node requires Bot API token and chat ID for notifications. See: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram                                                                           | Telegram integration                                                                                       |
| AI agents leverage OpenAI GPT-4o-mini model for generating professional emails and notification messages. Ensure OpenAI API credentials are configured correctly.                                                              | AI-powered content generation                                                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The process strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.