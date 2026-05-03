Push Dialpad call information to Syncro

https://n8nworkflows.xyz/workflows/push-dialpad-call-information-to-syncro-1490


# Push Dialpad call information to Syncro

### 1. Workflow Overview

This workflow automates the process of pushing call information from Dialpad into Syncro MSP as either new tickets or updates to existing tickets, linked to customers or contacts. It also logs calls and ticket IDs into a Google Sheet for tracking purposes and cross-referencing with another workflow (`dialpad_to_syncro_timer.json`). The workflow currently requires separate instances for each technician and can be configured to handle inbound or outbound calls selectively.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initial Filtering:** Receives call data via a webhook and filters based on call direction.
- **1.2 Environment Setup:** Loads environment variables such as Syncro API base URL and user ID.
- **1.3 Customer/Contact Lookup:** Searches Syncro for contacts and customers matching the call’s external number.
- **1.4 Ticket Retrieval and Decision Logic:** Checks for existing open tickets linked to the found contact.
- **1.5 Ticket Creation or Update:** Creates new tickets or adds comments to existing tickets based on prior search.
- **1.6 Google Sheets Logging:** Logs call-to-ticket mappings into Google Sheets for downstream processing and querying.
- **1.7 Fallback and No-Operation Nodes:** Handle cases where no contacts/customers or multiple tickets are found.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initial Filtering

- **Overview:**  
Receives incoming call data from Dialpad via an HTTP webhook and filters calls based on their direction (inbound only, by default).

- **Nodes Involved:**  
  - Webhook  
  - IF (filter on call direction)  
  - NoOp2 (discarded calls)

- **Node Details:**  
  - **Webhook**  
    - Type: HTTP Webhook  
    - Role: Entry point accepting POST requests at path `/moezdialpad` for call data.  
    - Config: Responds with all entries; HTTP method POST.  
    - Input: External HTTP POST call from Dialpad.  
    - Output: Passes data to IF node.  
    - Edge cases: Missing or malformed payload could cause errors; ensure Dialpad sends correct JSON.  
  - **IF**  
    - Type: Conditional  
    - Role: Filters to proceed only if the call direction is `"inbound"`.  
    - Config: Expression checks `$json["body"]["direction"] == "inbound"`.  
    - Input: Webhook output.  
    - Output: Passes inbound calls to EnvVariables node; others to NoOp2.  
    - Edge cases: If direction is undefined or different, call is ignored.  
  - **NoOp2**  
    - Type: No Operation  
    - Role: Ends processing for non-inbound calls gracefully with no action.  
    - Input: IF node output for non-inbound calls.

---

#### 1.2 Environment Setup

- **Overview:**  
Sets up necessary environment variables such as Syncro API base URL and Syncro user ID used for authentication and API calls.

- **Nodes Involved:**  
  - EnvVariables

- **Node Details:**  
  - **EnvVariables**  
    - Type: Set  
    - Role: Stores static configuration values accessible by other nodes.  
    - Config: Sets `syncro_baseurl` (e.g., `https://subdomain.syncromsp.com`) and `user_id` (e.g., `1234`).  
    - Input: IF node output (only inbound calls).  
    - Output: Passes to GetCustomer node.  
    - Edge cases: Incorrect base URL or user ID will cause authentication or request failures downstream.

---

#### 1.3 Customer/Contact Lookup

- **Overview:**  
Searches Syncro API for contacts matching the external phone number from the call, and separately for customers if no contacts are found.

- **Nodes Involved:**  
  - GetCustomer  
  - Contacts (function)  
  - IFContacts (conditional)  
  - GetTicket  
  - Customers (function)  
  - IFCustomers (conditional)  
  - CreateTicketForCustomer  
  - NoOp, NoOp1 (fallback no-op nodes)

