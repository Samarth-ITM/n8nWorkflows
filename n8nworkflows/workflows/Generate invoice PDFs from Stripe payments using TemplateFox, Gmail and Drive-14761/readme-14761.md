Generate invoice PDFs from Stripe payments using TemplateFox, Gmail and Drive

https://n8nworkflows.xyz/workflows/generate-invoice-pdfs-from-stripe-payments-using-templatefox--gmail-and-drive-14761


# Generate invoice PDFs from Stripe payments using TemplateFox, Gmail and Drive

# 1. Workflow Overview

This workflow automatically generates an invoice PDF after a successful Stripe Checkout payment, then delivers it to the customer by email and stores a copy in Google Drive.

Its main use case is post-payment invoicing for businesses using Stripe Checkout, with branded PDF generation handled by TemplateFox and delivery/storage handled by Gmail and Google Drive.

## 1.1 Payment Event Reception

The workflow starts when Stripe emits a `checkout.session.completed` event. This provides the checkout session data, including customer information, payment totals, and the session ID used later to fetch purchased line items.

## 1.2 Stripe Data Retrieval and Invoice Data Preparation

After the trigger fires, the workflow calls Stripe’s API to retrieve the line items associated with the completed checkout session. A Code node then transforms the trigger payload and line items into a structured invoice object matching the expected TemplateFox template fields.

## 1.3 PDF Generation and Delivery

The formatted invoice payload is sent to TemplateFox to generate a PDF invoice. Once generated, the workflow:
- sends the customer an email through Gmail containing a download link to the PDF
- downloads the PDF file from the generated URL
- uploads the PDF into Google Drive

---

# 2. Block-by-Block Analysis

## Block 1: Payment Event Reception

### Overview

This block listens for successful Stripe Checkout completions. It is the workflow entry point and provides the base transaction context used by all downstream steps.

### Nodes Involved

- Stripe Trigger

### Node Details

#### Stripe Trigger

- **Type and technical role:** `n8n-nodes-base.stripeTrigger`  
  Event-based trigger node that subscribes to Stripe webhook events.
- **Configuration choices:**
  - Configured to listen only to `checkout.session.completed`
  - Uses a fixed webhook ID: `stripe-checkout-trigger-001`
- **Key expressions or variables used:**
  - Downstream nodes reference `$('Stripe Trigger').first().json`
  - Session object is accessed later as `trigger.data.object`
- **Input and output connections:**
  - No input; this is the entry point
  - Outputs to **Get Line Items**
- **Version-specific requirements:**
  - Uses node version `1`
  - Requires valid Stripe webhook/credential setup in n8n
- **Edge cases or potential failure types:**
  - Stripe webhook misconfiguration
  - Invalid or missing Stripe credentials
  - Trigger not receiving events if webhook URL is not registered
  - Test/live mode mismatch between Stripe environment and credentials
- **Sub-workflow reference:**
  - None

---

## Block 2: Stripe Data Retrieval and Invoice Data Preparation

### Overview

This block enriches the Stripe event by fetching checkout line items, then reshapes all relevant payment and customer data into the exact invoice field structure expected by TemplateFox.

### Nodes Involved

- Get Line Items
- Format Invoice Data

### Node Details

#### Get Line Items

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Makes a direct authenticated Stripe API request to retrieve line items for the completed checkout session.
- **Configuration choices:**
  - Method: `GET`
  - URL is dynamically built from the Stripe session ID:
    `https://api.stripe.com/v1/checkout/sessions/{{ $json.data.object.id }}/line_items`
  - Authentication uses predefined credential type: `stripeApi`
  - `returnAll` is disabled
  - `limit` is set to `1` in the node configuration
- **Key expressions or variables used:**
  - `{{ $json.data.object.id }}` from the trigger payload
- **Input and output connections:**
  - Input from **Stripe Trigger**
  - Output to **Format Invoice Data**
- **Version-specific requirements:**
  - Uses node version `4.2`
  - Requires a configured Stripe API credential
- **Edge cases or potential failure types:**
  - Stripe credential missing or invalid
  - Session ID absent or malformed
  - API permission issues
  - Because `returnAll` is false and limit is `1`, there is a risk of only retrieving the first page or a limited set of items depending on actual Stripe response behavior and node interpretation
  - Network/API timeout or rate limiting
