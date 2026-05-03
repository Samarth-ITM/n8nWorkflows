Generate instant google meet links with a slack command

https://n8nworkflows.xyz/workflows/generate-instant-google-meet-links-with-a-slack-command-2156


# Generate instant google meet links with a slack command

### 1. Workflow Overview

This workflow enables users to generate instant Google Meet links directly from Slack through a custom slash command (`/meet`). Upon triggering the Slack command, it creates a short Google Calendar event with an embedded Google Meet link, posts the link back to the Slack channel, and then deletes the temporary calendar event to keep the calendar clean.

**Target Use Cases:**  
- Teams wanting quick, ephemeral Google Meet links without manually creating calendar events.  
- Slack users who prefer instant meeting links shared in their channels.  
- Automating meeting link generation integrated tightly with Google Calendar and Slack.

**Logical Blocks:**  
- **1.1 Input Reception:** Triggering via Slack slash command and receiving the webhook.  
- **1.2 Google Calendar Event Creation:** Creating a short calendar event with a Google Meet link.  
- **1.3 Slack Message Posting:** Sending the Google Meet link back to the Slack channel.  
- **1.4 Cleanup:** Deleting the temporary calendar event.  
- **1.5 Setup Instructions:** Sticky Notes explaining setup steps for Slack app, Google auth, and Slack messaging.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives the HTTP POST request from Slack when the `/meet` slash command is issued, acting as the workflow trigger.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  

| Node Name | Webhook |
|-----------|----------|
| Type | Webhook (Trigger node) |
| Configuration | HTTP POST method, path set to `slack-meet-trigger`, no response data (responds with no body), uses last node response mode to control response. |
| Key Expressions / Variables | None (trigger node) |
| Input Connections | None (trigger) |
| Output Connections | Connected to "Create event with google meet link" |
| Version Requirements | None specific |
| Edge Cases / Failures | Slack request may fail if URL is incorrect or webhook not accessible; missing or malformed Slack payload; network issues |
| Sub-workflow | None |

#### 1.2 Google Calendar Event Creation

- **Overview:**  
Creates a new Google Calendar event starting immediately, lasting 15 minutes, with Google Meet conference details enabled to generate a meeting link.

- **Nodes Involved:**  
  - Create event with google meet link

- **Node Details:**

| Node Name | Create event with google meet link |
|-----------|-----------------------------------|
| Type | Google Calendar node |
| Configuration | Creates an event with start time = now, end time = now + 15 minutes; calendar selected by user during setup; includes conferenceData with conferenceSolution = Hangouts Meet (Google Meet) |
| Key Expressions / Variables | `start = {{$now}}`, `end = {{$now.plus({minutes: 15})}}` |
| Input Connections | From Webhook node |
| Output Connections | To "Send msg with Google meet link" node |
| Version Requirements | Requires Google Calendar OAuth credentials with calendar write scope and conferenceData enabled API access |
| Edge Cases / Failures | Authentication errors, calendar permissions, API quota exceeded, conferenceData not enabled on calendar, time zone misconfigurations |
| Sub-workflow | None |

#### 1.3 Slack Message Posting

- **Overview:**  
Posts a message to the Slack channel where the slash command was issued, including the generated Google Meet link from the calendar event.

- **Nodes Involved:**  
  - Send msg with Google meet link

- **Node Details:**

| Node Name | Send msg with Google meet link |
|-----------|-------------------------------|
| Type | Slack node |
| Configuration | Sends message text: `Join me here: {{ $('Create event with google meet link').item.json.hangoutLink }}`; posts into channel ID extracted from Slack slash command payload (`channel_id`); disables link unfurling and workflow link inclusion |
| Key Expressions / Variables | Message text uses expression to fetch `hangoutLink` from previous node; channel ID from webhook JSON body field `channel_id` |
| Input Connections | From "Create event with google meet link" node |
| Output Connections | To "Delete temporary calendar event" node |
| Version Requirements | Slack credentials must have `chat:write` and `chat:write.public` scopes enabled |
| Edge Cases / Failures | Slack API rate limits, invalid or missing channel ID, auth token expiry, network errors |
| Sub-workflow | None |

