Recover Abandoned Shopify Carts with Email Follow-ups, HubSpot CRM & Google Sheets Tracking

https://n8nworkflows.xyz/workflows/recover-abandoned-shopify-carts-with-email-follow-ups--hubspot-crm---google-sheets-tracking-10818


# Recover Abandoned Shopify Carts with Email Follow-ups, HubSpot CRM & Google Sheets Tracking

### 1. Workflow Overview

This workflow automates the recovery of abandoned shopping carts from a Shopify store by sending personalized email follow-ups, syncing customer data with HubSpot CRM, and logging cart details into Google Sheets for performance tracking. It is designed for e-commerce teams aiming to increase conversion rates by targeting customers who initiated but did not complete checkout, focusing on carts older than 12 hours and valued over $50.

The workflow is logically divided into three main blocks:

- **1.1 Capture & Filter Carts**: Receiving Shopify checkout events, parsing cart data, and filtering carts based on age and value criteria.
- **1.2 Recovery Email & CRM Sync**: Sending a personalized recovery email to the customer, creating or updating their contact record in HubSpot, generating a detailed note about the cart, and associating this note with the contact.
- **1.3 Log Recovery Data**: Recording comprehensive cart and customer details into a Google Sheets spreadsheet for tracking and analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Capture & Filter Carts

**Overview:**  
This initial block listens for new checkout events from Shopify, extracts and processes relevant cart and customer information, then filters carts that are older than 12 hours and have a total value exceeding $50. Only carts meeting these criteria proceed further.

**Nodes Involved:**  
- Shopify Trigger  
- Parse Cart Data (Code)  
- Filter Qualified Carts (IF)  
- Sticky Note (Step 1 description)

**Node Details:**

- **Shopify Trigger**  
  - *Type & Role:* Event trigger node listening to Shopify's "checkouts/create" webhook.  
  - *Configuration:* Uses OAuth2 access token authentication connected to the Shopify store. Listens to the event when a new checkout is created.  
  - *Inputs/Outputs:* No input; outputs raw checkout JSON data.  
  - *Potential Failures:* Authentication token expiry, webhook misconfiguration, or Shopify API downtime.  
  - *Version:* v1.  
  - *Sticky Note:* Describes Step 1 of workflow.

- **Parse Cart Data (Code Node)**  
  - *Type & Role:* Custom JavaScript processing node that parses Shopify checkout data into a simplified cart object.  
  - *Configuration:* Extracts cart ID, customer email, name, cart total, checkout URL, creation timestamp, hours since creation, and line items (title and quantity). Uses optional chaining for missing shipping address fields.  
  - *Expressions:* Uses Javascript Date to calculate cart age in hours.  
  - *Inputs:* Receives checkout JSON from Shopify Trigger.  
  - *Outputs:* Emits array of simplified cart JSON objects.  
  - *Potential Failures:* Date parsing errors, missing fields, or unexpected JSON structure.  
  - *Version:* 2.

- **Filter Qualified Carts (IF Node)**  
  - *Type & Role:* Conditional node filtering carts based on age (>12 hours) and value (> $50).  
  - *Configuration:* Uses two numeric conditions combined with AND:  
    - `hoursSinceCreated > 12`  
    - `cartTotal > 50`  
  - *Inputs:* Cart JSON from Parse Cart Data node.  
  - *Outputs:* Only carts passing both conditions proceed.  
  - *Potential Failures:* Expression evaluation errors if fields are missing or malformed.  
  - *Version:* 2.

- **Sticky Note (Step 1)**  
  - *Role:* Documentation node describing the first step: capturing Shopify checkouts and filtering carts.

---

#### 1.2 Recovery Email & CRM Sync

**Overview:**  
This block sends a personalized reminder email to the customer about their abandoned cart, updates or creates the customer contact in HubSpot CRM, generates a detailed note about the cart activity, and associates that note with the contact for sales team visibility.

