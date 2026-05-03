Discord Channel Creation from Google Sheets with Member Notifications

https://n8nworkflows.xyz/workflows/discord-channel-creation-from-google-sheets-with-member-notifications-4719


# Discord Channel Creation from Google Sheets with Member Notifications

### 1. Workflow Overview

This workflow automates the creation of Discord channels for new projects tracked in a Google Sheet and sends notifications to project team members on Discord. It is designed for project onboarding use cases where each new project entry in a Google Sheet triggers channel creation, updates to the sheet, and a series of announcement messages in Discord.

The workflow is logically divided into these blocks:

- **1.1 Monitor New Project Entries:** Watches the Google Sheet for new rows representing new projects.
- **1.2 Filter Valid Project Entries:** Ensures only projects without existing Discord channels and with valid timestamps proceed.
- **1.3 Create Discord Channel:** Creates a new Discord channel named after the Project ID in a specified category.
- **1.4 Update Google Sheet with Channel Info:** Writes back the created Discord channel ID and updates the project status.
- **1.5 Send Project Announcement Messages:** Sends a sequence of formatted messages to the new Discord channel with project details.
- **1.6 Mark Onboarding Complete:** Updates the Google Sheet to mark the onboarding as fully complete after messages are sent.

---

### 2. Block-by-Block Analysis

#### 2.1 Monitor New Project Entries

- **Overview:**  
  This block triggers the workflow when a new project is added to the Google Sheet's "Project Onboarding" tab.

- **Nodes Involved:**  
  - Monitor New Project Entries (Google Sheets Trigger)

- **Node Details:**

  - **Monitor New Project Entries**  
    - Type: Google Sheets Trigger  
    - Configuration: Set to trigger on "rowAdded" event on a specific Google Sheet and tab named "Project Onboarding". Uses OAuth2 credentials for Google Sheets access.  
    - Key Expressions: None; triggers automatically on new row addition.  
    - Input Connections: None (trigger node)  
    - Output Connections: Connects to "Filter Valid Project Entries"  
    - Edge Cases:  
      - Google Sheets API rate limits or authorization errors.  
      - New rows without required data or malformed rows could trigger unnecessary workflow runs.  
    - Version Requirements: n8n Google Sheets Trigger node, version 1 or higher.

---

#### 2.2 Filter Valid Project Entries

- **Overview:**  
  Filters incoming new project rows to process only those without an existing Discord channel (empty "Discord ID") and that have a valid timestamp.

- **Nodes Involved:**  
  - Filter Valid Project Entries (Filter node)

- **Node Details:**

  - **Filter Valid Project Entries**  
    - Type: Filter  
    - Configuration:  
      - Condition 1: "Discord ID" field is empty (meaning no Discord channel created yet).  
      - Condition 2: "Timestamp" field is not empty (valid timestamp).  
      - Combines conditions with AND logic.  
    - Key Expressions:  
      - `{{$json['Discord ID']}}` empty check  
      - `{{$json.Timestamp}}` not empty check  
    - Input Connections: From "Monitor New Project Entries"  
    - Output Connections: To "Create Discord Channel"  
    - Edge Cases:  
      - Missing or malformed fields in the sheet rows may cause unexpected filtering results.  
      - Timestamp field format should be consistent to avoid false negatives.  
    - Version: Requires Filter node version supporting expression evaluation (v2.2 or later).

---

#### 2.3 Create Discord Channel

- **Overview:**  
  Creates a new Discord channel named after the Project ID in a designated category within the specified Discord guild/server.

- **Nodes Involved:**  
  - Create Discord Channel (Discord node)

