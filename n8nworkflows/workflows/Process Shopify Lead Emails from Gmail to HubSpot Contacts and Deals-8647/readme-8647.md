Process Shopify Lead Emails from Gmail to HubSpot Contacts and Deals

https://n8nworkflows.xyz/workflows/process-shopify-lead-emails-from-gmail-to-hubspot-contacts-and-deals-8647


# Process Shopify Lead Emails from Gmail to HubSpot Contacts and Deals

### 1. Workflow Overview

This workflow automates the processing of lead emails from Gmail that originate specifically from Shopify, extracting relevant lead details and creating or updating corresponding contacts and deals in HubSpot. It is targeted at businesses using Shopify for sales and HubSpot for CRM management, aiming to streamline lead capture and follow-up.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Poll Gmail for new emails and fetch full email data.
- **1.2 Sender Filtering:** Extract sender email and filter only Shopify-originated emails.
- **1.3 Email Content Parsing:** Parse the email body using regex to extract lead details (name, email, city, phone, message, product info).
- **1.4 Data Normalization:** Clean and normalize parsed fields into structured JSON.
- **1.5 HubSpot Integration:** Create or update a HubSpot contact and then create a linked deal representing the lead opportunity.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block polls Gmail every minute for new emails and fetches the complete email content including body and metadata.
- **Nodes Involved:** `Gmail Trigger`, `Get a message`

##### Gmail Trigger
- **Type:** Trigger node (Gmail Trigger)
- **Role:** Polls Gmail inbox every minute to emit new emails into the workflow.
- **Configuration:** Polls every minute; no additional filters set.
- **Expressions:** Uses `$json.id` to pass message ID downstream.
- **Connections:** Output to `Get a message`.
- **Failure Modes:** Gmail API auth failure; quota limits; network errors.
- **Sticky Note:** "Polls Gmail every minute and emits new emails into the flow."

##### Get a message
- **Type:** Gmail node (get message)
- **Role:** Retrieves full email content for a given message ID.
- **Configuration:** Operation set to "get"; uses messageId from trigger.
- **Expressions:** `messageId` set to `={{ $json.id }}`.
- **Connections:** Output to `Extract From Email`.
- **Failure Modes:** Message not found; API limits; auth issues.
- **Sticky Note:** "Fetches full email (body + metadata) for parsing."

---

#### 1.2 Sender Filtering

- **Overview:** Extract the sender email address from the fetched message and filter only emails sent from `mailer@shopify.com`.
- **Nodes Involved:** `Extract From Email`, `If Sender is Shopify`

##### Extract From Email
- **Type:** Function node
- **Role:** Extracts the sender’s email address from the email metadata.
- **Configuration:** JavaScript extracts `from.value[0].address` safely; adds as `extractedFromEmail` field.
- **Expressions:** `let fromEmail = $json.from?.value?.[0]?.address || null;`
- **Connections:** Output to `If Sender is Shopify`.
- **Failure Modes:** Missing or malformed `from` field; expression errors.
- **Sticky Note:** "Pulls sender address used for filtering/routing."

##### If Sender is Shopify
- **Type:** If node (string condition)
- **Role:** Filters flow to continue only if sender email exactly matches `mailer@shopify.com`.
- **Configuration:** Condition: `{{$json.extractedFromEmail}} == "mailer@shopify.com"`.
- **Connections:** True output to `Code` node (parsing); false output is implicitly ignored.
- **Failure Modes:** Case sensitivity issues; emails forwarded with altered sender fields.
- **Sticky Note:** "Processes only emails from Shopify (change to your source if needed)."

---

#### 1.3 Email Content Parsing

- **Overview:** Parses the email body text using regex to extract structured lead information including name, email, city, phone, message, product URL, and product title.
- **Nodes Involved:** `Code`

##### Code
- **Type:** Code node (JavaScript)
- **Role:** Applies regex patterns to clean email text and extract fields.
- **Configuration:** 
  - Cleans multi-line text to single-line.
  - Uses regex to extract fields by matching labels like "Name:", "Email:", "City:", etc.
  - Returns extracted fields as JSON.
- **Expressions:** Multiple regex matches like `/Name:\s*(.*?)\s*Email:/i`.
- **Connections:** Output to `Edit Fields`.
- **Failure Modes:** Email format changes breaking regex; missing fields; empty or malformed email body.
- **Sticky Note:** 
  ```
  Extracts: name, email, city, phone, message, product_url, product_title.
  Adjust regex if your email format differs.
  ```

