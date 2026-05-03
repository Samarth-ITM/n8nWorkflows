Generate Professional Invoices with Jotform, Xero, and GPT-powered Emails

https://n8nworkflows.xyz/workflows/generate-professional-invoices-with-jotform--xero--and-gpt-powered-emails-9797


# Generate Professional Invoices with Jotform, Xero, and GPT-powered Emails

### 1. Workflow Overview

This workflow automates the generation and distribution of professional invoices triggered by product or service orders submitted through a Jotform form. Upon receiving a form submission, the workflow processes customer and order data, ensures the customer exists or is updated in Xero accounting software, creates an invoice for the specified item, generates a professional email content for the invoice using AI, and finally emails the invoice to the customer.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Receives and captures the order submission from Jotform via webhook.

- **1.2 Data Formatting and Customer Management:** Extracts and structures submitted data, then creates or updates the customer contact in Xero.

- **1.3 Invoice Creation:** Generates a new invoice in Xero associated with the customer and the ordered product/service.

- **1.4 AI Email Generation and Sending:** Uses an AI language model to create a professional HTML invoice email, then sends it to the customer via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives the incoming product or service order submissions from Jotform via a webhook and prepares the raw data for processing.

- **Nodes Involved:**  
  - Receive form submission (Webhook)  
  - Sticky Note (Receive Submission)

- **Node Details:**

  - **Receive form submission**  
    - *Type & Role:* Webhook node acting as the starting trigger for the workflow, waiting for HTTP POST requests.  
    - *Configuration:*  
      - HTTP Method: POST  
      - Path: Unique webhook path (`bf87ee72-aecf-44e5-a466-3d0144da2221`) to receive Jotform submissions.  
      - No additional options specified.  
    - *Inputs/Outputs:* No input; output passes raw form submission JSON to the next node.  
    - *Potential Failures:*  
      - Webhook URL misconfiguration in Jotform resulting in no data received.  
      - HTTP request timeouts or delivery failures.  
      - Invalid or missing data in submissions.  
    - *Notes:* This node requires Jotform webhook setup; failure to configure webhook correctly will block workflow initiation.

  - **Sticky Note (Receive Submission)**  
    - Provides contextual annotation describing this block’s purpose: to receive form submissions from Jotform. No technical function.

---

#### 2.2 Data Formatting and Customer Management

- **Overview:**  
  This block extracts and formats customer and order details from the raw submission, then creates or updates the customer contact in Xero accounting software.

- **Nodes Involved:**  
  - Format data (Code)  
  - Create/Update the contact (Xero)  
  - Sticky Notes (Format Data, Create/Update Contact)

- **Node Details:**

  - **Format data**  
    - *Type & Role:* Code node performing JavaScript parsing to extract and structure the customer’s address, contact info, and ordered item from the Jotform submission.  
    - *Configuration:*  
      - Custom JS extracts fields like street address lines, city, state, postal code, country from a formatted string in `billingAddress`.  
      - Also extracts customer name, email, phone, and item name.  
      - Returns a structured object with `address`, `customer`, and `item` fields for downstream use.  
    - *Expressions:* Uses `$input.first().json.body` to access incoming webhook data.  
    - *Inputs/Outputs:* Input from webhook node; outputs formatted JSON object.  
    - *Potential Failures:*  
      - If the billingAddress format changes, regex extraction may fail leading to null fields.  
      - Missing fields in submission may cause incomplete data downstream.  
      - Expression errors if fields are undefined.  

  - **Create/Update the contact**  
    - *Type & Role:* Xero node that creates or updates a contact in Xero’s accounting system based on formatted customer data.  
    - *Configuration:*  
      - Resource: Contact  
      - Uses customer name, email, phone, and structured address fields mapped from previous node output.  
      - Phone type set as MOBILE.  
      - Address type set as STREET.  
      - Organization ID and OAuth2 credentials configured for Xero account access.  
    - *Expressions:* Dynamic mapping using expressions like `={{ $json.customer.name }}` etc.  
    - *Inputs/Outputs:* Input from Format data node; output includes Xero contact data with ContactID for use in invoice creation.  
    - *Potential Failures:*  
      - Authentication failures due to invalid or expired Xero OAuth2 credentials.  
      - API rate limits or service downtime.  
      - Data validation errors from Xero if fields are malformed.  
    - *Version:* Uses Xero OAuth2 API; ensure n8n version supports this integration.

  - **Sticky Note (Format Data)** and **Sticky Note (Create/Update Contact)**  
    - Provide descriptive comments on the respective nodes' responsibilities.

---

#### 2.3 Invoice Creation

- **Overview:**  
  This block creates a new invoice in Xero for the specified contact using the selected product/service item.

- **Nodes Involved:**  
  - Create the invoice (Xero)  
  - Sticky Note (Create The Invoice)