- **Node Details:**  
  - **GetCustomer**  
    - Type: HTTP Request  
    - Role: Queries Syncro’s search API with the external number (stripped of "+" and leading 0 or 1) to find matching contacts.  
    - Config: URL dynamically built from `syncro_baseurl` and the sanitized phone number from webhook payload. Uses header authentication with Syncro credentials.  
    - Input: EnvVariables output.  
    - Output: JSON results passed to Contacts and Customers functions.  
    - Edge cases: Network errors, invalid auth, or no results returned.  
  - **Contacts**  
    - Type: Function  
    - Role: Filters API results for the "contacts" index, extracts relevant fields (id, firstname, lastname, customer_id, etc.) into a structured array `contacts`.  
    - Input: GetCustomer output.  
    - Output: Passes contacts array JSON to IFContacts.  
    - Edge cases: Empty or malformed results cause empty contacts array.  
  - **IFContacts**  
    - Type: Conditional  
    - Role: Checks if exactly one contact is found.  
    - Input: Contacts output.  
    - Output: If one contact, proceeds to GetTicket; else NoOp (skips ticket logic for contacts).  
  - **GetTicket**  
    - Type: HTTP Request  
    - Role: Retrieves all open tickets for the identified contact by contact_id with status "Not Closed".  
    - Config: URL uses `syncro_baseurl` and contact_id from Contacts node; uses Syncro header auth.  
    - Input: IFContacts true branch.  
    - Output: Passes ticket array to IFMoreThanOne.  
    - Edge cases: No tickets found returns empty array; API errors possible.  
  - **Customers**  
    - Type: Function  
    - Role: Filters API results for "customers" index, extracts similar fields as Contacts function into a `customers` array.  
    - Input: GetCustomer output.  
    - Output: Passes customers array JSON to IFCustomers.  
  - **IFCustomers**  
    - Type: Conditional  
    - Role: Checks if exactly one customer is found.  
    - Input: Customers output.  
    - Output: If yes, proceeds to CreateTicketForCustomer node; else NoOp1.  
  - **CreateTicketForCustomer**  
    - Type: HTTP Request  
    - Role: Creates a new ticket in Syncro for the found customer.  
    - Config: POST to `/api/v1/tickets` with JSON body including customer_id, subject formatted with business name and phone, status "In Progress," and user_id from EnvVariables. Uses header auth.  
    - Input: IFCustomers true branch.  
    - Output: Passes newly created ticket info to ForGoogle2.  
    - Edge cases: API errors or invalid customer_id cause failure.  
  - **NoOp** and **NoOp1**  
    - Type: No Operation  
    - Role: End nodes for branches where no multiple tickets or customers are found to gracefully halt processing.

---

#### 1.4 Ticket Retrieval and Decision Logic

- **Overview:**  
Determines whether to update an existing ticket or create a new one, based on the number of open tickets found for the contact.

- **Nodes Involved:**  
  - IFMoreThanOne  
  - UpdateTicket  
  - CreateTicket

- **Node Details:**  
  - **IFMoreThanOne**  
    - Type: Conditional  
    - Role: Checks if exactly one ticket is found (`tickets.length == 1`), and tickets array is truthy.  
    - Input: GetTicket output.  
    - Output: If true, proceeds to UpdateTicket; else CreateTicket.  
  - **UpdateTicket**  
    - Type: HTTP Request  
    - Role: Adds a comment to the existing ticket to log the call.  
    - Config: POST to `/api/v1/tickets/{ticket_id}/comment` with JSON body including subject, body text with caller name, hidden flag true, and user_id. Subject and body are dynamically composed from GetCustomer and Webhook data. Uses header auth.  
    - Input: IFMoreThanOne true branch.  
    - Output: Passes to ForGoogle for logging.  
    - Edge cases: Failure if ticket ID invalid or API issues; expression errors if customer data missing.  
  - **CreateTicket**  
    - Type: HTTP Request  
    - Role: Creates a new ticket for the contact similarly to CreateTicketForCustomer but linked to contact_id.  
    - Config: POST to `/api/v1/tickets` with JSON body including customer_id, subject composed from contact’s first and last name and phone, status "In Progress," contact_id, and user_id.  
    - Input: IFMoreThanOne false branch.  
    - Output: Passes new ticket info to ForGoogle1 for logging.  
    - Edge cases: API errors or missing contact_id cause failure.

---

#### 1.5 Google Sheets Logging

- **Overview:**  
Logs the call ID and ticket ID pairs into Google Sheets for tracking and querying by the `dialpad_to_syncro_timer.json` workflow.

- **Nodes Involved:**  
  - ForGoogle, Google Sheets  
  - ForGoogle1, Google Sheets1  
  - ForGoogle2, Google Sheets2