- **Sub-workflow reference:**
  - None

#### Format Invoice Data

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that combines trigger data and fetched line items into a structured invoice payload.
- **Configuration choices:**
  - Uses inline JavaScript
  - Hardcodes company details directly in the script
  - Extracts:
    - customer identity and address
    - line items
    - invoice number
    - invoice date and due date
    - totals, tax, subtotal, and currency
  - Produces both template-facing fields and helper fields prefixed with `_`
- **Key expressions or variables used:**
  - `$('Stripe Trigger').first().json`
  - `$input.first().json.data || []`
  - `session.customer_details`
  - `session.customer_email`
  - `session.amount_subtotal`
  - `session.total_details?.amount_tax`
  - `session.amount_total`
  - `session.currency`
  - Generated helper fields:
    - `_client_email`
    - `_client_name`
    - `_pdf_filename`
- **Input and output connections:**
  - Input from **Get Line Items**
  - Output to **TemplateFox**
- **Version-specific requirements:**
  - Uses Code node version `2`
  - Requires JavaScript execution support in n8n
- **Edge cases or potential failure types:**
  - If Stripe payload shape differs, expressions like `trigger.data.object` may fail logically
  - Missing customer details can lead to fallback values like `Customer` or empty strings
  - `session.id.slice(-6)` will fail if `session.id` is missing
  - `session.amount_subtotal` or `session.amount_total` being undefined would break numeric formatting
  - Address fields may be absent; code handles this relatively safely with fallbacks
  - Hardcoded company information may become stale or inconsistent if not maintained
  - Invoice date is generated at execution time, not from Stripe payment timestamp
- **Sub-workflow reference:**
  - None

---

## Block 3: PDF Generation and Delivery

### Overview

This block sends the prepared invoice data to TemplateFox, then uses the generated PDF URL for customer delivery and file archiving. It branches into two parallel paths: emailing the invoice link and downloading/uploading the PDF to Google Drive.

### Nodes Involved

- TemplateFox
- Email Invoice
- Download PDF
- Save to Google Drive

### Node Details

#### TemplateFox

- **Type and technical role:** `n8n-nodes-templatefox.templateFox`  
  Community node that generates a PDF from a selected TemplateFox invoice template using mapped data fields.
- **Configuration choices:**
  - Operation: `generatePdf`
  - Data input mode: `fields`
  - Template field mapping mode: `defineBelow`
  - Filename is dynamically set from:
    `{{ $json._pdf_filename }}`
  - `templateId` is currently blank and must be selected/configured
  - `templateFields.value` is `null`, so field mapping is not yet fully defined in the provided configuration
- **Key expressions or variables used:**
  - `{{ $json._pdf_filename }}`
- **Input and output connections:**
  - Input from **Format Invoice Data**
  - Outputs to both **Email Invoice** and **Download PDF**
- **Version-specific requirements:**
  - Uses node version `1`
  - Requires installation of the `n8n-nodes-templatefox` community node
  - Requires TemplateFox API credentials
- **Edge cases or potential failure types:**
  - Community node not installed
  - Invalid or missing TemplateFox API key
  - No template selected because `templateId` is empty
  - Missing template field mapping if the template requires explicit field definitions
  - Output may not contain `url` if generation fails
- **Sub-workflow reference:**
  - None

#### Email Invoice

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email to the customer containing invoice details and a link to the generated PDF.
- **Configuration choices:**
  - Recipient:
    `{{ $('Format Invoice Data').first().json._client_email }}`
  - Subject:
    `Your Invoice {{ invoice_number }}`
  - Body includes:
    - customer name
    - PDF download URL from TemplateFox output
    - invoice number
    - total amount and currency
    - due date
    - company name
- **Key expressions or variables used:**
  - `{{ $('Format Invoice Data').first().json._client_email }}`
  - `{{ $('Format Invoice Data').first().json._client_name }}`
  - `{{ $json.url }}`
  - `{{ $('Format Invoice Data').first().json.invoice_number }}`
  - `{{ $('Format Invoice Data').first().json.total }}`
  - `{{ $('Format Invoice Data').first().json.currency }}`
  - `{{ $('Format Invoice Data').first().json.due_date }}`
  - `{{ $('Format Invoice Data').first().json.company_name }}`
