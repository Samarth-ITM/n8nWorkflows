Poll emails using JMAP 

https://n8nworkflows.xyz/workflows/poll-emails-using-jmap--2032


# Poll emails using JMAP 

### 1. Workflow Overview

This workflow demonstrates how to retrieve unread email information from any JMAP-compliant mail server, using Fastmail’s API as an example. Unlike typical n8n email nodes limited to Gmail or Outlook triggers, this workflow leverages JMAP’s HTTP API to perform custom email queries, enabling more flexible inbox monitoring such as counting unread messages or fetching specific message details on-demand.

The workflow is structured into the following logical blocks:

- **1.1 Input Trigger**: Manual initiation of the workflow.
- **1.2 API Session and Mailbox Identification**: Fetching API session details to retrieve the mail account ID and mailboxes.
- **1.3 Filtering and Fetching Unread Emails**: Executing a JMAP query to find unread emails and fetching details of the first three.
- **1.4 Result Formatting**: Structuring the output data for further use or display.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview**: Starts the workflow manually by user action; no automatic triggers are used.
- **Nodes Involved**:  
  - When clicking "Execute Workflow"
- **Node Details**:  
  - **Type & Role**: Manual Trigger node; initiates execution.
  - **Configuration**: Default manual trigger, no parameters.
  - **Expressions**: None.
  - **Input/Output**: No input; output flows to “Fetch API details”.
  - **Version requirements**: Supported in all recent n8n versions.
  - **Potential Failures**: None; manual trigger is stable.
  - **Sub-workflow**: None.

#### 2.2 API Session and Mailbox Identification

- **Overview**: Authenticates with the JMAP server and fetches account and mailbox details needed for subsequent email queries.
- **Nodes Involved**:  
  - Fetch API details  
  - Get mailboxes  
  - Format results  
  - Sticky Note (explaining caching optimization)
- **Node Details**:

  - **Fetch API details**  
    - Type & Role: HTTP Request node, GET request to JMAP session endpoint.  
    - Configuration: URL set to `https://api.fastmail.com/jmap/session`, authentication via HTTP Header Auth credential named “Fastmail” using Bearer token.  
    - Expressions: None.  
    - Input/Output: Input from manual trigger; output to “Get mailboxes”.  
    - Failures: Possible network errors, auth failures if token invalid or expired.  
    - Sub-workflow: None.

  - **Get mailboxes**  
    - Type & Role: HTTP Request node, POST request to JMAP API endpoint to fetch mailboxes.  
    - Configuration:  
      - URL: `https://api.fastmail.com/jmap/api/`  
      - Method: POST  
      - Body: JSON with `using` capabilities and `methodCalls` calling `Mailbox/get` with accountId from previous node’s JSON path `$json.primaryAccounts['urn:ietf:params:jmap:mail']`.  
      - Authentication: Same HTTP Header Auth credentials.  
    - Expressions: Uses expression to dynamically set accountId.  
    - Input/Output: Input from “Fetch API details”; output to “Format results”.  
    - Failures: Possible malformed request, API errors, auth errors.  
    - Sub-workflow: None.

  - **Format results**  
    - Type & Role: Set node; extracts and sets two key values: `account_id` and `mailbox_id` (ID of inbox).  
    - Configuration:  
      - `account_id` is set from the first node JSON path for mail account ID.  
      - `mailbox_id` is extracted from mailbox list, filtering for mailbox with role “inbox”.  
    - Expressions: Uses a JavaScript expression to find the inbox mailbox ID from the array of mailboxes.  
    - Input/Output: Input from “Get mailboxes”; output to “Get unread messages”.  
    - Failures: Expression could fail if mailbox list is empty or inbox role not found.  
    - Sub-workflow: None.

  - **Sticky Note**  
    - Content: Advises caching account and mailbox IDs to improve performance and reduce API load, as these values rarely change.  
    - Position: Visually associated with nodes fetching mailbox info.

#### 2.3 Filtering and Fetching Unread Emails

- **Overview**: Queries the JMAP server to find unread emails in the inbox and retrieves details for the top three most recent unread messages.
- **Nodes Involved**:  
  - Get unread messages
