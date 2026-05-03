Voiceflow Demo Support Chatbot 

https://n8nworkflows.xyz/workflows/voiceflow-demo-support-chatbot--2796


# Voiceflow Demo Support Chatbot 

### 1. Workflow Overview

This workflow, titled **Voiceflow Demo Support Chatbot**, is designed to extend the capabilities of Voiceflow-powered AI voice chatbots by integrating them with backend systems such as Zendesk, Google Calendar, Airtable, and Google Sheets. It targets businesses and developers who want to automate customer service processes including customer lookup, support ticket creation, meeting scheduling, and transcript reporting.

The workflow is logically divided into the following blocks:

- **1.1 Customer Lookup:** Receives a phone number from the chatbot, queries a Google Sheets customer database, and returns customer details or a "NOT_FOUND" status.
- **1.2 Zendesk Ticket Creation:** Creates or updates a customer in Zendesk and submits a support ticket based on chatbot input.
- **1.3 Meeting Scheduling:** Checks Google Calendar availability and schedules meetings, responding with success or error messages.
- **1.4 Transcript Reporting:** Receives transcript data and customer info, then stores it in Airtable for product team analysis.
- **1.5 Webhook Endpoints:** Four webhook nodes serve as entry points for Voiceflow, Zendesk, Google Calendar, and Airtable integrations.
- **1.6 Error Handling and Responses:** Handles various error cases and sends appropriate webhook responses back to the calling system.

---

### 2. Block-by-Block Analysis

#### 2.1 Customer Lookup

- **Overview:**  
  This block extracts a phone number from the Voiceflow webhook request, queries a Google Sheets customer database for matching records, and conditionally responds with customer data or a "NOT_FOUND" error.

- **Nodes Involved:**  
  - Voiceflow Endpoint  
  - Extract Phone Number  
  - Query Google Sheets for Phone  
  - Check if user found  
  - Respond to Webhook with Customer Data  
  - Set Error Data  
  - Respond to Webhook with Error

- **Node Details:**

  - **Voiceflow Endpoint**  
    - Type: Webhook  
    - Role: Entry point for Voiceflow chatbot requests  
    - Config: POST method, path `d9b20efe-9bb4-4d8b-b9aa-d568f43f78ea`, response mode set to respond node  
    - Inputs: External HTTP request from Voiceflow  
    - Outputs: Passes data to Extract Phone Number  
    - Edge Cases: Missing or malformed webhook payload

  - **Extract Phone Number**  
    - Type: Set  
    - Role: Normalize phone number by removing leading '+'  
    - Config: Sets `query.phone_number` to the incoming phone number with '+' stripped  
    - Inputs: Voiceflow Endpoint output  
    - Outputs: Passes normalized phone number to Google Sheets query  
    - Edge Cases: Phone number missing or invalid format

  - **Query Google Sheets for Phone**  
    - Type: Google Sheets  
    - Role: Searches customer database sheet for matching phone number  
    - Config: Filters on "Phone Number" column using normalized phone number  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Extract Phone Number output  
    - Outputs: Customer data if found, empty if not  
    - Edge Cases: API errors, no matching record

  - **Check if user found**  
    - Type: If  
    - Role: Checks if customer data contains a non-empty "Name" field  
    - Condition: `$json.Name` is not empty  
    - Inputs: Query Google Sheets output  
    - Outputs: True branch to Respond with Customer Data, False branch to Set Error Data  
    - Edge Cases: Missing or malformed data fields

  - **Respond to Webhook with Customer Data**  
    - Type: Respond to Webhook  
    - Role: Returns found customer data to Voiceflow  
    - Inputs: True branch of Check if user found  
    - Outputs: HTTP response to Voiceflow

  - **Set Error Data**  
    - Type: Set  
    - Role: Sets all customer fields to "NOT_FOUND" for error response  
    - Inputs: False branch of Check if user found  
    - Outputs: Passes error data to Respond to Webhook with Error

  - **Respond to Webhook with Error**  
    - Type: Respond to Webhook  
    - Role: Returns error response indicating customer not found  
    - Inputs: Set Error Data output  
    - Outputs: HTTP response to Voiceflow