- **Node Details:**

  - **Create the invoice**  
    - *Type & Role:* Xero node that generates an ACCREC (Accounts Receivable) invoice linked to the contact created or updated earlier.  
    - *Configuration:*  
      - Invoice type: ACCREC (accounts receivable)  
      - Contact ID dynamically assigned from previous node’s output (`ContactID`).  
      - Line items array with one item:  
        - Tax type: INPUT  
        - Item code: taken dynamically from formatted data’s item name (`$('Format data').item.json.item.name`) — important that this matches Xero's item codes exactly.  
        - Unit amount: Fixed at 10 (currency assumed default)  
        - Account code: 200 (must correspond to correct revenue account in Xero)  
      - Organization ID and OAuth2 credentials for Xero.  
    - *Inputs/Outputs:* Input from Create/Update the contact node; output includes full invoice data.  
    - *Potential Failures:*  
      - Invalid ContactID or missing contact.  
      - Item code mismatch causing invoice creation failure.  
      - Authentication or API rate limits on Xero side.  
      - Hardcoded unit amount may not reflect actual pricing; could require dynamic adjustment.  
    - *Version:* Uses Xero OAuth2 API integration.  

  - **Sticky Note (Create The Invoice)**  
    - Describes node purpose.

---

#### 2.4 AI Email Generation and Sending

- **Overview:**  
  This block generates a professional HTML invoice email using AI based on the newly created Xero invoice data and then sends the email to the customer using Gmail.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent node)  
  - OpenAI Chat Model (GPT-4o-mini)  
  - Send email (Gmail)  
  - Sticky Note (Send The Invoice)

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* LangChain Agent node that orchestrates interaction with an AI language model to generate professional invoice email content.  
    - *Configuration:*  
      - Receives JSON from the "Create the invoice" node as input.  
      - System message instructs AI to generate professional HTML email content based on Xero invoice response.  
      - Prompt type: "define" (custom prompt)  
      - Executes once per incoming data item.  
    - *Inputs/Outputs:* Input from Create the invoice; output is generated email content in HTML format.  
    - *Potential Failures:*  
      - AI API authentication failures.  
      - Model timeout or rate limit exceeded.  
      - Unexpected invoice data structure causing generation errors.  
      - HTML content may require validation before sending.  
    - *Version:* LangChain Agent v2.2.

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node that provides the underlying GPT-4o-mini model used by the AI Agent.  
    - *Configuration:*  
      - Model: gpt-4o-mini (lightweight GPT-4 variant)  
      - No additional options set.  
      - Uses OpenAI API credentials.  
    - *Inputs/Outputs:* Linked as language model backend to AI Agent.  
    - *Potential Failures:*  
      - Invalid or expired OpenAI API key.  
      - Model service interruptions or quota limits.  

  - **Send email**  
    - *Type & Role:* Gmail node that emails the generated invoice to the customer.  
    - *Configuration:*  
      - Recipient email dynamically set from created invoice contact’s email (`{{ $('Create the invoice').item.json.Contact.EmailAddress }}`)  
      - Subject fixed as "New Invoice"  
      - Message body uses AI Agent’s output HTML email content (`{{ $json.output }}`)  
      - Uses Gmail OAuth2 credentials.  
    - *Inputs/Outputs:* Input from AI Agent node; no outputs (terminal node).  
    - *Potential Failures:*  
      - Gmail credentials or OAuth token invalid or expired.  
      - Recipient email missing or malformed.  
      - Email sending blocked by Gmail policies or quota exceeded.  
    - *Version:* Gmail node v2.1.

  - **Sticky Note (Send The Invoice)**  
    - Describes the email sending purpose of this block.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                           | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                         |
|-------------------------|-------------------------|-----------------------------------------|--------------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| Receive form submission | Webhook                 | Receives Jotform form submissions       | -                        | Format data              | ## Receive Submission Receives the product/service form submission from Jotform                    |
| Sticky Note             | StickyNote              | Annotation for Receive Submission       | -                        | -                        | ## Receive Submission Receives the product/service form submission from Jotform                    |
| Format data             | Code                    | Extracts and formats customer/order data| Receive form submission  | Create/Update the contact | ## Format Data Formats the data thus making it easier to be used in other nodes                    |
| Sticky Note8            | StickyNote              | Annotation for Format Data               | -                        | -                        | ## Format Data Formats the data thus making it easier to be used in other nodes                    |
| Create/Update the contact| Xero                    | Creates or updates customer contact     | Format data              | Create the invoice       | ## Create/Update Contact Creates or updates the contact                                            |
| Sticky Note3            | StickyNote              | Annotation for Create/Update Contact    | -                        | -                        | ## Create/Update Contact Creates or updates the contact                                            |
| Create the invoice      | Xero                    | Creates invoice for the contact          | Create/Update the contact| AI Agent                 | ## Create The Invoice Creates a new invoice for that contact                                       |
| Sticky Note5            | StickyNote              | Annotation for Create The Invoice        | -                        | -                        | ## Create The Invoice Creates a new invoice for that contact                                       |
| OpenAI Chat Model       | LangChain LM Chat Model | Provides GPT-4o-mini model for AI Agent  | -                        | AI Agent                 |                                                                                                   |
| AI Agent                | LangChain Agent         | Generates professional invoice email HTML| Create the invoice, OpenAI Chat Model | Send email            | ## Send The Invoice Sends the newly created invoice for that customer(via email)                   |
| Sticky Note19           | StickyNote              | Annotation for Send The Invoice          | -                        | -                        | ## Send The Invoice Sends the newly created invoice for that customer(via email)                   |
| Send email              | Gmail                   | Sends invoice email to customer          | AI Agent                 | -                        |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add **Webhook** node named "Receive form submission".  
   - Set HTTP Method to POST.  
   - Define a unique webhook path (e.g., `bf87ee72-aecf-44e5-a466-3d0144da2221`).  
   - This node will receive Jotform submissions via webhook.