- **Node Details**:

  - **Get unread messages**  
    - Type & Role: HTTP Request node, POST to JMAP API to query and then get emails.  
    - Configuration:  
      - URL: `https://api.fastmail.com/jmap/api/`  
      - Method: POST  
      - Body contains two method calls:  
        1. `Email/query` with filter for messages in inbox mailbox ID and keyword “not $seen” (unread), sorted by received date descending, limited to 3, and request for total count.  
        2. `Email/get` to fetch detailed properties (id, receivedAt, from, subject, keywords) for the queried email IDs.  
      - Authentication: HTTP Header Auth with Fastmail token.  
    - Expressions: Uses expressions to dynamically set account and mailbox IDs from previous node’s JSON.  
    - Input/Output: Input from “Format results”; output is the final data set.  
    - Failures: Network errors, auth failures, expression failures if IDs missing, or JMAP API errors.  
    - Sub-workflow: None.

#### 2.4 Result Formatting (Implicit)

- The final output node “Get unread messages” returns JSON data including total unread count and details for the first three unread emails. The workflow does not explicitly include a node to format or display these results further, leaving that to downstream processes or manual inspection.

#### 2.5 Credentials & Authentication Notes

- **Sticky Note1**  
  - Content: Explains that JMAP supports various authentication methods; for Fastmail, Bearer token-based Header Auth is used. Credentials in n8n must be set up as HTTP Header Auth with header `Authorization: Bearer $apiToken`.  
  - Includes link to Fastmail token creation page.  
  - Associated nodes: All HTTP Request nodes using Fastmail credentials.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                                    | Input Node(s)                  | Output Node(s)           | Sticky Note                                                                                                  |
|------------------------------|---------------------|---------------------------------------------------|-------------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger      | Start workflow manually                            | -                             | Fetch API details         |                                                                                                              |
| Fetch API details             | HTTP Request        | Retrieve JMAP session info including account IDs  | When clicking "Execute Workflow" | Get mailboxes             | See Sticky Note1 about credentials                                                                           |
| Get mailboxes                | HTTP Request        | Fetch mailboxes for the mail account               | Fetch API details             | Format results            | See Sticky Note1 about credentials; linked to Sticky Note about caching mailbox/account IDs                  |
| Format results               | Set                 | Extract account_id and inbox mailbox_id            | Get mailboxes                | Get unread messages       | See Sticky Note about caching mailbox/account IDs                                                            |
| Get unread messages          | HTTP Request        | Query unread emails and fetch details for top 3   | Format results               | -                        | See Sticky Note1 about credentials                                                                           |
| Sticky Note                 | Sticky Note          | Advise caching of account and mailbox IDs         | -                             | -                        | "Replacing the initial nodes\n\nThese nodes fetch your account and mailbox IDs. Consider saving these values instead of querying them on every execution to improve performance and reduce the load on the JMAP API." |
| Sticky Note1                | Sticky Note          | Explain credential setup for Fastmail bearer token | -                            | -                        | "The JMAP standard does not limit the available authentication options. Fastmail supports Bearer authentication as well as OAuth2.\n\nIn n8n you can implement the Fastmail Bearer authentication by creating Header Auth credentials with a name of `Authorization` and a value of `Bearer $apiToken` (replacing `$apiToken` with your actual [token from Fastmail](https://www.fastmail.com/settings/security/tokens))." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking "Execute Workflow"`  
   - Type: Manual Trigger  
   - No configuration needed.

2. **Create HTTP Request node to fetch API session details**  
   - Name: `Fetch API details`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.fastmail.com/jmap/session`  
   - Authentication: Set to HTTP Header Auth credentials (see step 7)  
   - Connect output of manual trigger to this node.

