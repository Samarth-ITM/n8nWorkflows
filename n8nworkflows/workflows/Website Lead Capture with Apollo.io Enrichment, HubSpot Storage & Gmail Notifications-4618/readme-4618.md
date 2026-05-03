Website Lead Capture with Apollo.io Enrichment, HubSpot Storage & Gmail Notifications

https://n8nworkflows.xyz/workflows/website-lead-capture-with-apollo-io-enrichment--hubspot-storage---gmail-notifications-4618


# Website Lead Capture with Apollo.io Enrichment, HubSpot Storage & Gmail Notifications

### 1. Workflow Overview

This workflow automates the capture, enrichment, CRM entry, and notification process for inbound leads collected via a website form. It is designed for organizations seeking to streamline lead management by integrating real-time lead intake with data enrichment (via Apollo.io), CRM contact creation (in HubSpot), and email notifications (using Gmail).

**Logical Blocks:**

- **1.1 Lead Capture and Initial Acknowledgement:** Receives inbound lead data through a webhook and sends an immediate personalized thank-you email to the lead.
- **1.2 Internal Notification:** Sends an email alert to the internal sales or marketing team notifying them of the new lead.
- **1.3 Data Enrichment:** Queries Apollo.io API to enrich the lead data based on the email provided.
- **1.4 CRM Integration:** Creates or updates the contact in HubSpot with enriched and original lead data.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Capture and Initial Acknowledgement

- **Overview:** Captures lead data submitted via a website form using a webhook and sends a personalized thank-you email to the lead’s email address.
- **Nodes Involved:** 
  - Webhook - Collect Lead
  - Gmail - Send Thank You

##### Node: Webhook - Collect Lead
- **Type:** Webhook (HTTP Request Listener)
- **Role:** Entry point for inbound lead data; listens for HTTP POST requests at path `/lead-intake`.
- **Configuration:** 
  - HTTP Method: POST
  - Response Mode: Responds with output from the last node in the workflow
  - Path: `lead-intake`
- **Expressions:** None at this node; outputs the received JSON body.
- **Input/Output:** No input; outputs HTTP payload containing lead details (e.g., first_name, email).
- **Failure Modes:** 
  - Invalid or missing POST data
  - Network or endpoint unreachable issues on the client side
  - Potential security concerns if not protected (e.g., no authentication on webhook)
- **Version-Specific:** Type version 1

##### Node: Gmail - Send Thank You
- **Type:** Gmail node for sending emails
- **Role:** Sends a personalized thank-you email to the lead’s email address captured from the webhook.
- **Configuration:** 
  - Recipient: Email extracted dynamically via `{{$node["Webhook - Collect Lead"].json.body.email}}`
  - Subject: "Thanks for Getting in touch!"
  - Message: Plain text email with personalization using lead’s first name `{{$json.body.first_name}}`
  - Options: Attribution disabled
  - Credential: OAuth2 Gmail account configured
- **Expressions:** Uses expressions to pull email and first_name from webhook payload.
- **Input/Output:** Input from Webhook node; sends email and outputs sending status.
- **Failure Modes:** 
  - Authentication failure with Gmail OAuth2 credentials
  - Rate limiting or quota exceeded on Gmail API
  - Invalid or missing email in the webhook payload
- **Version-Specific:** Version 2.1

---

#### 1.2 Internal Notification

- **Overview:** Notifies internal team by email about new lead submission.
- **Nodes Involved:** 
  - Gmail - Notify Team

##### Node: Gmail - Notify Team
- **Type:** Gmail node for sending emails
- **Role:** Sends an internal notification email with lead details to a specified team email address.
- **Configuration:** 
  - Recipient: Static email `"your_email_address"` (should be replaced with actual team address)
  - Subject: "New Lead Notification"
  - Message: Contains lead’s first name and email extracted dynamically
  - Options: Attribution disabled
  - Credential: Same Gmail OAuth2 account as Thank You email
- **Expressions:** Uses dynamic fields to access webhook data for lead info.
- **Input/Output:** Input from Webhook node; outputs email sending status.
- **Failure Modes:** 
  - Invalid or missing team email address
  - Authentication failure or Gmail API issues
