Client Nurture & Testimonial Auto-Pilot for Gumroad with Notion and Email

https://n8nworkflows.xyz/workflows/client-nurture---testimonial-auto-pilot-for-gumroad-with-notion-and-email-8020


# Client Nurture & Testimonial Auto-Pilot for Gumroad with Notion and Email

### 1. Workflow Overview

This workflow automates a client nurturing and testimonial collection process for Gumroad product sales using Notion and email. It listens for new sales from Gumroad, records client information in Notion, sends a series of timed follow-up emails to nurture the client relationship, and requests testimonials. Incoming testimonials are captured via webhook, stored in Notion, and trigger email notifications to the owner. An error handling mechanism notifies administrators if any part of the workflow fails.

The workflow is logically divided into three main blocks:

- **1.1 Sales Processing & Client Creation:** Receives Gumroad sales data, maps it to a client format, stores it in Notion, and sends an initial delivery email.
- **1.2 Client Nurture Email Sequence:** Implements timed waits after the initial sale to send follow-up tips and testimonial request emails.
- **1.3 Testimonial Reception & Owner Notification:** Handles incoming testimonial submissions, records them in Notion, sends notification emails to the owner, and responds to the testimonial sender.
- **1.4 Error Handling:** Captures any errors in the workflow and alerts via email.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Sales Processing & Client Creation

**Overview:**  
This block captures new Gumroad sales using a webhook, transforms the data into a client format, stores the client information in Notion, and sends an immediate delivery email to the client.

**Nodes Involved:**  
- Gumroad Sale (Webhook)  
- Map Sale → Client (Function)  
- Notion — Create Client (Notion)  
- Email — Delivery (Immediate) (Email Send)

**Node Details:**

- **Gumroad Sale (Webhook)**  
  - *Type:* Webhook (Trigger)  
  - *Role:* Entry point for new Gumroad sales data via HTTP POST requests.  
  - *Configuration:* Default webhook settings; expects Gumroad sale payload.  
  - *Inputs:* External HTTP POST from Gumroad.  
  - *Outputs:* Sale data to next node.  
  - *Edge Cases:* Missing or malformed payloads, webhook authentication issues.  

- **Map Sale → Client (Function)**  
  - *Type:* Function node  
  - *Role:* Transforms raw Gumroad sale data into a structured client object suitable for Notion.  
  - *Configuration:* Custom JavaScript mapping sale fields (e.g., buyer name, email, purchase details) into client properties.  
  - *Inputs:* Data from webhook.  
  - *Outputs:* Mapped client data to Notion node.  
  - *Edge Cases:* Missing fields, data format changes from Gumroad API.  

- **Notion — Create Client (Notion)**  
  - *Type:* Notion node (Create operation)  
  - *Role:* Creates a new client record in a specified Notion database.  
  - *Configuration:* Connects to Notion via OAuth2 credentials; configured to map incoming client fields to Notion database properties.  
  - *Inputs:* Mapped client data.  
  - *Outputs:* Confirmation and created client data to next node.  
  - *Edge Cases:* Notion API rate limits, authentication errors, schema mismatches.  

- **Email — Delivery (Immediate) (Email Send)**  
  - *Type:* Email Send node  
  - *Role:* Sends a delivery confirmation or product access email immediately after client creation.  
  - *Configuration:* Uses configured SMTP or email service credentials; email template likely customized per client.  
  - *Inputs:* Client email and data from Notion node.  
  - *Outputs:* Passes control to “Wait — 3 days” node.  
  - *Edge Cases:* Email delivery failures, invalid email addresses.

---

#### 2.2 Client Nurture Email Sequence

**Overview:**  
This block manages timed delays and sends follow-up emails to nurture the client relationship, including tips and testimonial requests spaced at 3 and 7 days post-purchase.

**Nodes Involved:**  
- Wait — 3 days (Wait)  
- Email — Tips (Day 3) (Email Send)  
- Wait — 7 days (Wait)  
- Build Testimonial Link (Function)  
- Email — Testimonial Ask (Day 7) (Email Send)

**Node Details:**

