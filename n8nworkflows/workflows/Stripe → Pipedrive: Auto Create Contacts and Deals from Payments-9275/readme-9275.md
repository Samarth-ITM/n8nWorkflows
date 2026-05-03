Stripe → Pipedrive: Auto Create Contacts and Deals from Payments

https://n8nworkflows.xyz/workflows/stripe---pipedrive--auto-create-contacts-and-deals-from-payments-9275


# Stripe → Pipedrive: Auto Create Contacts and Deals from Payments

---
### 1. Workflow Overview

This workflow automates the synchronization of payment data from Stripe into Pipedrive CRM by creating or updating contacts ("Persons") and deals based on Stripe payment events. It is designed for businesses such as SaaS, subscription services, agencies, or sales teams that want to automatically mirror payment information inside their CRM without manual intervention.

The workflow is logically structured into these main blocks:

- **1.1 Stripe Event Reception and Validation:** Receives Stripe webhook events for payment success, extracts and validates the event data by confirming with Stripe API.
- **1.2 Pipedrive Contact Lookup and Creation:** Searches for an existing contact in Pipedrive based on the customer’s email from Stripe; if none exists, creates a new contact with custom payment-related fields.
- **1.3 Deal Creation and Deduplication:** Checks if a deal for the payment event already exists to avoid duplicates, then creates a new deal linked to the correct contact with payment details.
- **1.4 Control Flow and No-Op:** Manages decision branches and skips deal creation when duplicates are detected.

---
### 2. Block-by-Block Analysis

#### 1.1 Stripe Event Reception and Validation

- **Overview:** This block initiates with a webhook that listens for Stripe payment events, extracts key event information, and validates the event by retrieving it directly from Stripe’s API to ensure authenticity and prevent fraudulent or duplicate processing.
- **Nodes Involved:**  
  - Stripe Webhook  
  - Set field  
  - Check if event id is same  
- **Node Details:**

1. **Stripe Webhook**  
   - Type: Webhook (HTTP Trigger)  
   - Role: Receives POST requests from Stripe when payment events occur.  
   - Config: Path set to a unique webhook ID; method POST.  
   - Inputs: External HTTP requests from Stripe.  
   - Outputs: Raw Stripe event JSON payload.  
   - Edge Cases: Network failures, webhook misconfiguration, unauthorized requests.  
   
2. **Set field**  
   - Type: Set Node (Data Transformation)  
   - Role: Extracts and stores the Stripe event ID (`eventId`), event type (`eventType`), and the event payload object (`payload`) for downstream use.  
   - Config: Assigns variables using expressions from the webhook JSON body.  
   - Inputs: Output of Stripe Webhook node.  
   - Outputs: Structured data with extracted fields.  
   - Edge Cases: Missing or malformed event fields, JSON parsing errors.  
   
3. **Check if event id is same**  
   - Type: HTTP Request  
   - Role: Validates the Stripe event by fetching the event data from Stripe’s API endpoint using the event ID to confirm its legitimacy and freshness.  
   - Config: GET request to `https://api.stripe.com/v1/events/{{ eventId }}`, authenticated via HTTP header with Stripe secret key.  
   - Inputs: Data from Set field node.  
   - Outputs: Confirmed Stripe event JSON data.  
   - Edge Cases: API rate limits, authentication errors, invalid event ID, network timeouts.

---

#### 1.2 Pipedrive Contact Lookup and Creation

- **Overview:** This block attempts to locate a Pipedrive contact (Person) by the customer’s email from Stripe. If found, it proceeds to deal creation; if not, it creates a new contact with enriched payment-related custom fields.
- **Nodes Involved:**  
  - Search a person  
  - if person exists (IF node)  
  - Create a person  
- **Node Details:**

1. **Search a person**  
   - Type: Pipedrive Node (Search)  
   - Role: Searches Pipedrive persons by the email address obtained from Stripe customer details.  
   - Config: Search term set to customer email, returns all matches.  
   - Inputs: Output from "Check if event id is same" node.  
   - Outputs: List of matching persons or empty if none found.  
   - Edge Cases: API authentication errors, empty or malformed email field, multiple matches.  
   
2. **if person exists**  
   - Type: IF Node (Conditional Branch)  
   - Role: Checks if the search returned any existing person by verifying if the email exists in the returned data.  
   - Config: Condition checks existence of email in JSON path.  
   - Inputs: Output of Search a person node.  
   - Outputs: Two branches: yes (person exists), no (person does not exist).  
   - Edge Cases: False negatives due to data inconsistency, missing email field.  
   
3. **Create a person**  
   - Type: Pipedrive Node (Create)  
   - Role: Creates a new person/contact in Pipedrive if no existing person is found.  
   - Config: Sets name to the customer’s email; populates email and multiple custom properties with payment details such as amount, payment method, status, and source ("Stripe").  
   - Inputs: Output from "if person exists" node (no branch).  
   - Outputs: Newly created person record.  
   - Edge Cases: Validation errors for required fields, API errors, duplicate contact creation in race conditions.