**Nodes Involved:**  
- Send a message (Gmail node)  
- Create or update a contact (HubSpot node)  
- Generates Note Data (Set node)  
- Create HubSpot Note (HTTP Request node)  
- Associate Note with Contact in HubSpot (HTTP Request node)  
- Sticky Note1 (Step 2 description)

**Node Details:**

- **Send a message (Gmail Node)**  
  - *Type & Role:* Sends a plain text email via Gmail OAuth2.  
  - *Configuration:*  
    - To: customer email from filtered cart data.  
    - Subject: "You left something in your cart 👀"  
    - Message body: Personalized greeting with first name, reminder about saved cart, and encouragement to contact for help.  
  - *Credentials:* Gmail OAuth2 account connected.  
  - *Inputs:* Cart JSON from Filter Qualified Carts node.  
  - *Outputs:* Email sent confirmation.  
  - *Potential Failures:* OAuth token expiration, Gmail API rate limits, invalid email addresses.  
  - *Version:* 2.1.

- **Create or update a contact (HubSpot Node)**  
  - *Type & Role:* HubSpot CRM node to create or update a contact record.  
  - *Configuration:* Uses app token authentication. Maps email, first name, and last name from filtered cart data.  
  - *Credentials:* HubSpot app token with write access.  
  - *Inputs:* Cart JSON from Filter Qualified Carts node.  
  - *Outputs:* Returns contact record including HubSpot contact VID (unique ID).  
  - *Potential Failures:* Authentication failures, invalid data, HubSpot API limits.  
  - *Version:* 2.1.

- **Generates Note Data (Set Node)**  
  - *Type & Role:* Prepares JSON payload for creating a HubSpot note.  
  - *Configuration:* Assigns two properties:  
    - `hs_note_body`: Detailed message summarizing cart contents, customer name, cart creation time, hours since creation, item quantity and title, and cart total.  
    - `hs_timestamp`: Current timestamp in milliseconds.  
  - *Inputs:* Cart JSON from Filter Qualified Carts node.  
  - *Outputs:* JSON ready for HubSpot note creation.  
  - *Potential Failures:* Missing fields causing expression errors.  
  - *Version:* 3.4.

- **Create HubSpot Note (HTTP Request Node)**  
  - *Type & Role:* Makes REST API POST request to HubSpot to create a note object.  
  - *Configuration:*  
    - URL: `https://api.hubapi.com/crm/v3/objects/notes`  
    - Method: POST  
    - Headers: Authorization Bearer token (replace `"YOUR_TOKEN_HERE"` with actual token), Content-Type: application/json  
    - Body: JSON from previous node  
  - *Inputs:* JSON from Generates Note Data node.  
  - *Outputs:* Created note object with ID.  
  - *Potential Failures:* Authentication errors, invalid token, API rate limits, malformed body.  
  - *Version:* 4.2.

- **Associate Note with Contact in HubSpot (HTTP Request Node)**  
  - *Type & Role:* Makes REST API PUT request to associate the created note with the contact.  
  - *Configuration:*  
    - URL: `https://api.hubapi.com/crm/v3/objects/notes/{noteId}/associations/contacts/{contactVid}/note_to_contact` (IDs dynamically replaced via expressions)  
    - Method: PUT  
    - Headers: Authorization Bearer token (same as above), Content-Type: application/json  
  - *Inputs:* Note ID from Create HubSpot Note node and contact VID from Create or update a contact node.  
  - *Outputs:* Association confirmation.  
  - *Potential Failures:* Missing or invalid IDs, auth errors, API limits.  
  - *Version:* 4.2.

- **Sticky Note1 (Step 2)**  
  - *Role:* Documents email sending, HubSpot contact sync, note generation, and association.

---

#### 1.3 Log Recovery Data

**Overview:**  
This block logs all relevant cart and customer details into a Google Sheets document. This includes identifiers, item names, quantities, email, cart value, timestamps, and cart age to facilitate tracking the recovery campaign's effectiveness.

**Nodes Involved:**  
- Log to Google Sheets (Google Sheets node)  
- Sticky Note2 (Step 3 description)