- **Wait — 3 days (Wait)**  
  - *Type:* Wait node  
  - *Role:* Pauses execution for 3 days to delay sending tips email.  
  - *Configuration:* Fixed wait time of 3 days.  
  - *Inputs:* Triggered after immediate delivery email.  
  - *Outputs:* Triggers “Email — Tips (Day 3)”.  
  - *Edge Cases:* Workflow execution time limits; workflow persistence requirements.  

- **Email — Tips (Day 3) (Email Send)**  
  - *Type:* Email Send node  
  - *Role:* Sends helpful tips or follow-up content to the client on day 3 post-purchase.  
  - *Configuration:* Uses email credentials and likely a pre-defined tips email template.  
  - *Inputs:* Client email data from previous wait node.  
  - *Outputs:* Triggers “Wait — 7 days”.  
  - *Edge Cases:* Same as other email nodes (delivery failures, invalid emails).  

- **Wait — 7 days (Wait)**  
  - *Type:* Wait node  
  - *Role:* Pauses for 7 days before sending testimonial request.  
  - *Configuration:* Fixed wait of 7 days.  
  - *Inputs:* Triggered after tips email.  
  - *Outputs:* Triggers “Build Testimonial Link”.  
  - *Edge Cases:* Same as previous wait node.  

- **Build Testimonial Link (Function)**  
  - *Type:* Function node  
  - *Role:* Constructs a personalized testimonial submission link, likely embedding client identifiers or tokens.  
  - *Configuration:* Custom JavaScript generates URL including unique client data.  
  - *Inputs:* Client data from previous wait node.  
  - *Outputs:* Passes testimonial link to next email node.  
  - *Edge Cases:* Incorrect URL encoding, missing client data.  

- **Email — Testimonial Ask (Day 7) (Email Send)**  
  - *Type:* Email Send node  
  - *Role:* Sends an email requesting a testimonial with the generated link on day 7.  
  - *Configuration:* Uses email credentials and template including testimonial link.  
  - *Inputs:* Client email and testimonial link.  
  - *Outputs:* End of nurture sequence for this client branch.  
  - *Edge Cases:* Email delivery issues, broken links.

---

#### 2.3 Testimonial Reception & Owner Notification

**Overview:**  
This block handles incoming testimonial submissions via webhook, maps testimonial data, stores it in Notion, notifies the owner by email, and responds to the submitter with a thank-you message.

**Nodes Involved:**  
- Testimonial (Webhook)  
- Map Testimonial (Function)  
- Notion — Create Testimonial (Notion)  
- Email — Owner Notify (Email Send)  
- Respond — Thank You (Respond to Webhook)

**Node Details:**

- **Testimonial (Webhook)**  
  - *Type:* Webhook (Trigger)  
  - *Role:* Receives incoming testimonial submissions, presumably from clients clicking the testimonial link.  
  - *Configuration:* Default webhook expecting testimonial payloads.  
  - *Inputs:* External HTTP POST with testimonial data.  
  - *Outputs:* Passes data to mapping function.  
  - *Edge Cases:* Missing or malformed data, unauthorized submissions.  

- **Map Testimonial (Function)**  
  - *Type:* Function node  
  - *Role:* Transforms raw testimonial data into a format compatible with the Notion testimonial database schema.  
  - *Configuration:* Custom JavaScript mapping testimonial text, client info, ratings, etc.  
  - *Inputs:* Data from testimonial webhook.  
  - *Outputs:* Structured testimonial data to Notion node.  
  - *Edge Cases:* Data inconsistencies, missing fields.  

- **Notion — Create Testimonial (Notion)**  
  - *Type:* Notion node (Create operation)  
  - *Role:* Adds a new testimonial entry into the designated Notion database.  
  - *Configuration:* Connected via OAuth2; maps input fields to Notion properties.  
  - *Inputs:* Mapped testimonial data.  
  - *Outputs:* Passes control to owner notification email.  
  - *Edge Cases:* API limits, authentication errors.  

- **Email — Owner Notify (Email Send)**  
  - *Type:* Email Send node  
  - *Role:* Sends an email to the owner/admin notifying them of the new testimonial.  
  - *Configuration:* Uses configured email credentials; email content includes testimonial details.  
  - *Inputs:* Testimonial data from Notion node.  
  - *Outputs:* Triggers “Respond — Thank You”.  
  - *Edge Cases:* Email delivery failures.  