- **Node Details:**  
  - **ForGoogle, ForGoogle1, ForGoogle2**  
    - Type: Set  
    - Role: Formats the data to be appended to Google Sheets, setting "Call" as call_id and "Ticket" as ticket_id from previous nodes.  
    - Config: Extracts call_id from Webhook and ticket_id from respective ticket creation or update nodes.  
    - Input: UpdateTicket (ForGoogle), CreateTicket (ForGoogle1), CreateTicketForCustomer (ForGoogle2).  
    - Output: Passes formatted data to corresponding Google Sheets nodes.  
  - **Google Sheets, Google Sheets1, Google Sheets2**  
    - Type: Google Sheets Append  
    - Role: Appends rows to a Google Sheet with columns A:B containing Call ID and Ticket ID.  
    - Config: Uses `USER_ENTERED` input mode; sheet ID set to placeholder "xxx" (to be replaced).  
    - Credentials: Google API OAuth2 credentials required.  
    - Input: Corresponding ForGoogle Set nodes.  
    - Edge cases: API quota limits, authentication errors, or incorrect sheet ID cause failure.

---

#### 1.6 Fallback and No-Operation Nodes

- **Overview:**  
Gracefully handle workflow branches where no qualifying contacts, customers, or tickets are found, preventing errors and halting unnecessary downstream processing.

- **Nodes Involved:**  
  - NoOp, NoOp1, NoOp2

- **Node Details:**  
  - Each NoOp is a terminal node that does nothing and ends that branch of workflow execution cleanly.  
  - Used to handle cases like no contacts found, no customers found, or non-inbound calls filtered out.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                        | Input Node(s)          | Output Node(s)                   | Sticky Note                                                                                                    |
|---------------------|--------------------|-------------------------------------|-----------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook             | HTTP Webhook       | Receive Dialpad call data            | -                     | IF                              |                                                                                                               |
| IF                  | IF                 | Filter inbound calls only            | Webhook                | EnvVariables, NoOp2              |                                                                                                               |
| NoOp2               | NoOp               | End processing for non-inbound calls| IF                     | -                               |                                                                                                               |
| EnvVariables        | Set                | Load Syncro base URL and user ID     | IF                     | GetCustomer                     |                                                                                                               |
| GetCustomer         | HTTP Request       | Search Syncro for contacts/customers | EnvVariables            | Contacts, Customers              |                                                                                                               |
| Contacts            | Function           | Extract contacts from API results    | GetCustomer             | IFContacts                     |                                                                                                               |
| IFContacts          | IF                 | Check if exactly one contact found   | Contacts                | GetTicket, NoOp                 |                                                                                                               |
| GetTicket           | HTTP Request       | Retrieve open tickets for contact    | IFContacts              | IFMoreThanOne                  |                                                                                                               |
| IFMoreThanOne       | IF                 | Check if exactly one open ticket     | GetTicket               | UpdateTicket, CreateTicket      |                                                                                                               |
| UpdateTicket        | HTTP Request       | Add comment to existing ticket       | IFMoreThanOne           | ForGoogle                      |                                                                                                               |
| ForGoogle           | Set                | Prepare call/ticket data for logging | UpdateTicket            | Google Sheets                  |                                                                                                               |
| Google Sheets       | Google Sheets      | Append call/ticket log (update)      | ForGoogle               | -                             |                                                                                                               |
| CreateTicket        | HTTP Request       | Create new ticket for contact        | IFMoreThanOne           | ForGoogle1                     |                                                                                                               |
| ForGoogle1          | Set                | Prepare call/ticket data for logging | CreateTicket            | Google Sheets1                 |                                                                                                               |
| Google Sheets1      | Google Sheets      | Append call/ticket log (new ticket)  | ForGoogle1              | -                             |                                                                                                               |
| Customers           | Function           | Extract customers from API results   | GetCustomer             | IFCustomers                    |                                                                                                               |
| IFCustomers         | IF                 | Check if exactly one customer found  | Customers               | CreateTicketForCustomer, NoOp1 |                                                                                                               |
| CreateTicketForCustomer | HTTP Request    | Create new ticket for customer       | IFCustomers             | ForGoogle2                    |                                                                                                               |
| ForGoogle2          | Set                | Prepare call/ticket data for logging | CreateTicketForCustomer  | Google Sheets2                 |                                                                                                               |
| Google Sheets2      | Google Sheets      | Append call/ticket log (customer)    | ForGoogle2              | -                             |                                                                                                               |
| NoOp                | NoOp               | End branch for no matching tickets   | IFContacts false        | -                             |                                                                                                               |
| NoOp1               | NoOp               | End branch for no matching customers | IFCustomers false       | -                             |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: HTTP Webhook  
   - Path: `moezdialpad`  
   - HTTP Method: POST  
   - Response Data: All entries  
   - Response Mode: Last node  
   - Purpose: Entry point for Dialpad call data.

