Syncing Contacts with CentralStationCRM, Rapidmail, and Slack

https://n8nworkflows.xyz/workflows/syncing-contacts-with-centralstationcrm--rapidmail--and-slack-6125


# Syncing Contacts with CentralStationCRM, Rapidmail, and Slack

### 1. Workflow Overview

This workflow automates the synchronization of new contacts created daily in CentralStationCRM with a Rapidmail newsletter list, incorporating Slack for user approval when needed. It targets small teams or businesses using CentralStationCRM to manage contacts and Rapidmail for email marketing, adding an interactive approval step via Slack to ensure contacts receive appropriate newsletter tagging.

Logical blocks:

- **1.1 Scheduled Trigger**: Initiates workflow execution once daily at a configurable time (default 5 PM).
- **1.2 CentralStationCRM Data Retrieval**: Queries CentralStationCRM for new contacts created since the start of the day, including related tags, addresses, companies, and emails.
- **1.3 Newsletter Tag Check**: Evaluates if the contact already has the "Newsletter" tag.
- **1.4 Slack Approval for Tagging**: If no "Newsletter" tag is present, sends an interactive Slack message to request user approval to add the tag.
- **1.5 Tag Assignment and Rapidmail Update**: Depending on approval, assigns the "Newsletter" tag in CentralStationCRM and adds the contact to the Rapidmail recipient list.
- **1.6 Workflow Termination**: Ends the workflow after processing each contact.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Triggers the workflow daily at a specified hour (default 5 PM).
- **Nodes Involved:** 
  - Workflow triggers daily at 5PM
- **Node Details:**

  - **Workflow triggers daily at 5PM**
    - Type: Schedule Trigger
    - Configuration: Set to trigger daily at 17:00 (5 PM)
    - Key parameters: `"triggerAtHour": 17`
    - Input connections: None (entry point)
    - Output connections: Connects to "Get new people per day from CentralStationCRM"
    - Failure types: Misconfiguration of time zone or trigger interval might cause no execution
    - Version requirements: v1.2 or later recommended for daily scheduling

#### 2.2 CentralStationCRM Data Retrieval

- **Overview:** Fetches contacts created during the current day from CentralStationCRM API, requesting related tags, addresses, companies, and emails.
- **Nodes Involved:**
  - Get new people per day from CentralStationCRM
- **Node Details:**

  - **Get new people per day from CentralStationCRM**
    - Type: HTTP Request
    - Configuration:
      - URL: `https://api.centralstationcrm.net/api/people`
      - Method: GET (default)
      - Query: Filters contacts with `created_at` greater than the start of the current day, includes `tags, addrs, companies, emails`.
      - Authentication: Uses CentralStationCRM API key via HTTP Header Auth (`X-apikey` header).
      - Headers: `accept: application/json`
    - Key expressions:
      - Query JSON: filters using `{{ new Date().beginningOf('day') }}` to get today's contacts.
    - Input connections: from Schedule Trigger node
    - Output connections: to "Does a person have a \"Newsletter\" tag?"
    - Failure types: API key authentication failure, API rate limiting, malformed query, empty response if no new contacts
    - Version requirements: n8n HTTP Request v4.2 or later recommended

#### 2.3 Newsletter Tag Check

- **Overview:** Checks if each contact retrieved has the "Newsletter" tag.
- **Nodes Involved:**
  - Does a person have a "Newsletter" tag?
- **Node Details:**

  - **Does a person have a "Newsletter" tag?**
    - Type: If Node (Condition)
    - Configuration:
      - Condition: Checks if the `person.tags` JSON string contains `"Newsletter"`.
      - Case-sensitive search using `contains` operator.
    - Input connections: from "Get new people per day from CentralStationCRM"
    - Output connections:
      - True branch: to "Write person to Rapidmail list"
      - False branch: to "ask in Slack: add \"Newsletter\" tag?"
    - Failure types: Missing or malformed `person.tags` field causing expression errors
    - Version requirements: If node v2.2 or later for advanced conditions

#### 2.4 Slack Approval for Tagging

- **Overview:** Sends an interactive Slack message to a specified user asking whether to add the "Newsletter" tag to the contact.
- **Nodes Involved:**
  - ask in Slack: add "Newsletter" tag?
  - Did the user click the "Yes" button?