- **Input and output connections:**
  - Input from **TemplateFox**
  - No downstream output
- **Version-specific requirements:**
  - Uses node version `2.1`
  - Requires Gmail OAuth2 credentials
- **Edge cases or potential failure types:**
  - Missing or invalid Gmail OAuth2 credential
  - Empty recipient email
  - TemplateFox output missing `url`
  - Gmail sending restrictions, quota, or permission issues
  - Email body sends a link, not an actual PDF attachment
- **Sub-workflow reference:**
  - None

#### Download PDF

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the generated PDF file from the URL returned by TemplateFox.
- **Configuration choices:**
  - Method: `GET`
  - URL:
    `{{ $json.url }}`
  - Response format configured as `file`
- **Key expressions or variables used:**
  - `{{ $json.url }}`
- **Input and output connections:**
  - Input from **TemplateFox**
  - Output to **Save to Google Drive**
- **Version-specific requirements:**
  - Uses node version `4.2`
- **Edge cases or potential failure types:**
  - URL missing or expired
  - File download timeout
  - HTTP 403/404 from the TemplateFox file endpoint
  - Very large files may affect memory or execution duration
- **Sub-workflow reference:**
  - None

#### Save to Google Drive

- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the downloaded invoice PDF binary file into a selected Google Drive folder.
- **Configuration choices:**
  - Operation: `upload`
  - File name:
    `{{ $('Format Invoice Data').first().json._pdf_filename }}.pdf`
  - `folderId` is blank and must be filled
- **Key expressions or variables used:**
  - `{{ $('Format Invoice Data').first().json._pdf_filename }}.pdf`
- **Input and output connections:**
  - Input from **Download PDF**
  - No downstream output
- **Version-specific requirements:**
  - Uses node version `3`
  - Requires Google Drive OAuth2 credentials
  - Depends on binary data from the previous HTTP Request node
