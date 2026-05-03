Automated Stripe to KlickTipp: Send Purchase Confirmation Emails via Tagging

https://n8nworkflows.xyz/workflows/automated-stripe-to-klicktipp--send-purchase-confirmation-emails-via-tagging-6092


# Automated Stripe to KlickTipp: Send Purchase Confirmation Emails via Tagging

### 1. Workflow Overview

This n8n workflow automates the process of capturing Stripe checkout session completions and syncing relevant purchase data into KlickTipp to send automated purchase confirmation emails. It targets digital product sellers, course creators, and service providers who want a seamless, end-to-end automated solution for confirming sales and triggering follow-up campaigns.

**Logical blocks included:**

- **1.1 Data Reception & Initial Capture**  
  Listens to Stripe webhook events for completed checkout sessions and retrieves related line items and payment details.

- **1.2 Data Enrichment & Processing**  
  Fetches associated charge and invoice information (such as receipt URL, amount) required for contact enrichment.

- **1.3 Contact Synchronization & Tagging in KlickTipp**  
  Creates or updates a KlickTipp subscriber record with the buyer’s information, purchase details, and assigns a specific tag to trigger a post-purchase campaign.

- **1.4 Documentation & Setup Guidance (Sticky Notes)**  
  Provides detailed textual documentation within the workflow for users to understand setup, benefits, and customization options.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Reception & Initial Capture

- **Overview:**  
  This block listens for Stripe checkout session completed events, then retrieves line items from the completed session to prepare purchase data for further processing.

- **Nodes Involved:**  
  - New checkout session completed (Stripe Trigger)  
  - Getting line items (HTTP Request)  
  - Getting charge ID (HTTP Request)  

- **Node Details:**

1. **New checkout session completed**  
   - *Type & Role:* Stripe Trigger node; triggers on `checkout.session.completed` event.  
   - *Configuration:* Watches Stripe events using Stripe API credentials; webhook configured with ID for Stripe to call upon checkout completion.  
   - *Expressions:* Emits event data containing checkout session details.  
   - *Connections:* Output connects to "Getting line items".  
   - *Edge Cases:*  
     - Stripe webhook delivery failures (retry or webhook secret mismatch).  
     - Event propagation delay causing missing or delayed triggers.

2. **Getting line items**  
   - *Type & Role:* HTTP Request node; fetches the line items for the specific checkout session from Stripe API.  
   - *Configuration:* URL dynamically built using `{{ $json.data.object.id }}` from Stripe event; sends authorization header with Stripe API key; specifies Stripe API version in headers to ensure compatibility.  
   - *Expressions:* Uses the checkout session ID from the previous node.  
   - *Connections:* Output feeds into “Getting charge ID”.  
   - *Edge Cases:*  
     - HTTP errors (401 Unauthorized if API key invalid, 404 if session ID invalid).  
     - API rate limits from Stripe.  
     - API version mismatches causing unexpected response structure.

3. **Getting charge ID**  
   - *Type & Role:* HTTP Request node; retrieves payment intent details to obtain charge ID and receipt URL.  
   - *Configuration:* URL uses payment_intent ID from the checkout event JSON; sends authorization header with Stripe API key (must be base64 encoded or use Basic Auth format).  
   - *Expressions:* Uses `{{ $('New checkout session completed').item.json.data.object.payment_intent }}` to get payment intent ID.  
   - *Connections:* Output feeds into "Getting invoice link".  
   - *Edge Cases:*  
     - Authorization failures if Stripe key is incorrect or missing.  
     - Payment intent may be null or incomplete if payment fails or is pending.  
     - API response changes if Stripe updates endpoint.

---

#### 2.2 Data Enrichment & Processing

- **Overview:**  
  This block fetches detailed charge information from Stripe, including receipt URLs and payment amounts, to enrich contact data before syncing it to KlickTipp.

- **Nodes Involved:**  
  - Getting invoice link (Stripe node)  

- **Node Details:**

1. **Getting invoice link**  
   - *Type & Role:* Stripe node (Stripe charge resource); retrieves charge details using charge ID from previous HTTP request.  
   - *Configuration:* Charge ID set dynamically as `{{ $json.latest_charge }}` from the prior node’s output.  
   - *Expressions:* Uses latest charge ID to fetch payment details including receipt URL, amount, and billing information.  
   - *Connections:* Output feeds into "Subscribe buyer to KlickTipp".  
   - *Edge Cases:*  
     - Charge ID may be missing or invalid if payment not completed.  
     - Network or Stripe API downtime could cause retrieval failures.  
     - Receipt URL may be null if Stripe has not generated it yet.

