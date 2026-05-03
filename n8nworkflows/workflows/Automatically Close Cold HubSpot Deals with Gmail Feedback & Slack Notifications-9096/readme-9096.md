Automatically Close Cold HubSpot Deals with Gmail Feedback & Slack Notifications

https://n8nworkflows.xyz/workflows/automatically-close-cold-hubspot-deals-with-gmail-feedback---slack-notifications-9096


# Automatically Close Cold HubSpot Deals with Gmail Feedback & Slack Notifications

### 1. Workflow Overview

This workflow automates the process of identifying and closing stale (cold) deals in HubSpot that have not been modified for over 21 days. It enriches these deals by retrieving associated contact information, sends personalized feedback request emails via Gmail to the contacts, and notifies the internal team on Slack about the deals closed. The workflow is designed for sales and customer success teams aiming to maintain clean pipelines by automatically closing cold deals while keeping communication channels active for potential future opportunities.

Logical blocks:

- **1.1 Trigger & Deal Retrieval:** Scheduled initiation and fetching of all HubSpot deals with relevant properties.
- **1.2 Lead Qualification (Cold Deals):** Filtering deals untouched for 21+ days and updating their status to "Closed Lost".
- **1.3 Deal to Contact Association:** Retrieving contacts linked to the closed deals and extracting their detailed information.
- **1.4 Follow-up & Notifications:** Sending personalized feedback request emails via Gmail and posting notifications to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Deal Retrieval

- **Overview:**  
  This block triggers the workflow based on a scheduled interval and fetches all deals from HubSpot, extracting key deal properties for further processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get HubSpot Deals  
  - Extract Deal Fields

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow on a defined schedule (e.g., daily or weekly).  
    - Configuration: Uses default interval (can be customized).  
    - Inputs: None (start node).  
    - Outputs: Triggers "Get HubSpot Deals".  
    - Edge Cases: Misconfiguration of schedule; no trigger if disabled.  
    - Version: 1.1

  - **Get HubSpot Deals**  
    - Type: HubSpot Node  
    - Role: Retrieves all deals from HubSpot with selected properties: dealname, hs_lastmodifieddate, notes_last_updated, notes_last_contacted.  
    - Configuration: "Get All" operation with returnAll=true, authenticated via HubSpot App Token.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Sends raw deal data to "Extract Deal Fields".  
    - Edge Cases: API rate limits, expired credentials, empty deal list, network errors.  
    - Version: 2

  - **Extract Deal Fields**  
    - Type: Code Node (JavaScript)  
    - Role: Parses and restructures deal data into a simplified format with consistent fields (dealId, dealname, lastmodifieddate, notes_last_updated, notes_last_contacted).  
    - Configuration: Custom JS code mapping input items to output with safe property access.  
    - Inputs: Raw deals from "Get HubSpot Deals".  
    - Outputs: Cleaned deal objects to "Filter Cold Leads (21+ days)".  
    - Edge Cases: Missing properties, unexpected data structure.  
    - Version: 2

---

#### 1.2 Lead Qualification (Cold Deals)

- **Overview:**  
  Filters deals that have not been modified in the last 21 days and updates their stage in HubSpot to "Closed Lost".

- **Nodes Involved:**  
  - Filter Cold Leads (21+ days)  
  - Update Deal to Closed Lost

- **Node Details:**

  - **Filter Cold Leads (21+ days)**  
    - Type: Filter Node  
    - Role: Filters deals where the last modified date is older than 21 days from the current date.  
    - Configuration: Uses an expression comparing lastmodifieddate timestamp to current time minus 21 days (in milliseconds).  
    - Inputs: Cleaned deal data from "Extract Deal Fields".  
    - Outputs: Passes filtered cold deals to "Update Deal to Closed Lost".  
    - Edge Cases: Invalid or missing lastmodifieddate, timezone considerations, expression evaluation errors.  
    - Version: 2

  - **Update Deal to Closed Lost**  
    - Type: HubSpot Node  
    - Role: Updates the stage of filtered deals to "closedlost" in HubSpot.  
    - Configuration: Uses dealId from filtered deals; authenticated with HubSpot App Token.  
    - Inputs: Cold deals from "Filter Cold Leads (21+ days)".  
    - Outputs: Updated deals data to "Fetch Deal Associations".  
    - Edge Cases: API update failures, invalid dealId, permission issues, stale token.  
    - Version: 2.1

