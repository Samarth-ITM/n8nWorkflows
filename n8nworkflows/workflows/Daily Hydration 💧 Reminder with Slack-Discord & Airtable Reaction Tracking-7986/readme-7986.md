Daily Hydration 💧 Reminder with Slack/Discord & Airtable Reaction Tracking

https://n8nworkflows.xyz/workflows/daily-hydration----reminder-with-slack-discord---airtable-reaction-tracking-7986


# Daily Hydration 💧 Reminder with Slack/Discord & Airtable Reaction Tracking

### 1. Workflow Overview

This workflow, titled **"Daily Hydration Hug 💧"**, automates friendly hydration reminders twice daily via Slack and/or Discord, tracks user engagement through reaction emojis, and logs participation data to Airtable for ongoing analytics such as leaderboards. It targets wellness communities or teams aiming to encourage regular hydration habits with gentle nudges and community interaction tracking.

The workflow is structured into the following functional blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow twice daily at fixed times.
- **1.2 Hydration Message Preparation:** Selects a random hydration-themed GIF to accompany the message.
- **1.3 Message Dispatch:** Sends the hydration reminder with the GIF to Slack and/or Discord.
- **1.4 Engagement Waiting Period:** Pauses workflow execution to allow time for reactions.
- **1.5 Reaction Retrieval:** Fetches reaction data from Slack for the posted message.
- **1.6 Reaction Filtering:** Determines if relevant "checkmark" reactions were given.
- **1.7 Reaction Logging:** Records confirmed reactions in Airtable for tracking and reporting.

Each block directly connects to the next, forming a clear linear flow with a branching node for Slack and Discord message dispatch.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Triggers the entire workflow twice daily at 10:00 AM and 3:00 PM local time, establishing the heartbeat for the hydration reminders.

- **Nodes Involved:**  
  - Every Day at 10 AM & 3 PM (Schedule Trigger)

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Configuration:** Set to fire at 10:00 AM and 3:00 PM daily. Timezone must be configured in n8n system settings to align with the target community’s local time.  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Connects to "Pick Random GIF" node.  
  - **Potential Failures:** Misconfigured timezone can cause reminders to trigger at incorrect local times.  
  - **Edge Cases:** System downtime at trigger times will skip execution.  

---

#### 2.2 Hydration Message Preparation

- **Overview:**  
  Selects a random GIF from a curated list of hydration-themed images to enhance message appeal and engagement.

- **Nodes Involved:**  
  - Pick Random GIF (Set node)

- **Node Details:**  
  - **Type:** Set node (used to assign or manipulate data)  
  - **Configuration:** Contains an array or list of URLs pointing to hydration-related GIFs hosted on platforms like Giphy, Imgur, or a CDN. The node randomly selects one GIF URL per execution.  
  - **Inputs:** Receives trigger event from schedule node.  
  - **Outputs:** Passes selected GIF URL downstream for message dispatch.  
  - **Potential Failures:** Empty or unreachable GIF URLs lead to broken images in messages.  
  - **Edge Cases:** If the GIF list is empty or malformed, no GIF will be sent.  

---

#### 2.3 Message Dispatch

- **Overview:**  
  Sends the hydration reminder message, combined with the selected GIF, to Slack and/or Discord channels via webhook HTTP requests.

- **Nodes Involved:**  
  - Send to Slack (HTTP Request)  
  - Send to Discord (HTTP Request)