- **Edge cases or potential failure types:**
  - Missing Google Drive OAuth2 credential
  - Empty or invalid folder ID
  - Binary property mismatch if file data is not where Google Drive expects it
  - Permission denied on the target Drive/folder
  - Duplicate file naming may create multiple files unless overwrite handling is added
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Stripe Trigger | n8n-nodes-base.stripeTrigger | Receives successful Stripe Checkout completion events |  | Get Line Items | ### Stripe payment → Invoice PDF → Gmail + Google Drive<br>Automatically generate and deliver professional invoice PDFs when a Stripe payment succeeds.<br><br>**Full tutorial:** [pdftemplateapi.com/n8n/automate-invoices](https://pdftemplateapi.com/n8n/automate-invoices)<br><br>### How it works<br>1. **Stripe Trigger** fires on every successful checkout session<br>2. **HTTP Request** fetches the line items for that session<br>3. **Code** node formats the data to match your invoice template fields<br>4. **TemplateFox** generates a branded PDF invoice<br>5. Invoice is **emailed** to the customer and **saved to Google Drive**<br><br>### Setup<br>- [ ] Add your **Stripe Secret Key** (`sk_live_` or `sk_test_`) from [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)<br>- [ ] Install the **TemplateFox** community node (`n8n-nodes-templatefox`)<br>- [ ] Add your **TemplateFox** API key — [get one free](https://app.pdftemplateapi.com)<br>- [ ] Connect **Gmail** OAuth2<br>- [ ] Connect **Google Drive** OAuth2<br>- [ ] Select your invoice template in the TemplateFox node<br>- [ ] Set your company details directly in the template (not in the Code node)<br>- [ ] Set your **Google Drive folder ID** in the Google Drive node<br><br>### Customize<br>- Change the email body in the Gmail node<br>- Adjust the invoice number format in the Code node<br>- Add a Slack notification node after Google Drive for team alerts<br>## 1. Receive payment<br>Stripe fires a webhook when a checkout session completes. |
| Get Line Items | n8n-nodes-base.httpRequest | Retrieves Stripe checkout session line items | Stripe Trigger | Format Invoice Data | ### Stripe payment → Invoice PDF → Gmail + Google Drive<br>Automatically generate and deliver professional invoice PDFs when a Stripe payment succeeds.<br><br>**Full tutorial:** [pdftemplateapi.com/n8n/automate-invoices](https://pdftemplateapi.com/n8n/automate-invoices)<br><br>### How it works<br>1. **Stripe Trigger** fires on every successful checkout session<br>2. **HTTP Request** fetches the line items for that session<br>3. **Code** node formats the data to match your invoice template fields<br>4. **TemplateFox** generates a branded PDF invoice<br>5. Invoice is **emailed** to the customer and **saved to Google Drive**<br><br>### Setup<br>- [ ] Add your **Stripe Secret Key** (`sk_live_` or `sk_test_`) from [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)<br>- [ ] Install the **TemplateFox** community node (`n8n-nodes-templatefox`)<br>- [ ] Add your **TemplateFox** API key — [get one free](https://app.pdftemplateapi.com)<br>- [ ] Connect **Gmail** OAuth2<br>- [ ] Connect **Google Drive** OAuth2<br>- [ ] Select your invoice template in the TemplateFox node<br>- [ ] Set your company details directly in the template (not in the Code node)<br>- [ ] Set your **Google Drive folder ID** in the Google Drive node<br><br>### Customize<br>- Change the email body in the Gmail node<br>- Adjust the invoice number format in the Code node<br>- Add a Slack notification node after Google Drive for team alerts<br>## 2. Fetch & format data<br>Retrieve line items from Stripe and format them for the invoice template. |
| Format Invoice Data | n8n-nodes-base.code | Converts Stripe data into invoice template fields | Get Line Items | TemplateFox | ### Stripe payment → Invoice PDF → Gmail + Google Drive<br>Automatically generate and deliver professional invoice PDFs when a Stripe payment succeeds.<br><br>**Full tutorial:** [pdftemplateapi.com/n8n/automate-invoices](https://pdftemplateapi.com/n8n/automate-invoices)<br><br>### How it works<br>1. **Stripe Trigger** fires on every successful checkout session<br>2. **HTTP Request** fetches the line items for that session<br>3. **Code** node formats the data to match your invoice template fields<br>4. **TemplateFox** generates a branded PDF invoice<br>5. Invoice is **emailed** to the customer and **saved to Google Drive**<br><br>### Setup<br>- [ ] Add your **Stripe Secret Key** (`sk_live_` or `sk_test_`) from [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)<br>- [ ] Install the **TemplateFox** community node (`n8n-nodes-templatefox`)<br>- [ ] Add your **TemplateFox** API key — [get one free](https://app.pdftemplateapi.com)<br>- [ ] Connect **Gmail** OAuth2<br>- [ ] Connect **Google Drive** OAuth2<br>- [ ] Select your invoice template in the TemplateFox node<br>- [ ] Set your company details directly in the template (not in the Code node)<br>- [ ] Set your **Google Drive folder ID** in the Google Drive node<br><br>### Customize<br>- Change the email body in the Gmail node<br>- Adjust the invoice number format in the Code node<br>- Add a Slack notification node after Google Drive for team alerts<br>## 2. Fetch & format data<br>Retrieve line items from Stripe and format them for the invoice template. |
| TemplateFox | n8n-nodes-templatefox.templateFox | Generates the invoice PDF from template data | Format Invoice Data | Email Invoice, Download PDF | ### Stripe payment → Invoice PDF → Gmail + Google Drive<br>Automatically generate and deliver professional invoice PDFs when a Stripe payment succeeds.<br><br>**Full tutorial:** [pdftemplateapi.com/n8n/automate-invoices](https://pdftemplateapi.com/n8n/automate-invoices)<br><br>### How it works<br>1. **Stripe Trigger** fires on every successful checkout session<br>2. **HTTP Request** fetches the line items for that session<br>3. **Code** node formats the data to match your invoice template fields<br>4. **TemplateFox** generates a branded PDF invoice<br>5. Invoice is **emailed** to the customer and **saved to Google Drive**<br><br>### Setup<br>- [ ] Add your **Stripe Secret Key** (`sk_live_` or `sk_test_`) from [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)<br>- [ ] Install the **TemplateFox** community node (`n8n-nodes-templatefox`)<br>- [ ] Add your **TemplateFox** API key — [get one free](https://app.pdftemplateapi.com)<br>- [ ] Connect **Gmail** OAuth2<br>- [ ] Connect **Google Drive** OAuth2<br>- [ ] Select your invoice template in the TemplateFox node<br>- [ ] Set your company details directly in the template (not in the Code node)<br>- [ ] Set your **Google Drive folder ID** in the Google Drive node<br><br>### Customize<br>- Change the email body in the Gmail node<br>- Adjust the invoice number format in the Code node<br>- Add a Slack notification node after Google Drive for team alerts<br>## 3. Generate & deliver<br>Create the PDF and send it via email + save to Drive. |
| Email Invoice | n8n-nodes-base.gmail | Emails the invoice link and payment details to the customer | TemplateFox |  | ### Stripe payment → Invoice PDF → Gmail + Google Drive<br>Automatically generate and deliver professional invoice PDFs when a Stripe payment succeeds.<br><br>**Full tutorial:** [pdftemplateapi.com/n8n/automate-invoices](https://pdftemplateapi.com/n8n/automate-invoices)<br><br>### How it works<br>1. **Stripe Trigger** fires on every successful checkout session<br>2. **HTTP Request** fetches the line items for that session<br>3. **Code** node formats the data to match your invoice template fields<br>4. **TemplateFox** generates a branded PDF invoice<br>5. Invoice is **emailed** to the customer and **saved to Google Drive**<br><br>### Setup<br>- [ ] Add your **Stripe Secret Key** (`sk_live_` or `sk_test_`) from [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)<br>- [ ] Install the **TemplateFox** community node (`n8n-nodes-templatefox`)<br>- [ ] Add your **TemplateFox** API key — [get one free](https://app.pdftemplateapi.com)<br>- [ ] Connect **Gmail** OAuth2<br>- [ ] Connect **Google Drive** OAuth2<br>- [ ] Select your invoice template in the TemplateFox node<br>- [ ] Set your company details directly in the template (not in the Code node)<br>- [ ] Set your **Google Drive folder ID** in the Google Drive node<br><br>### Customize<br>- Change the email body in the Gmail node<br>- Adjust the invoice number format in the Code node<br>- Add a Slack notification node after Google Drive for team alerts<br>## 3. Generate & deliver<br>Create the PDF and send it via email + save to Drive. |
| Download PDF | n8n-nodes-base.httpRequest | Downloads the generated PDF as a binary file | TemplateFox | Save to Google Drive | ### Stripe payment → Invoice PDF → Gmail + Google Drive<br>Automatically generate and deliver professional invoice PDFs when a Stripe payment succeeds.<br><br>**Full tutorial:** [pdftemplateapi.com/n8n/automate-invoices](https://pdftemplateapi.com/n8n/automate-invoices)<br><br>### How it works<br>1. **Stripe Trigger** fires on every successful checkout session<br>2. **HTTP Request** fetches the line items for that session<br>3. **Code** node formats the data to match your invoice template fields<br>4. **TemplateFox** generates a branded PDF invoice<br>5. Invoice is **emailed** to the customer and **saved to Google Drive**<br><br>### Setup<br>- [ ] Add your **Stripe Secret Key** (`sk_live_` or `sk_test_`) from [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)<br>- [ ] Install the **TemplateFox** community node (`n8n-nodes-templatefox`)<br>- [ ] Add your **TemplateFox** API key — [get one free](https://app.pdftemplateapi.com)<br>- [ ] Connect **Gmail** OAuth2<br>- [ ] Connect **Google Drive** OAuth2<br>- [ ] Select your invoice template in the TemplateFox node<br>- [ ] Set your company details directly in the template (not in the Code node)<br>- [ ] Set your **Google Drive folder ID** in the Google Drive node<br><br>### Customize<br>- Change the email body in the Gmail node<br>- Adjust the invoice number format in the Code node<br>- Add a Slack notification node after Google Drive for team alerts<br>## 3. Generate & deliver<br>Create the PDF and send it via email + save to Drive. |
| Save to Google Drive | n8n-nodes-base.googleDrive | Uploads the invoice PDF file to Google Drive | Download PDF |  | ### Stripe payment → Invoice PDF → Gmail + Google Drive<br>Automatically generate and deliver professional invoice PDFs when a Stripe payment succeeds.<br><br>**Full tutorial:** [pdftemplateapi.com/n8n/automate-invoices](https://pdftemplateapi.com/n8n/automate-invoices)<br><br>### How it works<br>1. **Stripe Trigger** fires on every successful checkout session<br>2. **HTTP Request** fetches the line items for that session<br>3. **Code** node formats the data to match your invoice template fields<br>4. **TemplateFox** generates a branded PDF invoice<br>5. Invoice is **emailed** to the customer and **saved to Google Drive**<br><br>### Setup<br>- [ ] Add your **Stripe Secret Key** (`sk_live_` or `sk_test_`) from [dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)<br>- [ ] Install the **TemplateFox** community node (`n8n-nodes-templatefox`)<br>- [ ] Add your **TemplateFox** API key — [get one free](https://app.pdftemplateapi.com)<br>- [ ] Connect **Gmail** OAuth2<br>- [ ] Connect **Google Drive** OAuth2<br>- [ ] Select your invoice template in the TemplateFox node<br>- [ ] Set your company details directly in the template (not in the Code node)<br>- [ ] Set your **Google Drive folder ID** in the Google Drive node<br><br>### Customize<br>- Change the email body in the Gmail node<br>- Adjust the invoice number format in the Code node<br>- Add a Slack notification node after Google Drive for team alerts<br>## 3. Generate & deliver<br>Create the PDF and send it via email + save to Drive. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Stripe payment → Invoice PDF → Gmail + Google Drive`.

