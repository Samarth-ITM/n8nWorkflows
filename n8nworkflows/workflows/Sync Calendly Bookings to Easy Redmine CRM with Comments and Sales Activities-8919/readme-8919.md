Sync Calendly Bookings to Easy Redmine CRM with Comments and Sales Activities

https://n8nworkflows.xyz/workflows/sync-calendly-bookings-to-easy-redmine-crm-with-comments-and-sales-activities-8919


# Sync Calendly Bookings to Easy Redmine CRM with Comments and Sales Activities

### 1. Workflow Overview

This workflow automates synchronization of Calendly bookings with Easy Redmine CRM by creating comments and sales activities on the corresponding lead record, and sending an email notification via Outlook. It targets sales, customer success, or support teams who schedule meetings using Calendly and want to maintain up-to-date CRM records and calendar notifications without manual data entry.

**Logical Blocks:**

- **1.1 Calendly Event Trigger:** Listens for new meeting bookings in Calendly.
- **1.2 Lead Lookup in Easy Redmine CRM:** Finds the CRM lead matching the invitee’s email.
- **1.3 Add Comment to Lead:** Posts a comment on the lead profile summarizing the meeting.
- **1.4 Create Sales Activity:** Logs the meeting as a sales activity linked to the lead.
- **1.5 Outlook Email Notification:** Sends a meeting notification email to the manager or assigned user.

---

### 2. Block-by-Block Analysis

#### 2.1 Calendly Event Trigger

- **Overview:**  
  This block triggers the workflow automatically when a new booking is created in Calendly for the organization scope.

- **Nodes Involved:**  
  - Calendly Trigger

- **Node Details:**

  - **Calendly Trigger**  
    - *Type:* Calendly Trigger node  
    - *Role:* Initiates the workflow on new booking events (`invitee.created`).  
    - *Configuration:*  
      - Scope set to "organization" to listen for all bookings in the organization.  
      - Event filtered to `invitee.created`.  
      - Authentication via OAuth2 with connected Calendly developer credentials.  
    - *Key Expressions:* Payload data accessed downstream for invitee name, email, meeting time, and questions/answers.  
    - *Input:* None (trigger)  
    - *Output:* Emits booking payload JSON to downstream nodes.  
    - *Edge Cases / Failures:*  
      - Authentication failures on OAuth2 token expiration or revocation.  
      - Network or API rate limiting from Calendly.  
      - Missed events if webhook is unsubscribed or misconfigured.  

#### 2.2 Lead Lookup in Easy Redmine CRM

- **Overview:**  
  Searches for an Easy Redmine CRM lead matching the invitee’s email address from the Calendly booking.

- **Nodes Involved:**  
  - Get ID of Account from email

- **Node Details:**

  - **Get ID of Account from email**  
    - *Type:* HTTP Request node  
    - *Role:* Queries Easy Redmine API to find leads filtered by email.  
    - *Configuration:*  
      - GET request to `https://yourappname.easyredmine.com/easy_leads.json` with a filter on the invitee email extracted from Calendly payload.  
      - Authentication via HTTP Header Auth using API key/token credential named "Jane Doe Account".  
    - *Key Expressions:* URL dynamically constructed using `{{$json.payload.email}}` from Calendly trigger.  
    - *Input:* Output from Calendly Trigger.  
    - *Output:* JSON array of matching lead(s) with their IDs.  
    - *Edge Cases / Failures:*  
      - No leads found for email – downstream nodes must handle empty response gracefully.  
      - API authentication errors or endpoint unavailability.  
      - URL or filter syntax errors.  

#### 2.3 Add Comment to Lead

- **Overview:**  
  Adds a comment on the matched lead’s profile in Easy Redmine with meeting details from Calendly.

- **Nodes Involved:**  
  - Add Comment

- **Node Details:**

  - **Add Comment**  
    - *Type:* Easy Redmine node (custom, from easy-redmine n8n package)  
    - *Role:* Posts a comment to the lead entity with meeting info.  
    - *Configuration:*  
      - Resource set to `easy_leads` and operation `add-comment`.  
      - Lead ID dynamically retrieved from previous node’s output (`{{$json.easy_leads[0].id}}`).  
      - Comment content formatted using invitee name, scheduled event start time, and meeting description (answers to Calendly questions).  
    - *Key Expressions:* Comment built using expressions accessing Calendly Trigger node's JSON payload.  
    - *Input:* Lead ID and booking details from previous HTTP Request node.  
    - *Output:* Confirmation of comment creation.  
    - *Edge Cases / Failures:*  
      - Missing lead ID if no matching lead found.  
      - API errors posting comment due to permissions or endpoint issues.  
      - Malformed comment text or missing booking fields.  