---

#### 1.3 Deal Creation and Deduplication

- **Overview:** This block creates a new deal associated with the identified or newly created person. It includes a deduplication check based on the Stripe event ID to prevent creating multiple deals for the same payment.
- **Nodes Involved:**  
  - If event id exist (IF node)  
  - Append a deal  
  - Create a deal  
  - Do nothing (NoOp)  
- **Node Details:**

1. **If event id exist**  
   - Type: IF Node  
   - Role: Checks if a deal for the current Stripe event ID has already been created to avoid duplicates.  
   - Config: Compares event IDs from the Stripe event and existing deals.  
   - Inputs: Output from "if person exists" node (yes branch) or from "Create a person" node.  
   - Outputs: Two branches: yes (event already exists), no (new event).  
   - Edge Cases: Missing or inconsistent event IDs, delayed API responses causing race conditions.  
   
2. **Append a deal**  
   - Type: Pipedrive Node (Create)  
   - Role: Creates a new deal (named "Deposit — [email] — [amount]") linked to the existing person with payment amount and custom Stripe event ID property.  
   - Config: Uses expressions to set title, value, person ID, and custom properties referencing the Stripe event ID.  
   - Inputs: Output from "If event id exist" node (no branch).  
   - Outputs: Newly created deal record.  
   - Edge Cases: API limits, invalid person ID, failed deal creation.  
   
3. **Create a deal**  
   - Type: Pipedrive Node (Create)  
   - Role: Similar to Append a deal; creates a deal linked to the person created in "Create a person" node.  
   - Config: Same as Append a deal for deal title and custom fields.  
   - Inputs: Output from "Create a person" node.  
   - Outputs: Deal record.  
   - Edge Cases: Same as Append a deal node.  
   
4. **Do nothing**  
   - Type: No Operation (NoOp)  
   - Role: Acts as a sink node when a duplicate event is detected; no action is taken.  
   - Inputs: Output from "If event id exist" node (yes branch).  
   - Outputs: None.  
   - Edge Cases: None.

---

#### 1.4 Control Flow and Data Routing

- **Overview:** This block manages logical branching between nodes, ensuring proper flow based on conditions (e.g., existence of person, event duplication), and prevents duplicate deal creation.
- **Nodes Involved:**  
  - if person exists (IF node)  
  - If event id exist (IF node)  
- **Node Details:** Covered above as part of blocks 1.2 and 1.3.

---

### 3. Summary Table

| Node Name           | Node Type              | Functional Role                                | Input Node(s)            | Output Node(s)                   | Sticky Note                                                                                       |
|---------------------|------------------------|-----------------------------------------------|--------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Stripe Webhook      | Webhook                | Receives Stripe payment webhook events         | External HTTP Request    | Set field                       | # Step 1 - Initial setup: Stripe webhook trigger                                                |
| Set field           | Set                    | Extracts eventId, eventType, and payload       | Stripe Webhook           | Check if event id is same       | # Step 1 - Initial setup: Extracts event data                                                   |
| Check if event id is same | HTTP Request       | Validates event by fetching from Stripe API    | Set field                | Search a person                 | # Step 1 - Initial setup: Confirms event via Stripe API                                        |
| Search a person      | Pipedrive Search       | Searches for existing contact by email         | Check if event id is same| if person exists                | # Step 2 - Search contact: Looks up Pipedrive contact by email                                  |
| if person exists     | IF                     | Checks if person exists in Pipedrive            | Search a person          | If event id exist, Create a person | # Step 2 - Search contact: Branches on person existence                                         |
| Create a person      | Pipedrive Create       | Creates new contact with payment details        | if person exists (no)    | Create a deal                   | # Step 2 - Search contact: Creates new Person with custom fields                                |
| Create a deal        | Pipedrive Create       | Creates deal linked to newly created person     | Create a person          |                                 | # Step 2 - Search contact: Creates deal linked to new person                                    |
| If event id exist    | IF                     | Checks for duplicate deals by event ID          | if person exists (yes)   | Do nothing, Append a deal       | # Step 3 - Checks if event is not same to avoid duplicate deal creation                         |
| Append a deal        | Pipedrive Create       | Creates deal linked to existing person          | If event id exist (no)   |                                 | # Step 3 - Creates deal linked to existing Person                                              |
| Do nothing           | NoOp                   | Skips deal creation for duplicate events        | If event id exist (yes)  |                                 |                                                                                                 |
| Sticky Note          | Sticky Note            | Setup guide and instructions                     |                          |                                 | ## Setup Guide: Author Krishna Sharma; prerequisites include n8n, Stripe and Pipedrive accounts |
| Sticky Note1         | Sticky Note            | Workflow Description                             |                          |                                 | ## Description: Explains automation purpose and ideal users                                    |
| Sticky Note2         | Sticky Note            | Step-by-step workflow explanation                |                          |                                 | ## How it works: Detailed node-by-node logic explanation                                       |
| Sticky Note3         | Sticky Note            | Explanation of event ID deduplication logic      |                          |                                 | # Step 3 - Checks if event is not same to avoid duplicate deal creation                       |
| Sticky Note4         | Sticky Note            | Initial setup explanation                         |                          |                                 | # Step 1 - Initial setup: Stripe webhook and event validation                                 |
| Sticky Note5         | Sticky Note            | Contact search and creation explanation          |                          |                                 | # Step 2 - Search contact: Search and conditional person creation                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Stripe Webhook node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use a unique webhook path (e.g., `stripe/<unique-id>`)  
   - Purpose: To receive payment event notifications from Stripe.  
   - Note: Ensure your n8n instance is publicly accessible over HTTPS and the URL is registered in Stripe webhook settings.

