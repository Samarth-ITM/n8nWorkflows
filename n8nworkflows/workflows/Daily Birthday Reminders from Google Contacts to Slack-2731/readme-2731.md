Daily Birthday Reminders from Google Contacts to Slack

https://n8nworkflows.xyz/workflows/daily-birthday-reminders-from-google-contacts-to-slack-2731


# Daily Birthday Reminders from Google Contacts to Slack

### 1. Workflow Overview

This workflow automates daily birthday reminders by integrating Google Contacts with Slack. It runs every day at a specified time, fetches all Google Contacts, filters those whose birthdays fall on the current day, and sends a personalized birthday message to a designated Slack channel. The workflow is structured into four main logical blocks:

- **1.1 Schedule Trigger:** Initiates the workflow daily at a fixed time.
- **1.2 Retrieve Contacts:** Fetches all contacts from Google Contacts including birthday information.
- **1.3 Filter Birthdays:** Filters contacts to only those whose birthday matches the current date.
- **1.4 Send Slack Notifications:** Sends personalized birthday messages to a Slack channel.

This design ensures timely and automated birthday notifications, helping teams or individuals never miss celebrating important birthdays.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block triggers the entire workflow once daily at a specified hour, ensuring the birthday check and notification process runs automatically every day.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - *Type:* Schedule Trigger node  
    - *Role:* Initiates workflow execution daily at a fixed time.  
    - *Configuration:* Set to trigger daily at 8 AM (triggerAtHour: 8).  
    - *Input:* None (trigger node).  
    - *Output:* Triggers the Google Contacts node.  
    - *Version:* 1.2  
    - *Edge Cases:*  
      - Workflow will not run if n8n instance is offline at trigger time.  
      - Timezone considerations: Ensure n8n server timezone aligns with intended schedule.  
    - *Notes:* None.

#### 2.2 Retrieve Contacts

- **Overview:**  
  Retrieves all contacts from Google Contacts, including their names, email addresses, nicknames, and birthdays, to prepare for birthday filtering.

- **Nodes Involved:**  
  - Google Contacts

- **Node Details:**  
  - **Google Contacts**  
    - *Type:* Google Contacts node  
    - *Role:* Fetches all contacts with relevant fields.  
    - *Configuration:*  
      - Operation: getAll  
      - Fields retrieved: emailAddresses, birthdays, names, nicknames  
      - Return all contacts (returnAll: true)  
    - *Input:* Triggered by Schedule Trigger node.  
    - *Output:* Passes contact list to Filter Contact node.  
    - *Version:* 1  
    - *Edge Cases:*  
      - Authentication errors if Google OAuth2 credentials expire or are invalid.  
      - Contacts without birthday data will still be retrieved but filtered later.  
      - API rate limits could cause failures if contact list is very large.  
    - *Notes:* Contains a sticky note: "Get the contact details".

#### 2.3 Filter Birthdays

- **Overview:**  
  Filters the retrieved contacts to only those who have birthday information and whose birthday matches the current date.

- **Nodes Involved:**  
  - Filter Contact  
  - If

- **Node Details:**  
  - **Filter Contact**  
    - *Type:* Filter node  
    - *Role:* Removes contacts without birthday data to optimize processing.  
    - *Configuration:*  
      - Condition: birthday field is not empty (strict string validation).  
    - *Input:* Receives all contacts from Google Contacts node.  
    - *Output:* Passes filtered contacts with birthdays to the If node.  
    - *Version:* 2.2  
    - *Edge Cases:*  
      - Contacts missing birthday data are excluded here.  
      - Expression failures if birthday field structure changes.  
    - *Notes:* None.

  - **If**  
    - *Type:* If node  
    - *Role:* Compares each contact’s birthday with the current date to identify today’s birthdays.  
    - *Configuration:*  
      - Condition: Checks if the contact’s birthday equals the current date stored in `$json.today`.  
      - Uses strict string equality.  
    - *Input:* Receives contacts with birthdays from Filter Contact node.  
    - *Output:* Contacts matching today’s birthday proceed to Slack notification.  
    - *Version:* 2.2  
    - *Edge Cases:*  
      - The `$json.today` variable must be correctly set in the workflow context or via a preceding node (not explicitly shown in the JSON).  
      - Date format mismatches between contact birthday and `$json.today` can cause false negatives.  
      - Contacts with incomplete or incorrectly formatted birthday data may be skipped.  
    - *Notes:* None.

#### 2.4 Send Slack Notifications

- **Overview:**  
  Sends a personalized birthday message to a specified Slack channel for each contact whose birthday is today.

- **Nodes Involved:**  
  - Slack