- **Version-Specific:** Version 2.1

---

#### 1.3 Data Enrichment

- **Overview:** Enriches lead data by querying Apollo.io’s contacts API using the lead’s email.
- **Nodes Involved:** 
  - Enrich Data from Apollo

##### Node: Enrich Data from Apollo
- **Type:** HTTP Request node
- **Role:** Sends a POST request to Apollo.io API to retrieve enriched contact data based on the lead’s email.
- **Configuration:** 
  - URL: `https://api.apollo.io/api/v1/contacts/search`
  - Method: POST
  - Query Parameters: `q_keywords` set to lead email; `sort_ascending` set to false
  - Headers: Includes `Cache-Control: no-cache`, `accept: application/json`, and `x-api-key` with Apollo API key (to be configured)
- **Expressions:** Email extracted from webhook payload dynamically for the query parameter.
- **Input/Output:** Input from Webhook node; outputs JSON response containing enriched contact information.
- **Failure Modes:** 
  - Invalid or missing Apollo API key
  - API rate limits or downtime
  - Lead email not found in Apollo database (resulting in empty or null response)
  - Network timeouts
- **Version-Specific:** Version 4.2

---

#### 1.4 CRM Integration

- **Overview:** Creates or updates the lead as a contact in HubSpot CRM, using both original and enriched data.
- **Nodes Involved:** 
  - HubSpot - Create Contact

##### Node: HubSpot - Create Contact
- **Type:** HubSpot node (Contact creation)
- **Role:** Creates a new contact in HubSpot with fields from Apollo enrichment and webhook input.
- **Configuration:** 
  - Email: Lead email from webhook
  - Additional Fields:
    - City, Country, Job Title, Twitter Username, Company Name, Website URL, Person LinkedIn, Company LinkedIn, Company Phone, Message (composed with LinkedIn and phone data)
    - Most fields extracted from Apollo enrichment response JSON paths (e.g., `$json.contacts[0].city`)
    - First Name from webhook input
  - Authentication: HubSpot App Token
- **Expressions:** Uses complex expressions to map enriched data to HubSpot contact fields.
- **Input/Output:** Input from Apollo enrichment node; outputs HubSpot API response.
- **Failure Modes:** 
  - Invalid or expired HubSpot authentication token
  - Missing required fields or malformed data from Apollo response
  - API rate limits or connectivity issues
- **Version-Specific:** Version 2

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                         | Input Node(s)           | Output Node(s)                    | Sticky Note                                        |
|-------------------------|--------------------|---------------------------------------|------------------------|---------------------------------|---------------------------------------------------|
| Webhook - Collect Lead  | Webhook            | Receives inbound lead data             | None                   | Gmail - Send Thank You, Gmail - Notify Team, Enrich Data from Apollo |                                                   |
| Gmail - Send Thank You  | Gmail              | Sends thank-you email to lead          | Webhook - Collect Lead | None                            |                                                   |
| Gmail - Notify Team     | Gmail              | Sends internal notification email      | Webhook - Collect Lead | None                            |                                                   |
| Enrich Data from Apollo | HTTP Request       | Enriches lead data using Apollo API    | Webhook - Collect Lead | HubSpot - Create Contact         |                                                   |
| HubSpot - Create Contact| HubSpot            | Creates/updates contact in HubSpot CRM | Enrich Data from Apollo| None                            |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Webhook - Collect Lead"**
   - Type: Webhook
   - HTTP Method: POST
   - Path: `lead-intake`
   - Response Mode: Last Node
   - No authentication (consider adding security for production)
   - Position: (Optional) e.g., x=-440, y=100

2. **Create Gmail Node: "Gmail - Send Thank You"**
   - Type: Gmail
   - Credential: Configure Gmail OAuth2 credentials
   - Send To: Expression `{{$node["Webhook - Collect Lead"].json.body.email}}`
   - Subject: "Thanks for Getting in touch!"
   - Message (text):
     ```
     Thank You for Reaching Out!

     Dear {{$json.body.first_name}}

     I hope this message finds you well. Thank you for reaching out to us at InfyOm Technologies. Your interest and support mean a lot to us.

     We have received your request and our team will soon get back to you.

     Thanks,
     InfyOm Sales Team
     ```
   - Options: Disable attribution
   - Position: e.g., x=-180, y=-80
   - Connect input from "Webhook - Collect Lead"

