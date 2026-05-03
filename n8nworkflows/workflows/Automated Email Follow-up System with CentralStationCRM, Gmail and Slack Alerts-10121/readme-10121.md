Automated Email Follow-up System with CentralStationCRM, Gmail and Slack Alerts

https://n8nworkflows.xyz/workflows/automated-email-follow-up-system-with-centralstationcrm--gmail-and-slack-alerts-10121


# Automated Email Follow-up System with CentralStationCRM, Gmail and Slack Alerts

### 1. Workflow Overview

This workflow automates personalized email follow-ups for contacts tagged "Outreach" in CentralStationCRM. It runs on weekdays at 17:00, fetching updated contacts from the CRM, sending an initial email, then waiting 7 days to check for replies. If no reply is detected, it sends a follow-up email and alerts a user via Slack when a response is received. The workflow is structured into these main logical blocks:

- **1.1 Scheduled Trigger and CRM Data Fetch:** Trigger the workflow on weekdays and retrieve updated contacts from CentralStationCRM.
- **1.2 Tag Filtering:** Identify contacts tagged "Outreach" to target only relevant individuals.
- **1.3 Initial Email Sending:** Send a predefined initial email via Gmail to each filtered contact.
- **1.4 Wait and Response Checking:** Wait for 7 days, then query Gmail to check if the contact has replied.
- **1.5 Follow-up Email and Notification:** If no reply, send a follow-up email and notify the user via Slack upon any reply detection.
- **1.6 Duplicate Removal:** Ensure no repeated processing of the same email addresses.
- **1.7 Alerts:** Slack notifications for user awareness upon contact replies.
- **1.8 Documentation and Setup Notes:** Sticky notes providing setup instructions and workflow rationale.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and CRM Data Fetch

- **Overview:**  
Runs the workflow on weekdays at 17:00 to fetch CRM contacts updated on the current day.

- **Nodes Involved:**  
  - Cron: weekday 17:00  
  - get CRM updates

- **Node Details:**

  - **Cron: weekday 17:00**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression set to trigger at 17:00 Monday through Friday.  
    - Input: None (trigger node)  
    - Output: Activates "get CRM updates" node.  
    - Edge cases: Cron misconfiguration could cause no trigger; time zone differences may affect timing.

  - **get CRM updates**  
    - Type: HTTP Request  
    - Configuration: GET request to CentralStationCRM API endpoint `/api/people`, filtering for people updated since the start of the day, including related tags, addresses, companies, and emails. Uses HTTP header authentication with an API key named `X-apikey`.  
    - Key expression: Filters contacts updated since start of current day using `new Date().beginningOf('day')`.  
    - Input: Trigger from Cron node  
    - Output: Passes CRM data to "test for tag" node.  
    - Edge cases: API key invalid or expired, network timeouts, rate limits, malformed response.  
    - Version: HTTP Request node v4.2.

#### 2.2 Tag Filtering

- **Overview:**  
Filters CRM contacts to only those tagged with "Outreach".

- **Nodes Involved:**  
  - test for tag

- **Node Details:**

  - **test for tag**  
    - Type: If  
    - Configuration: Checks if the `tags` field of the person JSON contains the string `"Outreach"`. Uses string "contains" operator with case sensitivity.  
    - Input: CRM updates from previous node.  
    - Output: True branch leads to "Send a message" node; false branch ends flow for that item.  
    - Edge cases: Empty or missing tags array, case sensitivity causing false negatives, JSON path errors.  
    - Version: If node v2.2.

#### 2.3 Initial Email Sending

- **Overview:**  
Sends an initial email to the filtered contact's primary email address.

- **Nodes Involved:**  
  - Send a message  
  - wait 7 days