2. **Add a Stripe Trigger node**
   - Node type: **Stripe Trigger**
   - Event: `checkout.session.completed`
   - Connect Stripe credentials using your Stripe Secret Key.
   - Ensure the webhook is properly registered in Stripe.
   - Use the test key for test mode and live key for production.

3. **Add an HTTP Request node named `Get Line Items`**
   - Connect it after **Stripe Trigger**.
   - Node type: **HTTP Request**
   - Method: `GET`
   - URL:
     ```text
     https://api.stripe.com/v1/checkout/sessions/{{ $json.data.object.id }}/line_items
     ```
   - Authentication: **Predefined Credential Type**
   - Credential type: **Stripe API**
   - Keep the response in JSON format.
   - If available in your n8n version, consider enabling full retrieval of line items instead of a limited response if multi-item carts are expected.

4. **Add a Code node named `Format Invoice Data`**
   - Connect it after **Get Line Items**.
   - Paste JavaScript that:
     - defines company metadata
     - reads the Stripe session from the trigger node
     - reads line items from the HTTP response
     - formats customer address and invoice items
     - calculates invoice date and due date
     - returns a single JSON object with template fields
   - Use the same field names as in the provided workflow:
     - `company_name`
     - `company_tagline`
     - `company_address_1`
     - `company_address_2`
     - `company_email`
     - `company_reg`
     - `company_vat`
     - `bank_name`
     - `account_name`
     - `iban`
     - `bic`
     - `payment_terms`
     - `notes`
     - `client_company`
     - `client_address_1`
     - `client_address_2`
     - `client_email`
     - `invoice_number`
     - `invoice_date`
     - `due_date`
     - `items`
     - `subtotal`
     - `tax`
     - `total`
     - `currency`
     - `_client_email`
     - `_client_name`
     - `_pdf_filename`
   - Important implementation details:
     - Build line items from the Stripe response array
     - Convert Stripe amounts from cents to decimal currency strings
     - Generate invoice number from current date plus trailing session ID characters
     - Store helper fields for later nodes