2. **Add a Set node ("Set field")**  
   - Extract these fields from webhook JSON payload:  
     - `eventId` = `{{$json.body.id}}`  
     - `eventType` = `{{$json.body.type}}`  
     - `payload` = `{{$json.body.data.object}}`  
   - Connect Stripe Webhook output to this node.

3. **Add an HTTP Request node ("Check if event id is same")**  
   - HTTP Method: GET  
   - URL: `https://api.stripe.com/v1/events/{{$json.eventId}}`  
   - Authentication: HTTP Header with Stripe Secret API key  
   - Connect Set field output to this node to validate the event.

4. **Add a Pipedrive node ("Search a person")**  
   - Resource: Person  
   - Operation: Search  
   - Search Term: Use the customer's email from Stripe event `{{$json.data.object.customer_details.email}}`  
   - Return all matches.  
   - Connect output of "Check if event id is same" to this node.

5. **Add an IF node ("if person exists")**  
   - Condition: Check if `{{$json.item.emails[0]}}` exists (i.e., person found)  
   - Connect output of "Search a person" to this IF node.

6. **Add a Pipedrive node ("Create a person")**  
   - Resource: Person  
   - Operation: Create  
   - Name: Use customer email from Stripe payload `{{$json.data.object.customer_details.email}}`  
   - Email: Same email as name.  
   - Add custom properties (using your Pipedrive custom field IDs):  
     - Amount: Convert amount from Stripe (e.g., `Number(...) / 100`)  
     - Source: "Stripe"  
     - Payment method: Stripe payment method type from event  
     - Payment status from event  
   - Connect the IF node "no" branch (person does not exist) to this node.

7. **Add a Pipedrive node ("Create a deal")**  
   - Resource: Deal  
   - Operation: Create  
   - Title: Format as `Deposit — {{email}} — {{amount}}`  
   - Value: Amount from Stripe event (convert cents to dollars)  
   - Person ID: Use the ID of the created person.  
   - Custom property: Stripe Event ID to track duplicates.  
   - Connect "Create a person" node's output here.

8. **Add an IF node ("If event id exist")**  
   - Condition: Compare event IDs to check if a deal for this event already exists to avoid duplicates.  
   - Connect the IF node "yes" branch (person exists) to this node.

9. **Add a Pipedrive node ("Append a deal")**  
   - Same configuration as "Create a deal" node but linked to the existing person.  
   - Connect the "If event id exist" node's "no" branch here.

10. **Add a NoOp node ("Do nothing")**  
    - Connect the "If event id exist" node's "yes" branch to this node to skip duplicate deals.

11. **Connect the nodes appropriately:**  
    - Stripe Webhook → Set field → Check if event id is same → Search a person → if person exists  
    - if person exists → (yes) → If event id exist → (yes) → Do nothing  
    - if person exists → (yes) → If event id exist → (no) → Append a deal  
    - if person exists → (no) → Create a person → Create a deal

12. **Credentials Setup:**  
    - Stripe HTTP Request node requires HTTP Header Auth with Stripe Secret Key.  
    - Pipedrive nodes require Pipedrive API credentials with permission to create/search persons and deals.

13. **Custom Fields:**  
    - Before running, create required custom fields in Pipedrive (amount, payment method, status, Stripe Event ID) and note their field IDs to use in the Create Person and Deal nodes.

14. **Test the workflow:**  
    - Trigger test payments in Stripe and confirm contacts and deals are created or updated in Pipedrive accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Author: Krishna Sharma                                                                                           | Setup Guide Sticky Note                                                                         |
| Prerequisites: n8n instance with HTTPS, Stripe account with secret key, Pipedrive account with API access        | Setup Guide Sticky Note                                                                         |
| Workflow ideal for SaaS, subscription businesses, agencies, and teams syncing payments into CRM                 | Description Sticky Note                                                                         |
| Detailed step-by-step explanation of workflow logic including webhook reception, validation, contact search, creation, deal creation, and deduplication | How it works Sticky Note                                                                        |
| Avoids duplicate deal creation by checking Stripe event IDs before deal creation                                | Deduplication Sticky Note                                                                       |

---

**Disclaimer:** The provided workflow JSON is an automated integration built with n8n respecting all content policies. It processes only legal and public data from Stripe and Pipedrive APIs.

If you require further assistance with customization or error handling improvements, feel free to ask.