- **Node Details:**

  - **Send a message**  
    - Type: Gmail node (Send Email)  
    - Configuration:  
      - Sends email to the first email address of the person.  
      - Subject preset to "Unser Gespräch".  
      - Message text includes salutation and name placeholders with static placeholders for content and signature to be customized.  
      - Email type: plain text.  
      - Attribution disabled.  
    - Input: True branch output of "test for tag".  
    - Output: Connects to "wait 7 days".  
    - Edge cases: Invalid email addresses, Gmail API auth failures, rate limits, message formatting errors.  
    - Version: Gmail node v2.1.

  - **wait 7 days**  
    - Type: Wait  
    - Configuration: Pauses workflow execution for 7 days before continuing.  
    - Input: Continues from email sending.  
    - Output: To "get last 7 days".  
    - Edge cases: Workflow interruptions during wait, time zone considerations.  
    - Version: Wait node v1.1.

#### 2.4 Wait and Response Checking

- **Overview:**  
After waiting 7 days, queries Gmail to check if the contact replied to the initial email.

- **Nodes Involved:**  
  - get last 7 days  
  - answered?  
  - Remove Duplicates  
  - alert user

- **Node Details:**

  - **get last 7 days**  
    - Type: Gmail node (Get Emails)  
    - Configuration:  
      - Searches Gmail inbox for emails from the contact's email address within the last 7 days.  
      - Limit set to 10 messages max.  
      - Uses Gmail search query with `from:` filter dynamically injected from the contact email.  
    - Input: Output from "wait 7 days".  
    - Output: To "answered?" node.  
    - Edge cases: Gmail API limits, no emails found, incorrect query syntax.  
    - Version: Gmail node v2.1.

  - **answered?**  
    - Type: If  
    - Configuration: Checks if any email in response has an ID (i.e., if any replies exist).  
    - Input: Emails from "get last 7 days".  
    - Output: True branch goes to "Remove Duplicates"; false branch goes to "send another message".  
    - Edge cases: Empty reply list, JSON path errors.  
    - Version: If node v2.2.

  - **Remove Duplicates**  
    - Type: Remove Duplicates  
    - Configuration: Removes duplicated emails based on sender address (`from.value[0].address`).  
    - Input: Replies from "answered?".  
    - Output: To "alert user".  
    - Edge cases: Addresses missing or malformed, duplicates not detected correctly.  
    - Version: Remove Duplicates node v2.

  - **alert user**  
    - Type: Slack  
    - Configuration:  
      - Sends Slack message to specific user (user ID U063KFE3KNW).  
      - Message informs user that the contact has responded, including a direct Gmail link to the thread.  
      - Authenticated with OAuth2 credentials.  
    - Input: Filtered unique replies.  
    - Output: Ends flow.  
    - Edge cases: Slack API auth failure, user ID incorrect, message formatting errors.  
    - Version: Slack node v2.3.

#### 2.5 Follow-up Email and Notification

- **Overview:**  
If no reply is detected after 7 days, sends a follow-up email, then checks again for replies and alerts user if detected.

- **Nodes Involved:**  
  - send another message  
  - get last 7 days again  
  - replied now?  
  - Remove Duplicates1  
  - alert user1

- **Node Details:**

  - **send another message**  
    - Type: Gmail node (Send Email)  
    - Configuration:  
      - Sends a follow-up email to the contact's first email address.  
      - Subject same as initial email ("Unser Gespräch").  
      - Message text placeholders for second message content and signature.  
      - Sender name overridden to "Christian Lipowsky".  
      - Attribution disabled.  
    - Input: False branch from "answered?" node.  
    - Output: To "get last 7 days again".  
    - Edge cases: Same as initial email sending.  
    - Version: Gmail node v2.1.

  - **get last 7 days again**  
    - Type: Gmail node (Get Emails)  
    - Configuration: Same as "get last 7 days", querying emails from contact within past 7 days.  
    - Input: Output from "send another message".  
    - Output: To "replied now?".  
    - Edge cases: Same as earlier Gmail search node.  
    - Version: Gmail node v2.1.

  - **replied now?**  
    - Type: If  
    - Configuration: Checks if any email reply exists (email ID exists).  
    - Input: Emails from "get last 7 days again".  
    - Output: True branch to "Remove Duplicates1"; false branch ends flow.  
    - Edge cases: Same as previous If node checking replies.  
    - Version: If node v2.2.

  - **Remove Duplicates1**  
    - Type: Remove Duplicates  
    - Configuration: Removes duplicates based on sender address like prior Remove Duplicates node.  
    - Input: Replies from "replied now?".  
    - Output: To "alert user1".  
    - Edge cases: Same as previous Remove Duplicates node.  
    - Version: Remove Duplicates node v2.

  - **alert user1**  
    - Type: Slack  
    - Configuration: Same as "alert user", sends Slack alert to user about contact response with Gmail thread link.  
    - Input: Unique replies from "Remove Duplicates1".  
    - Output: Ends flow.  
    - Edge cases: Same as previous Slack node.  
    - Version: Slack node v2.3.

