Send Zendesk tickets to Pipedrive contacts and assign tasks

https://n8nworkflows.xyz/workflows/send-zendesk-tickets-to-pipedrive-contacts-and-assign-tasks-1806


# Send Zendesk tickets to Pipedrive contacts and assign tasks

### 1. Workflow Overview

This workflow automates the synchronization of Zendesk tickets with Pipedrive contacts by assigning ticket tasks to the respective Pipedrive contact owners. It runs every 5 minutes, fetching newly created Zendesk tickets, identifying the requester’s email, searching for the requester in Pipedrive, retrieving the contact owner, and updating the Zendesk ticket assignee accordingly. If no matching contact is found, a note is added to the ticket.

**Logical Blocks:**

- **1.1 Trigger and Timestamp Management:** Manages periodic triggering and tracks last execution time to fetch only new tickets.
- **1.2 Ticket and Requester Data Retrieval:** Fetches tickets created after the last execution and retrieves requester details from Zendesk.
- **1.3 Pipedrive Contact Search and Owner Retrieval:** Searches Pipedrive for the requester, gets the contact owner information, and merges agent data.
- **1.4 Ticket Assignment Update:** Updates Zendesk tickets by assigning them to Pipedrive contact owners or adding a note if not found.
- **1.5 Execution Timestamp Update:** Updates the stored last execution timestamp after processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Timestamp Management

**Overview:**  
This block triggers the workflow every 5 minutes and manages the last execution timestamp stored in static data to ensure incremental processing of tickets.

**Nodes Involved:**  
- Every 5 minutes (Cron)  
- Get last execution timestamp (Function Item)  
- Set new last execution timestamp (Function Item)

**Node Details:**

- **Every 5 minutes**  
  - Type: Cron  
  - Role: Triggers workflow every 5 minutes  
  - Configuration: Runs every X minutes with X=5  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "Get last execution timestamp"  
  - Edge Cases: Workflow not triggered if cron fails; ensure workflow is active

- **Get last execution timestamp**  
  - Type: Function Item  
  - Role: Reads or initializes last execution timestamp from static data; adds current timestamp and lastExecution to item JSON  
  - Configuration: Uses n8n global static data; if no lastExecution exists, initializes with current time  
  - Key Expressions: Uses `getWorkflowStaticData('global')`  
  - Inputs: Trigger from Cron  
  - Outputs: Provides item with `lastExecution` and `executionTimeStamp` fields  
  - Edge Cases: On first run, sets lastExecution to current time, so no tickets processed before that  
  - Version: Compatible with n8n v1+ (using static data and Function Item)

- **Set new last execution timestamp**  
  - Type: Function Item  
  - Role: Updates static data with the most recent execution timestamp after processing  
  - Configuration: Sets `staticData.lastExecution` to the `executionTimeStamp` from "Get last execution timestamp" node’s first item  
  - Inputs: From final processing nodes (e.g., after assignment update or note addition)  
  - Outputs: None (end of flow)  
  - Execute Once: true (executes only once regardless of multiple inputs)  
  - Edge Cases: Ensures timestamps correctly update to prevent missing tickets or duplicate processing

---

#### 1.2 Ticket and Requester Data Retrieval

**Overview:**  
Fetches all Zendesk tickets created after the last execution timestamp and obtains requester information for each ticket.

**Nodes Involved:**  
- Get tickets created after last execution (Zendesk)  
- Get requester information (Zendesk)  
- Keep only needed requester information (Set)  
- Add requester information to ticket data (Merge)

**Node Details:**

- **Get tickets created after last execution**  
  - Type: Zendesk  
  - Role: Retrieve all tickets created after `lastExecution` timestamp  
  - Configuration: Operation `getAll` with query filter `created>{{ $json["lastExecution"] }}`; sorts tickets by `updated_at` descending  
  - Credentials: Zendesk API (OAuth2 or API token)  
  - Inputs: From "Get last execution timestamp"  
  - Outputs: List of tickets matching query  
  - Edge Cases: API rate limits, no tickets found (empty output), expression failure if lastExecution missing

