Connect Airtable Contacts to telli for Automated AI Voice Call Scheduling

https://n8nworkflows.xyz/workflows/connect-airtable-contacts-to-telli-for-automated-ai-voice-call-scheduling-3803


# Connect Airtable Contacts to telli for Automated AI Voice Call Scheduling

### 1. Workflow Overview

This workflow automates the process of uploading CRM contacts from an Airtable base into the telli platform and subsequently schedules AI voice-agent calls to those contacts. It targets businesses wanting to streamline outbound calling tasks such as lead qualification, appointment reminders, and customer feedback collection by integrating Airtable CRM with telli’s AI-powered voice agents.

The workflow is logically structured into two primary functional blocks:

- **1.1 Input Reception and Contact Detection:** Uses an Airtable Trigger node to monitor new entries in the CRM contacts table.
- **1.2 API Integration with telli Platform:** Sequentially performs two HTTP POST requests—first to add a contact to telli, then to schedule a call for that contact.

This design ensures that each new contact from Airtable is automatically onboarded into telli and assigned a voice-agent call per configured parameters.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Contact Detection

**Overview:**  
This block continuously monitors the specified Airtable base and table for newly created contact records. Upon detecting a new contact, it triggers the workflow to begin processing.

**Nodes Involved:**  
- Airtable Trigger

**Node Details:**

- **Airtable Trigger**  
  - *Type & Role:* Trigger node that listens for new records in an Airtable base/table.  
  - *Configuration:*  
    - Connected to a specific Airtable base (`appjsUaPrbH6ph7cB`) and table (`tblVXEWTj7dErmNsa`) that store CRM contacts.  
    - Polls every minute to check for new entries based on the "Created Time" field.  
    - Uses Airtable API token authentication (credential named "Airtable account").  
  - *Expressions/Variables:* None internally, but outputs new contact data in JSON format.  
  - *Input/Output:* No input (trigger node), outputs new contact data to the next node.  
  - *Version Requirements:* Compatible with n8n version 1.x and above.  
  - *Potential Failures:*  
    - Authentication errors if Airtable API token is invalid or expired.  
    - Rate limiting or API unavailability from Airtable.  
    - Polling frequency might miss very rapid updates if too slow.  
  - *Sub-workflow:* None.

---

#### 2.2 API Integration with telli Platform

**Overview:**  
This block performs two sequential HTTP POST requests to the telli API. First, it adds the new contact into telli’s system. Next, it schedules an AI voice-agent call for that contact. Both steps rely on data from the Airtable Trigger and require an API key for authorization.

**Nodes Involved:**  
- Add contact request (HTTP Request)  
- Schedule Calls Request (HTTP Request)

**Node Details:**

- **Add contact request**  
  - *Type & Role:* HTTP Request node executing a POST request to add a contact to telli.  
  - *Configuration:*  
    - URL: `https://api.telli.com/v1/add-contact`  
    - Method: POST  
    - Headers:  
      - `Authorization`: Must be replaced with the user's telli API key (`<YOUR-API-KEY>` placeholder).  
      - `Content-Type`: `application/json` (implied by contentType: json)  
    - Body: Configured to send JSON payload matching telli’s expected contact schema. However, the current node’s body parameters are empty (`parameters: [{}]`), indicating that the actual field mapping from Airtable data to telli API fields must be added during setup.  
  - *Expressions/Variables:* Not explicitly set in the current configuration; requires mapping from Airtable Trigger output fields such as external_contact_id, first_name, last_name, phone_number, email, etc.  
  - *Input/Output:*  
    - Input: Receives contact data from Airtable Trigger node.  
    - Output: Sends API response data (which should include the telli contact_id) to the next node.  
  - *Version Requirements:* HTTP Request node version 4.2 or higher recommended for consistent JSON body handling.  
  - *Potential Failures:*  
    - Authorization failure if API key is incorrect.  
    - Payload errors if required fields are missing or malformed.  
    - Network issues or API downtime.  
    - Unexpected API response format causing downstream errors.  
  - *Sub-workflow:* None.

- **Schedule Calls Request**  
  - *Type & Role:* HTTP Request node executing a POST request to schedule AI voice-agent calls in telli.  
  - *Configuration:*  
    - URL: `https://api.telli.com/v1/schedule-call`  
    - Method: POST  
    - Headers:  
      - `Authorization`: Must be replaced with the user's telli API key (`<YOUR-API-KEY>` placeholder).  
      - `Content-Type`: `application/json` (implied)  
    - Body Parameters: Includes at least the `contact_id` field dynamically set using `{{$json.contact_id}}` from the previous node’s output. Other required fields such as `agent_id`, `max_retry_days`, and `call_details` are expected to be included but are missing in this base configuration, so users must add them.  
  - *Expressions/Variables:* Uses expression for `contact_id` from the previous node’s JSON response.  
  - *Input/Output:*  
    - Input: Receives telli contact ID from "Add contact request" node.  
    - Output: API response confirming scheduling.  
  - *Version Requirements:* HTTP Request node 4.2+ recommended.  
  - *Potential Failures:*  
    - Authorization errors if API key is invalid.  
    - Failure if `contact_id` is missing or incorrect.  
    - Missing or malformed scheduling parameters causing API errors.  
    - Network or API downtime.  
  - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                                      | Input Node(s)       | Output Node(s)          | Sticky Note                                                                                          |