#### 2.6 Duplicate Removal

- **Overview:**  
Removes duplicate emails based on sender address to avoid multiple alerts for the same contact.

- **Nodes Involved:**  
  - Remove Duplicates  
  - Remove Duplicates1

- **Node Details:**  
  - Both nodes configured identically, filtering duplicates by comparing the sender's email address.  
  - Inputs come from Gmail search results filtered by reply checks.  
  - Outputs to Slack alert nodes.

#### 2.7 Alerts

- **Overview:**  
Send Slack notifications to a specific user when the contact has responded.

- **Nodes Involved:**  
  - alert user  
  - alert user1

- **Node Details:**  
  - Both nodes send Slack messages to user ID `U063KFE3KNW` (cached username "christian").  
  - Messages contain contact name and salutation plus a link to the Gmail conversation thread.  
  - Use OAuth2 authentication.  
  - Triggered on detection of responses after initial or follow-up messages.

#### 2.8 Documentation and Setup Notes

- **Overview:**  
Sticky notes provide guidance and references for setup and usage.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note8

- **Node Details:**

  - **Sticky Note**  
    - Contains a logo image and a textual overview of the workflow logic and usage notes.  
    - Warns to test workflow carefully and how to disable wait nodes for tests.

  - **Sticky Note1**  
    - Lists the external tools integrated: CentralStationCRM, Gmail, Slack.

  - **Sticky Note2**  
    - Step-by-step instructions on generating the CentralStationCRM API key with screenshots.

  - **Sticky Note8**  
    - Instructions for configuring CentralStationCRM credentials in n8n (header auth with `X-apikey`).

  - **Sticky Note3**  
    - Setup instructions for Slack and Gmail credentials with screenshots and usage tips.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                     | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                         |