#### 1.4 Cleanup

- **Overview:**  
Deletes the temporary Google Calendar event created earlier to avoid clutter, ensuring only the meeting link remains in Slack.

- **Nodes Involved:**  
  - Delete temporary calendar event

- **Node Details:**

| Node Name | Delete temporary calendar event |
|-----------|--------------------------------|
| Type | Google Calendar node |
| Configuration | Deletes event with ID obtained from the "Create event with google meet link" node's output (`id` field); uses the same calendar as event creation |
| Key Expressions / Variables | `eventId = {{ $('Create event with google meet link').item.json["id"]}}` |
| Input Connections | From "Send msg with Google meet link" node |
| Output Connections | None (end node) |
| Version Requirements | Same Google Calendar OAuth credentials as event creation, with delete permission |
| Edge Cases / Failures | Event already deleted, permission issues, network errors, API quota limits |
| Sub-workflow | None |

#### 1.5 Setup Instructions (Sticky Notes)

- **Overview:**  
Sticky Notes provide detailed guidance on setting up Slack app, Google credentials, and Slack message customization.

- **Nodes Involved:**  
  - Sticky Note (Slack App Setup)  
  - Sticky Note1 (Google Auth & Calendar Setup)  
  - Sticky Note2 (Slack Node Authentication & Message Setup)  
  - Sticky Note3 (Google Calendar Account Selection)  
  - Sticky Note4 (Workflow description and purpose)

- **Node Details:**

| Node Name | Sticky Note | Purpose |
|-----------|-------------|---------|
| Sticky Note | Explains Slack app creation, required OAuth scopes, slash command creation, and app installation. Includes URL to Slack API docs. | Slack app setup |
| Sticky Note1 | Guides on setting up Google OAuth credentials and selecting calendar for Google Meet events. Links to official n8n docs for Google OAuth. | Google auth setup |
| Sticky Note2 | Describes configuring Slack node authentication and customizing Slack message, emphasizing use of hangoutLink expression. | Slack message configuration |
| Sticky Note3 | Reminds to select the same Google calendar account in Slack node for consistency. | Google calendar account selection |
| Sticky Note4 | Provides a high-level description of workflow purpose and usage. | Workflow overview |

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                              | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                          |
|-----------------------------------|---------------------|----------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook                           | Webhook             | Receive Slack slash command trigger          | None                             | Create event with google meet link |                                                                                                    |
| Create event with google meet link| Google Calendar     | Create short calendar event with Meet link   | Webhook                         | Send msg with Google meet link   |                                                                                                    |
| Send msg with Google meet link    | Slack               | Post Google Meet link back into Slack channel| Create event with google meet link| Delete temporary calendar event  |                                                                                                    |
| Delete temporary calendar event   | Google Calendar     | Delete temporary calendar event               | Send msg with Google meet link  | None                            |                                                                                                    |
| Sticky Note                       | Sticky Note         | Slack app setup instructions                   | None                             | None                            | "### 1. Setup: Add a Slack App..." (Slack app creation and slash command setup)                     |
| Sticky Note1                      | Sticky Note         | Google auth and calendar setup instructions   | None                             | None                            | "### 2. Setup: Google auth & calendar..." (Google OAuth credential setup)                           |
| Sticky Note2                      | Sticky Note         | Slack node auth and message customization     | None                             | None                            | "### 3. Setup: Configure slack node authentication and your message..."                            |
| Sticky Note3                      | Sticky Note         | Google calendar account selection instructions| None                             | None                            | "### 3. Setup: Select google calendar account..."                                                  |
| Sticky Note4                      | Sticky Note         | Workflow overview and description              | None                             | None                            | "## Generate google meet links with a slack command\nSpin up instant google meet links..."         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a new Webhook node named `Webhook`.  
   - Set HTTP Method to `POST`.  
   - Set Path to `slack-meet-trigger`.  
   - Set Response Data to `No Data`.  
   - Set Response Mode to `Last Node`.  
   - This node will be the starting trigger for the workflow.