|---------------------|-----------------------|-----------------------------------------------------|---------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Airtable Trigger    | airtableTrigger       | Detect new contacts in Airtable base/table          | -                   | Add contact request      | Connect it to the table where you store information about your leads or contacts in general.       |
| Add contact request | httpRequest           | Add new contact to telli via API call                | Airtable Trigger    | Schedule Calls Request   | Here you perform a POST request to telli's API to bring your CRM contacts into the telli system.  |
| Schedule Calls Request | httpRequest         | Schedule AI voice-agent call for the contact         | Add contact request | -                       | Right after the contacts have been added, schedule calls based on smart calling strategy.          |
| Sticky Note         | stickyNote            | Documentation and detailed instructions              | -                   | -                       | # Upload your CRM contacts to telli and schedule AI voice-agent calls (detailed guide and API info)|
| Sticky Note1        | stickyNote            | Guidance on Airtable Trigger node                     | -                   | -                       | ## CRM node Connect it to the table where you store information about your leads or contacts.     |
| Sticky Note2        | stickyNote            | Explanation for Add contact request node              | -                   | -                       | ## Add contacts to telli Perform POST request to telli's API to add CRM contacts.                  |
| Sticky Note3        | stickyNote            | Explanation for Schedule Calls Request node           | -                   | -                       | ## Schedule calls for your new contacts Perform POST request to schedule calls in telli.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create New Workflow in n8n**

2. **Add Airtable Trigger Node**
   - Node Type: Airtable Trigger  
   - Configure credentials with your Airtable API token.  
   - Set Base ID to your Airtable base containing contacts.  
   - Set Table ID to the table storing your contacts.  
   - Set trigger to poll every minute for new records based on the "Created Time" field.  
   - Save node as "Airtable Trigger".

3. **Add HTTP Request Node for Adding Contact to telli**
   - Node Type: HTTP Request  
   - Name: "Add contact request"  
   - HTTP Method: POST  
   - URL: `https://api.telli.com/v1/add-contact`  
   - Authentication: None (use header)  
   - Headers: Add a header: `Authorization` with value set to your telli API key (e.g., `Bearer <YOUR-API-KEY>` or as per telli docs).  
   - Content-Type: JSON  
   - Body Parameters: Map the Airtable contact fields to telli’s expected JSON schema fields:  
     - `external_contact_id`: Map to Airtable record ID or unique contact ID  
     - `salutation`: (optional) e.g., Mr., Ms.  
     - `first_name`: Map from Airtable first name field  
     - `last_name`: Map from Airtable last name field  
     - `phone_number`: Map from Airtable phone field (ensure international format)  
     - `email`: Map from Airtable email field  
     - `contact_details`: (optional) additional JSON object with custom data  
     - `timezone`: (optional) timezone string if available  
   - Connect "Airtable Trigger" node output to this node input.

4. **Add HTTP Request Node for Scheduling Calls in telli**
   - Node Type: HTTP Request  
   - Name: "Schedule Calls Request"  
   - HTTP Method: POST  
   - URL: `https://api.telli.com/v1/schedule-call`  
   - Headers: Same Authorization header with your telli API key.  
   - Content-Type: JSON  
   - Body Parameters:  
     - `contact_id`: Use expression to set from previous node’s response (e.g., `{{$json["contact_id"]}}`)  
     - `agent_id`: Set to your telli agent ID string  
     - `max_retry_days`: Set as integer (e.g., 3 or 7)  
     - `call_details`: JSON object containing:  
       - `message`: String with your call message (e.g., "Hello, this is your friendly reminder!")  
       - `questions`: Array of objects defining questions you want the AI agent to ask (optional)  
     - `override_from_number`: (optional) phone number to override caller ID  
   - Connect "Add contact request" node output to this node input.

5. **Configure Credentials**
   - Airtable node: Use valid Airtable API token credentials.  
   - HTTP Request nodes: No dedicated credential required, but ensure the Authorization header contains your valid telli API key.

6. **Test Execution**
   - Save the workflow.  
   - Trigger a test by adding a new contact in Airtable.  
   - Observe the workflow execution in n8n’s UI.  
   - Confirm the contact is added in telli and a call is scheduled.

7. **Activate Workflow**
   - Once verified, activate the workflow for continuous operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                      | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow template supports bulk contact upload via Loop node or telli batch API endpoints (`/add-contacts-batch` and `/schedule-calls-batch`). See telli API documentation for batch payload format.                                                                                          | Workflow description section under "Uploading Multiple Contacts"                               |
| For detailed API payloads and examples, refer to telli’s API docs or developer portal at: https://api.telli.com/v1/                                                                                                                                                                              | API endpoint details section in workflow description                                           |
| Ensure phone numbers are in international E.164 format to avoid API errors.                                                                                                                                                                                                                       | General best practice for telephony APIs                                                      |
| Use n8n version 0.150 or higher for optimal HTTP Request node stability and JSON body handling.                                                                                                                                                                                                  | Recommended n8n version for HTTP Request node                                                  |
| This workflow demonstrates basic integration; users should enhance error handling using n8n’s error workflows or try/catch nodes for production use.                                                                                                                                             | Best practice advice                                                                          |
| For more information on telli and AI voice agent features, visit https://www.telli.com                                                                                                                                                                                                           | Branding and product info                                                                     |

---

This document provides a comprehensive understanding and stepwise reconstruction guide for the "Connect Airtable Contacts to telli for Automated AI Voice Call Scheduling" n8n workflow, facilitating both human users and automation agents to manage, modify, and troubleshoot this integration effectively.