Automated Slack Channel Inactivity Audit & Report Generator for Workspace Cleanup

https://n8nworkflows.xyz/workflows/automated-slack-channel-inactivity-audit---report-generator-for-workspace-cleanup-7728


# Automated Slack Channel Inactivity Audit & Report Generator for Workspace Cleanup

### 1. Workflow Overview

This workflow automates the auditing of Slack channels within a workspace to identify inactive channels and generate a weekly inactivity report. It targets Slack workspace administrators, IT/operations managers, and HR/compliance teams who wish to maintain a clean, relevant, and efficient communication environment by reviewing channels with no recent activity.

The workflow logic is divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow on a recurring weekly schedule.
- **1.2 Slack Channel Retrieval:** Fetches all public and private Slack channels from the workspace.
- **1.3 Channel Activity Check:** Retrieves the most recent message history for each channel.
- **1.4 Inactivity Filtering:** Filters out channels with no messages in the last 30 days.
- **1.5 Metadata Collection:** Extracts key metadata from inactive channels for reporting.
- **1.6 Report Generation:** Creates a formatted Slack-friendly inactivity report with recommendations.
- **1.7 Report Dispatch:** Sends the generated report to a specified Slack user/channel for administrative action.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Starts the entire workflow automatically on a weekly basis, eliminating manual initiation.

- **Nodes Involved:**  
  - Weekly Schedule Trigger

- **Node Details:**  
  - **Node:** Weekly Schedule Trigger  
  - **Type:** Schedule Trigger  
  - **Configuration:** Set to trigger every week on Monday (day 1). This can be customized to other days or frequencies.  
  - **Input/Output:** No input; output triggers downstream nodes.  
  - **Potential Failures:** None typical; ensure n8n server uptime for scheduled execution.  
  - **Sticky Note:** Explains the node’s role to automate weekly review without manual intervention.

#### 1.2 Slack Channel Retrieval

- **Overview:**  
  Retrieves a comprehensive list of all public and private channels in the Slack workspace using Slack API with OAuth2 credentials.

- **Nodes Involved:**  
  - Get many channels

- **Node Details:**  
  - **Node:** Get many channels  
  - **Type:** Slack node (API integration)  
  - **Configuration:** Operation `getAll` on resource `channel`; filters set to include `public_channel` and `private_channel`. Uses OAuth2 authentication via configured Slack credentials.  
  - **Input/Output:** Triggered by schedule; outputs channel list items for each channel.  
  - **Potential Failures:** OAuth token expiration; insufficient API scopes (`channels:read` required); rate limiting by Slack API.  
  - **Sticky Note:** Describes the node fetching all public Slack channels.

#### 1.3 Channel Activity Check

- **Overview:**  
  For each channel retrieved, fetches the most recent message (limit 1) to determine last activity timestamp.

- **Nodes Involved:**  
  - Get the history of a channel

- **Node Details:**  
  - **Node:** Get the history of a channel  
  - **Type:** Slack node  
  - **Configuration:** Operation `history` on resource `channel` with `limit` set to 1 to get the latest message only. Channel ID dynamically set from the previous node's output. Uses OAuth2 credentials.  
  - **Input/Output:** Receives channel ID from "Get many channels"; outputs last message timestamp per channel.  
  - **Potential Failures:** Slack API rate limits; empty channel history (no messages ever); OAuth token issues.  
  - **Sticky Note:** Notes this node retrieves recent message history to assess activity.

#### 1.4 Inactivity Filtering

- **Overview:**  
  Filters channels that have had no messages in the past 30 days, identifying them as inactive or expired.

- **Nodes Involved:**  
  - Filter channel with last discussion 30 days ago

- **Node Details:**  
  - **Node:** Filter channel with last discussion 30 days ago  
  - **Type:** Filter node  
  - **Configuration:** Compares last message timestamp converted to datetime against current date minus 30 days, passing only those channels with last activity before this threshold.  
  - **Input/Output:** Input from "Get the history of a channel"; outputs only inactive channels.  
  - **Potential Failures:** Incorrect date parsing if Slack timestamp format changes; timezone mismatches; channels with no messages (may lack timestamp).  
  - **Sticky Note:** Explains filtering of channels with no activity in last 30 days.