---

#### 1.4 Data Normalization

- **Overview:** Takes parsed data and assigns normalized field names and values for downstream HubSpot integration.
- **Nodes Involved:** `Edit Fields`

##### Edit Fields
- **Type:** Set node
- **Role:** Maps extracted fields to normalized JSON keys, ensuring consistent naming.
- **Configuration:** Assigns string values for `name`, `email`, `city`, `phone`, `body`, `product_title`, `product_url`.
- **Expressions:** Directly maps from previous node’s JSON fields.
- **Connections:** Output to `Create or update a contact`.
- **Failure Modes:** Missing or null values propagate; no validation on data format.
- **Sticky Note:** "Normalizes extracted values into clean JSON for HubSpot."

---

#### 1.5 HubSpot Integration

- **Overview:** Creates or updates a HubSpot Contact with the lead information, then creates a Deal linked to that contact representing the sales opportunity.
- **Nodes Involved:** `Create or update a contact`, `Create a deal`

##### Create or update a contact
- **Type:** HubSpot node (contact resource)
- **Role:** Creates or updates a HubSpot contact by email.
- **Configuration:**
  - Auth via HubSpot App Token.
  - Fields set: email, city, firstName, websiteUrl (static: "interwood.pk"), mobilePhoneNumber.
- **Expressions:** Maps fields from normalized JSON.
- **Connections:** Output to `Create a deal`.
- **Failure Modes:** API auth failure; email conflicts; HubSpot rate limits.
- **Sticky Note:** "Creates/updates a Contact (email, name, phone, city)."

##### Create a deal
- **Type:** HubSpot node (deal resource)
- **Role:** Creates a deal linked to the contact’s HubSpot VID.
- **Configuration:**
  - Pipeline stage ID must be replaced (`YOUR_STAGE_ID` placeholder).
  - Amount hardcoded to 5.
  - Deal name set to lead name.
  - Description combines message body and product title.
  - Associates deal with contact VID.
  - Custom properties: campaign set to "Website", message set to combined lead message.
- **Expressions:** Uses expressions referencing previous nodes, e.g., `$('Edit Fields').item.json.body`.
- **Connections:** None (end node).
- **Failure Modes:** Missing or invalid pipeline stage; missing VID; API failures.
- **Sticky Note:** 
  ```
  Creates a Deal linked to the Contact.
  Set your pipeline stage (replace `YOUR_STAGE_ID`).
  ```

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                       |
|-------------------------|---------------------|-----------------------------------------------|------------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| Gmail Trigger           | Gmail Trigger       | Polls Gmail every minute for new emails       | —                      | Get a message            | Polls Gmail every minute and emits new emails into the flow.                                    |
| Get a message           | Gmail               | Fetches full email content                     | Gmail Trigger          | Extract From Email        | Fetches full email (body + metadata) for parsing.                                               |
| Extract From Email      | Function            | Extracts sender email for filtering            | Get a message          | If Sender is Shopify      | Pulls sender address used for filtering/routing.                                                |
| If Sender is Shopify    | If                  | Filters only emails from Shopify sender        | Extract From Email     | Code                     | Processes only emails from Shopify (change to your source if needed).                           |
| Code                    | Code (JavaScript)   | Parses email body with regex to extract fields | If Sender is Shopify   | Edit Fields               | Extracts: name, email, city, phone, message, product_url, product_title. Adjust regex if needed.|
| Edit Fields             | Set                 | Normalizes and cleans extracted fields         | Code                   | Create or update a contact| Normalizes extracted values into clean JSON for HubSpot.                                        |
| Create or update a contact | HubSpot            | Creates or updates a HubSpot Contact            | Edit Fields             | Create a deal             | Creates/updates a Contact (email, name, phone, city).                                            |
| Create a deal           | HubSpot             | Creates a Deal linked to the Contact            | Create or update a contact | —                      | Creates a Deal linked to the Contact. Set your pipeline stage (replace `YOUR_STAGE_ID`).         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add node: Gmail Trigger**
   - Type: Gmail Trigger
   - Parameters:
     - Poll interval: Every minute
     - No filters
   - Connect its output to the next node.

3. **Add node: Get a message (Gmail)**
   - Type: Gmail
   - Operation: Get
   - Message ID: `={{ $json.id }}`
   - Connect input from Gmail Trigger.