---

#### 2.2 Zendesk Ticket Creation

- **Overview:**  
  This block receives ticket details from the Zendesk webhook, extracts relevant fields, creates or updates the customer in Zendesk, then creates a support ticket. It responds with success or error status.

- **Nodes Involved:**  
  - Zendesk Endpoint  
  - Extract Zendesk Fields  
  - Create Customer in DB  
  - Create Ticket  
  - Check if submitted successfully  
  - Ticket Created Successfully  
  - Error Creating Ticket

- **Node Details:**

  - **Zendesk Endpoint**  
    - Type: Webhook  
    - Role: Entry point for Zendesk ticket creation requests  
    - Config: POST method, path `9c15c8ac-8f3a-40d3-8ad5-e40468388968`, response mode set to respond node  
    - Inputs: External HTTP request from Zendesk or Voiceflow  
    - Outputs: Passes data to Extract Zendesk Fields

  - **Extract Zendesk Fields**  
    - Type: Set  
    - Role: Extracts email, name, transcript, and summary from request body  
    - Config: Maps `body.email`, `body.name`, `body.transcript`, `body.summary` from incoming JSON  
    - Inputs: Zendesk Endpoint output  
    - Outputs: Passes data to Create Customer in DB

  - **Create Customer in DB**  
    - Type: HTTP Request  
    - Role: Creates or updates user in Zendesk via API  
    - Config: POST to Zendesk `/api/v2/users/create_or_update` with user email and name in JSON body  
    - Credentials: Zendesk API (predefined credential)  
    - Inputs: Extract Zendesk Fields output  
    - Outputs: Passes Zendesk user data to Create Ticket  
    - Edge Cases: API errors, invalid user data

  - **Create Ticket**  
    - Type: HTTP Request  
    - Role: Creates a Zendesk ticket for the user  
    - Config: POST to Zendesk `/api/v2/tickets` with requester ID, subject, and comment body including transcript and summary  
    - Credentials: Zendesk API (predefined credential)  
    - Inputs: Create Customer in DB output and Extract Zendesk Fields for comment content  
    - Outputs: Passes ticket creation response to Check if submitted successfully  
    - Error Handling: Continues on error to allow conditional response

  - **Check if submitted successfully**  
    - Type: If  
    - Role: Checks if ticket creation response contains a non-empty ticket URL  
    - Condition: `$json.ticket.url` is not empty  
    - Inputs: Create Ticket output  
    - Outputs: True branch to Ticket Created Successfully, False branch to Error Creating Ticket

  - **Ticket Created Successfully**  
    - Type: Respond to Webhook  
    - Role: Sends success status JSON response  
    - Inputs: True branch of Check if submitted successfully

  - **Error Creating Ticket**  
    - Type: Respond to Webhook  
    - Role: Sends error status JSON response with HTTP 400 code  
    - Inputs: False branch of Check if submitted successfully

---

#### 2.3 Meeting Scheduling

- **Overview:**  
  This block handles meeting scheduling requests by extracting requested datetime and user info, validating the datetime, checking calendar availability, creating a calendar event if available, and responding with success or error messages.

- **Nodes Involved:**  
  - Gcal Endpoint  
  - Extract Gcal Data  
  - Check for malformed date  
  - Check Calendar Availability  
  - Check if available  
  - Create Calendar Event  
  - Set Calendar Success Message  
  - Respond with Success  
  - Set Calendar Error Data  
  - Respond With Calendar Error data  
  - Set Invalid Data Error  
  - Respond with Generic Error