#### 1.5 Metadata Collection

- **Overview:**  
  Extracts relevant metadata from each inactive channel to prepare data for reporting, including ID, name, creation date, member count, and purpose.

- **Nodes Involved:**  
  - Collect expired channel information

- **Node Details:**  
  - **Node:** Collect expired channel information  
  - **Type:** Code node (JavaScript)  
  - **Configuration:** Runs once per channel item, referencing channel data from the "Get many channels" node to extract and format metadata fields such as:  
    - ChannelID  
    - ChannelName  
    - CreatedDate (converted from Unix timestamp)  
    - TotalMember count  
    - Purpose (defaulting to "N/A" if empty)  
  - **Input/Output:** Input from filtered inactive channels; outputs structured channel metadata objects.  
  - **Potential Failures:** Missing or malformed JSON data references; empty or null fields; unexpected data types.  
  - **Sticky Note:** Details the metadata fields extracted for reporting.

#### 1.6 Report Generation

- **Overview:**  
  Aggregates all inactive channel metadata into a human-readable Slack Markdown report, including a table-like structure and recommendations.

- **Nodes Involved:**  
  - Consume slack report

- **Node Details:**  
  - **Node:** Consume slack report  
  - **Type:** Code node (JavaScript)  
  - **Configuration:**  
    - Collects all incoming items (inactive channels).  
    - Builds a multi-line string report with Markdown syntax: header, table with columns (Channel ID, Name, Members, Created On, Purpose), and recommendations for archiving or engagement.  
    - Adds a timestamp for report generation date.  
    - Outputs JSON with a `text` field containing the report.  
  - **Input/Output:** Input from "Collect expired channel information"; outputs formatted report text.  
  - **Potential Failures:** Formatting errors if data is missing; large channel lists may cause message size issues in Slack; character encoding issues.  
  - **Sticky Note:** Describes report generation and formatting steps.

#### 1.7 Report Dispatch

- **Overview:**  
  Sends the generated inactivity report as a message to a specified Slack user or channel, alerting workspace admins for cleanup actions.

- **Nodes Involved:**  
  - Send Channel Inactivity Report

- **Node Details:**  
  - **Node:** Send Channel Inactivity Report  
  - **Type:** Slack node  
  - **Configuration:** Posts message with text from previous node to a Slack user (username `@trung.tran`) or a channel. Uses Slack API credentials with `chat:write` scope. Message options enable Markdown formatting.  
  - **Input/Output:** Input is the report text JSON; output is Slack API response (not used downstream).  
  - **Potential Failures:** Invalid or missing Slack credentials; user/channel not found; message size limits exceeded; API rate limits.  
  - **Sticky Note:** Explains message dispatch to notify admins.

---

### 3. Summary Table