- **Node Details:**  
  - **Type:** HTTP Request nodes  
  - **Configuration:**  
    - **Slack:** Sends message using Slack’s Block Kit JSON format. Requires Slack webhook or Bot Token URL. Message includes soothing text and the selected GIF as an accessory.  
    - **Discord:** Sends embed message with hydration GIF and soft blue color (#2C7873). Uses Discord webhook URL.  
  - **Inputs:** Receives GIF URL from "Pick Random GIF".  
  - **Outputs:** Both nodes connect to "Wait 24 Hours" node.  
  - **Potential Failures:**  
    - Authentication errors if webhook URLs or tokens are invalid.  
    - Rate limits from Slack or Discord API may cause message send failures.  
  - **Edge Cases:**  
    - Only one of these nodes should be active if sending to a single platform, or dynamic routing logic should be implemented to prevent duplicate messages.  
  - **Notes:** Replace Slack webhook with Discord webhook URL if only Discord is used.  

---

#### 2.4 Engagement Waiting Period

- **Overview:**  
  Pauses the workflow for 24 hours to allow members to react to the hydration message with emojis such as ✅.

- **Nodes Involved:**  
  - Wait 24 Hours (Wait node)

- **Node Details:**  
  - **Type:** Wait node  
  - **Configuration:** Waits exactly 24 hours before continuing. Parameter can be adjusted to 12 hours for faster cycles.  
  - **Inputs:** Receives from both Slack and Discord message nodes.  
  - **Outputs:** Proceeds to fetch reactions from Slack.  
  - **Potential Failures:** Workflow executions might time out if there are system constraints on max execution time.  
  - **Edge Cases:** If reactions happen after wait expires, they won’t be captured in this cycle.  
  - **Important:** This wait node is necessary to ensure reaction data can be collected.  

---

#### 2.5 Reaction Retrieval

- **Overview:**  
  Fetches the reactions on the sent Slack message to detect engagement (specifically ✅ reactions).

- **Nodes Involved:**  
  - Get Slack Reactions (HTTP Request)

- **Node Details:**  
  - **Type:** HTTP Request node  
  - **Configuration:** Calls Slack API (such as reactions.get endpoint) using a Bot Token with scopes: `reactions:read` and `channels:read`. Requires message timestamp from the original posted message to query reactions accurately.  
  - **Inputs:** Connected from "Wait 24 Hours".  
  - **Outputs:** Passes reaction data to filter node.  
  - **Potential Failures:**  
    - Invalid or missing Bot Token results in auth errors.  
    - Incorrect message timestamp causes empty or failed retrieval.  
  - **Edge Cases:**  
    - If multiple messages are posted, message timestamp management is critical to avoid mixing data.  

---

#### 2.6 Reaction Filtering

- **Overview:**  
  Filters the retrieved reactions to continue workflow only if at least one ✅ (white_check_mark) reaction is present.

- **Nodes Involved:**  
  - Filter ✅ Reactions (Switch node)

- **Node Details:**  
  - **Type:** Switch node  
  - **Configuration:** Checks if the reaction list includes the emoji `white_check_mark`. Can be extended to filter other emojis like 💧 or 🫶 for richer tracking.  
  - **Inputs:** Reaction data from Slack API node.  
  - **Outputs:** Only continues to Airtable logging if condition met.  
  - **Potential Failures:** If reaction data structure changes, expression may fail.  
  - **Edge Cases:** No reactions or only non-checkmark reactions result in workflow termination here.  

---

#### 2.7 Reaction Logging

- **Overview:**  
  Logs each confirmed ✅ reaction as a new record in Airtable for tracking hydration engagement and building leaderboards or reports.

- **Nodes Involved:**  
  - Log in Airtable (Airtable node)

- **Node Details:**  
  - **Type:** Airtable node  
  - **Configuration:** Inserts a new record per reaction, potentially including user identifiers, timestamp, and reaction details. Can link to user profiles or segment by tags for analytics.  
  - **Inputs:** Filtered reaction data.  
  - **Outputs:** None (terminal node).  
  - **Potential Failures:**  
    - Airtable API authentication errors (requires valid API key).  
    - Rate limits or data schema mismatches causing record creation failure.  
  - **Edge Cases:** Duplicate or missing user IDs may affect data quality of leaderboards.  

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                 | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                       |
|-------------------------|-------------------|--------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Every Day at 10 AM & 3 PM | Schedule Trigger  | Initiate workflow twice daily  | -                           | Pick Random GIF              | 🕒 Schedule: Daily Wellness Nudges<br>Triggers the workflow twice daily:<br>- 10:00 AM<br>- 3:00 PM<br>🔹 Set timezone in Make settings to match your community.<br>🔹 This is the heartbeat of the hydration reminder system. |
| Pick Random GIF          | Set node          | Select random hydration GIF    | Every Day at 10 AM & 3 PM   | Send to Slack, Send to Discord | 🖼️ Random GIF Selector<br>Rotates through a curated list of calming hydration-themed GIFs.<br>💡 Hosted on Giphy, Imgur, or CDN.<br>You can expand the list for seasonal variety. |
| Send to Slack            | HTTP Request      | Send hydration reminder to Slack | Pick Random GIF             | Wait 24 Hours               | 📤 Send Message to Slack<br>Posts a friendly hydration reminder with soothing text and GIF.<br>✅ Uses Slack's Block Kit.<br>🔧 Replace URL with Discord webhook if needed. |
| Send to Discord          | HTTP Request      | Send hydration reminder to Discord | Pick Random GIF             | Wait 24 Hours               | 📤 Send Message to Discord<br>Sends embed-based message with hydration GIF.<br>🎨 Color: Soft blue (#2C7873).<br>💡 Use only one send node (Slack OR Discord) unless routing dynamically. |
| Wait 24 Hours            | Wait node         | Pause to allow reactions       | Send to Slack, Send to Discord | Get Slack Reactions         | ⏸️ Wait for Reactions<br>Pauses 24 hours to allow members to react with ✅.<br>🔹 Can reduce to 12h for faster cycles.<br>⚠️ Do not skip: needed to capture engagement. |
| Get Slack Reactions      | HTTP Request      | Fetch reactions on Slack message | Wait 24 Hours               | Filter ✅ Reactions          | 🔍 Fetch Slack Reactions<br>Uses Slack API to check if ✅ was added.<br>🔐 Requires Bot Token with reactions:read, channels:read.<br>💡 Store message timestamp from initial post. |
| Filter ✅ Reactions       | Switch node       | Continue only if ✅ reaction present | Get Slack Reactions          | Log in Airtable             | ✅ Filter for Checkmark Reactions<br>Only continues if at least one ✅ (white_check_mark) was added.<br>🔧 Can add more emoji filters (💧, 🫶) for expanded tracking. |
| Log in Airtable          | Airtable node     | Log confirmed reactions        | Filter ✅ Reactions          | -                           | 📊 Log Reaction in Airtable<br>Saves each ✅ reaction as a new record.<br>🔁 Used later for monthly leaderboards.<br>💡 Link to user profiles or add tags for segmentation. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run twice daily at 10:00 AM and 3:00 PM.  
   - Configure timezone in n8n settings to your community’s local time.  
   - Name it: "Every Day at 10 AM & 3 PM".

2. **Create Set Node to Pick Random GIF**  
   - Type: Set node  
   - Create a field (e.g., `gifURL`) with an array of hydration-themed GIF URLs.  
   - Use an expression or JavaScript snippet to pick one URL randomly per execution.  
   - Name it: "Pick Random GIF".  
   - Connect output of Schedule Trigger node to this node.

3. **Create HTTP Request Node for Slack Message**  
   - Type: HTTP Request  
   - Configure with Slack webhook URL or Bot Token.  
   - Set Method: POST.  
   - Compose payload in Slack Block Kit JSON format including text and the randomly selected GIF URL from the previous node.  
   - Name it: "Send to Slack".  
   - Connect output of "Pick Random GIF" node here.

4. **Create HTTP Request Node for Discord Message**  
   - Type: HTTP Request  
   - Configure with Discord webhook URL.  
   - Set Method: POST.  
   - Compose payload as Discord embed JSON including hydration GIF URL and soft blue color (#2C7873).  
   - Name it: "Send to Discord".  
   - Connect output of "Pick Random GIF" node here.

5. **Connect Both Message Nodes to Wait Node**  
   - Create a Wait node named "Wait 24 Hours".  
   - Set wait time to 24 hours (adjustable to 12 hours if preferred).  
   - Connect "Send to Slack" and "Send to Discord" outputs to this Wait node.

6. **Create HTTP Request Node to Fetch Slack Reactions**  
   - Type: HTTP Request  
   - Configure to call Slack API endpoint for reactions (e.g., `reactions.get`).  
   - Use Bot Token with `reactions:read` and `channels:read` scopes.  
   - Pass message timestamp (saved from "Send to Slack" response) as query parameter.  
   - Name it: "Get Slack Reactions".  
   - Connect output of Wait node here.

7. **Create Switch Node to Filter for ✅ Reactions**  
   - Type: Switch node  
   - Add condition to check if reaction array contains `white_check_mark` emoji.  
   - Name it: "Filter ✅ Reactions".  
   - Connect output of Slack reaction fetch node here.

8. **Create Airtable Node to Log Reactions**  
   - Type: Airtable node  
   - Connect to your Airtable base and table where reactions will be logged.  
   - Map fields to log user info, timestamp, reaction emoji, etc.  
   - Name it: "Log in Airtable".  
   - Connect output of switch node’s positive path here.

9. **Ensure Credentials Are Configured**  
   - Slack: Valid Bot Token or webhook URL with required scopes.  
   - Discord: Valid webhook URL if using Discord node.  
   - Airtable: API key with access to the base and table.

10. **Test Workflow**  
    - Trigger manually or wait for scheduled time.  
    - Verify messages sent with random GIFs.  
    - React with ✅ in Slack channel.  
    - Confirm reaction logging in Airtable.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow encourages hydration reminders twice daily with community engagement tracking via emoji reactions.                                                              | Workflow concept and engagement strategy.                                                      |
| Slack message formatting uses Block Kit for rich, user-friendly messages.                                                                                                | Slack API documentation: https://api.slack.com/block-kit                                      |
| Discord embeds use soft blue color #2C7873 (decimal 2899551) for calming visual consistency.                                                                              | Discord message embed color guides.                                                            |
| Reaction tracking focuses on ✅ emoji but can be extended to other emojis like 💧 or 🫶 for richer analytics.                                                             | Emoji Unicode documentation, Slack emoji API behavior.                                        |
| Airtable logging enables building monthly leaderboards ("Hydration Heroes") and user segmentation by tags or profiles.                                                  | Airtable API docs: https://airtable.com/api                                                    |
| Timezone configuration in n8n is critical to ensure reminders trigger at correct local times.                                                                             | n8n timezone settings: https://docs.n8n.io/integrations/builtin/triggers/schedule-trigger/     |
| Slack Bot Token must have `reactions:read` and `channels:read` OAuth scopes to successfully fetch reactions.                                                             | Slack OAuth scopes: https://api.slack.com/scopes                                              |
| Wait node is essential to allow users time to react before fetching reaction data; shortening wait time reduces engagement window but speeds processing cycles.           | Workflow timing considerations.                                                               |
| For multi-platform use, dynamically route messages to Slack or Discord to avoid duplicate notifications unless both are desired.                                         | Workflow design best practices for multi-channel messaging.                                   |

---

**Disclaimer:** The provided content is derived solely from an automated workflow constructed with n8n, a tool for integration and automation. It complies fully with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.