- **Get requester information**  
  - Type: Zendesk  
  - Role: For each ticket, fetch user details of the requester by requester_id  
  - Configuration: Operation `get` on resource `user`, id from `{{ $json["requester_id"] }}`  
  - Credentials: Zendesk API  
  - Inputs: From "Get tickets created after last execution" (parallel processing)  
  - Outputs: User objects with requester details  
  - Edge Cases: Requester may not exist or deleted, API errors, missing requester_id

- **Keep only needed requester information**  
  - Type: Set  
  - Role: Retain only requester ID and email fields for downstream processing  
  - Configuration: Keeps only fields: `requester_id` (from `$json["id"]`), `requester_email` (from `$json["email"]`)  
  - Inputs: From "Get requester information"  
  - Outputs: Streamlined requester data  
  - Edge Cases: Missing email or id fields may cause downstream lookup issues

- **Add requester information to ticket data**  
  - Type: Merge (Merge by Key)  
  - Role: Merge ticket data with requester information on `requester_id`  
  - Configuration: Merge by key on property `requester_id` in both inputs  
  - Inputs: From "Get tickets created after last execution" and "Keep only needed requester information"  
  - Outputs: Combined item with ticket and requester email/id  
  - Edge Cases: If merge keys don't match, incomplete data; check for null merges

---

#### 1.3 Pipedrive Contact Search and Owner Retrieval

**Overview:**  
Searches Pipedrive for the requester’s email, retrieves the contact owner’s details, merges with Zendesk agents/admins data to identify valid owners.

**Nodes Involved:**  
- Search requester in pipedrive (Pipedrive)  
- Get owner information of Pipedrive contact (HTTP Request)  
- Keep only requester owner email (Set)  
- Get agents and admins (Zendesk)  
- Keep only email and Id (Set)  
- Add Pipedrive agent data to pipedrive contact information (Merge)  
- Add contact owner to ticket data (Merge)

**Node Details:**

- **Search requester in pipedrive**  
  - Type: Pipedrive  
  - Role: Searches person records in Pipedrive matching requester email  
  - Configuration: Operation `search` on resource `person` with term `{{$json["requester_email"]}}`; fields include email for accuracy  
  - Credentials: Pipedrive API  
  - Inputs: From "Add requester information to ticket data"  
  - Outputs: Pipedrive person search results, including primary email and owner ID  
  - Edge Cases: No match found, multiple matches, API rate limits

- **Get owner information of Pipedrive contact**  
  - Type: HTTP Request  
  - Role: Retrieves full user info for the owner of the Pipedrive contact (owner ID)  
  - Configuration: GET request to `https://n8n.pipedrive.com/api/v1/users/{{$json["owner"]["id"]}}`  
  - Authentication: Uses Pipedrive API credentials predefined for this node  
  - Inputs: From "Search requester in pipedrive"  
  - Outputs: Owner user data, including email  
  - Edge Cases: Owner ID missing, HTTP errors, auth failures

- **Keep only requester owner email**  
  - Type: Set  
  - Role: Retain only the Pipedrive requester primary email and owner’s email fields for merging  
  - Configuration: Keeps `requester_pipedrive_email` from search result's `primary_email`, `requester_pipedrive_owner_email` from HTTP request response's `data.email`  
  - Inputs: From "Get owner information of Pipedrive contact"  
  - Outputs: Streamlined owner email data  
  - Edge Cases: Missing emails, mismatches

- **Get agents and admins**  
  - Type: Zendesk  
  - Role: Retrieves all Zendesk users with roles "agent" or "admin" for validation  
  - Configuration: Operation `getAll` on resource `user` filtered by roles `agent` and `admin`, returns all results  
  - Credentials: Zendesk API  
  - Inputs: None (runs in parallel after trigger)  
  - Outputs: List of Zendesk agents/admins with email and id  
  - Edge Cases: Large user sets, API limits

- **Keep only email and Id**  
  - Type: Set  
  - Role: Retains only user email and id fields from Zendesk agents/admins for matching  
  - Configuration: Keeps `agent_email` and `agent_id` fields  
  - Inputs: From "Get agents and admins"  
  - Outputs: Simplified agent list  
  - Edge Cases: Missing fields, empty list

- **Add Pipedrive agent data to pipedrive contact information**  
  - Type: Merge (Merge by Key)  
  - Role: Merges Pipedrive contact owner data with Zendesk agent data by matching emails  
  - Configuration: Merge by key on `requester_pipedrive_owner_email` (left) and `agent_email` (right)  
  - Inputs: From "Keep only requester owner email" and "Keep only email and Id"  
  - Outputs: Adds `agent_id` to the contact owner info if matched  
  - Edge Cases: No match found, causing no agent_id; important for conditional assignment