- **Node Details:**

  - **Gcal Endpoint**  
    - Type: Webhook  
    - Role: Entry point for Google Calendar scheduling requests  
    - Config: POST method, path `c1020b94-603c-4981-ab48-51e208d17223`, response mode set to respond node  
    - Inputs: External HTTP request  
    - Outputs: Passes data to Extract Gcal Data

  - **Extract Gcal Data**  
    - Type: Set  
    - Role: Extracts and converts datetime, name, email, and summary from query parameters  
    - Config: Converts `query.datetime` to ISO datetime string, assigns other fields  
    - Inputs: Gcal Endpoint output  
    - Outputs: Passes data to Check for malformed date  
    - Edge Cases: Missing or invalid datetime format

  - **Check for malformed date**  
    - Type: If  
    - Role: Validates that requested datetime is after current time  
    - Condition: `$json.availability` is after `$now`  
    - Inputs: Extract Gcal Data output  
    - Outputs: True branch to Check Calendar Availability, False branch to Set Invalid Data Error

  - **Check Calendar Availability**  
    - Type: Google Calendar  
    - Role: Queries calendar events between requested start and 30 minutes later  
    - Config: Uses OAuth2 credential, calendar email `angel@n8n.io`, timeMin and timeMax set dynamically  
    - Inputs: Check for malformed date true branch  
    - Outputs: Passes data to Check if available

  - **Check if available**  
    - Type: If  
    - Role: Checks if calendar availability response indicates free slot (boolean `available` true)  
    - Inputs: Check Calendar Availability output  
    - Outputs: True branch to Create Calendar Event, False branch to Set Calendar Error Data

  - **Create Calendar Event**  
    - Type: Google Calendar  
    - Role: Creates a calendar event for 30 minutes starting at requested datetime  
    - Config: Sets start and end times, summary including customer name, attendees email, and description from summary  
    - Inputs: True branch of Check if available  
    - Outputs: Passes data to Set Calendar Success Message

  - **Set Calendar Success Message**  
    - Type: Set  
    - Role: Sets status message "MEETING_BOOKED_SUCCESSFULLY"  
    - Inputs: Create Calendar Event output  
    - Outputs: Passes data to Respond with Success

  - **Respond with Success**  
    - Type: Respond to Webhook  
    - Role: Sends success response to caller  
    - Inputs: Set Calendar Success Message output

  - **Set Calendar Error Data**  
    - Type: Set  
    - Role: Sets status message "CSM_UNAVAILABLE" indicating calendar slot unavailable  
    - Inputs: False branch of Check if available  
    - Outputs: Passes data to Respond With Calendar Error data

  - **Respond With Calendar Error data**  
    - Type: Respond to Webhook  
    - Role: Sends error response indicating calendar unavailability  
    - Inputs: Set Calendar Error Data output

  - **Set Invalid Data Error**  
    - Type: Set  
    - Role: Sets status message "INVALID_DATA_ERROR" for malformed or past datetime  
    - Inputs: False branch of Check for malformed date  
    - Outputs: Passes data to Respond with Generic Error

  - **Respond with Generic Error**  
    - Type: Respond to Webhook  
    - Role: Sends generic error response for invalid data  
    - Inputs: Set Invalid Data Error output

---

#### 2.4 Transcript Reporting

- **Overview:**  
  This block receives transcript data and customer info via webhook, extracts relevant fields, and creates a record in Airtable for product team analysis.

- **Nodes Involved:**  
  - Airtable Endpoint  
  - Extract Airtable Data  
  - Create Airtable Data

- **Node Details:**

  - **Airtable Endpoint**  
    - Type: Webhook  
    - Role: Entry point for transcript data submission  
    - Config: POST method, path `9a52822c-0304-4dad-a86a-ae662161243c`  
    - Inputs: External HTTP request  
    - Outputs: Passes data to Extract Airtable Data

  - **Extract Airtable Data**  
    - Type: Set  
    - Role: Extracts phone, summary, transcript, and customer type from query parameters  
    - Inputs: Airtable Endpoint output  
    - Outputs: Passes data to Create Airtable Data

  - **Create Airtable Data**  
    - Type: Airtable  
    - Role: Creates a new record in Airtable base and table specified for product customer analysis  
    - Config: Maps extracted fields to Airtable columns: Phone, Summary, Transcript, Customer Type  
    - Credentials: Airtable API token  
    - Inputs: Extract Airtable Data output  
    - Outputs: Final node in this block  
    - Edge Cases: API errors, invalid data format

---

#### 2.5 Webhook Endpoints Summary