- **Node Details:**

  - **ask in Slack: add "Newsletter" tag?**
    - Type: Slack Node (Send and Wait)
    - Configuration:
      - Sends a message to user `@Christian` with contact name.
      - Message includes interactive buttons "Ja" (Yes) and "Nein" (No).
      - Waits for user response.
      - Uses Slack OAuth2 credentials.
    - Input connections: from False branch of "Does a person have a \"Newsletter\" tag?"
    - Output connections: to "Did the user click the \"Yes\" button?"
    - Failure types: Slack API authentication failure, user not responding, message formatting errors
    - Version requirements: Slack node v2.3 or later for interactive approval

  - **Did the user click the "Yes" button?**
    - Type: If Node
    - Configuration:
      - Checks if Slack response `data.approved` equals `true`.
    - Input connections: from "ask in Slack: add \"Newsletter\" tag?"
    - Output connections:
      - True branch: to "give person a \"Newsletter\" tag"
      - False branch: to "end Workflow"
    - Failure types: Missing or malformed Slack response data

#### 2.5 Tag Assignment and Rapidmail Update

- **Overview:** Assigns the "Newsletter" tag to the contact in CentralStationCRM if approved, then adds the contact to the Rapidmail recipient list.
- **Nodes Involved:**
  - give person a "Newsletter" tag
  - Write person to Rapidmail list
- **Node Details:**

  - **give person a "Newsletter" tag**
    - Type: HTTP Request
    - Configuration:
      - URL: `https://api.centralstationcrm.net/api/people/{{person_id}}/tags`
      - Method: POST
      - JSON Body: `{ "tag": { "name": "Newsletter" } }`
      - Headers: `accept: application/json`, `Content-Type: application/json`
      - Authentication: CentralStationCRM API key via HTTP Header Auth
    - Input connections: from True branch of "Did the user click the \"Yes\" button?"
    - Output connections: to "Write person to Rapidmail list"
    - Failure types: API errors, invalid person ID, tag already present, authentication failure

  - **Write person to Rapidmail list**
    - Type: HTTP Request
    - Configuration:
      - URL: `https://apiv3.emailsys.net/v1/recipients`
      - Method: POST
      - JSON Body includes:
        - status: active
        - recipientlist_id: placeholder `ENTER-RAPIDMAIL-LIST-ID` (must be replaced)
        - email: first email of person
        - firstname and lastname fields
      - Headers: `Content-Type: application/json`, `Accept: application/json`
      - Authentication: Basic Auth using Rapidmail credentials
    - Input connections:
      - From True branch of "Does a person have a \"Newsletter\" tag?" (already tagged)
      - From "give person a \"Newsletter\" tag" after tagging
    - Output connections: none (end of data flow)
    - Failure types: Invalid Rapidmail API credentials, invalid recipient list ID, missing email address, API errors

#### 2.6 Workflow Termination

- **Overview:** Gracefully ends the workflow when no further action is needed.
- **Nodes Involved:**
  - end Workflow
- **Node Details:**

  - **end Workflow**
    - Type: No Operation (NoOp) Node
    - Configuration: None
    - Input connections: from False branch of "Did the user click the \"Yes\" button?"
    - Output connections: none
    - Role: Stops execution cleanly without error

---

### 3. Summary Table