**Node Details:**

- **Log to Google Sheets (Google Sheets Node)**  
  - *Type & Role:* Appends a new row to a specified Google Sheets spreadsheet.  
  - *Configuration:*  
    - Document ID: Points to a specific Google Sheet (editable).  
    - Sheet Name: Points to the first sheet (`gid=0`).  
    - Columns mapped from Filter Qualified Carts node: id, first and last names, email, cart total, creation timestamp, hours since creation, first line item title and quantity.  
    - Operation: Append row (no overwrite).  
  - *Credentials:* Google Sheets OAuth2 connection.  
  - *Inputs:* Cart JSON from Filter Qualified Carts node.  
  - *Outputs:* Confirmation of append operation.  
  - *Potential Failures:* OAuth token expiry, permission issues, spreadsheet missing or renamed, rate limits.  
  - *Version:* 4.5.

- **Sticky Note2 (Step 3)**  
  - *Role:* Describes the logging of recovery data into Google Sheets.

---

### 3. Summary Table

| Node Name                        | Node Type           | Functional Role                                          | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                              |
|---------------------------------|---------------------|----------------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Shopify Trigger                 | Shopify Trigger     | Receives new Shopify checkout events                      |                             | Parse Cart Data              | Step 1: Capture & Filter Carts                                                                          |
| Parse Cart Data                 | Code                | Parses raw Shopify checkout JSON into simplified cart data | Shopify Trigger             | Filter Qualified Carts       | Step 1: Capture & Filter Carts                                                                          |
| Filter Qualified Carts          | IF                  | Filters carts older than 12 hours and > $50 value         | Parse Cart Data             | Send a message              | Step 1: Capture & Filter Carts                                                                          |
| Send a message                 | Gmail               | Sends personalized recovery email                          | Filter Qualified Carts      | Create or update a contact   | Step 2: Recovery Email & CRM Sync                                                                       |
| Create or update a contact      | HubSpot              | Creates or updates customer contact in HubSpot CRM         | Send a message              | Generates Note Data          | Step 2: Recovery Email & CRM Sync                                                                       |
| Generates Note Data             | Set                  | Prepares detailed note content for HubSpot                 | Create or update a contact  | Create HubSpot Note          | Step 2: Recovery Email & CRM Sync                                                                       |
| Create HubSpot Note             | HTTP Request         | Creates a note object in HubSpot CRM                        | Generates Note Data         | Associate Note with Contact in HubSpot | Step 2: Recovery Email & CRM Sync                                                                       |
| Associate Note with Contact in HubSpot | HTTP Request         | Links the note to the HubSpot contact                       | Create HubSpot Note         | Log to Google Sheets         | Step 2: Recovery Email & CRM Sync                                                                       |
| Log to Google Sheets            | Google Sheets        | Logs cart and customer details to Google Sheets             | Associate Note with Contact in HubSpot |                             | Step 3: Log Recovery Data                                                                               |
| Sticky Note                    | Sticky Note          | Describes Step 1 logic                                      |                             |                             | Step 1: Capture & Filter Carts                                                                          |
| Sticky Note1                   | Sticky Note          | Describes Step 2 logic                                      |                             |                             | Step 2: Recovery Email & CRM Sync                                                                       |
| Sticky Note2                   | Sticky Note          | Describes Step 3 logic                                      |                             |                             | Step 3: Log Recovery Data                                                                               |
| Sticky Note3                   | Sticky Note          | Overview and setup instructions                             |                             |                             | Provides summary overview and setup steps for entire workflow                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger Node**  
   - Type: Shopify Trigger  
   - Parameters:  
     - Topic: `checkouts/create`  
     - Authentication: Use OAuth2 Access Token credentials linked to your Shopify store.  
   - Position: Start of workflow.