- **Node Details:**

  - **Create Discord Channel**  
    - Type: Discord node (Channel creation)  
    - Configuration:  
      - Channel name set dynamically to the Project ID (`{{$json['Project ID']}}`).  
      - Guild ID and Category ID configured via credential-linked dropdown lists.  
      - Uses Discord Bot API credentials for permissions.  
    - Key Expressions:  
      - Channel name expression: `={{ $json['Project ID'] }}`  
    - Input Connections: From "Filter Valid Project Entries"  
    - Output Connections: To "Update Sheet with Discord Channel ID"  
    - Edge Cases:  
      - Discord API rate limits or permission errors (bot must have channel creation rights).  
      - Invalid Guild or Category IDs cause failure.  
      - Duplicate channel names handled by Discord server rules (may cause errors).  
    - Version: Discord node v2 or higher to support channel creation.

---

#### 2.4 Update Google Sheet with Channel Info

- **Overview:**  
  Updates the Google Sheet row for the project with the newly created Discord channel ID and marks the status as "Discord Created".

- **Nodes Involved:**  
  - Update Sheet with Discord Channel ID (Google Sheets node)

- **Node Details:**

  - **Update Sheet with Discord Channel ID**  
    - Type: Google Sheets (Write/Update)  
    - Configuration:  
      - Updates columns "Domain ID" (matching column), "Discord ID" (set to new channel ID), and "Discord Server Creation" (set to "Discord Created").  
      - Uses "Domain ID" as the matching key for the update operation.  
      - Target sheet and document specified with OAuth2 credentials.  
    - Key Expressions:  
      - `"Domain ID":"={{ $('Filter Valid Project Entries').item.json['Domain ID'] }}"`  
      - `"Discord ID":"={{ $json.id }}"` (Discord channel ID from previous node)  
      - `"Discord Server Creation":"Discord Created"`  
    - Input Connections: From "Create Discord Channel"  
    - Output Connections: To "Check Message Sending Status"  
    - Edge Cases:  
      - Update failures due to mismatched Domain ID or sheet access issues.  
      - Rate limits or credential expiration.  
    - Version: Google Sheets node v4.5 or higher recommended.

---

#### 2.5 Send Project Announcement Messages

- **Overview:**  
  Sends two consecutive messages to the newly created Discord channel: an announcement with main project details and a follow-up with additional information.

- **Nodes Involved:**  
  - Check Message Sending Status (Filter node)  
  - Send Project Announcement Message (Discord node)  
  - Send Additional Project Details (Discord node)

- **Node Details:**

  - **Check Message Sending Status**  
    - Type: Filter  
    - Configuration:  
      - Ensures timestamp is present and "Discord Server Creation" status equals "Discord Created".  
      - Acts as a gate to allow messages to be sent only after channel creation is confirmed.  
    - Key Expressions:  
      - `{{$('Monitor New Project Entries').item.json.Timestamp}}` not empty  
      - Status equals "Discord Created"  
    - Input Connections: From "Update Sheet with Discord Channel ID"  
    - Output Connections: To "Send Project Announcement Message"  
    - Edge Cases: Incorrect status or missing timestamp blocks message sending.

  - **Send Project Announcement Message**  
    - Type: Discord (Message send)  
    - Configuration:  
      - Sends a formatted message tagging `@everyone` and `@ProjectManager` with key project fields like Project ID, Product Type, Client Name, goals, immediate tasks, and commitments.  
      - Message content uses mustache expressions referencing filtered project row data.  
      - Sends message to the Discord channel ID updated in the sheet.  
    - Key Expressions:  
      - Dynamic content built from fields such as `{{$('Filter Valid Project Entries').item.json['Project ID']}}` and others.  
      - Channel ID from `{{$('Update Sheet with Discord Channel ID').item.json['Discord ID']}}`  
    - Input Connections: From "Check Message Sending Status"  
    - Output Connections: To "Send Additional Project Details"  
    - Edge Cases:  
      - Discord API message sending failures (rate limits, permission issues).  
      - Invalid channel ID reference leads to failure.  

  - **Send Additional Project Details**  
    - Type: Discord (Message send)  
    - Configuration:  
      - Sends a second message with additional project information including access credentials, competitors, call recording, and communication methods.  
      - Uses expressions referencing project sheet data similar to the announcement message.  
      - Sends to the same Discord channel ID (`{{$json.channel_id}}` from previous message output).  
    - Input Connections: From "Send Project Announcement Message"  
    - Output Connections: To "Mark Onboarding Complete"  
    - Edge Cases: Same as above, plus dependency on successful previous message send.