#### 2.4 Create Sales Activity

- **Overview:**  
  Creates a sales activity entry in Easy Redmine tied to the lead for tracking purposes.

- **Nodes Involved:**  
  - Sales Activity POST

- **Node Details:**

  - **Sales Activity POST**  
    - *Type:* HTTP Request node  
    - *Role:* POSTs a new sales activity to Easy Redmine’s activities endpoint.  
    - *Configuration:*  
      - URL: `https://yourappname.easyredmine.com/easy_entity_activities.json`  
      - Method: POST  
      - Body: JSON with `entity_type` as `EasyLead`, `entity_id` from lead lookup, activity name "Book a demo", description with booking details, and `activity_type_id` set to 1 (assumed to be phone call or meeting).  
      - Authentication: HTTP Header Auth with same credential as above.  
    - *Key Expressions:* Uses Calendly Trigger payload and lead ID from lead lookup node.  
    - *Input:* Lead ID and booking details.  
    - *Output:* JSON response with created sales activity entity details.  
    - *Edge Cases / Failures:*  
      - API errors creating activity (permissions, invalid data).  
      - Missing or invalid lead ID.  
      - Network or timeout issues.  

#### 2.5 Outlook Email Notification

- **Overview:**  
  Sends an email via Microsoft Outlook to notify the manager or assigned user about the new meeting and sales activity.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Send a message**  
    - *Type:* Microsoft Outlook node  
    - *Role:* Sends an email notification about the new sales activity and meeting.  
    - *Configuration:*  
      - Subject dynamically includes invitee name from Calendly.  
      - Body content includes a link to the CRM lead and meeting details (time and description).  
      - Recipient email taken from first event member's user email in Calendly payload.  
      - Authentication via OAuth2 connected Outlook account.  
    - *Key Expressions:* Uses Calendly Trigger payload and sales activity entity ID from previous node.  
    - *Input:* Sales activity creation output to get entity ID; Calendly booking details.  
    - *Output:* Confirmation of email sent.  
    - *Edge Cases / Failures:*  
      - OAuth2 token expiration or revocation.  
      - Invalid recipient email or missing data in Calendly payload.  
      - Email sending failures due to Outlook API limits or network issues.  

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                                   | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                       |
|-----------------------------|--------------------------------|-------------------------------------------------|---------------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note1                | Sticky Note                    | Workflow overview and detailed usage instructions| None                      | None                         | Contains a comprehensive explanation of the workflow purpose, setup steps, and use cases.       |
| Sticky Note3                | Sticky Note                    | Stepwise summary of workflow blocks              | None                      | None                         | Summarizes the main functional stages of the workflow.                                          |
| Sticky Note4                | Sticky Note                    | Example of final outputs                           | None                      | None                         | Shows sample comment, sales activity, and Outlook email output formatting.                       |
| Calendly Trigger            | Calendly Trigger               | Starts workflow on new Calendly booking           | None                      | Get ID of Account from email  |                                                                                                 |
| Get ID of Account from email| HTTP Request                  | Finds Easy Redmine lead by invitee email          | Calendly Trigger          | Add Comment, Sales Activity POST |                                                                                                 |
| Add Comment                 | Easy Redmine (custom node)     | Adds meeting details as comment on lead           | Get ID of Account from email | None                         |                                                                                                 |
| Sales Activity POST         | HTTP Request                  | Creates sales activity linked to lead              | Get ID of Account from email | Send a message               |                                                                                                 |
| Send a message              | Microsoft Outlook              | Sends notification email about meeting and activity| Sales Activity POST       | None                         |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Calendly Trigger" node:**
   - Type: Calendly Trigger  
   - Set Scope to "organization"  
   - Select event `invitee.created`  
   - Authenticate using OAuth2 credentials connected to your Calendly developer app.

2. **Create "Get ID of Account from email" node:**
   - Type: HTTP Request  
   - Method: GET  
   - URL template: `https://yourappname.easyredmine.com/easy_leads.json?set_filter=1&f[email]={{ $json.payload.email }}`  
   - Authentication: HTTP Header Auth with your Easy Redmine API key/token (credential named e.g. "Jane Doe Account").

3. **Connect "Calendly Trigger" node output to "Get ID of Account from email" input.**