2. **Create Code Node: Parse Cart Data**  
   - Type: Code (JavaScript)  
   - Parameters: Use this script to parse checkout data and calculate cart age in hours:  
     ```javascript
     const inputData = $input.all();
     const carts = [];

     for (const item of inputData) {
       const data = item.json;

       const createdAt = new Date(data.created_at);
       const now = new Date();
       const hoursSinceCreated = (now - createdAt) / (1000 * 60 * 60);

       carts.push({
         id: data.id,
         email: data.email,
         firstName: data.shipping_address?.first_name || 'Customer',
         lastName: data.shipping_address?.last_name || '',
         cartTotal: parseFloat(data.total_price),
         checkoutUrl: data.checkout_url || null,
         createdAt: data.created_at,
         hoursSinceCreated: hoursSinceCreated,
         lineItems: (data.line_items || []).map(item => ({
           title: item.title,
           quantity: item.quantity
         }))
       });
     }

     return carts.map(cart => ({ json: cart }));
     ```
   - Connect Shopify Trigger output to this node input.

3. **Create IF Node: Filter Qualified Carts**  
   - Type: IF  
   - Parameters:  
     - Condition 1: Number > `hoursSinceCreated` > 12  
     - Condition 2: Number > `cartTotal` > 50  
     - Combine with AND operator.  
   - Connect Parse Cart Data output to this node input.

4. **Create Gmail Node: Send a message**  
   - Type: Gmail  
   - Parameters:  
     - Send To: `={{ $json.email }}`  
     - Subject: `You left something in your cart 👀`  
     - Message (plain text):  
       ```
       Hi {{ $json.firstName }},

       We noticed you didn’t get a chance to complete your checkout, and your selected items are still waiting for you.

       Take another look whenever you’re ready — your cart has been saved, so it’ll be quick and easy to finish up.

       If you have any questions or need help along the way, just reply to this email — we’re always happy to assist.
       ```
   - Credentials: Configure Gmail OAuth2 account.  
   - Connect the "true" output of Filter Qualified Carts to this node input.

5. **Create HubSpot Node: Create or update a contact**  
   - Type: HubSpot  
   - Parameters:  
     - Email: `={{ $('Filter Qualified Carts').item.json.email }}`  
     - Additional Fields:  
       - First Name: `={{ $('Filter Qualified Carts').item.json.firstName }}`  
       - Last Name: `={{ $('Filter Qualified Carts').item.json.lastName }}`  
     - Authentication: App token (HubSpot API Token)  
   - Credentials: HubSpot app token with CRM write permissions.  
   - Connect Send a message node output to this node input.

6. **Create Set Node: Generates Note Data**  
   - Type: Set  
   - Parameters: Assign two fields:  
     - `properties.hs_note_body` (string):  
       ```
       ={{ $('Filter Qualified Carts').item.json.firstName }} {{ $('Filter Qualified Carts').item.json.lastName }} started a checkout on {{ $('Filter Qualified Carts').item.json.createdAt }}, about {{ $('Filter Qualified Carts').item.json.hoursSinceCreated }} hours ago. 
       They added {{ $('Filter Qualified Carts').item.json.lineItems[0].quantity }} item — “{{ $('Filter Qualified Carts').item.json.lineItems[0].title }}” — to their cart with a total value of {{ $('Filter Qualified Carts').item.json.cartTotal }}, but haven’t completed the purchase yet.

       A follow-up email is sent regarding the same
       ```
     - `properties.hs_timestamp` (string): `={{ Date.now() }}`  
   - Connect Create or update a contact node output to this node input.