- **Respond — Thank You (Respond to Webhook)**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends an HTTP response to the testimonial submitter confirming receipt and thanking them.  
  - *Configuration:* Typical HTTP 200 response with thank-you message.  
  - *Inputs:* Triggered after owner email sent.  
  - *Outputs:* Ends webhook execution.  
  - *Edge Cases:* Network errors, delayed response time.

---

#### 2.4 Error Handling

**Overview:**  
This block captures any errors occurring anywhere in the workflow and sends an alert email to the administrator or support team.

**Nodes Involved:**  
- On Error (Error Trigger)  
- Email — Error Alert (Email Send)

**Node Details:**

- **On Error (Error Trigger)**  
  - *Type:* Error Trigger node  
  - *Role:* Global error catcher for the workflow, activating when any node throws an error.  
  - *Configuration:* Default error trigger settings.  
  - *Inputs:* None (listens globally).  
  - *Outputs:* Passes error details to alert email node.  
  - *Edge Cases:* None (intended for error catching).  

- **Email — Error Alert (Email Send)**  
  - *Type:* Email Send node  
  - *Role:* Sends an alert email with error details to owners or administrators.  
  - *Configuration:* Uses email credentials; message includes error context.  
  - *Inputs:* Error data from trigger node.  
  - *Outputs:* End of error handling branch.  
  - *Edge Cases:* Email delivery failures during error state.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                              | Input Node(s)            | Output Node(s)                    | Sticky Note                                                         |
|----------------------------|---------------------|----------------------------------------------|--------------------------|----------------------------------|--------------------------------------------------------------------|
| 📌 Sticky — Overview & Setup| Sticky Note         | Informational / setup guidance                |                          |                                  |                                                                    |
| Gumroad Sale (Webhook)       | Webhook             | Receives Gumroad sales data                    |                          | Map Sale → Client                 |                                                                    |
| Map Sale → Client            | Function            | Maps sale data to client format                | Gumroad Sale (Webhook)    | Notion — Create Client           |                                                                    |
| Notion — Create Client       | Notion              | Creates client record in Notion                 | Map Sale → Client         | Email — Delivery (Immediate)     |                                                                    |
| Email — Delivery (Immediate) | Email Send          | Sends immediate delivery email                   | Notion — Create Client    | Wait — 3 days                   |                                                                    |
| Wait — 3 days               | Wait                | Waits 3 days before next email                   | Email — Delivery (Immediate) | Email — Tips (Day 3)           |                                                                    |
| Email — Tips (Day 3)         | Email Send          | Sends tips email on day 3                        | Wait — 3 days             | Wait — 7 days                   |                                                                    |
| Wait — 7 days               | Wait                | Waits 7 days before testimonial ask email       | Email — Tips (Day 3)      | Build Testimonial Link           |                                                                    |
| Build Testimonial Link       | Function            | Generates personalized testimonial submission link | Wait — 7 days             | Email — Testimonial Ask (Day 7) |                                                                    |
| Email — Testimonial Ask (Day 7) | Email Send       | Sends testimonial request email with link      | Build Testimonial Link    |                                  |                                                                    |
| Testimonial (Webhook)        | Webhook             | Receives testimonial submissions                |                          | Map Testimonial                 |                                                                    |
| Map Testimonial             | Function            | Maps testimonial data to Notion format          | Testimonial (Webhook)     | Notion — Create Testimonial      |                                                                    |
| Notion — Create Testimonial  | Notion              | Creates testimonial record in Notion             | Map Testimonial           | Email — Owner Notify             |                                                                    |
| Email — Owner Notify         | Email Send          | Notifies owner of new testimonial                | Notion — Create Testimonial | Respond — Thank You            |                                                                    |
| Respond — Thank You          | Respond to Webhook  | Sends thank-you response to testimonial submitter | Email — Owner Notify      |                                  |                                                                    |
| On Error                   | Error Trigger       | Global error catcher                             |                          | Email — Error Alert              |                                                                    |
| Email — Error Alert          | Email Send          | Sends error alert email                           | On Error                  |                                  |                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note Node:**  
   - Name: "📌 Sticky — Overview & Setup"  
   - Purpose: Place a note for overview and setup instructions.