---

#### 2.6 Mark Onboarding Complete

- **Overview:**  
  Marks the onboarding process as complete by updating the Google Sheet to reflect that onboarding messages have been sent.

- **Nodes Involved:**  
  - Mark Onboarding Complete (Google Sheets node)

- **Node Details:**

  - **Mark Onboarding Complete**  
    - Type: Google Sheets (Update)  
    - Configuration:  
      - Updates the "Discord Server Creation" column to "Discord Created, Onboarding Message Sent".  
      - Uses "Discord ID" as the matching column.  
      - Authenticated with Google Sheets OAuth2 credentials.  
    - Key Expressions:  
      - `"Discord ID":"={{ $('Check Message Sending Status').item.json['Discord ID'] }}"`  
      - `"Discord Server Creation":"Discord Created, Onboarding Message Sent"`  
    - Input Connections: From "Send Additional Project Details"  
    - Output Connections: None (workflow end)  
    - Edge Cases: Failed update due to mismatched Discord ID or sheet access.  
    - Version: Google Sheets node v4.6 or higher.

---

### 3. Summary Table

| Node Name                         | Node Type          | Functional Role                              | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                               |
|----------------------------------|--------------------|----------------------------------------------|------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------|
| Monitor New Project Entries       | Google Sheets Trigger | Trigger on new project row addition          | None                         | Filter Valid Project Entries     | ## Monitor New Project Entries<br>Monitors Google Sheets for new project entries and initiates workflow.  |
| Filter Valid Project Entries      | Filter             | Filter projects without Discord ID and valid timestamp | Monitor New Project Entries   | Create Discord Channel           | ## Filter Valid Project Entries<br>Filters projects without Discord channels and valid timestamp.          |
| Create Discord Channel            | Discord            | Create new Discord channel named after Project ID | Filter Valid Project Entries  | Update Sheet with Discord Channel ID | ## Create New Discord Channel<br>Creates Discord channel in specified category.                            |
| Update Sheet with Discord Channel ID | Google Sheets      | Update sheet with Discord channel ID and status | Create Discord Channel        | Check Message Sending Status     | ## Update Google Sheets with Channel Info<br>Updates sheet with Discord channel ID and status.            |
| Check Message Sending Status      | Filter             | Verify project timestamp and Discord creation status | Update Sheet with Discord Channel ID | Send Project Announcement Message |                                                                                                           |
| Send Project Announcement Message| Discord            | Send main project announcement message       | Check Message Sending Status  | Send Additional Project Details  | ## Send Project Announcement<br>Sends formatted message announcing new project and team assignments.      |
| Send Additional Project Details   | Discord            | Send supplementary project details message    | Send Project Announcement Message | Mark Onboarding Complete         |                                                                                                           |
| Mark Onboarding Complete          | Google Sheets      | Mark onboarding as complete in sheet          | Send Additional Project Details | None                           | ## Complete Onboarding Process<br>Marks onboarding complete after messages sent.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Name: Monitor New Project Entries  
   - Type: Google Sheets Trigger  
   - Event: rowAdded  
   - Sheet Name: "Project Onboarding" (or your actual sheet tab)  
   - Document URL: Your Google Sheet URL  
   - Credentials: OAuth2 for Google Sheets  
   - Configure polling as needed (e.g., every X seconds/minutes).

2. **Add Filter Node**  
   - Name: Filter Valid Project Entries  
   - Type: Filter  
   - Conditions (AND):  
     - `Discord ID` field is empty  
     - `Timestamp` field is not empty  
   - Connect input from "Monitor New Project Entries".