| Node Name                         | Node Type              | Functional Role                              | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                                                  |
|----------------------------------|------------------------|----------------------------------------------|-------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Weekly Schedule Trigger           | Schedule Trigger       | Starts workflow weekly on scheduled interval | None                          | Get many channels                | Starts the workflow on a recurring schedule (e.g., every Monday at 9 AM).                                                    |
| Get many channels                | Slack Node              | Fetches all public/private Slack channels    | Weekly Schedule Trigger        | Get the history of a channel     | Fetches a list of all public Slack channels in the workspace using Slack’s `channels:read` scope.                            |
| Get the history of a channel      | Slack Node              | Gets latest message of each channel           | Get many channels              | Filter channel with last discussion 30 days ago | Retrieves recent message history from each channel using Slack’s `channels:history` scope.                                   |
| Filter channel with last discussion 30 days ago | Filter Node          | Filters channels with no activity in 30 days | Get the history of a channel   | Collect expired channel information | Filters out channels that have had no messages or discussion in the last 30 days.                                             |
| Collect expired channel information | Code Node              | Extracts metadata for inactive channels       | Filter channel with last discussion 30 days ago | Consume slack report            | Extracts relevant metadata for each inactive channel, including: Channel ID, name, member count, created date, purpose.      |
| Consume slack report              | Code Node              | Generates formatted Slack inactivity report   | Collect expired channel information | Send Channel Inactivity Report  | Generates a clean and Slack-friendly Markdown report listing all inactive channels with contextual recommendations.          |
| Send Channel Inactivity Report    | Slack Node              | Sends inactivity report to Slack user/channel | Consume slack report           | None                           | Posts the formatted report to a specified Slack channel using `chat:write`, notifying admins or moderators to take action.   |
| Sticky Note                      | Sticky Note Node        | Visual documentation/image                     | None                          | None                           | ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-22+at+4.04.52%E2%80%AFPM.png)                    |
| Sticky Note1                     | Sticky Note Node        | Full workflow description and instructions    | None                          | None                           | Detailed workflow overview, setup instructions, customization options, and Slack permissions reference.                      |
| Sticky Note2                     | Sticky Note Node        | Description of Schedule Trigger node           | None                          | None                           | Starts the workflow on a recurring schedule (e.g., every Monday at 9 AM) to automate the review process without manual intervention. |
| Sticky Note3                     | Sticky Note Node        | Description of Get many channels node          | None                          | None                           | Fetches a list of all public Slack channels in the workspace using Slack’s `channels:read` scope.                            |
| Sticky Note4                     | Sticky Note Node        | Description of Get the history of a channel    | None                          | None                           | Retrieves recent message history from each channel using Slack’s `channels:history` scope.                                   |
| Sticky Note5                     | Sticky Note Node        | Description of Filter inactive channels node   | None                          | None                           | Filters out channels that have had no messages or discussion in the last 30 days.                                             |
| Sticky Note6                     | Sticky Note Node        | Description of Collect expired channel info    | None                          | None                           | Extracts relevant metadata for each inactive channel, including ID, name, member count, created date, purpose.                |
| Sticky Note7                     | Sticky Note Node        | Description of report generation node           | None                          | None                           | Generates a clean and Slack-friendly Markdown report listing all inactive channels with contextual recommendations.          |
| Sticky Note8                     | Sticky Note Node        | Description of report dispatch node             | None                          | None                           | Posts the formatted report to a specified Slack channel using `chat:write`, notifying admins or moderators to take cleanup action. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Weekly Schedule Trigger`  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger every week on Monday (day 1). Adjust time if needed.

3. **Add a Slack node to get all channels:**  
   - Name: `Get many channels`  
   - Type: Slack  
   - Credentials: Connect with Slack OAuth2 credentials that have `channels:read` scope.  
   - Parameters:  
     - Resource: Channel  
     - Operation: Get All  
     - Filters: Include both `public_channel` and `private_channel` types.  
   - Connect `Weekly Schedule Trigger` output to this node input.

4. **Add a Slack node to get message history:**  
   - Name: `Get the history of a channel`  
   - Type: Slack  
   - Credentials: Same Slack OAuth2 credentials.  
   - Parameters:  
     - Resource: Channel  
     - Operation: History  
     - Channel ID: Set dynamically from `Get many channels` node using expression `={{ $json.id }}`  
     - Limit: 1 (to get the latest message only)  
   - Connect `Get many channels` output to this node input.

5. **Add a Filter node to filter inactive channels:**  
   - Name: `Filter channel with last discussion 30 days ago`  
   - Type: Filter  
   - Parameters:  
     - Condition: Only pass items where the timestamp of the last message is before today minus 30 days.  
     - Expression:  
       - Left value: `={{ $json.ts.toDateTime('s') }}` (convert Slack timestamp to datetime)  
       - Operator: Before  
       - Right value: `={{ $today.minus(30,'days') }}`  
   - Connect `Get the history of a channel` output to this node input.

6. **Add a Code node to collect metadata:**  
   - Name: `Collect expired channel information`  
   - Type: Code  
   - Parameters:  
     - Mode: Run Once For Each Item  
     - Code (JavaScript):  
       ```javascript
       const channel = $('Get many channels').item.json;

       return {
         json: {
           ChannelID: channel.id,
           ChannelName: channel.name,
           CreatedDate: new Date(channel.created * 1000).toLocaleDateString('en-US'),
           TotalMember: channel.num_members,
           Purpose: channel.purpose.value || 'N/A'
         }
       };
       ```  
   - Connect `Filter channel with last discussion 30 days ago` output to this node input.

7. **Add a Code node to generate the report:**  
   - Name: `Consume slack report`  
   - Type: Code  
   - Parameters:  
     - Code (JavaScript):  
       ```javascript
       const channels = $input.all().map(item => item.json);

       let report = '*📊 Weekly Slack Channel Inactivity Report*\n\n';
       report += 'The following channels have had *no discussion in the past 30 days*. Please review and take action if needed:\n\n';
       report += '```';
       report += 'Channel ID       Name                        Members  Created On   Purpose\n';
       report += '-------------   --------------------------  -------   ----------   ------------------------------\n';

       for (const c of channels) {
         report += `${c.ChannelID.padEnd(15)} ${('#' + c.ChannelName).padEnd(26)} ${String(c.TotalMember).padEnd(9)} ${c.CreatedDate.padEnd(12)} ${c.Purpose || 'N/A'}\n`;
       }

       report += '```\n\n';

       report += '\n⚠️ *Recommendation:*\n';
       report += '- Channels with *0 members* or *no defined purpose* should be reviewed for archiving.\n';
       report += '- Channels like `#aws-study-group` and `#team-coffee` may benefit from engagement prompts or repurposing.\n\n';

       report += `📅 _Report generated on:_ ${new Date().toLocaleDateString('en-GB')}`;

       return [
         {
           json: {
             text: report
           }
         }
       ];
       ```  
   - Connect `Collect expired channel information` output to this node input.

8. **Add a Slack node to send the report:**  
   - Name: `Send Channel Inactivity Report`  
   - Type: Slack  
   - Credentials: Use Slack API credentials with `chat:write` scope (can be Bot User OAuth Token).  
   - Parameters:  
     - Text: Set dynamically from previous node with expression `={{ $json.text }}`  
     - User: Set to Slack username or channel where report is sent, e.g., `@trung.tran` or a dedicated channel ID  
     - Message options: Enable Markdown formatting (mrkdwn)  
   - Connect `Consume slack report` output to this node input.

9. **Test the workflow:**  
   - Run manually or wait for scheduled trigger to verify channel retrieval, filtering, report generation, and message posting.

10. **Credential setup:**  
    - Create Slack credentials in n8n with OAuth2 authentication.  
    - Ensure the Slack App has the following OAuth scopes:  
      - `channels:read`  
      - `channels:history`  
      - `chat:write`  
    - Install the app into the Slack workspace and provide the Bot User OAuth Token to n8n credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Full workflow overview, setup instructions, customization options, and Slack permissions reference are included in an attached sticky note with image and detailed text. | See “Sticky Note1” content for comprehensive workflow documentation and setup guidance.                              |
| Slack permission scopes required: `channels:read`, `channels:history`, `chat:write` (optional: `users:read` for personalization). | Slack API OAuth scopes documentation: https://api.slack.com/scopes                                                       |
| Recommended to customize schedule frequency and inactivity threshold by adjusting the Schedule Trigger node and Filter node respectively. | Workflow flexibility allows adapting to different workspace needs and policies.                                      |
| The workflow can be extended to include auto-archiving or engagement messages based on the report findings.                      | Enhancements may require additional Slack API permissions and nodes.                                                  |
| An example screenshot of the workflow UI is linked in a sticky note for reference.                                                | ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-22+at+4.04.52%E2%80%AFPM.png)             |

---

**Disclaimer:** The text and workflow described originate exclusively from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or offensive material. All handled data is legal and public.