|---------------------|---------------------|-----------------------------------|------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------|
| Cron: weekday 17:00 | Schedule Trigger    | Starts workflow on weekdays at 17:00 | None                   | get CRM updates           |                                                                                                                     |
| get CRM updates     | HTTP Request        | Fetches today's updated contacts from CentralStationCRM | Cron: weekday 17:00     | test for tag              |                                                                                                                     |
| test for tag        | If                  | Filters contacts tagged "Outreach" | get CRM updates         | Send a message            |                                                                                                                     |
| Send a message      | Gmail (Send Email)  | Sends initial email to contact     | test for tag             | wait 7 days               |                                                                                                                     |
| wait 7 days         | Wait                | Pauses 7 days before checking reply | Send a message           | get last 7 days           |                                                                                                                     |
| get last 7 days     | Gmail (Get Emails)  | Looks for replies from contact in last 7 days | wait 7 days             | answered?                 |                                                                                                                     |
| answered?           | If                  | Checks if contact replied          | get last 7 days          | Remove Duplicates (true), send another message (false) |                                                                                                                     |
| Remove Duplicates   | Remove Duplicates   | Removes duplicate replies          | answered? (true)         | alert user                |                                                                                                                     |
| alert user          | Slack               | Notifies user contact has replied | Remove Duplicates        | None                      |                                                                                                                     |
| send another message| Gmail (Send Email)  | Sends follow-up email if no reply  | answered? (false)        | get last 7 days again     |                                                                                                                     |
| get last 7 days again | Gmail (Get Emails) | Checks again for replies after follow-up | send another message     | replied now?              |                                                                                                                     |
| replied now?        | If                  | Checks if contact replied after follow-up | get last 7 days again    | Remove Duplicates1 (true) |                                                                                                                     |
| Remove Duplicates1  | Remove Duplicates   | Removes duplicate replies after follow-up | replied now? (true)      | alert user1               |                                                                                                                     |
| alert user1         | Slack               | Notifies user contact replied after follow-up | Remove Duplicates1       | None                      |                                                                                                                     |
| Sticky Note         | Sticky Note         | Workflow overview and usage notes  | None                    | None                      | ![CSCRM Logo](https://s3.42he.com/cscrm-marketing-page-production/Logo_Central_Station_CRM_0dd02e23d2.jpeg) \nWarning to test carefully and disable wait nodes during testing. |
| Sticky Note1        | Sticky Note         | Lists external tools used          | None                    | None                      | Explains integrated tools: CentralStationCRM, Gmail, Slack                                                         |
| Sticky Note2        | Sticky Note         | CentralStationCRM API key setup    | None                    | None                      | Step-by-step API key generation instructions with screenshots                                                      |
| Sticky Note8        | Sticky Note         | CentralStationCRM n8n credential setup | None                    | None                      | Instructions to create header auth credential with X-apikey                                                        |
| Sticky Note3        | Sticky Note         | Slack and Gmail credentials setup  | None                    | None                      | Slack and Gmail credential setup with screenshots and user instructions                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Cron: weekday 17:00`  
   - Type: Schedule Trigger  
   - Set Cron Expression: `0 17 * * 1-5` (At 17:00 on Monday through Friday)  
   - No inputs.

2. **Create HTTP Request Node to Fetch CRM Updates**  
   - Name: `get CRM updates`  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://api.centralstationcrm.net/api/people`  
   - Send Query Parameters as JSON:  
     ```json
     {
       "filter": {
         "updated_at": { "larger_than": "{{ new Date().beginningOf('day') }}" }
       },
       "includes": "tags addrs companies emails"
     }
     ```  
   - Authentication: HTTP Header Auth with header `X-apikey` (CentralStationCRM API key)  
   - Header: `accept: application/json`  
   - Connect output of `Cron: weekday 17:00` to this node.

3. **Create If Node to Test for Tag "Outreach"**  
   - Name: `test for tag`  
   - Type: If  
   - Condition: Check if `.person.tags` contains string `"Outreach"` (case sensitive)  
   - Connect output of `get CRM updates` to this node.

4. **Create Gmail Node to Send Initial Message**  
   - Name: `Send a message`  
   - Type: Gmail (Send Email)  
   - Set recipient to `{{ $json.person.emails[0].name }}`  
   - Subject: `"Unser Gespräch"`  
   - Message body:  
     ```
     Hi {{ $json.person.salutation }} {{ $json.person.name }},

     #INSERT YOUR TEXT#

     Best regards

     #SIGNATURE#
     ```  
   - Disable attribution  
   - Connect True output of `test for tag` to this node.  
   - Configure Gmail credentials (OAuth2).

5. **Create Wait Node**  
   - Name: `wait 7 days`  
   - Type: Wait  
   - Duration: 7 days  
   - Connect output of `Send a message` to this node.

6. **Create Gmail Node to Get Last 7 Days Emails**  
   - Name: `get last 7 days`  
   - Type: Gmail (Get Emails)  
   - Filters:  
     - Query: `from: {{ $('test for tag').item.json.person.emails[0].name }}`  
     - Received after: `{{ new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString() }}`  
   - Limit: 10 messages  
   - Connect output of `wait 7 days` to this node.

7. **Create If Node to Check If Replied**  
   - Name: `answered?`  
   - Type: If  
   - Condition: Check if any email has an ID (exists)  
   - Connect output of `get last 7 days` to this node.

8. **Create Remove Duplicates Node for Replies**  
   - Name: `Remove Duplicates`  
   - Type: Remove Duplicates  
   - Fields to compare: `from.value[0].address`  
   - Connect True output of `answered?` to this node.

9. **Create Slack Node to Alert User on Reply**  
   - Name: `alert user`  
   - Type: Slack  
   - Message:  
     ```
     Hi #YOURNAME#,

     your contact {{ $('test for tag').item.json.person.salutation }} {{ $('test for tag').item.json.person.name }} has responded.

     Look it up: 

     https://mail.google.com/mail/u/0/#all/{{ $json.threadId }}
     ```  
   - User: Select your Slack user via OAuth2 credential  
   - Connect output of `Remove Duplicates` to this node.

10. **Create Gmail Node to Send Follow-up Message**  
    - Name: `send another message`  
    - Type: Gmail (Send Email)  
    - Recipient: `{{ $('get CRM updates').item.json.person.emails[0].name }}`  
    - Subject: `"Unser Gespräch"`  
    - Message body:  
      ```
      Hi {{ $('get CRM updates').item.json.person.salutation }} {{ $json.person.name }},

      ##CONTENT OF YOUR SECOND MESSAGE##

      Best regards

      ##SIGNATURE##
      ```  
    - Sender name: `"Christian Lipowsky"`  
    - Disable attribution  
    - Connect False output of `answered?` to this node.

11. **Create Gmail Node to Get Last 7 Days Emails Again**  
    - Name: `get last 7 days again`  
    - Type: Gmail (Get Emails)  
    - Same filter as previous "get last 7 days" node  
    - Connect output of `send another message` to this node.

12. **Create If Node to Check Reply After Follow-up**  
    - Name: `replied now?`  
    - Type: If  
    - Same condition as `answered?` node  
    - Connect output of `get last 7 days again` to this node.

13. **Create Remove Duplicates Node for Follow-up Replies**  
    - Name: `Remove Duplicates1`  
    - Type: Remove Duplicates  
    - Same configuration as previous Remove Duplicates node  
    - Connect True output of `replied now?` to this node.

14. **Create Slack Node to Alert User for Follow-up Replies**  
    - Name: `alert user1`  
    - Type: Slack  
    - Same configuration as previous alert user node  
    - Connect output of `Remove Duplicates1` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| ![CSCRM Logo](https://s3.42he.com/cscrm-marketing-page-production/Logo_Central_Station_CRM_0dd02e23d2.jpeg) \n\n* A simple email automation for tagged persons in CentralStationCRM \n* Uses cron time trigger \n* Fetches today's updated contacts with tags, companies, emails \n* Targets contacts tagged 'Outreach' \n* Sends initial message, waits 7 days, checks for replies \n* Alerts user on reply via Slack \n* Repeat process with follow-up messages \n\n*Test carefully — disable wait nodes to avoid spamming* | Workflow overview and usage notes (Sticky Note)                                                           |
| Lists integrated tools:\n- CentralStationCRM: https://centralstationcrm.de\n- Gmail by Google\n- Slack messaging tool                                                                                                                                                                                                                                                                                                            | Sticky Note1                                                                                            |
| CentralStationCRM API Key setup instructions with screenshots:\n- Log in\n- Access account settings\n- Create API key\n- Save key (cannot be retrieved again)\n- Use key in n8n header auth credential                                                                                                                                                                                                                            | Sticky Note2                                                                                            |
| CentralStationCRM n8n credential setup guide:\n- Use Header Auth\n- Header name: X-apikey\n- Enter your API key as value                                                                                                                                                                                                                                                                                                          | Sticky Note8                                                                                            |
| Slack node setup:\n- Create Slack OAuth2 credentials\n- Select your credential in Slack nodes\n- Use user (by username) for notifications\n- Gmail node setup with Google OAuth2 credentials                                                                                                                                                                                                                                     | Sticky Note3                                                                                            |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.