- **Add contact owner to ticket data**  
  - Type: Merge (Merge by Key)  
  - Role: Merges contact owner info with ticket/requester data using `requester_email` and `requester_pipedrive_email` keys  
  - Configuration: Merge by key on `requester_email` and `requester_pipedrive_email`  
  - Inputs: From "Add requester information to ticket data" and "Add Pipedrive agent data to pipedrive contact information"  
  - Outputs: Final combined data including ticket, requester, and contact owner info  
  - Edge Cases: Missing keys, incomplete merges

---

#### 1.4 Ticket Assignment Update

**Overview:**  
Checks if a matching Pipedrive contact owner was found and updates the Zendesk ticket assignee accordingly or adds a note if not found.

**Nodes Involved:**  
- Contact exists in Pipedrive (If)  
- Change assignee to Pipedrive contact owner (Zendesk)  
- Add a note requester not found (Zendesk)

**Node Details:**

- **Contact exists in Pipedrive**  
  - Type: If  
  - Role: Checks if `agent_id` field is non-empty, indicating a matching Pipedrive contact owner was found  
  - Configuration: Condition checks if `{{$json["agent_id"]}}` is not empty  
  - Inputs: From "Add contact owner to ticket data"  
  - Outputs: Two branches — true (contact exists) and false (not found)

- **Change assignee to Pipedrive contact owner**  
  - Type: Zendesk  
  - Role: Updates the ticket to assign it to the Pipedrive contact owner by email  
  - Configuration: Operation `update` on ticket with id `{{$json["id"]}}`, sets `assigneeEmail` to `{{$json["requester_pipedrive_owner_email"]}}`  
  - Credentials: Zendesk API  
  - Inputs: From "Contact exists in Pipedrive" (true branch)  
  - Outputs: Updated ticket data  
  - Edge Cases: Assignee email invalid, API error, permission issues

- **Add a note requester not found**  
  - Type: Zendesk  
  - Role: Adds an internal note to the ticket indicating the requester was not found in Pipedrive  
  - Configuration: Operation `update` on ticket with id `{{$json["id"]}}`, sets `internalNote` to "Requester not found in Pipedrive"  
  - Credentials: Zendesk API  
  - Inputs: From "Contact exists in Pipedrive" (false branch)  
  - Outputs: Updated ticket data with note  
  - Edge Cases: API errors, ticket id missing

---

#### 1.5 Execution Timestamp Update

**Overview:**  
Updates the workflow’s static data with the timestamp of the last execution to ensure incremental ticket processing.

**Nodes Involved:**  
- Set new last execution timestamp (Function Item)

**Node Details:**  
(See node details in 1.1)

---

### 3. Summary Table