5. **Edit the company details in the Code node**
   - Replace the placeholder values with your business information.
   - Note that the sticky note advises keeping company details in the TemplateFox template rather than the Code node. In the current workflow, they are still hardcoded in the code, so decide which source of truth you want to keep.

6. **Install the TemplateFox community node if not already available**
   - Package name: `n8n-nodes-templatefox`
   - Restart n8n if required after installation.

7. **Add a TemplateFox node named `TemplateFox`**
   - Connect it after **Format Invoice Data**.
   - Node type: **TemplateFox**
   - Operation: `generatePdf`
   - Data input mode: `fields`
   - Select your TemplateFox template ID
   - Set filename with the expression:
     ```text
     {{ $json._pdf_filename }}
     ```
   - Map the invoice fields from the Code node to the template fields expected by your selected template.
   - Configure TemplateFox credentials with your API key.

8. **Add a Gmail node named `Email Invoice`**
   - Connect it from **TemplateFox**.
   - Node type: **Gmail**
   - Operation: send email
   - Authenticate with Gmail OAuth2.
   - `To` field:
     ```text
     {{ $('Format Invoice Data').first().json._client_email }}
     ```
   - Subject:
     ```text
     Your Invoice {{ $('Format Invoice Data').first().json.invoice_number }}
     ```
   - Message body:
     ```text
     Hi {{ $('Format Invoice Data').first().json._client_name }},

     Thank you for your purchase! Please find your invoice attached below.

     📄 Download PDF: {{ $json.url }}

     Invoice #: {{ $('Format Invoice Data').first().json.invoice_number }}
     Amount: {{ $('Format Invoice Data').first().json.total }} {{ $('Format Invoice Data').first().json.currency }}
     Due date: {{ $('Format Invoice Data').first().json.due_date }}

     If you have any questions, reply to this email.

     Best regards,
     {{ $('Format Invoice Data').first().json.company_name }}
     ```
   - Note: this sends a download link, not a binary attachment.