- **Node Details:**  
  - **Slack**  
    - *Type:* Slack node  
    - *Role:* Sends birthday reminder messages to Slack channel.  
    - *Configuration:*  
      - Message type: block message  
      - Text template: "Today is {{first_name}} {{last_name}}'s birthday! 🎉" (uses contact’s first and last name from JSON)  
      - Channel: Configured via channelId parameter (expects Slack channel URL to extract ID)  
      - Authentication: OAuth2 credentials for Slack API  
    - *Input:* Receives contacts filtered by If node.  
    - *Output:* None (end of workflow).  
    - *Version:* 2.3  
    - *Edge Cases:*  
      - Slack OAuth2 token expiration or invalid credentials cause message failures.  
      - Incorrect or missing channel ID leads to message delivery failure.  
      - Message formatting errors if contact name fields are missing.  
      - Rate limits on Slack API may cause delays or failures.  
    - *Notes:* Contains a sticky note: "Reminds to the birthday message".

---

### 3. Summary Table

| Node Name        | Node Type               | Functional Role                     | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                          |
|------------------|-------------------------|-----------------------------------|---------------------|---------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger  | Schedule Trigger        | Triggers workflow daily at 8 AM   | None                | Google Contacts     |                                                                                                    |
| Google Contacts   | Google Contacts         | Retrieves all contacts with birthdays | Schedule Trigger    | Filter Contact      | Get the contact details                                                                            |
| Filter Contact    | Filter                  | Filters contacts with birthday data | Google Contacts     | If                  |                                                                                                    |
| If               | If                      | Filters contacts whose birthday is today | Filter Contact      | Slack                |                                                                                                    |
| Slack            | Slack                   | Sends birthday message to Slack channel | If                  | None                | Reminds to the birthday message                                                                    |
| Sticky Note      | Sticky Note             | Workflow title                    | None                | None                | Send Daily Birthday Reminders from Google Contacts to Slack                                        |
| Sticky Note1     | Sticky Note             | Workflow description              | None                | None                | This workflow automates the process of retrieving your Google Contacts, filtering out the ones with birthdays on the current day, and sending a reminder to a designated Slack channel. By scheduling it to run daily at a specific time, the workflow ensures that you never miss a birthday reminder. Whether for team celebrations, personal reminders, or simply keeping track of important dates, this workflow can be easily customized to notify you or your team about upcoming birthdays directly in Slack. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Add a **Schedule Trigger** node.  
   - Set it to trigger daily at 8 AM (`triggerAtHour: 8`).  
   - No credentials needed.

2. **Add Google Contacts Node**  
   - Add a **Google Contacts** node.  
   - Set operation to `getAll`.  
   - Select fields: `emailAddresses`, `birthdays`, `names`, `nicknames`.  
   - Enable `Return All` to fetch all contacts.  
   - Connect Schedule Trigger output to this node’s input.  
   - Configure Google OAuth2 credentials with access to Google Contacts API.

3. **Add Filter Contact Node**  
   - Add a **Filter** node named "Filter Contact".  
   - Set condition to check that the `birthdays` field is not empty (`notEmpty`).  
   - Connect Google Contacts node output to this node’s input.

4. **Add If Node**  
   - Add an **If** node.  
   - Configure condition to compare the contact’s birthday with the current date:  
     - Left value: `={{ $('Google Contacts').item.json.birthdays }}`  
     - Operator: equals  
     - Right value: `={{ $json.today }}`  
   - Connect Filter Contact node output to this node’s input.  
   - **Important:** Ensure the workflow sets `$json.today` to the current date in the same format as birthdays before this node runs (e.g., via a Set node or Function node that formats today’s date). This is not shown in the original workflow JSON and must be added for correct operation.

5. **Add Slack Node**  
   - Add a **Slack** node.  
   - Set message type to `block`.  
   - Configure message text as:  
     `Today is {{$json["first_name"]}} {{$json["last_name"]}}'s birthday! 🎉`  
   - Set the target Slack channel by providing the channel URL or ID.  
   - Connect If node’s "true" output to Slack node input.  
   - Configure Slack OAuth2 credentials with permissions to post messages.

6. **Add Sticky Notes (Optional)**  
   - Add sticky notes for workflow title and description as per original workflow.

7. **Activate Workflow**  
   - Save and activate the workflow.  
   - Ensure all credentials are valid and tested.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow automates birthday reminders by integrating Google Contacts with Slack, running daily to keep teams informed.      | Workflow description sticky note                                                                       |
| Developed by WeblineIndia, showcasing innovative automation solutions to enhance productivity and relationships.                 | https://www.weblineindia.com/process-automation-solutions.html                                        |
| Ensure Google Contacts API and Slack API credentials are properly configured and have required scopes for reading contacts and posting messages. | Credential setup note                                                                                   |
| Date format consistency between Google Contacts birthdays and the workflow’s current date variable is critical for correct filtering. | Important integration note                                                                             |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Send Daily Birthday Reminders from Google Contacts to Slack" workflow. It highlights key configurations, potential failure points, and integration considerations to ensure robust operation.