| Node Name                                    | Node Type         | Functional Role                                  | Input Node(s)                          | Output Node(s)                                    | Sticky Note                                   |
|----------------------------------------------|-------------------|-------------------------------------------------|--------------------------------------|--------------------------------------------------|-----------------------------------------------|
| Every 5 minutes                              | Cron              | Triggers workflow every 5 minutes                | None                                 | Get last execution timestamp                       |                                               |
| Get last execution timestamp                 | Function Item     | Reads/initializes last execution timestamp       | Every 5 minutes                      | Get tickets created after last execution           |                                               |
| Get tickets created after last execution     | Zendesk           | Fetches tickets created after last timestamp     | Get last execution timestamp         | Add requester information to ticket data, Get requester information |                                               |
| Get requester information                    | Zendesk           | Retrieves requester user info by requester_id     | Get tickets created after last execution | Keep only needed requester information           |                                               |
| Keep only needed requester information       | Set               | Retains requester ID and email                     | Get requester information            | Add requester information to ticket data           |                                               |
| Add requester information to ticket data    | Merge             | Merges tickets with requester info by requester_id | Get tickets created after last execution, Keep only needed requester information | Search requester in pipedrive, Add contact owner to ticket data |                                               |
| Search requester in pipedrive                | Pipedrive         | Searches Pipedrive for requester by email         | Add requester information to ticket data | Get owner information of Pipedrive contact        |                                               |
| Get owner information of Pipedrive contact   | HTTP Request      | Retrieves owner info of Pipedrive contact          | Search requester in pipedrive        | Keep only requester owner email                     |                                               |
| Keep only requester owner email               | Set               | Keeps Pipedrive requester email and owner email    | Get owner information of Pipedrive contact | Add Pipedrive agent data to pipedrive contact information |                                               |
| Get agents and admins                        | Zendesk           | Retrieves all Zendesk agents and admins           | None                                 | Keep only email and Id                             |                                               |
| Keep only email and Id                       | Set               | Keeps only email and id fields from Zendesk agents | Get agents and admins                | Add Pipedrive agent data to pipedrive contact information |                                               |
| Add Pipedrive agent data to pipedrive contact information | Merge | Merges Pipedrive owner info with Zendesk agents by email | Keep only requester owner email, Keep only email and Id | Add contact owner to ticket data                   |                                               |
| Add contact owner to ticket data             | Merge             | Merges contact owner info with ticket/requester data | Add requester information to ticket data, Add Pipedrive agent data to pipedrive contact information | Contact exists in Pipedrive                       |                                               |
| Contact exists in Pipedrive                  | If                | Checks if Pipedrive contact owner exists           | Add contact owner to ticket data     | Change assignee to Pipedrive contact owner, Add a note requester not found |                                               |
| Change assignee to Pipedrive contact owner  | Zendesk           | Updates ticket assignee to Pipedrive contact owner | Contact exists in Pipedrive (true)   | Set new last execution timestamp                   |                                               |
| Add a note requester not found               | Zendesk           | Adds internal note if requester not found           | Contact exists in Pipedrive (false)  | Set new last execution timestamp                   |                                               |
| Set new last execution timestamp             | Function Item     | Updates stored last execution timestamp             | Change assignee to Pipedrive contact owner, Add a note requester not found | None                                              |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node: "Every 5 minutes"**  
   - Type: Cron  
   - Set to trigger every 5 minutes (everyX minutes = 5)  
   - Connect output to "Get last execution timestamp"

2. **Create Function Item Node: "Get last execution timestamp"**  
   - Reads `lastExecution` from global static data or initializes it  
   - Adds `executionTimeStamp` (current time) and `lastExecution` fields to item JSON  
   - Connect input from "Every 5 minutes"  
   - Output to "Get tickets created after last execution"

3. **Create Zendesk Node: "Get tickets created after last execution"**  
   - Operation: getAll tickets  
   - Options: Query `=created>{{ $json["lastExecution"] }}`, sort by `updated_at` descending  
   - Credentials: Zendesk account (OAuth2 or API token)  
   - Input from "Get last execution timestamp"  
   - Output branches:  
     a) To "Add requester information to ticket data"  
     b) To "Get requester information"

4. **Create Zendesk Node: "Get requester information"**  
   - Operation: get user  
   - Resource: user  
   - ID: `={{ $json["requester_id"] }}`  
   - Credentials: Zendesk account  
   - Input from "Get tickets created after last execution"  
   - Output to "Keep only needed requester information"

5. **Create Set Node: "Keep only needed requester information"**  
   - Keep only the fields:  
     - `requester_id` = `{{$json["id"]}}`  
     - `requester_email` = `{{$json["email"]}}`  
   - Input from "Get requester information"  
   - Output to "Add requester information to ticket data"

6. **Create Merge Node: "Add requester information to ticket data"**  
   - Mode: Merge by Key  
   - PropertyName1: `requester_id`  
   - PropertyName2: `requester_id`  
   - Inputs:  
     1) From "Get tickets created after last execution"  
     2) From "Keep only needed requester information"  
   - Outputs to:  
     a) "Search requester in pipedrive"  
     b) "Add contact owner to ticket data"

7. **Create Pipedrive Node: "Search requester in pipedrive"**  
   - Operation: search person  
   - Term: `={{ $json["requester_email"] }}`  
   - Additional Fields: fields = email  
   - Credentials: Pipedrive API  
   - Input from "Add requester information to ticket data"  
   - Output to "Get owner information of Pipedrive contact"