---

#### 2.3 Contact Synchronization & Tagging in KlickTipp

- **Overview:**  
  This block writes the collected Stripe checkout data into KlickTipp contact records, enriching them with purchase details and applying a tag to trigger confirmation email campaigns.

- **Nodes Involved:**  
  - Subscribe buyer to KlickTipp (CUSTOM.klicktipp node)  

- **Node Details:**

1. **Subscribe buyer to KlickTipp**  
   - *Type & Role:* Custom KlickTipp node; subscribes or updates a contact in KlickTipp with Stripe purchase data and tags them.  
   - *Configuration:*  
     - Email taken from Stripe billing details `{{ $json.billing_details.email }}`.  
     - Tag ID hardcoded to `13201265` to identify the Stripe purchase confirmation campaign.  
     - Custom fields populated with:  
       - Amount paid (`{{ $json.amount }}`)  
       - Receipt URL (`{{ $json.receipt_url }}`)  
       - Stripe payment ID (`{{ $('New checkout session completed').item.json.data.object.id }}`)  
       - First and last name parsed from full name string with JavaScript expressions trimming and splitting name by whitespace for accurate assignment.  
       - Product descriptions concatenated from line items.  
     - List ID set to `358895` for the KlickTipp list.  
   - *Expressions:* Complex expressions used to parse names and map product details dynamically.  
   - *Connections:* This is the last node in the chain.  
   - *Credentials:* Uses KlickTipp API credentials (username/password with API access).  
   - *Edge Cases:*  
     - Invalid or missing email will cause subscription failure.  
     - API errors due to invalid credentials, tag ID, or list ID.  
     - Parsing errors if billing name format is unusual or missing.  
     - Network or API downtime on KlickTipp side.

---

#### 2.4 Documentation & Setup Guidance (Sticky Notes)

- **Overview:**  
  Several sticky note nodes provide detailed inline documentation, describing workflow purpose, setup instructions, benefits, and customization tips.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  

- **Node Details:**

1. **Sticky Note**  
   - *Type & Role:* Visual note; marks the block “Data reception & collection via Webhook & HTTP Requests”.  
   - *Content:* Section header for clarity in workflow canvas.

2. **Sticky Note1**  
   - *Type & Role:* Visual note; marks the block “Writing checkout data into contact record”.  
   - *Content:* Section header to delineate contact sync logic.

3. **Sticky Note2**  
   - *Type & Role:* Visual note; contains comprehensive introduction, benefits, feature descriptions, setup instructions, testing, and ideas for expansion/customization.  
   - *Content:*  
     - Explains the overall workflow purpose.  
     - Describes key features such as instant confirmation emails, structured contact data, smart campaign triggering.  
     - Details setup steps for KlickTipp custom fields, credentials, and mapping.  
     - Advises testing steps and notes on Stripe API key usage.  
     - Suggests campaign expansion and customization ideas, including branching with Switch nodes and integrating with other tools.  

---

### 3. Summary Table

| Node Name                   | Node Type                | Functional Role                                      | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                                             |
|-----------------------------|--------------------------|-----------------------------------------------------|----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| New checkout session completed | Stripe Trigger           | Listens for Stripe checkout.session.completed event | —                          | Getting line items            | Data reception & collection via Webhook & HTTP Requests                                                                                 |
| Getting line items           | HTTP Request             | Retrieves line items for the checkout session       | New checkout session completed | Getting charge ID             | Data reception & collection via Webhook & HTTP Requests                                                                                 |
| Getting charge ID            | HTTP Request             | Gets payment intent details including charge ID     | Getting line items          | Getting invoice link          | Data reception & collection via Webhook & HTTP Requests                                                                                 |
| Getting invoice link         | Stripe                   | Fetches charge details including receipt URL        | Getting charge ID           | Subscribe buyer to KlickTipp  | Data reception & collection via Webhook & HTTP Requests                                                                                 |
| Subscribe buyer to KlickTipp | CUSTOM.klicktipp         | Subscribes/updates contact in KlickTipp with purchase data and tag | Getting invoice link        | —                            | Writing checkout data into contact record                                                                                              |
| Sticky Note                 | Sticky Note              | Visual label for data reception block                | —                          | —                            | Data reception & collection via Webhook & HTTP Requests                                                                                 |
| Sticky Note1                | Sticky Note              | Visual label for contact writing block                | —                          | —                            | Writing checkout data into contact record                                                                                              |
| Sticky Note2                | Sticky Note              | Detailed documentation and instructions               | —                          | —                            | Introduction, benefits, setup instructions, testing, and customization ideas                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Stripe Trigger node:**  
   - Type: Stripe Trigger  
   - Event: `checkout.session.completed`  
   - Credentials: Connect your Stripe account via API key (test or live)  
   - Purpose: Listen to real-time checkout completions.