7. **Create HTTP Request Node: Create HubSpot Note**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.hubapi.com/crm/v3/objects/notes`  
     - Method: POST  
     - Headers:  
       - Authorization: `Bearer YOUR_HUBSPOT_TOKEN` (replace accordingly)  
       - Content-Type: `application/json`  
     - Body: Use JSON from previous Set node (`={{ $json }}`)  
     - Body type: JSON  
   - Connect Generates Note Data output to this node input.

8. **Create HTTP Request Node: Associate Note with Contact in HubSpot**  
   - Type: HTTP Request  
   - Parameters:  
     - URL:  
       ```
       =https://api.hubapi.com/crm/v3/objects/notes/{{ $json.id }}/associations/contacts/{{ $('Create or update a contact').item.json.vid }}/note_to_contact
       ```  
     - Method: PUT  
     - Headers:  
       - Authorization: `Bearer YOUR_HUBSPOT_TOKEN`  
       - Content-Type: `application/json`  
   - Connect Create HubSpot Note node output to this node input.

9. **Create Google Sheets Node: Log to Google Sheets**  
   - Type: Google Sheets  
   - Parameters:  
     - Operation: Append  
     - Document ID: Your target spreadsheet ID (e.g., `10W6XFIKZAbQ-Ni-GSNMpMQQbj4y6geFKkgtw9f9KRvQ`)  
     - Sheet Name: Usually first sheet, e.g., `gid=0` or `"Sheet1"`  
     - Columns to map:  
       - Id: `={{ $('Filter Qualified Carts').item.json.id }}`  
       - item: `={{ $('Filter Qualified Carts').item.json.lineItems[0].title }}`  
       - email: `={{ $('Filter Qualified Carts').item.json.email }}`  
       - lastName: `={{ $('Filter Qualified Carts').item.json.lastName }}`  
       - quantity: `={{ $('Filter Qualified Carts').item.json.lineItems[0].quantity }}`  
       - cartTotal: `={{ $('Filter Qualified Carts').item.json.cartTotal }}`  
       - createdAt: `={{ $('Filter Qualified Carts').item.json.createdAt }}`  
       - firstName: `={{ $('Filter Qualified Carts').item.json.firstName }}`  
       - hoursSinceCreated: `={{ $('Filter Qualified Carts').item.json.hoursSinceCreated }}`  
   - Credentials: Google Sheets OAuth2 account with edit access.  
   - Connect Associate Note with Contact in HubSpot node output to this node input.

10. **Add Sticky Notes**  
    - Add descriptive sticky notes to document each step: capture/filter, email & CRM sync, logging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires valid OAuth2 credentials for Shopify, Gmail, HubSpot (app token), and Google Sheets. Ensure tokens have appropriate scopes (e.g., read checkouts, send emails, CRM contacts write, sheets edit).                           | Credentials Setup                                                                                                                           |
| HubSpot API tokens used in HTTP Request nodes must be replaced with actual tokens with sufficient permissions. For production, token management and rotation are recommended.                                                                  | HubSpot API Docs: https://developers.hubspot.com/docs/api/crm/notes                                                                         |
| Google Sheets document must exist and the connected user must have edit rights. The sheet's columns should match the specified mapping for smooth data append operations.                                                                     | Google Sheets API Docs: https://developers.google.com/sheets/api                                                                              |
| The workflow filters carts older than 12 hours and worth more than $50. Adjust these thresholds in the Filter node to fit your business needs.                                                                                                | Business Logic                                                                                                                              |
| Email personalization relies on extracted first name and uses a plain text email format for compatibility. Consider HTML templates for richer email content.                                                                                 | Gmail Node Documentation                                                                                                                    |
| The workflow is designed to run on new Shopify checkout creation events; however, Shopify webhook reliability and network issues may cause missed events. Consider periodic reconciliation jobs if needed.                                     | Shopify Webhooks                                                                                                                            |
| The custom note created in HubSpot includes cart details and timestamps, enabling sales teams to understand customer behavior and follow up accordingly.                                                                                      | HubSpot Notes Feature                                                                                                                       |
| Sticky notes in the workflow provide step-by-step explanations for easy maintainability and handover to other team members or automation agents.                                                                                              | Workflow Documentation                                                                                                                      |
| The workflow is currently inactive (`active: false`). Activate after testing with sample data to start receiving live events.                                                                                                                | Workflow Status                                                                                                                             |

---

**Disclaimer:**  
This documentation is generated from an n8n workflow automation and fully complies with content policies. All integrated data sources and APIs are used within their legal and public usage scopes.