8. **Create HTTP Request Node: "Get owner information of Pipedrive contact"**  
   - Method: GET  
   - URL: `https://n8n.pipedrive.com/api/v1/users/{{$json["owner"]["id"]}}`  
   - Authentication: Use Pipedrive API credentials (predefined for HTTP Request node)  
   - Input from "Search requester in pipedrive"  
   - Output to "Keep only requester owner email"

9. **Create Set Node: "Keep only requester owner email"**  
   - Keep fields:  
     - `requester_pipedrive_email` = `{{$node["Search requester in pipedrive"].json["primary_email"]}}`  
     - `requester_pipedrive_owner_email` = `{{$json["data"].email}}`  
   - Input from "Get owner information of Pipedrive contact"  
   - Output to "Add Pipedrive agent data to pipedrive contact information"

10. **Create Zendesk Node: "Get agents and admins"**  
    - Operation: getAll users  
    - Filters: role = agent, admin  
    - Credentials: Zendesk account  
    - No input connection (runs independently)  
    - Output to "Keep only email and Id"

11. **Create Set Node: "Keep only email and Id"**  
    - Keep fields:  
      - `agent_email` = `{{$json["email"]}}`  
      - `agent_id` = `{{$json["id"]}}`  
    - Input from "Get agents and admins"  
    - Output to "Add Pipedrive agent data to pipedrive contact information"

12. **Create Merge Node: "Add Pipedrive agent data to pipedrive contact information"**  
    - Mode: Merge by Key  
    - PropertyName1: `requester_pipedrive_owner_email`  
    - PropertyName2: `agent_email`  
    - Inputs:  
      1) From "Keep only requester owner email"  
      2) From "Keep only email and Id"  
    - Output to "Add contact owner to ticket data"

13. **Create Merge Node: "Add contact owner to ticket data"**  
    - Mode: Merge by Key  
    - PropertyName1: `requester_email`  
    - PropertyName2: `requester_pipedrive_email`  
    - Inputs:  
      1) From "Add requester information to ticket data"  
      2) From "Add Pipedrive agent data to pipedrive contact information"  
    - Output to "Contact exists in Pipedrive"

14. **Create If Node: "Contact exists in Pipedrive"**  
    - Condition: Check if `{{$json["agent_id"]}}` isNotEmpty  
    - Input from "Add contact owner to ticket data"  
    - True output to "Change assignee to Pipedrive contact owner"  
    - False output to "Add a note requester not found"

15. **Create Zendesk Node: "Change assignee to Pipedrive contact owner"**  
    - Operation: update ticket  
    - Ticket ID: `{{$json["id"]}}`  
    - Update Fields: assigneeEmail = `{{$json["requester_pipedrive_owner_email"]}}`  
    - Credentials: Zendesk account  
    - Input from "Contact exists in Pipedrive" (true branch)  
    - Output to "Set new last execution timestamp"

16. **Create Zendesk Node: "Add a note requester not found"**  
    - Operation: update ticket  
    - Ticket ID: `{{$json["id"]}}`  
    - Update Fields: internalNote = "Requester not found in Pipedrive"  
    - Credentials: Zendesk account  
    - Input from "Contact exists in Pipedrive" (false branch)  
    - Output to "Set new last execution timestamp"

17. **Create Function Item Node: "Set new last execution timestamp"**  
    - Sets static data `lastExecution` to the `executionTimeStamp` value from "Get last execution timestamp" node  
    - Execute Once enabled  
    - Inputs from "Change assignee to Pipedrive contact owner" and "Add a note requester not found"  
    - No outputs (ends workflow)

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Pipedrive and Zendesk accounts must be created by the same person or with the same email to ensure matching | Prerequisites section in workflow description                                                   |
| For Pipedrive credentials setup and details, see: https://docs.n8n.io/integrations/builtin/credentials/pipedrive/ | Official n8n Pipedrive credential documentation                                                |
| For Zendesk credentials setup and details, see: https://docs.n8n.io/integrations/builtin/credentials/zendesk/ | Official n8n Zendesk credential documentation                                                  |
| This workflow relies on static data storage to keep track of last execution timestamp to avoid duplicate ticket processing | Static data usage within Function Item nodes                                                    |
| The HTTP Request node uses Pipedrive API authentication with predefined credential type to fetch owner info | Pipedrive API endpoint: https://n8n.pipedrive.com/api/v1/users/{userId}                        |