---

#### 1.3 Deal to Contact Association

- **Overview:**  
  Retrieves contacts associated with the closed deals, extracts contact IDs, and fetches detailed contact information including email and names.

- **Nodes Involved:**  
  - Fetch Deal Associations  
  - Extract Contact IDs  
  - Get Contact Details  
  - Extract Contact Email

- **Node Details:**

  - **Fetch Deal Associations**  
    - Type: HTTP Request Node  
    - Role: Calls HubSpot API to get contacts associated with each closed deal by dealId.  
    - Configuration: GET request to HubSpot CRM API endpoint with Authorization Bearer token header.  
    - Inputs: Updated deals from "Update Deal to Closed Lost".  
    - Outputs: JSON response with deal-contact associations to "Extract Contact IDs".  
    - Edge Cases: API rate limits, expired or invalid access token, dealId not found, network errors.  
    - Version: 4.2

  - **Extract Contact IDs**  
    - Type: Code Node (JavaScript)  
    - Role: Parses association results to extract contactId, dealId, and dealName for each contact linked to deals.  
    - Configuration: Custom JS iterating over associations, safely extracting contact data.  
    - Inputs: Association data from "Fetch Deal Associations".  
    - Outputs: Array of contact objects to "Get Contact Details".  
    - Edge Cases: Missing associations, empty results, malformed API responses.  
    - Version: 2

  - **Get Contact Details**  
    - Type: HubSpot Node  
    - Role: Fetches detailed contact properties (email, firstname, lastname, hs_full_name_or_email) from HubSpot by contactId.  
    - Configuration: Get operation on contact resource, authenticated via HubSpot App Token.  
    - Inputs: Contact IDs from "Extract Contact IDs".  
    - Outputs: Detailed contact data to "Extract Contact Email".  
    - Edge Cases: Contact not found, rate limits, expired credentials, missing properties.  
    - Version: 2.1

  - **Extract Contact Email**  
    - Type: Code Node (JavaScript)  
    - Role: Extracts and safely returns the contact's email address from the detailed contact data.  
    - Configuration: JS code checks for existence of email property before returning it.  
    - Inputs: Detailed contact data from "Get Contact Details".  
    - Outputs: Simplified email JSON objects to "Send Gmail Feedback Request".  
    - Edge Cases: Missing email field, null or empty emails.  
    - Version: 2

---

#### 1.4 Follow-up & Notifications

- **Overview:**  
  Sends personalized feedback requests via Gmail to the contacts of closed deals and posts notifications in Slack to inform the team.

- **Nodes Involved:**  
  - Send Gmail Feedback Request  
  - Send Slack Notification