| Node Name                                | Node Type           | Functional Role                        | Input Node(s)                              | Output Node(s)                              | Sticky Note                                                                                                          |
|-----------------------------------------|---------------------|-------------------------------------|-------------------------------------------|---------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Sticky Note1                            | Sticky Note         | Overview and branding                | None                                      | None                                        | ![CSCRM Logo](https://s3.42he.com/cscrm-marketing-page-production/Logo_Central_Station_CRM_0dd02e23d2.jpeg) # Sync New CentralStationCRM Contacts to Rapidmail with Slack Approval ... |
| Sticky Note2                            | Sticky Note         | Instructions for CentralStationCRM API key setup | None                                      | None                                        | ## 1. set up CentralStationCRM API Key ...                                                                            |
| Sticky Note8                            | Sticky Note         | Instructions for CentralStationCRM n8n credential setup | None                                      | None                                        | ## 2. set up CentralStationCRM n8n credentials ...                                                                    |
| Sticky Note                            | Sticky Note         | Notes on Schedule Trigger setup     | None                                      | None                                        | ## 3. set up Schedule Trigger ...                                                                                      |
| Sticky Note10                          | Sticky Note         | Instructions for setting up new people query node | None                                      | None                                        | ## 4. set up new people/ day ...                                                                                       |
| Sticky Note3                            | Sticky Note         | Slack setup instructions            | None                                      | None                                        | ## 5. set up Slack ...                                                                                                 |
| Sticky Note4                            | Sticky Note         | Setup for the Newsletter tag HTTP node | None                                      | None                                        | ## 6. set up "Newsletter" tag Node ...                                                                                 |
| Sticky Note5                            | Sticky Note         | Rapidmail setup instructions        | None                                      | None                                        | ## 7. set up Rapidmail ...                                                                                             |
| Sticky Note9                            | Sticky Note         | Tools involved in the workflow      | None                                      | None                                        | ## Tools in this workflow ...                                                                                          |
| Workflow triggers daily at 5PM          | Schedule Trigger    | Initiates daily workflow execution  | None                                      | Get new people per day from CentralStationCRM |                                                                                                                      |
| Get new people per day from CentralStationCRM | HTTP Request       | Retrieves new contacts from CRM     | Workflow triggers daily at 5PM            | Does a person have a "Newsletter" tag?       |                                                                                                                      |
| Does a person have a "Newsletter" tag? | If                  | Checks for presence of "Newsletter" tag | Get new people per day from CentralStationCRM | Write person to Rapidmail list; ask in Slack: add "Newsletter" tag? |                                                                                                                      |
| ask in Slack: add "Newsletter" tag?    | Slack               | Requests Slack approval for tagging | Does a person have a "Newsletter" tag? (False branch) | Did the user click the "Yes" button?           |                                                                                                                      |
| Did the user click the "Yes" button?   | If                  | Determines user response            | ask in Slack: add "Newsletter" tag?       | give person a "Newsletter" tag; end Workflow |                                                                                                                      |
| give person a "Newsletter" tag         | HTTP Request        | Assigns "Newsletter" tag in CRM     | Did the user click the "Yes" button? (True branch) | Write person to Rapidmail list                 |                                                                                                                      |
| Write person to Rapidmail list          | HTTP Request        | Adds person to Rapidmail recipient list | Does a person have a "Newsletter" tag? (True branch); give person a "Newsletter" tag | None                                          |                                                                                                                      |
| end Workflow                           | NoOp                | Ends workflow execution              | Did the user click the "Yes" button? (False branch) | None                                          |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Type: Schedule Trigger
   - Set rule to trigger daily at 17:00 (5 PM).
   - Position appropriately (e.g., top-left).
   
2. **Create an HTTP Request node to get new contacts:**
   - Name: "Get new people per day from CentralStationCRM"
   - Type: HTTP Request
   - URL: `https://api.centralstationcrm.net/api/people`
   - Method: GET
   - Authentication: Set up generic HTTP Header Auth credential with CentralStationCRM API key:
     - Credential Type: Header Auth
     - Header Name: `X-apikey`
     - Header Value: your CentralStationCRM API key
   - Headers: Add header `accept: application/json`
   - Query Parameters: Use JSON query with filter for `created_at` greater than start of current day:
     ```json
     {
       "filter": {
         "created_at": {
           "larger_than": "{{ new Date().beginningOf('day') }}"
         }
       },
       "includes": "tags addrs companies emails"
     }
     ```
   - Connect Schedule Trigger node output to this node input.

3. **Create an If node to check for "Newsletter" tag:**
   - Name: "Does a person have a \"Newsletter\" tag?"
   - Type: If
   - Condition: Check if `person.tags` contains substring `"Newsletter"` (case-sensitive).
   - Connect output of "Get new people per day from CentralStationCRM" to this node.

4. **Create Slack node for approval:**
   - Name: "ask in Slack: add \"Newsletter\" tag?"
   - Type: Slack node (Send and Wait)
   - Authentication: OAuth2 with Slack account credentials
   - User: enter the Slack username (e.g., `@Christian`)
   - Message: 
     ```
     Hey!

     Would you like to add

     {{ $json.person.first_name }} {{ $json.person.name }}

     to your Newsletter?
     ```
   - Approval Options:
     - Approval Type: double
     - Approve Label: "Ja"
     - Disapprove Label: "Nein"
   - Connect False output of the If node ("Does a person have a \"Newsletter\" tag?") to this Slack node.

5. **Create an If node to handle Slack response:**
   - Name: "Did the user click the \"Yes\" button?"
   - Type: If
   - Condition: Check if `data.approved` equals `true`
   - Connect Slack node output to this If node.

6. **Create HTTP Request node to assign "Newsletter" tag:**
   - Name: "give person a \"Newsletter\" tag"
   - Type: HTTP Request
   - URL: `https://api.centralstationcrm.net/api/people/{{ $json.person.id }}/tags`
   - Method: POST
   - Headers:
     - `accept: application/json`
     - `Content-Type: application/json`
   - Body (JSON):
     ```json
     {
       "tag": {
         "name": "Newsletter"
       }
     }
     ```
   - Authentication: CentralStationCRM API key via HTTP Header Auth (same as step 2)
   - Connect True branch of "Did the user click the \"Yes\" button?" to this node.

7. **Create HTTP Request node to write to Rapidmail:**
   - Name: "Write person to Rapidmail list"
   - Type: HTTP Request
   - URL: `https://apiv3.emailsys.net/v1/recipients`
   - Method: POST
   - Headers:
     - `Content-Type: application/json`
     - `Accept: application/json`
   - Body (JSON):
     ```json
     {
       "status": "active",
       "recipientlist_id": ENTER-RAPIDMAIL-LIST-ID,
       "email": "{{ $json.person.emails[0].name }}",
       "firstname": "{{ $json.person.first_name }}",
       "lastname": "{{ $json.person.name }}"
     }
     ```
   - Authentication: Basic Auth with Rapidmail credentials (username and API key)
   - Connect:
     - True branch of "Does a person have a \"Newsletter\" tag?" directly to this node
     - Output of "give person a \"Newsletter\" tag" node to this node

8. **Create NoOp node to end workflow:**
   - Name: "end Workflow"
   - Type: NoOp
   - Connect False branch of "Did the user click the \"Yes\" button?" to this node.

9. **Verify connections:**

   - Schedule Trigger → Get new people per day from CentralStationCRM
   - Get new people per day from CentralStationCRM → Does a person have a "Newsletter" tag?
   - Does a person have a "Newsletter" tag? True → Write person to Rapidmail list
   - Does a person have a "Newsletter" tag? False → ask in Slack: add "Newsletter" tag?
   - ask in Slack: add "Newsletter" tag? → Did the user click the "Yes" button?
   - Did the user click the "Yes" button? True → give person a "Newsletter" tag → Write person to Rapidmail list
   - Did the user click the "Yes" button? False → end Workflow

10. **Setup Credentials:**

    - CentralStationCRM API Key:
      - Obtain API key from CentralStationCRM account settings.
      - Create n8n HTTP Header Auth credentials with header name `X-apikey`.
    - Slack OAuth2:
      - Connect n8n Slack OAuth2 credentials with appropriate scopes to send messages and receive interactive responses.
    - Rapidmail Basic Auth:
      - Create Basic Auth credentials in n8n using Rapidmail API username and key.
    - Replace `ENTER-RAPIDMAIL-LIST-ID` in the Rapidmail node JSON body with the actual recipient list ID from Rapidmail.

11. **Activate the workflow** after testing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                     | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow includes branding images and detailed setup instructions accessible by double-clicking sticky note nodes within n8n editor.                                                                                                                                           | Sticky Notes 1-10 in the workflow                                                                         |
| CentralStationCRM API documentation: https://api.centralstationcrm.net/api-docs/index.html                                                                                                                                                                                     | CentralStationCRM API reference                                                                             |
| Slack message customization warns not to alter JSON formatting for interactive approval buttons.                                                                                                                                                                               | Sticky Note 3                                                                                              |
| Rapidmail API key creation and list ID extraction instructions are crucial for correct operation.                                                                                                                                                                              | Sticky Note 5 and Sticky Note 7                                                                             |
| The workflow is designed for daily operation but schedule trigger frequency can be adjusted as needed.                                                                                                                                                                          | Sticky Note                                                                                            |
| Use n8n version supporting HTTP Request v4.2 and Slack v2.3 nodes for best compatibility.                                                                                                                                                                                       | Version notes in node details                                                                              |

---

**Disclaimer:** The provided text is generated exclusively from an automated n8n workflow configuration. It fully complies with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.