2. **Create Gumroad Sale Webhook Node:**  
   - Type: Webhook  
   - Purpose: Trigger on new Gumroad sales; accept POST requests.  
   - Configure webhook URL and method.  

3. **Add Function Node "Map Sale → Client":**  
   - Connect from Gumroad Sale webhook.  
   - Purpose: Map raw Gumroad sale payload to client properties (e.g., name, email).  
   - Write JavaScript to extract and format data accordingly.

4. **Add Notion Node "Notion — Create Client":**  
   - Connect from "Map Sale → Client".  
   - Operation: Create  
   - Configure Notion credentials (OAuth2).  
   - Set target Notion database and map incoming fields to database properties.

5. **Add Email Send Node "Email — Delivery (Immediate)":**  
   - Connect from "Notion — Create Client".  
   - Configure email credentials (SMTP or external service).  
   - Compose delivery confirmation email using client data.

6. **Add Wait Node "Wait — 3 days":**  
   - Connect from "Email — Delivery (Immediate)".  
   - Set wait time: 3 days.

7. **Add Email Send Node "Email — Tips (Day 3)":**  
   - Connect from "Wait — 3 days".  
   - Configure email credentials.  
   - Compose tip email content.

8. **Add Wait Node "Wait — 7 days":**  
   - Connect from "Email — Tips (Day 3)".  
   - Set wait time: 7 days.

9. **Add Function Node "Build Testimonial Link":**  
   - Connect from "Wait — 7 days".  
   - Write JavaScript to generate unique testimonial submission URL, embedding client identifiers.

10. **Add Email Send Node "Email — Testimonial Ask (Day 7)":**  
    - Connect from "Build Testimonial Link".  
    - Configure email credentials.  
    - Compose testimonial request email including the generated link.

11. **Create Testimonial Webhook Node "Testimonial (Webhook)":**  
    - Type: Webhook  
    - Purpose: Receive testimonial submissions from clients.  
    - Configure webhook URL.

12. **Add Function Node "Map Testimonial":**  
    - Connect from "Testimonial (Webhook)".  
    - Map testimonial payload to Notion testimonial database format.

13. **Add Notion Node "Notion — Create Testimonial":**  
    - Connect from "Map Testimonial".  
    - Operation: Create  
    - Configure Notion credentials and target testimonial database.

14. **Add Email Send Node "Email — Owner Notify":**  
    - Connect from "Notion — Create Testimonial".  
    - Configure email credentials.  
    - Compose notification email including testimonial details.

15. **Add Respond to Webhook Node "Respond — Thank You":**  
    - Connect from "Email — Owner Notify".  
    - Set response message thanking the submitter.

16. **Add Error Trigger Node "On Error":**  
    - Type: Error Trigger  
    - Listens globally for any workflow errors.

17. **Add Email Send Node "Email — Error Alert":**  
    - Connect from "On Error".  
    - Configure email credentials.  
    - Compose alert email with error details for administrators.

18. **Connect all nodes according to the flow described in section 1 and 2.**

19. **Test all webhook URLs externally to ensure they accept data and trigger the workflow properly.**

20. **Validate Notion database schemas match expected properties and email templates are correctly formatted.**

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow automates nurturing and testimonial gathering for Gumroad product sales with Notion integration. | N8N Workflow: "Client Nurture & Testimonial Auto-Pilot (Gumroad → Notion → Email)"                                   |
| Ensure Notion databases have appropriate schemas for clients and testimonials with required properties.       | Notion API documentation: https://developers.notion.com/                                                           |
| Configure email credentials properly to avoid spam filters or delivery failures.                               | Use verified SMTP or transactional email services like SendGrid, Mailgun, etc.                                      |
| Webhook nodes require publicly accessible URLs; consider using n8n cloud or tunneling tools for local testing.| n8n Webhooks docs: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/                                                 |
| Error handling email alerts help maintain workflow reliability and prompt issue resolution.                   |                                                                                                                     |

---

**Disclaimer:** This document is generated exclusively from an n8n automated workflow. It adheres strictly to content policies and contains no illegal or protected material. All data handled is legal and public.