2. **Add an IF Node (Call Direction Filter)**  
   - Condition: `$json["body"]["direction"] == "inbound"`  
   - True output: Proceed; False output: NoOp2 node.

3. **Add a NoOp Node (NoOp2)**  
   - Purpose: Terminate non-inbound calls cleanly.

4. **Add a Set Node (EnvVariables)**  
   - Set string variables:  
     - `syncro_baseurl` = `https://subdomain.syncromsp.com` (replace with actual URL)  
     - `user_id` = `1234` (replace with actual user ID)  
   - Input: IF true branch.

5. **Add an HTTP Request Node (GetCustomer)**  
   - Method: GET  
   - URL Expression: `{{$node["EnvVariables"].json["syncro_baseurl"]}}/api/v1/search?query={{$json["body"]["external_number"].replace(/\+/g, '').replace(/^[01]/, '')}}`  
   - Authentication: Header Auth with Syncro credentials configured.  
   - Input: EnvVariables output.

6. **Add a Function Node (Contacts)**  
   - Code: Filters results by `_index === 'contacts'` and maps fields: id, firstname, lastname, customer_id, email, business_name, phones.  
   - Input: GetCustomer output.

7. **Add an IF Node (IFContacts)**  
   - Condition: number of contacts equals 1.  
   - True branch: Go to GetTicket node.  
   - False branch: Go to NoOp node.

8. **Add an HTTP Request Node (GetTicket)**  
   - Method: GET  
   - URL Expression: `{{$node["EnvVariables"].json["syncro_baseurl"]}}/api/v1/tickets?contact_id={{$json["contacts"][0]["id"]}}&status=Not%20Closed`  
   - Authentication: Header Auth with Syncro credentials.  
   - Input: IFContacts true branch.

9. **Add an IF Node (IFMoreThanOne)**  
   - Condition:  
     - Number: `{{$node["GetTicket"].json["tickets"].length}} == 1`  
     - Boolean: tickets array exists (truthy)  
   - True branch: UpdateTicket  
   - False branch: CreateTicket

10. **Add an HTTP Request Node (UpdateTicket)**  
    - Method: POST  
    - URL Expression: `{{$node["EnvVariables"].json["syncro_baseurl"]}}/api/v1/tickets/{{$json["tickets"][0]["id"]}}/comment`  
    - Authentication: Header Auth with Syncro credentials.  
    - Body (JSON):  
      - `subject`: `Phone call from {{ $node["GetCustomer"].json["results"][0]["table"]["_source"]["table"]["firstname"] }} {{ $node["GetCustomer"].json["results"][0]["table"]["_source"]["table"]["lastname"] }} ({{ $node["Webhook"].json["body"]["contact"]["phone"] }})`  
      - `body`: `{{ $node["GetCustomer"].json["results"][0]["table"]["_source"]["table"]["firstname"] }} {{ $node["GetCustomer"].json["results"][0]["table"]["_source"]["table"]["lastname"] }} called.`  
      - `hidden`: `true`  
      - `user_id`: From EnvVariables `user_id`  
    - Input: IFMoreThanOne true branch.

11. **Add a Set Node (ForGoogle)**  
    - Set strings:  
      - `Call`: `{{$node["Webhook"].json["body"]["call_id"]}}`  
      - `Ticket`: `{{$node["GetTicket"].json["tickets"][0]["id"]}}`  
    - Input: UpdateTicket output.

12. **Add a Google Sheets Node (Google Sheets)**  
    - Operation: Append  
    - Sheet ID: Replace `"xxx"` with actual Google Sheet ID  
    - Range: `A:B`  
    - Value Input Mode: `USER_ENTERED`  
    - Credentials: Google API OAuth2  
    - Input: ForGoogle output.

13. **Add an HTTP Request Node (CreateTicket)**  
    - Method: POST  
    - URL Expression: `{{$node["EnvVariables"].json["syncro_baseurl"]}}/api/v1/tickets`  
    - Authentication: Header Auth with Syncro credentials.  
    - Body (JSON):  
      - `customer_id`: `{{$node["Contacts"].json["contacts"][0]["customer_id"]}}`  
      - `subject`: `Phone call from {{$node["Function"].json["contacts"][0]["firstname"]}} {{$node["Function"].json["contacts"][0]["lastname"]}} ({{$node["Webhook"].json["body"]["contact"]["phone"]}})`  
      - `status`: `"In Progress"`  
      - `contact_id`: `{{$node["Contacts"].json["contacts"][0]["id"]}}`  
      - `user_id`: From EnvVariables `user_id`  
    - Input: IFMoreThanOne false branch.