3. **Create Gmail Node: "Gmail - Notify Team"**
   - Type: Gmail
   - Credential: Same Gmail OAuth2 credentials
   - Send To: Replace `"your_email_address"` with actual team email address
   - Subject: "New Lead Notification"
   - Message (text):
     ```
     A new form has been submitted on the website.

     Name : {{$node["Webhook - Collect Lead"].json.body.first_name}}
     Email : {{$json.body.email}}
     ```
   - Options: Disable attribution
   - Position: e.g., x=-180, y=280
   - Connect input from "Webhook - Collect Lead"

4. **Create HTTP Request Node: "Enrich Data from Apollo"**
   - Type: HTTP Request
   - HTTP Method: POST
   - URL: `https://api.apollo.io/api/v1/contacts/search`
   - Query Parameters:
     - `q_keywords` = expression `{{$json.body.email}}` (from webhook)
     - `sort_ascending` = `false`
   - Headers:
     - `Cache-Control`: `no-cache`
     - `accept`: `application/json`
     - `x-api-key`: Your Apollo API key (configure securely)
   - Position: e.g., x=-180, y=100
   - Connect input from "Webhook - Collect Lead"

5. **Create HubSpot Node: "HubSpot - Create Contact"**
   - Type: HubSpot (Contact)
   - Authentication: HubSpot App Token credentials configured
   - Email: `{{$node["Webhook - Collect Lead"].json.body.email}}`
   - Additional Fields (mapped with expressions from Apollo response JSON):
     - City: `{{$json.contacts[0].city}}`
     - Country: `{{$json.contacts[0].country}}`
     - Job Title: `{{$json.contacts[0].title}}`
     - Twitter Username: `{{$json.contacts[0].twitter_url}}`
     - Company Name: `{{$json.contacts[0].organization_name}}`
     - Website URL: `{{$json.contacts[0].account.website_url}}`
     - First Name: `{{$node["Webhook - Collect Lead"].json.body.first_name}}`
     - Message (custom text field):
       ```
       Person LinkedIn: {{$json.contacts[0].linkedin_url}}
       Company Phone: {{$json.contacts[0].account.primary_phone.number}}
       Company LinkedIn: {{$json.contacts[0].account.linkedin_url}}
       ```
   - Position: e.g., x=40, y=100
   - Connect input from "Enrich Data from Apollo"

6. **Connect the Nodes:**
   - "Webhook - Collect Lead" main output connects to three nodes:
     - "Gmail - Send Thank You"
     - "Gmail - Notify Team"
     - "Enrich Data from Apollo"
   - "Enrich Data from Apollo" main output connects to "HubSpot - Create Contact"

7. **Activate Workflow and Test:**
   - Ensure all credentials are properly configured and tested.
   - Test webhook reception by sending POST requests with appropriate JSON payload containing fields: `first_name` and `email`.
   - Check email deliveries and CRM contact creation.
   - Monitor for errors and adjust node configurations or credentials as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Replace `"your_email_address"` in "Gmail - Notify Team" node with actual internal notification email.    | Internal notifications require valid recipient email.                                           |
| Apollo.io API requires a valid API key; ensure rate limits and API quota are monitored to prevent errors.| Apollo API documentation: https://apollo.io/api/docs                                                |
| HubSpot node uses App Token authentication; ensure the token has permissions for contact creation.       | HubSpot API docs: https://developers.hubspot.com/docs/api/crm/contacts                            |
| Gmail nodes use OAuth2 credentials; token refresh and scopes must allow sending emails.                   | Gmail OAuth2 setup: https://developers.google.com/identity/protocols/oauth2                        |
| The webhook endpoint `/lead-intake` should ideally be secured (e.g., with API keys or IP restrictions).  | Security best practices for webhooks: https://n8n.io/integrations/built-in/webhook                |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created in n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.