9. **Add an HTTP Request node named `Download PDF`**
   - Also connect it from **TemplateFox** as a second branch.
   - Method: `GET`
   - URL:
     ```text
     {{ $json.url }}
     ```
   - In response settings, choose **File** as the response format so the node outputs binary data.

10. **Add a Google Drive node named `Save to Google Drive`**
    - Connect it after **Download PDF**.
    - Node type: **Google Drive**
    - Operation: `upload`
    - Authenticate with Google Drive OAuth2.
    - File name:
      ```text
      {{ $('Format Invoice Data').first().json._pdf_filename }}.pdf
      ```
    - Set the destination folder ID manually.
    - Ensure the node reads the binary file output from **Download PDF**.

11. **Verify the connection order**
    - `Stripe Trigger` → `Get Line Items` → `Format Invoice Data` → `TemplateFox`
    - `TemplateFox` → `Email Invoice`
    - `TemplateFox` → `Download PDF` → `Save to Google Drive`

12. **Configure credentials**
    - **Stripe API / Stripe Trigger**
      - Secret key from Stripe dashboard
      - Match test/live mode consistently
    - **TemplateFox**
      - API key from TemplateFox
    - **Gmail**
      - OAuth2 with send permission
    - **Google Drive**
      - OAuth2 with file upload permission

13. **Fill missing required parameters**
    - TemplateFox `templateId`
    - TemplateFox field mapping
    - Google Drive `folderId`

14. **Test with a real or test Stripe Checkout session**
    - Complete a checkout in Stripe test mode
    - Confirm the trigger fires
    - Confirm line items return correctly
    - Check that invoice fields match your template
    - Verify TemplateFox returns a valid `url`
    - Confirm the email is sent and the file appears in Drive

15. **Recommended hardening improvements**
    - Add an IF node before Gmail to ensure `_client_email` is not empty
    - Add error handling for TemplateFox generation failures
    - Add retry or fallback logic for Google Drive upload
    - Consider storing invoice metadata in a database or sheet
    - If true attachment delivery is required, replace the email link approach with an attachment-capable email configuration using the downloaded binary

### Sub-workflow setup

This workflow does not invoke any sub-workflow and does not contain any Execute Workflow node. It has a single entry point: **Stripe Trigger**.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Full workflow explanation and setup article | [https://pdftemplateapi.com/n8n/automate-invoices](https://pdftemplateapi.com/n8n/automate-invoices) |
| Stripe API keys can be obtained from the Stripe dashboard | [https://dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys) |
| TemplateFox API key signup | [https://app.pdftemplateapi.com](https://app.pdftemplateapi.com) |
| Suggested customizations: change email body, adjust invoice number format, add Slack notification after Google Drive | Workflow design note |
| Important setup note: select your invoice template in TemplateFox and set your Google Drive folder ID in the Drive node | Workflow design note |
| Important design note: sticky note advises setting company details in the template rather than in the Code node, but the current implementation hardcodes them in JavaScript | Implementation consistency note |