- **Voiceflow Endpoint:** Receives customer lookup requests from Voiceflow chatbot.
- **Zendesk Endpoint:** Receives ticket creation requests.
- **Gcal Endpoint:** Receives meeting scheduling requests.
- **Airtable Endpoint:** Receives transcript data for reporting.

Each webhook node is configured with a unique path and POST method, serving as the entry point for their respective functional blocks.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                           | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                          |
|-------------------------------|------------------------|-----------------------------------------|----------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| Sticky Note                   | Sticky Note            | Visual documentation for Customer Lookup|                                  |                                       | ![voiceflow](https://uploads.n8n.io/templates/voiceflow.png) Find Customer: queries customer DB     |
| Check if user found           | If                     | Checks if customer found in DB           | Query Google Sheets for Phone    | Respond to Webhook with Customer Data, Set Error Data |                                                                                                    |
| Sticky Note1                  | Sticky Note            | Visual documentation for Zendesk Ticket |                                  |                                       | ![zendesk](https://uploads.n8n.io/templates/zendesk.png) Create Zendesk Ticket                      |
| Sticky Note2                  | Sticky Note            | Visual documentation for Meeting Scheduling |                                  |                                       | ![Gcal](https://uploads.n8n.io/templates/calendar.png) Schedule a meeting                          |
| Sticky Note3                  | Sticky Note            | Visual documentation for Transcript Reporting |                                  |                                       | ![voiceflow](https://uploads.n8n.io/templates/airtable.png) Give Product team transcripts for analysis |
| Check if available            | If                     | Checks calendar availability             | Check Calendar Availability     | Create Calendar Event, Set Calendar Error Data |                                                                                                    |
| Check for malformed date      | If                     | Validates requested datetime             | Extract Gcal Data               | Check Calendar Availability, Set Invalid Data Error |                                                                                                    |
| Create Ticket                | HTTP Request           | Creates Zendesk support ticket           | Create Customer in DB           | Check if submitted successfully        |                                                                                                    |
| Create Customer in DB         | HTTP Request           | Creates or updates Zendesk user          | Extract Zendesk Fields          | Create Ticket                          |                                                                                                    |
| Check if submitted successfully | If                   | Checks if ticket creation succeeded      | Create Ticket                  | Ticket Created Successfully, Error Creating Ticket |                                                                                                    |
| Ticket Created Successfully   | Respond to Webhook     | Sends success response for ticket creation | Check if submitted successfully |                                       |                                                                                                    |
| Error Creating Ticket         | Respond to Webhook     | Sends error response for ticket creation | Check if submitted successfully |                                       |                                                                                                    |
| Airtable Endpoint            | Webhook                | Entry point for transcript reporting     |                                  | Extract Airtable Data                  |                                                                                                    |
| Gcal Endpoint                | Webhook                | Entry point for meeting scheduling       |                                  | Extract Gcal Data                      |                                                                                                    |
| Zendesk Endpoint             | Webhook                | Entry point for Zendesk ticket creation  |                                  | Extract Zendesk Fields                 |                                                                                                    |
| Voiceflow Endpoint           | Webhook                | Entry point for customer lookup           |                                  | Extract Phone Number                   |                                                                                                    |
| Extract Phone Number         | Set                    | Normalizes phone number for lookup        | Voiceflow Endpoint             | Query Google Sheets for Phone          |                                                                                                    |
| Query Google Sheets for Phone | Google Sheets          | Queries customer database by phone number | Extract Phone Number           | Check if user found                    |                                                                                                    |
| Respond to Webhook with Customer Data | Respond to Webhook | Returns customer data to Voiceflow        | Check if user found            |                                       |                                                                                                    |
| Set Error Data               | Set                    | Sets "NOT_FOUND" error data                | Check if user found            | Respond to Webhook with Error          |                                                                                                    |
| Respond to Webhook with Error | Respond to Webhook     | Returns error response for customer lookup | Set Error Data                |                                       |                                                                                                    |
| Extract Zendesk Fields       | Set                    | Extracts ticket fields from request       | Zendesk Endpoint              | Create Customer in DB                  |                                                                                                    |
| Extract Gcal Data            | Set                    | Extracts and formats meeting scheduling data | Gcal Endpoint               | Check for malformed date               |                                                                                                    |
| Extract Airtable Data        | Set                    | Extracts transcript data for Airtable     | Airtable Endpoint             | Create Airtable Data                   |                                                                                                    |
| Create Airtable Data         | Airtable               | Creates record in Airtable for transcripts | Extract Airtable Data         |                                       |                                                                                                    |
| Check Calendar Availability  | Google Calendar        | Checks calendar for availability           | Check for malformed date       | Check if available                     |                                                                                                    |
| Create Calendar Event        | Google Calendar        | Creates calendar event if available        | Check if available             | Set Calendar Success Message           |                                                                                                    |
| Set Calendar Success Message | Set                    | Sets success status for meeting booking    | Create Calendar Event          | Respond with Success                   |                                                                                                    |
| Respond with Success         | Respond to Webhook     | Sends success response for meeting booking | Set Calendar Success Message  |                                       |                                                                                                    |
| Set Calendar Error Data      | Set                    | Sets error status if calendar slot unavailable | Check if available           | Respond With Calendar Error data       |                                                                                                    |
| Respond With Calendar Error data | Respond to Webhook | Sends error response for unavailable slot  | Set Calendar Error Data        |                                       |                                                                                                    |
| Set Invalid Data Error       | Set                    | Sets error status for invalid datetime      | Check for malformed date       | Respond with Generic Error             |                                                                                                    |
| Respond with Generic Error   | Respond to Webhook     | Sends generic error response                 | Set Invalid Data Error         |                                       |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes (4 total):**  
   - **Voiceflow Endpoint:** POST, path `d9b20efe-9bb4-4d8b-b9aa-d568f43f78ea`, response mode: respond node.  
   - **Zendesk Endpoint:** POST, path `9c15c8ac-8f3a-40d3-8ad5-e40468388968`, response mode: respond node.  
   - **Gcal Endpoint:** POST, path `c1020b94-603c-4981-ab48-51e208d17223`, response mode: respond node.  
   - **Airtable Endpoint:** POST, path `9a52822c-0304-4dad-a86a-ae662161243c`.

2. **Customer Lookup Block:**  
   - Add **Set** node "Extract Phone Number": assign `query.phone_number` = incoming phone number with leading '+' removed. Connect from Voiceflow Endpoint.  
   - Add **Google Sheets** node "Query Google Sheets for Phone": configure with your customer database spreadsheet and sheet, filter on "Phone Number" column using `query.phone_number`. Connect from Extract Phone Number. Use Google Sheets OAuth2 credentials.  
   - Add **If** node "Check if user found": condition check if `$json.Name` is not empty. Connect from Query Google Sheets for Phone.  
   - Add **Respond to Webhook** node "Respond to Webhook with Customer Data": connect from true branch of Check if user found.  
   - Add **Set** node "Set Error Data": set all customer fields (`row_number`, `Name`, `Email Address`, `Tier`, `Phone Number`) to "NOT_FOUND". Connect from false branch of Check if user found.  
   - Add **Respond to Webhook** node "Respond to Webhook with Error": connect from Set Error Data.

3. **Zendesk Ticket Creation Block:**  
   - Add **Set** node "Extract Zendesk Fields": map `body.email`, `body.name`, `body.transcript`, `body.summary` from webhook body. Connect from Zendesk Endpoint.  
   - Add **HTTP Request** node "Create Customer in DB": POST to Zendesk `/api/v2/users/create_or_update` with JSON body containing user email and name. Use Zendesk API credentials. Connect from Extract Zendesk Fields.  
   - Add **HTTP Request** node "Create Ticket": POST to Zendesk `/api/v2/tickets` with JSON body including requester_id from created user, subject, and comment body combining summary and transcript (escaped). Use Zendesk API credentials. Connect from Create Customer in DB. Set onError to continue.  
   - Add **If** node "Check if submitted successfully": check if `$json.ticket.url` is not empty. Connect from Create Ticket.  
   - Add **Respond to Webhook** node "Ticket Created Successfully": connect from true branch.  
   - Add **Respond to Webhook** node "Error Creating Ticket": connect from false branch, set response code 400.

4. **Meeting Scheduling Block:**  
   - Add **Set** node "Extract Gcal Data": assign `availability` as ISO datetime from `query.datetime`, and map `query.name`, `query.email`, `query.summary`. Connect from Gcal Endpoint.  
   - Add **If** node "Check for malformed date": check if `availability` is after current time. Connect from Extract Gcal Data.  
   - Add **Google Calendar** node "Check Calendar Availability": query events between `availability` and `availability + 30 minutes` on calendar `angel@n8n.io`. Use Google Calendar OAuth2 credentials. Connect from true branch of Check for malformed date.  
   - Add **If** node "Check if available": check if calendar response indicates availability (boolean `available` true). Connect from Check Calendar Availability.  
   - Add **Google Calendar** node "Create Calendar Event": create event starting at `availability` for 30 minutes, summary includes customer name, attendees include customer email, description from summary. Use Google Calendar OAuth2 credentials. Connect from true branch of Check if available.  
   - Add **Set** node "Set Calendar Success Message": set `status` to "MEETING_BOOKED_SUCCESSFULLY". Connect from Create Calendar Event.  
   - Add **Respond to Webhook** node "Respond with Success": connect from Set Calendar Success Message.  
   - Add **Set** node "Set Calendar Error Data": set `status` to "CSM_UNAVAILABLE". Connect from false branch of Check if available.  
   - Add **Respond to Webhook** node "Respond With Calendar Error data": connect from Set Calendar Error Data.  
   - Add **Set** node "Set Invalid Data Error": set `status` to "INVALID_DATA_ERROR". Connect from false branch of Check for malformed date.  
   - Add **Respond to Webhook** node "Respond with Generic Error": connect from Set Invalid Data Error.

5. **Transcript Reporting Block:**  
   - Add **Set** node "Extract Airtable Data": map `phone`, `summary`, `transcript`, and `type` from webhook query parameters. Connect from Airtable Endpoint.  
   - Add **Airtable** node "Create Airtable Data": create record in Airtable base and table configured for product customer analysis, mapping fields accordingly. Use Airtable API credentials. Connect from Extract Airtable Data.

6. **Credential Setup:**  
   - Configure Google Sheets OAuth2 credentials with access to your customer database spreadsheet.  
   - Configure Zendesk API credentials with permissions for user and ticket creation.  
   - Configure Google Calendar OAuth2 credentials with access to the target calendar.  
   - Configure Airtable API token with write access to the target base and table.

7. **Testing and Deployment:**  
   - Deploy the workflow on your n8n instance.  
   - Configure Voiceflow chatbot to call the Voiceflow Endpoint webhook for customer lookup.  
   - Configure chatbot or backend systems to call Zendesk, Gcal, and Airtable endpoints as appropriate.  
   - Test each integration path thoroughly, including error cases.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow template adheres to n8n submission guidelines ensuring clarity and broad applicability | n8n official guidelines                                                                           |
| Voiceflow branding image used in sticky notes for customer lookup block                         | ![voiceflow](https://uploads.n8n.io/templates/voiceflow.png)                                    |
| Zendesk branding image used in sticky notes for ticket creation block                          | ![zendesk](https://uploads.n8n.io/templates/zendesk.png)                                        |
| Google Calendar branding image used in sticky notes for meeting scheduling block              | ![Gcal](https://uploads.n8n.io/templates/calendar.png)                                          |
| Airtable branding image used in sticky notes for transcript reporting block                    | ![voiceflow](https://uploads.n8n.io/templates/airtable.png)                                     |
| For customizing, adjust database queries, API payloads, and Airtable fields as needed          | See workflow description for customization tips                                                 |
| Ensure all API credentials are properly configured with required scopes and permissions        | Zendesk, Google Calendar, Google Sheets, Airtable APIs                                          |

---

This documentation provides a comprehensive, structured reference for understanding, reproducing, and modifying the Voiceflow Demo Support Chatbot workflow in n8n.