4. **Add node: Extract From Email (Function)**
   - Type: Function
   - Code:
     ```javascript
     let fromEmail = $json.from?.value?.[0]?.address || null;
     return [{ json: { ...$json, extractedFromEmail: fromEmail } }];
     ```
   - Connect input from Get a message node.

5. **Add node: If Sender is Shopify (If)**
   - Type: If
   - Condition: String equals
     - Value1: `={{ $json.extractedFromEmail }}`
     - Value2: `mailer@shopify.com`
   - Connect input from Extract From Email node.

6. **Add node: Code (JavaScript)**
   - Type: Code
   - Code:
     ```javascript
     const rawText = $json.text || $json.from?.text || null;
     if (!rawText) throw new Error('Email body not found.');
     const cleanMessage = rawText.replace(/\r?\n|\r/g, ' ').replace(/\s+/g, ' ');
     const nameMatch = cleanMessage.match(/Name:\s*(.*?)\s*Email:/i);
     const emailMatch = cleanMessage.match(/Email:\s*(.*?)\s*City:/i);
     const cityMatch = cleanMessage.match(/City:\s*(.*?)\s*Phone:/i);
     const phoneMatch = cleanMessage.match(/Phone:\s*(.*?)\s*Body:/i);
     const bodyMatch = cleanMessage.match(/Body:\s*(.*?)\s*(Product Url:|Product Title:|$)/i);
     const urlMatch = cleanMessage.match(/Product Url:\s*(.*?)\s*(Product Title:|$)/i);
     const titleMatch = cleanMessage.match(/Product Title:\s*(.*)/i);
     return [{
       json: {
         name: nameMatch?.[1]?.trim() || null,
         email: emailMatch?.[1]?.trim() || null,
         city: cityMatch?.[1]?.trim() || null,
         phone: phoneMatch?.[1]?.trim() || null,
         body: bodyMatch?.[1]?.trim() || null,
         product_url: urlMatch?.[1]?.trim() || null,
         product_title: titleMatch?.[1]?.trim() || null
       }
     }];
     ```
   - Connect input from If node's "true" output.

7. **Add node: Edit Fields (Set)**
   - Type: Set
   - Assign fields:
     - name: `={{ $json.name }}`
     - email: `={{ $json.email }}`
     - city: `={{ $json.city }}`
     - phone: `={{ $json.phone }}`
     - body: `={{ $json.body }}`
     - product_title: `={{ $json.product_title }}`
     - product_url: `={{ $json.product_url }}`
   - Connect input from Code node.

8. **Add node: Create or update a contact (HubSpot)**
   - Type: HubSpot
   - Resource: Contact
   - Operation: Create or Update
   - Authentication: Use HubSpot App Token credentials
   - Parameters:
     - Email: `={{ $json.email }}`
     - Additional Fields:
       - City: `={{ $json.city }}`
       - First Name: `={{ $json.name }}`
       - Website URL: `interwood.pk` (static)
       - Mobile Phone Number: `={{ $json.phone }}`
   - Connect input from Edit Fields node.

9. **Add node: Create a deal (HubSpot)**
   - Type: HubSpot
   - Resource: Deal
   - Operation: Create
   - Authentication: Use HubSpot App Token credentials
   - Parameters:
     - Stage: Replace `YOUR_STAGE_ID` with your actual HubSpot pipeline stage ID
     - Amount: `5` (static)
     - Deal Name: `={{ $json.name }}`
     - Description: `={{ $('Edit Fields').item.json.body }} of {{ $('Edit Fields').item.json.product_title }}`
     - Associated VIDs: `={{ $json.vid }}`
     - Custom Properties:
       - campaign: `Website`
       - message: `={{ $('Edit Fields').item.json.body }} of {{ $('Edit Fields').item.json.product_title }}`
   - Connect input from Create or update a contact node.

10. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                           |
|-------------------------------------------------------------------------------------------------|------------------------------------------|
| Replace `YOUR_STAGE_ID` in the "Create a deal" node with the actual HubSpot pipeline stage ID.  | HubSpot pipeline setup                    |
| Regex patterns in the Code node are tightly coupled to the email format; adjust if format changes.| Email parsing customization               |
| Website URL is statically set to `interwood.pk` in HubSpot contact creation, change if needed.  | Contact data enrichment                    |
| Workflow assumes emails come directly from `mailer@shopify.com`; modify the filter for other senders.| Sender filtering customization            |
| HubSpot credentials must be configured as App Token authentication in n8n credentials settings. | HubSpot API authentication                |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.