2. **Create Google Calendar Node for Event Creation**  
   - Add a Google Calendar node named `Create event with google meet link`.  
   - Set Operation to `Create`.  
   - Set Start to `={{ $now }}` (current datetime).  
   - Set End to `={{ $now.plus({minutes:15}) }}` (15 minutes after start).  
   - Select the Google Calendar account and the calendar to create events on.  
   - Under Additional Fields, enable `conferenceData`. Set `conferenceSolution` to `hangoutsMeet` to generate Google Meet links.  
   - Connect the output of `Webhook` node to this node.

3. **Create Slack Node to Send Message**  
   - Add a Slack node named `Send msg with Google meet link`.  
   - Select the Slack credentials with bot token scopes `chat:write` and `chat:write.public`.  
   - Set Operation to `Post Message`.  
   - Set Channel to `={{ $('Webhook').item.json.body.channel_id }}` to post in the channel where the command was triggered.  
   - Set Text to `=Join me here: {{ $('Create event with google meet link').item.json.hangoutLink }}` to include the Google Meet link in the message.  
   - Disable `Unfurl Links` and `Include Link To Workflow` options for cleaner messages.  
   - Connect the output of `Create event with google meet link` node to this node.

4. **Create Google Calendar Node for Event Deletion**  
   - Add another Google Calendar node named `Delete temporary calendar event`.  
   - Set Operation to `Delete`.  
   - Use the same Google Calendar credentials and calendar as the event creation node.  
   - Set Event ID to `={{ $('Create event with google meet link').item.json["id"] }}` to delete the event just created.  
   - Connect the output of `Send msg with Google meet link` node to this node.

5. **Add Sticky Notes for Setup Instructions (Optional but Recommended)**  
   - Add multiple Sticky Note nodes with detailed instructions for:  
     - Slack app creation and slash command setup (including required scopes and URLs).  
     - Google OAuth credential setup for Google Calendar access.  
     - Slack node authentication and message customization notes.  
     - Reminder to select the same Google calendar on all relevant nodes.  
     - Workflow purpose and overview.

6. **Credentials Setup**  
   - Slack: Create a Slack app with bot token scopes `chat:write` and `chat:write.public`. Create the `/meet` slash command and set the Request URL to the Webhook URL generated by the `Webhook` node. Install the app to your workspace.  
   - Google: Setup OAuth credentials for Google Calendar with scopes that allow creating and deleting events and accessing conferenceData (Google Meet). Follow n8n docs for OAuth single-service setup.

7. **Workflow Activation**  
   - Activate the workflow.  
   - Use `/meet` command in Slack to test instant Google Meet link generation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Slack app creation steps including adding `chat:write` and `chat:write.public` scopes, creating slash command `/meet`, and installing the app. | https://api.slack.com/apps (referenced in Sticky Note) |
| Google OAuth credential setup instructions for Google Calendar with conferenceData API enabled. | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/ (linked in Sticky Note1) |
| This workflow deletes the temporary Google Calendar event after posting the Google Meet link, ensuring no clutter on calendar. | Workflow design detail |
| Slack message customization uses expression to dynamically insert the Google Meet link from calendar event output. | Slack node configuration detail |
| The Slack slash command payload is used to extract the channel ID to post the message back to the correct channel. | Webhook node and Slack node integration detail |
| Workflow enables instant meetings without manual calendar event creation, improving team productivity and seamless integration. | Workflow purpose overview |

---

This documentation fully captures the workflow's structure, logic, and setup requirements, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.