- **Node Details:**

  - **Send Gmail Feedback Request**  
    - Type: Gmail Node  
    - Role: Sends a personalized email to each contact, thanking them and requesting feedback.  
    - Configuration: Uses OAuth2 Gmail credentials; message includes dynamic fields like contact firstname for personalization; subject customized accordingly.  
    - Inputs: Contact emails from "Extract Contact Email".  
    - Outputs: Trigger to "Send Slack Notification".  
    - Edge Cases: OAuth token expiry, email sending limits, invalid email addresses, dynamic content errors.  
    - Version: 2.1

  - **Send Slack Notification**  
    - Type: Slack Node  
    - Role: Posts a message to a specified Slack channel summarizing deals moved to Closed Lost.  
    - Configuration: OAuth2 Slack credentials; channel ID specified; message includes dealId dynamically.  
    - Inputs: Trigger from "Send Gmail Feedback Request".  
    - Outputs: Workflow ends here.  
    - Edge Cases: OAuth token expiry, channel ID misconfiguration, API rate limits.  
    - Version: 2.3

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                        | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                 |
|----------------------------|-------------------------|-------------------------------------|----------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger        | Initiates workflow on schedule      | -                          | Get HubSpot Deals            | ## Trigger & Deal Fetching *Schedule Trigger: Starts the workflow on a defined interval.*  |
| Get HubSpot Deals           | HubSpot                 | Fetches all deals with key fields   | Schedule Trigger            | Extract Deal Fields          | ## Trigger & Deal Fetching *Get HubSpot Deals: Pulls all deals from HubSpot...*             |
| Extract Deal Fields         | Code                    | Cleans and structures deal data     | Get HubSpot Deals           | Filter Cold Leads (21+ days) | ## Trigger & Deal Fetching *Extract Deal Fields: Cleans up and restructures deal data.*     |
| Filter Cold Leads (21+ days)| Filter                  | Filters deals older than 21 days    | Extract Deal Fields         | Update Deal to Closed Lost   | ## Lead Qualification (Cold Deals) *Filters deals untouched for 21+ days.*                  |
| Update Deal to Closed Lost  | HubSpot                 | Updates deal stage to Closed Lost   | Filter Cold Leads (21+ days)| Fetch Deal Associations      | ## Lead Qualification (Cold Deals) *Updates stale deals to Closed Lost.*                    |
| Fetch Deal Associations     | HTTP Request            | Gets contacts associated with deals| Update Deal to Closed Lost  | Extract Contact IDs          | ## Deal → Contact Mapping *Fetch Deal Associations: Calls HubSpot API for linked contacts.* |
| Extract Contact IDs         | Code                    | Extracts contactId, dealId, dealName| Fetch Deal Associations     | Get Contact Details          | ## Deal → Contact Mapping *Extract Contact IDs: Parses association response.*               |
| Get Contact Details         | HubSpot                 | Retrieves contact properties        | Extract Contact IDs         | Extract Contact Email        | ## Deal → Contact Mapping *Get Contact Details: Retrieves enriched contact details.*        |
| Extract Contact Email       | Code                    | Extracts email from contact details | Get Contact Details         | Send Gmail Feedback Request  | ## Deal → Contact Mapping *Extract Contact Email: Keeps only the email safely.*             |
| Send Gmail Feedback Request | Gmail                   | Sends personalized feedback emails  | Extract Contact Email       | Send Slack Notification      | ## Follow-up & Notifications *Sends polite follow-up emails.*                               |
| Send Slack Notification     | Slack                   | Posts notification about closed deals| Send Gmail Feedback Request | -                           | ## Follow-up & Notifications *Posts message to Slack channel about closed deals.*          |
| Sticky Note                | Sticky Note             | Workflow documentation              | -                          | -                           | ## Trigger & Deal Fetching (covers Schedule Trigger, Get HubSpot Deals, Extract Deal Fields)|
| Sticky Note1               | Sticky Note             | Workflow documentation              | -                          | -                           | ## Lead Qualification (Cold Deals) (covers Filter Cold Leads, Update Deal to Closed Lost)   |
| Sticky Note2               | Sticky Note             | Workflow documentation              | -                          | -                           | ## Deal → Contact Mapping (covers Fetch Deal Associations, Extract Contact IDs, etc.)       |
| Sticky Note3               | Sticky Note             | Workflow documentation              | -                          | -                           | ## Follow-up & Notifications (covers Send Gmail, Send Slack)                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure the interval for workflow execution (e.g., daily).  
   - Connect output to "Get HubSpot Deals".

2. **Create "Get HubSpot Deals" node:**  
   - Type: HubSpot  
   - Operation: Get All deals  
   - Properties to retrieve: dealname, hs_lastmodifieddate, notes_last_updated, notes_last_contacted  
   - Authentication: Use HubSpot App Token credential  
   - Connect input from Schedule Trigger, output to "Extract Deal Fields".

3. **Create "Extract Deal Fields" node:**  
   - Type: Code (JavaScript)  
   - Code: Map each deal item to a JSON object with dealId, dealname, lastmodifieddate, notes_last_updated, notes_last_contacted safely extracted.  
   - Connect input from "Get HubSpot Deals", output to "Filter Cold Leads (21+ days)".