4. **Create "Add Comment" node:**
   - Type: Easy Redmine node (from `@easysoftware/n8n-nodes-easy-redmine` package)  
   - Operation: `add-comment`  
   - Resource: `easy_leads`  
   - ID: `={{ $json.easy_leads[0].id }}` (from previous node output)  
   - Comment: Use expression to format:  
     `Book a demo with {{ $('Calendly Trigger').item.json.payload.name }} on {{ $('Calendly Trigger').item.json.payload.scheduled_event.start_time }}. Meeting description: {{ $('Calendly Trigger').item.json.payload.questions_and_answers[0].answer }}.`

5. **Connect "Get ID of Account from email" node output to "Add Comment" node input.**

6. **Create "Sales Activity POST" node:**
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://yourappname.easyredmine.com/easy_entity_activities.json`  
   - Body (JSON):  
     ```json
     {
       "easy_entity_activity": {
         "entity_type": "EasyLead",
         "entity_id": "{{ $json.easy_leads[0].id }}",
         "name": "Book a demo",
         "description": "Book a demo with {{ $('Calendly Trigger').item.json.payload.name }} on {{ $('Calendly Trigger').item.json.payload.scheduled_event.start_time }}. Meeting description: {{ $('Calendly Trigger').item.json.payload.questions_and_answers[0].answer }}.",
         "activity_type_id": 1
       }
     }
     ```  
   - Authentication: HTTP Header Auth with the same Easy Redmine API credential.

7. **Connect "Get ID of Account from email" node output to "Sales Activity POST" node input.**

8. **Create "Send a message" node:**
   - Type: Microsoft Outlook  
   - Subject: `Demo meeting with {{ $('Calendly Trigger').item.json.payload.name }}`  
   - Body content:  
     ```
     New Sales Activity was created https://easyredmine.com/easy_leads/{{ $json.easy_entity_activity.entity.id }}
     Please check your calendar meeting with {{ $('Calendly Trigger').item.json.payload.name }} on {{ $('Calendly Trigger').item.json.payload.scheduled_event.start_time }}
     ```  
   - To recipients: `={{ $('Calendly Trigger').item.json.payload.scheduled_event.event_memberships[0].user_email }}`  
   - Authentication: OAuth2 via connected Microsoft Outlook account.

9. **Connect "Sales Activity POST" node output to "Send a message" node input.**

10. **Verify all credentials are configured:**
    - Calendly OAuth2 credentials with developer app access.  
    - Easy Redmine API key/token credential with permission to read leads, post comments, and create activities.  
    - Outlook OAuth2 credential for sending emails.

11. **Test workflow by scheduling a test booking in Calendly, ensure:**
    - Lead is found in Easy Redmine.  
    - Comment is added with correct details.  
    - Sales activity is created and linked to the lead.  
    - Email notification is sent to the specified Outlook user.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow is designed for seamless synchronization between Calendly, Easy Redmine CRM, and Microsoft Outlook to automate sales and support meeting logging.                                                                                                                                                      | Workflow purpose overview (Sticky Note1)                                                               |
| For setting up Calendly API OAuth2 credentials, visit the [Calendly Developer Portal](https://developer.calendly.com/) and follow n8n’s guide: https://docs.n8n.io/integrations/builtin/credentials/calendly/                                                                                                         | Calendly API credential setup                                                                           |
| Easy Redmine API must be enabled with permission to access leads, add comments, and post sales activities. Ensure correct API endpoint URLs and authentication tokens.                                                                                                                                               | Easy Redmine API setup                                                                                   |
| Outlook integration requires OAuth2 connection with proper mailbox permissions to send emails.                                                                                                                                                                                                                        | Microsoft Outlook OAuth2 credential setup                                                              |
| To extend the workflow: add conditional filters on meeting types, create leads if none found, customize email content, or add tags to comments for better categorization.                                                                                                                                             | Possible workflow customizations                                                                        |
| Community support and help available via n8n community [Easy8.ai user](https://community.n8n.io/u/easy8.ai), direct contact with Easy8.ai team, and YouTube channel [easy8ai](https://www.youtube.com/@easy8ai)                                                                                                         | Support and contact information                                                                         |
| Example final output formatting includes clear and concise comments, sales activity labels, and email messages with direct CRM links for easy access and tracking.                                                                                                                                                   | Sticky Note4 example output formatting                                                                 |

---

This document provides a complete and detailed reference for understanding, reproducing, and maintaining the "Sync Calendly Bookings to Easy Redmine CRM with Comments and Sales Activities" workflow in n8n.