14. **Add a Set Node (ForGoogle1)**  
    - Set strings:  
      - `Call`: `{{$node["Webhook"].json["body"]["call_id"]}}`  
      - `Ticket`: `{{$node["CreateTicket"].json["ticket"]["id"]}}`  
    - Input: CreateTicket output.

15. **Add a Google Sheets Node (Google Sheets1)**  
    - Same config as Google Sheets node, with same sheet ID and range.  
    - Input: ForGoogle1 output.

16. **Add a Function Node (Customers)**  
    - Code: Similar to Contacts node but filters for `_index === 'customers'`.  
    - Input: GetCustomer output.

17. **Add an IF Node (IFCustomers)**  
    - Condition: number of customers equals 1.  
    - True branch: CreateTicketForCustomer  
    - False branch: NoOp1 node.

18. **Add an HTTP Request Node (CreateTicketForCustomer)**  
    - Method: POST  
    - URL Expression: `{{$node["EnvVariables"].json["syncro_baseurl"]}}/api/v1/tickets`  
    - Authentication: Header Auth with Syncro credentials.  
    - Body (JSON):  
      - `customer_id`: `{{$node["Customers"].json["customers"][0]["id"]}}`  
      - `subject`: `Phone call from {{$node["Customers"].json["customers"][0]["business_name"]}} ({{$node["Webhook"].json["body"]["contact"]["phone"]}})`  
      - `status`: `"In Progress"`  
      - `user_id`: From EnvVariables `user_id`  
    - Input: IFCustomers true branch.

19. **Add a Set Node (ForGoogle2)**  
    - Set strings:  
      - `Call`: `{{$node["Webhook"].json["body"]["call_id"]}}`  
      - `Ticket`: `{{$node["CreateTicketForCustomer"].json["ticket"]["id"]}}`  
    - Input: CreateTicketForCustomer output.

20. **Add a Google Sheets Node (Google Sheets2)**  
    - Same config as previous Google Sheets nodes.  
    - Input: ForGoogle2 output.

21. **Add NoOp Nodes for fallback:**  
    - NoOp: For IFContacts false branch.  
    - NoOp1: For IFCustomers false branch.

22. **Connect nodes according to the described flow:**  
    - Webhook → IF → EnvVariables / NoOp2  
    - EnvVariables → GetCustomer → Contacts and Customers  
    - Contacts → IFContacts → GetTicket / NoOp  
    - GetTicket → IFMoreThanOne → UpdateTicket / CreateTicket  
    - UpdateTicket → ForGoogle → Google Sheets  
    - CreateTicket → ForGoogle1 → Google Sheets1  
    - Customers → IFCustomers → CreateTicketForCustomer / NoOp1  
    - CreateTicketForCustomer → ForGoogle2 → Google Sheets2

23. **Set up credentials:**  
    - Syncro Header Auth: For all HTTP Request nodes accessing Syncro API.  
    - Google API OAuth2: For all Google Sheets nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow is part of an MSP collection; the original source and additional resources can be found here: https://github.com/bionemesis/n8nsyncro          | Project repository and reference                              |
| You need to create one workflow instance per technician currently; multi-technician support is not automated.                                                  | Workflow deployment guidance                                  |
| The workflow logs calls and ticket IDs to a Google Sheet for cross-referencing with another workflow (`dialpad_to_syncro_timer.json`) which handles call matching. | Integration note                                              |
| To restrict to inbound or outbound calls exclusively, modify the IF node filtering on the call direction accordingly.                                           | Customization tip                                            |
| Google Sheet ID placeholders ("xxx") must be replaced with actual Google Sheet IDs and OAuth credentials configured properly.                                   | Setup requirement                                            |
| Syncro API user_id and base URL must be correctly set in EnvVariables node to ensure API calls work.                                                            | Configuration requirement                                     |

---

This documentation enables understanding, reproduction, and maintenance of the Dialpad to Syncro call logging workflow, with clear separation of concerns and error anticipation.