4. **Create "Filter Cold Leads (21+ days)" node:**  
   - Type: Filter  
   - Condition: lastmodifieddate < (Date.now() - 21 days in milliseconds)  
   - Connect input from "Extract Deal Fields", output to "Update Deal to Closed Lost".

5. **Create "Update Deal to Closed Lost" node:**  
   - Type: HubSpot  
   - Operation: Update deal  
   - Parameters: stage = "closedlost"  
   - Use dealId from input item  
   - Authentication: HubSpot App Token  
   - Connect input from filter, output to "Fetch Deal Associations".

6. **Create "Fetch Deal Associations" node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.hubapi.com/crm/v3/objects/deals/{{ $json.dealId }}?associations=contacts`  
   - Headers: Authorization Bearer token (HubSpot OAuth2 or API Token)  
   - Connect input from "Update Deal to Closed Lost", output to "Extract Contact IDs".

7. **Create "Extract Contact IDs" node:**  
   - Type: Code (JavaScript)  
   - Code: Iterate over deals, extract all associated contacts' IDs, dealId, and dealName into an array.  
   - Connect input from "Fetch Deal Associations", output to "Get Contact Details".

8. **Create "Get Contact Details" node:**  
   - Type: HubSpot  
   - Operation: Get contact by ID  
   - Properties: email, firstname, lastname, hs_full_name_or_email  
   - Authentication: HubSpot App Token  
   - Connect input from "Extract Contact IDs", output to "Extract Contact Email".

9. **Create "Extract Contact Email" node:**  
   - Type: Code (JavaScript)  
   - Code: Safely extract email from contact properties and return JSON with email field.  
   - Connect input from "Get Contact Details", output to "Send Gmail Feedback Request".

10. **Create "Send Gmail Feedback Request" node:**  
    - Type: Gmail  
    - Authentication: Gmail OAuth2 credential  
    - To: email from previous node  
    - Subject: "Thank You — We'd Value Your Feedback"  
    - Message Body: Personalized HTML message including contact's firstname from "Get Contact Details", thanking for engagement, requesting feedback.  
    - Connect input from "Extract Contact Email", output to "Send Slack Notification".

11. **Create "Send Slack Notification" node:**  
    - Type: Slack  
    - Authentication: Slack OAuth2 credential  
    - Channel ID: your target Slack channel (e.g., #general)  
    - Message: "Deals which are moved to Closed Lost: {{ dealId }}" dynamically referencing deal IDs from "Extract Contact IDs" node.  
    - Connect input from "Send Gmail Feedback Request".

12. **Test the workflow:**  
    - Verify all credentials (HubSpot, Gmail, Slack) are valid.  
    - Validate API tokens and OAuth setups.  
    - Run manually or wait for schedule trigger.  
    - Check logs and outputs for errors or missing data.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow automates closing stale HubSpot deals (21+ days inactive) and sends feedback requests via Gmail.     | Primary use case: sales pipeline maintenance and customer feedback gathering.                           |
| Slack notifications keep internal teams informed about deal status changes.                                   | Useful for sales managers and customer success teams.                                                   |
| Gmail message uses HTML formatting and personalized fields (firstname).                                       | Ensure Gmail OAuth2 credentials have appropriate scopes for sending emails on behalf of users.          |
| HubSpot API calls require correct scopes and token validity; app token or OAuth2 can be used.                 | HubSpot API docs: https://developers.hubspot.com/docs/api/crm/deals                                    |
| Slack messages require channel ID and proper OAuth token with chat:write scope.                               | Slack API docs: https://api.slack.com/messaging                                                        |
| Time calculations use JavaScript Date.now() and millisecond conversions for 21 days filter.                   | Consider timezone of HubSpot timestamps when debugging date filters.                                   |
| The workflow assumes all deals have associated contacts; handle cases where no contacts are linked gracefully.| Add error handling or fallback if contact associations are missing or empty.                            |
| For sending personalized emails, ensure the firstname field exists and is not null to avoid empty greetings. | Optionally add default fallback name or skip email if data is insufficient.                             |