3. **Add Discord Node for Channel Creation**  
   - Name: Create Discord Channel  
   - Type: Discord node (channel create)  
   - Channel Name: Expression `{{$json['Project ID']}}`  
   - Guild ID: Select your Discord server via credential dropdown  
   - Category ID: Select category for new projects  
   - Credentials: Discord Bot API with channel creation permission  
   - Connect input from "Filter Valid Project Entries".

4. **Add Google Sheets Node to Update Channel Info**  
   - Name: Update Sheet with Discord Channel ID  
   - Type: Google Sheets (update)  
   - Sheet Name: Same as in trigger  
   - Document URL: Same as trigger  
   - Matching Columns: Domain ID  
   - Columns to update:  
     - Domain ID: `={{ $('Filter Valid Project Entries').item.json['Domain ID'] }}`  
     - Discord ID: `={{ $json.id }}` (Discord channel ID from previous node)  
     - Discord Server Creation: "Discord Created"  
   - Credentials: OAuth2 for Google Sheets  
   - Connect input from "Create Discord Channel".

5. **Add Filter Node to Check Message Sending Status**  
   - Name: Check Message Sending Status  
   - Type: Filter  
   - Conditions (AND):  
     - Timestamp (from trigger node) is not empty  
     - Discord Server Creation equals "Discord Created"  
   - Connect input from "Update Sheet with Discord Channel ID".

6. **Add Discord Node to Send Project Announcement Message**  
   - Name: Send Project Announcement Message  
   - Type: Discord (send message)  
   - Guild ID: Select your Discord server  
   - Channel ID: Expression `={{ $('Update Sheet with Discord Channel ID').item.json['Discord ID'] }}`  
   - Content: Custom formatted message using expressions referencing filtered project data, including tags @everyone and @ProjectManager and project fields (Project ID, Product Type, Client Name, goals, immediate tasks, commitments).  
   - Credentials: Discord Bot API  
   - Connect input from "Check Message Sending Status".

7. **Add Discord Node to Send Additional Project Details**  
   - Name: Send Additional Project Details  
   - Type: Discord (send message)  
   - Guild ID: Same as above  
   - Channel ID: Use previous message channel ID (`{{$json.channel_id}}`)  
   - Content: Additional project information with expressions for access info, competitors, call recording, communication method.  
   - Credentials: Discord Bot API  
   - Connect input from "Send Project Announcement Message".

8. **Add Google Sheets Node to Mark Onboarding Complete**  
   - Name: Mark Onboarding Complete  
   - Type: Google Sheets (update)  
   - Sheet Name and Document URL: Same as above  
   - Matching Column: Discord ID  
   - Columns to update:  
     - Discord Server Creation: "Discord Created, Onboarding Message Sent"  
   - Credentials: OAuth2 for Google Sheets  
   - Connect input from "Send Additional Project Details".

9. **Review and Test Workflow**  
   - Validate credential permissions for Google Sheets and Discord Bot.  
   - Test with a sample new project row in Google Sheets.  
   - Monitor logs for errors and adjust filter conditions or expressions if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses dynamic expressions heavily; ensure that column names in Google Sheets exactly match those used here.| Critical for expression accuracy and filtering logic.                                           |
| Discord Bot must have permissions to create channels and send messages in the specified guild and category.            | Discord Bot API credential setup requirement.                                                  |
| Google Sheets OAuth2 credentials must have edit access to the target spreadsheet.                                       | Google Sheets API permissions.                                                                 |
| Sticky notes in the workflow provide step explanations, useful for onboarding new users on this workflow logic.        | Visible in n8n editor for easier maintenance.                                                  |
| For more information on Discord Bot permissions and channel creation, see Discord Developer Portal: https://discord.com/developers | External documentation link for Discord API setup.                                             |
| Google Sheets Trigger polling interval should be configured to balance latency and API usage.                          | Consider rate limits and workflow responsiveness.                                             |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. It adheres strictly to content policies and contains no illegal or offensive material. All processed data is legal and publicly available.