2. **Add Code Node for Data Formatting**  
   - Add **Code** node named "Format data".  
   - Paste JavaScript code to extract customer and address information from the webhook payload.  
   - Use `$input.first().json.body` to access incoming data fields like `billingAddress`, `name`, `email`, `phone`, and `itemName`.  
   - Output structured JSON object with `address`, `customer`, and `item` keys.  
   - Connect output of "Receive form submission" to this node.

3. **Add Xero Node to Create or Update Contact**  
   - Add **Xero** node named "Create/Update the contact".  
   - Set Resource to "Contact".  
   - Map fields from "Format data" output using expressions:  
     - Name: `{{$json.customer.name}}`  
     - Email: `{{$json.customer.email}}`  
     - Phone (type MOBILE): `{{$json.customer.phone}}`  
     - Address (type STREET): map line1, line2, city, region, postal code, country from `address` object.  
   - Set Organization ID to your Xero organization.  
   - Configure Xero OAuth2 credentials.  
   - Connect "Format data" output to this node.

4. **Add Xero Node to Create Invoice**  
   - Add another **Xero** node named "Create the invoice".  
   - Set Type to ACCREC.  
   - For Contact ID, set expression to the ContactID returned by the previous node (`{{$json.ContactID}}`).  
   - For line items, add one item:  
     - Tax Type: INPUT  
     - Item Code: `{{$('Format data').item.json.item.name}}` (must match Xero item codes exactly)  
     - Unit Amount: 10 (can be customized)  
     - Account Code: 200 (adjust to your accounts)  
   - Use the same Organization ID and Xero credentials as before.  
   - Connect output of "Create/Update the contact" node to this node.

5. **Add LangChain Agent Node for AI Email Generation**  
   - Add **LangChain Agent** node named "AI Agent".  
   - Set the input text to the JSON output of "Create the invoice" node (`={{ $json }}`).  
   - Provide a system message instructing the AI to generate a professional HTML invoice email based on the Xero invoice data.  
   - Set prompt type to "define".  
   - Connect output of "Create the invoice" node to this node.

6. **Add OpenAI Chat Model Node**  
   - Add **LangChain LM Chat Model** node named "OpenAI Chat Model".  
   - Select model `gpt-4o-mini` or preferred GPT model.  
   - Configure OpenAI API credentials.  
   - Connect as language model to the "AI Agent" node (via the ai_languageModel input).

7. **Add Gmail Node to Send Email**  
   - Add **Gmail** node named "Send email".  
   - Set recipient email to `{{$('Create the invoice').item.json.Contact.EmailAddress}}`.  
   - Set email subject to "New Invoice".  
   - Set message body to the AI Agent’s output (`{{$json.output}}`), which is the generated HTML email content.  
   - Configure Gmail OAuth2 credentials.  
   - Connect output of "AI Agent" node to this node.

8. **Add Sticky Notes**  
   - Add sticky notes for each block to document the workflow purpose, such as:  
     - "Receive Submission" near webhook node.  
     - "Format Data" near the Code node.  
     - "Create/Update Contact" near Xero contact node.  
     - "Create The Invoice" near Xero invoice node.  
     - "Send The Invoice" near AI Agent and Gmail nodes.  
   - Include the provided descriptive content for clarity.

9. **Finalize and Test**  
   - Ensure all credentials (Xero OAuth2, OpenAI, Gmail) are valid and authorized.  
   - Set Jotform to send submissions to the webhook URL.  
   - Test form submissions and monitor execution logs for errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow automates receiving Jotform submissions, creating/updating contacts and invoices in Xero, generating AI-powered professional invoice emails, and sending them via Gmail. | Workflow purpose overview                                                                                       |
| Jotform webhook setup instructions: https://www.jotform.com/help/245-how-to-setup-a-webhook-with-jotform/                                                        | Jotform integration requirement                                                                                 |
| Xero OAuth2 credentials setup documentation: https://docs.n8n.io/integrations/builtin/credentials/xero                                                             | Xero API integration and credential setup                                                                       |
| Gmail OAuth2 setup documentation: https://docs.n8n.io/integrations/builtin/credentials/google                                                                       | Email sending configuration                                                                                      |
| Ensure product/service values in Jotform exactly match the item `Code` in Xero for invoice creation to succeed.                                                     | Data consistency requirement                                                                                      |
| The AI agent uses the GPT-4o-mini model to generate professional invoice emails in HTML format for customer communication.                                        | AI email content generation                                                                                       |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.