3. **Create HTTP Request node to get mailboxes**  
   - Name: `Get mailboxes`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.fastmail.com/jmap/api/`  
   - Authentication: Use same HTTP Header Auth credentials  
   - Body Parameters (JSON):  
     ```json
     {
       "using": [
         "urn:ietf:params:jmap:core",
         "urn:ietf:params:jmap:mail"
       ],
       "methodCalls": [
         [
           "Mailbox/get",
           {
             "accountId": "={{ $json.primaryAccounts['urn:ietf:params:jmap:mail'] }}",
             "ids": null
           },
           "mailboxes"
         ]
       ]
     }
     ```  
   - Ensure “Send Body” is enabled and content-type is JSON.  
   - Connect output of “Fetch API details” node to this node.

4. **Create Set node to format results**  
   - Name: `Format results`  
   - Type: Set  
   - Remove all fields except two:  
     - `account_id` with value: `={{ $('Fetch API details').first().json.primaryAccounts['urn:ietf:params:jmap:mail'] }}`  
     - `mailbox_id` with value: `={{ $json.methodResponses.find(e => e[2] == 'mailboxes')[1].list.find(e => e.role == 'inbox').id }}`  
   - Connect output of “Get mailboxes” to this node.

5. **Create HTTP Request node to get unread messages**  
   - Name: `Get unread messages`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.fastmail.com/jmap/api/`  
   - Authentication: Use same HTTP Header Auth credentials  
   - Body Parameters (JSON):  
     ```json
     {
       "using": [
         "urn:ietf:params:jmap:core",
         "urn:ietf:params:jmap:mail"
       ],
       "methodCalls": [
         [
           "Email/query",
           {
             "accountId": "={{ $json.account_id }}",
             "filter": {
               "inMailbox": "={{ $json.mailbox_id }}",
               "notKeyword": "$seen"
             },
             "sort": [
               {
                 "property": "receivedAt",
                 "isAscending": false
               }
             ],
             "limit": 3,
             "calculateTotal": true
           },
           "messages"
         ],
         [
           "Email/get",
           {
             "accountId": "={{ $json.account_id }}",
             "#ids": {
               "name": "Email/query",
               "path": "/ids",
               "resultOf": "messages"
             },
             "properties": [
               "id",
               "receivedAt",
               "from",
               "subject",
               "keywords"
             ]
           },
           "emails"
         ]
       ]
     }
     ```  
   - Ensure “Send Body” is enabled and content-type is JSON.  
   - Connect output of “Format results” node to this node.

6. **Configure Credentials**  
   - In n8n credentials section, create a new HTTP Header Auth credential named `Fastmail`.  
   - Set header name to `Authorization`.  
   - Set header value to `Bearer <your Fastmail API token>`. Replace `<your Fastmail API token>` with your actual token from Fastmail’s security settings.  
   - Assign this credential to all HTTP Request nodes in steps 2, 3, and 5.

7. **Optional: Add Sticky Notes**  
   - Add one sticky note near the mailbox and account ID nodes explaining the recommendation to cache account and mailbox IDs to reduce API calls.  
   - Add another sticky note near the HTTP Request nodes explaining how to set up the Fastmail Bearer token credentials, including a link to https://www.fastmail.com/settings/security/tokens.

This completes the full workflow recreation with proper node connections, parameter settings, and credential configuration.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                             | Context or Link                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The JMAP standard is an IETF standard aiming to replace legacy mail/calendar/contact protocols like IMAP, CalDAV, and CardDAV with a modern, efficient HTTP-based API.                                                   | https://jmap.io/spec.html                                      |
| Fastmail is the primary sponsor of JMAP and offers API access with bearer token authentication, making it ideal for testing and production use with n8n.                                                                 | https://ref.fm/u30544864                                       |
| If your email provider does not support JMAP yet, consider contacting them to express interest in this modern protocol.                                                                                                  | -                                                             |
| This workflow was built with n8n version 1.20 but should work with any version supporting HTTP Request nodes and header authentication.                                                                                   | -                                                             |
| For more info on creating tokens in Fastmail, visit their token management page to generate a personal API token for authentication.                                                                                     | https://www.fastmail.com/settings/security/tokens             |

---

This detailed analysis and reproduction guide should help users and AI agents fully understand, modify, and recreate the “Poll emails using JMAP” workflow in n8n.