2. **Add an HTTP Request node named "Getting line items":**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.stripe.com/v1/checkout/sessions/{{ $json.data.object.id }}/line_items`  
   - Headers:  
     - `Stripe-Version`: Use `"2025-05-28.basil"`  
     - `Authorization`: Basic auth with your Stripe API key  
   - Input: Connect from Stripe Trigger output.  
   - Purpose: Fetch purchased product line items.

3. **Add an HTTP Request node named "Getting charge ID":**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.stripe.com/v1/payment_intents/{{ $('New checkout session completed').item.json.data.object.payment_intent }}`  
   - Headers:  
     - `Authorization`: Basic auth with your Stripe API key  
   - Input: Connect from "Getting line items".  
   - Purpose: Obtain payment intent details to extract charge ID.

4. **Add a Stripe node named "Getting invoice link":**  
   - Type: Stripe  
   - Resource: Charge  
   - Operation: Get  
   - Charge ID: `{{ $json.latest_charge }}` (from previous node output)  
   - Credentials: Use Stripe API credentials  
   - Input: Connect from "Getting charge ID".  
   - Purpose: Retrieve charge details including receipt URL and amount.

5. **Add a Custom KlickTipp node named "Subscribe buyer to KlickTipp":**  
   - Type: CUSTOM.klicktipp  
   - Operation: Subscribe (create or update subscriber)  
   - List ID: Use your KlickTipp list ID (e.g., `358895`)  
   - Tag ID: Set your campaign tag ID (e.g., `13201265`)  
   - Email: `{{ $json.billing_details.email }}` from Stripe charge data  
   - Custom Fields Mapping:  
     - Amount: `{{ $json.amount }}`  
     - Receipt URL: `{{ $json.receipt_url }}`  
     - Payment ID: `{{ $('New checkout session completed').item.json.data.object.id }}`  
     - First Name: Extract first word from `{{ $json.billing_details.name }}` (trim and split by whitespace)  
     - Last Name: Extract last word from `{{ $json.billing_details.name }}` (trim and split by whitespace)  
     - Products: Map descriptions from line items via `{{ $('Getting line items').item.json.data.map(item => item.description) }}`  
   - Credentials: Configure KlickTipp API credentials (username/password with API access)  
   - Input: Connect from "Getting invoice link".  
   - Purpose: Add buyer to KlickTipp list with enriched purchase data and tag to trigger confirmation campaign.

6. **Add Sticky Notes (optional but recommended):**  
   - Add sticky notes to visually separate and document workflow blocks as per the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| This workflow enables automated confirmation email sequences by syncing Stripe checkout data into KlickTipp using tags and custom fields, ideal for digital product sellers and service providers. It captures Stripe checkout completion events, enriches contact data with purchase details, and triggers KlickTipp campaigns automatically.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow purpose and use case                                                                                            |
| Ensure KlickTipp custom fields are created matching the mapped Stripe fields: `Stripe_Beleg_URL` (URL), `Stripe_Gesamtbetrag` (Decimal), `Stripe_Zahlungs_ID` (Text), `Stripe_Produkte` (Text). Create and assign tags for each product or confirmation flow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | KlickTipp setup instructions                                                                                             |
| Use Stripe API keys carefully: test keys for development, live keys for production. Stripe events may be delayed by a few seconds; webhook reliability and retries should be monitored.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Stripe integration notes                                                                                                 |
| The workflow can be expanded with Switch nodes to branch logic by product or payment link, enabling targeted upsell flows, membership activations, or CRM entries. Use KlickTipp placeholders like `[[Stripe_Beleg_URL]]` in email templates to personalize communications.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Customization and expansion ideas                                                                                        |
| Credential setup is crucial: Stripe API key (with correct permissions), KlickTipp username/password with API access. Validate all credentials before activating the workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Credential management                                                                                                    |
| The workflow uses advanced JavaScript expressions to parse buyer names safely and concatenate product descriptions, which must be tested with various input formats to avoid errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Expression usage and error handling                                                                                      |
| Testing sequence: Set workflow active, perform a test Stripe checkout, verify contact creation and tag assignment in KlickTipp, confirm that purchase data is correctly stored and confirmation emails are triggered.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Testing instructions                                                                                                    |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.