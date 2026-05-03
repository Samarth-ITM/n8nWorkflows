Nightly Discord Channel Cleanup

https://n8nworkflows.xyz/workflows/nightly-discord-channel-cleanup-2980


# Nightly Discord Channel Cleanup

### 1. Workflow Overview

This workflow, titled **Nightly Discord Channel Cleanup**, is designed to automate the maintenance of Discord channels by deleting messages older than seven days. It is intended for Discord server administrators who want to enforce message retention policies and keep channels tidy without manual intervention.

The workflow runs once daily at 9:00 p.m. and performs the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every day at 9 p.m.
- **1.2 Retrieve Channels:** Fetches all channels from a specified Discord server.
- **1.3 Rate Limit Pause (Post-Channels):** Waits briefly to respect Discord API rate limits after fetching channels.
- **1.4 Loop Over Channels:** Iterates through each channel to process messages.
- **1.5 Fetch Messages per Channel:** Retrieves all messages from the current channel.
- **1.6 Rate Limit Pause (Post-Messages Fetch):** Waits to avoid hitting Discord’s message retrieval rate limits.
- **1.7 Filter Old Messages:** Filters messages older than seven days.
- **1.8 Loop Over Old Messages:** Iterates through each old message to delete.
- **1.9 Delete Messages:** Deletes each filtered message.
- **1.10 Rate Limit Pause (Post-Deletion):** Waits to comply with Discord’s message deletion rate limits.

Supporting the main logic are sticky notes providing setup instructions, tips, and reminders about error handling.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Starts the workflow automatically every day at 9:00 p.m.
- **Nodes Involved:** `Every day at 9pm`
- **Node Details:**
  - Type: Schedule Trigger
  - Configuration: Trigger set to fire at hour 21 (9 p.m.) daily.
  - Inputs: None (trigger node)
  - Outputs: Connects to `Get all Discord channels`
  - Edge Cases: Workflow will not run if n8n instance is offline or disabled.
  - Version: 1.2

#### 1.2 Retrieve Channels

- **Overview:** Retrieves all channels from the specified Discord server.
- **Nodes Involved:** `Get all Discord channels`
- **Node Details:**
  - Type: Discord Node
  - Operation: `getAll` channels for a given guild (Discord server)
  - Parameters: 
    - `guildId`: Must be set to the target Discord server ID.
    - `filter`: Includes only text and voice channels (filter values `[0,2]`).
    - `returnAll`: true (fetch all channels without pagination)
  - Inputs: From `Every day at 9pm`
  - Outputs: To `Cool down Discord API rate limits`
  - Edge Cases: 
    - Authentication errors if Discord credentials are invalid.
    - Empty channel list if guildId is incorrect.
  - Version: 2

#### 1.3 Rate Limit Pause (Post-Channels)

- **Overview:** Waits 2 seconds to avoid hitting Discord API rate limits after fetching channels.
- **Nodes Involved:** `Cool down Discord API rate limits`
- **Node Details:**
  - Type: Wait Node
  - Parameters: Wait for 2 seconds
  - Inputs: From `Get all Discord channels`
  - Outputs: To `Loop Over Channels`
  - Edge Cases: Minimal risk; wait node may delay workflow if n8n is busy.
  - Version: 1.1

#### 1.4 Loop Over Channels

- **Overview:** Splits the list of channels into batches to process each channel individually.
- **Nodes Involved:** `Loop Over Channels`
- **Node Details:**
  - Type: SplitInBatches
  - Parameters: Default batch size (1 by default)
  - Inputs: From `Cool down Discord API rate limits`
  - Outputs: Two outputs:
    - Main output: To `Filter Messages older than 7 days`
    - Secondary output: To `Get messages from Channel`
  - Edge Cases: 
    - If no channels, loop does nothing.
    - Batch size can be adjusted to optimize rate limits.
  - Version: 3

#### 1.5 Fetch Messages per Channel

- **Overview:** Retrieves all messages from the current channel.
- **Nodes Involved:** `Get messages from Channel`
- **Node Details:**
  - Type: Discord Node
  - Operation: `getAll` messages for a given channel
  - Parameters:
    - `guildId`: Same as server ID used in channels node.
    - `channelId`: Dynamically set to current channel ID (`={{ $json.id }}`)
    - `returnAll`: true (fetch all messages)
  - Inputs: From `Loop Over Channels` (secondary output)
  - Outputs: To `Cool down Get messages API rate limits`
  - Error Handling: On error, continues regular output (does not stop workflow)
  - Edge Cases:
    - Large channels may have many messages, potentially causing timeouts.
    - API limits on message retrieval.
  - Version: 2

#### 1.6 Rate Limit Pause (Post-Messages Fetch)

- **Overview:** Waits 2 seconds to respect Discord API rate limits after fetching messages.
- **Nodes Involved:** `Cool down Get messages API rate limits`
- **Node Details:**
  - Type: Wait Node
  - Parameters: Wait for 2 seconds
  - Inputs: From `Get messages from Channel`
  - Outputs: To `Loop Over Channels` (secondary output)
  - Edge Cases: Minimal; ensures compliance with API limits.
  - Version: 1.1

#### 1.7 Filter Old Messages

- **Overview:** Filters messages to keep only those older than 7 days.
- **Nodes Involved:** `Filter Messages older than 7 days`
- **Node Details:**
  - Type: Filter Node
  - Conditions:
    - Checks if `timestamp` exists on message.
    - Checks if `timestamp` is before current date minus 7 days.
  - Inputs: From `Loop Over Channels` (main output)
  - Outputs: To `Loop Over Messages`
  - Edge Cases:
    - Messages without timestamp are excluded.
    - Timezone considerations depend on Discord timestamp format (ISO 8601).
  - Version: 2.2

#### 1.8 Loop Over Old Messages

- **Overview:** Splits filtered old messages into batches to delete them one by one.
- **Nodes Involved:** `Loop Over Messages`
- **Node Details:**
  - Type: SplitInBatches
  - Parameters: Default batch size (1)
  - Inputs: From `Filter Messages older than 7 days`
  - Outputs: Two outputs:
    - Main output: None (empty)
    - Secondary output: To `Delete Message`
  - Edge Cases:
    - If no old messages, loop does nothing.
  - Version: 3

#### 1.9 Delete Messages

- **Overview:** Deletes each message identified as older than 7 days.
- **Nodes Involved:** `Delete Message`
- **Node Details:**
  - Type: Discord Node
  - Operation: `deleteMessage`
  - Parameters:
    - `guildId`: Same server ID as before.
    - `channelId`: Dynamically set from message's `channel_id`
    - `messageId`: Dynamically set from message's `id`
  - Inputs: From `Loop Over Messages` (secondary output)
  - Outputs: To `Cool down Message deletion API rate limits`
  - Error Handling: On error, continues regular output (does not stop workflow)
  - Edge Cases:
    - Message may have been deleted already.
    - Permissions errors if bot lacks delete rights.
    - Rate limits on deletion.
  - Version: 2

#### 1.10 Rate Limit Pause (Post-Deletion)

- **Overview:** Waits 1 second after each message deletion to comply with Discord’s deletion rate limits.
- **Nodes Involved:** `Cool down Message deletion API rate limits`
- **Node Details:**
  - Type: Wait Node
  - Parameters: Wait for 1 second
  - Inputs: From `Delete Message`
  - Outputs: To `Loop Over Messages` (secondary output)
  - Edge Cases: Minimal; ensures API compliance.
  - Version: 1.1

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                         | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                              |
|-----------------------------------|---------------------|---------------------------------------|--------------------------------|----------------------------------------|--------------------------------------------------------------------------------------------------------|
| Every day at 9pm                  | Schedule Trigger    | Starts workflow daily at 9 p.m.       | None                           | Get all Discord channels                |                                                                                                        |
| Get all Discord channels          | Discord             | Retrieves all channels from server    | Every day at 9pm               | Cool down Discord API rate limits       |                                                                                                        |
| Cool down Discord API rate limits | Wait                | Pauses 2 seconds after fetching channels | Get all Discord channels       | Loop Over Channels                      |                                                                                                        |
| Loop Over Channels                | SplitInBatches      | Iterates over each Discord channel    | Cool down Discord API rate limits | Filter Messages older than 7 days, Get messages from Channel |                                                                                                        |
| Filter Messages older than 7 days | Filter              | Filters messages older than 7 days    | Loop Over Channels             | Loop Over Messages                      |                                                                                                        |
| Loop Over Messages               | SplitInBatches      | Iterates over each old message        | Filter Messages older than 7 days | Delete Message                         |                                                                                                        |
| Delete Message                   | Discord             | Deletes each filtered old message     | Loop Over Messages             | Cool down Message deletion API rate limits |                                                                                                        |
| Cool down Message deletion API rate limits | Wait                | Pauses 1 second after each deletion   | Delete Message                 | Loop Over Messages                      |                                                                                                        |
| Get messages from Channel        | Discord             | Retrieves all messages from a channel | Loop Over Channels             | Cool down Get messages API rate limits |                                                                                                        |
| Cool down Get messages API rate limits | Wait                | Pauses 2 seconds after fetching messages | Get messages from Channel      | Loop Over Channels                      |                                                                                                        |
| Sticky Note                      | Sticky Note         | Setup instructions                    | None                           | None                                   | ### 👨‍🎤 Setup 1. Add your **Discord** credentials 2. Change the server in each **Discord** node to the correct one 3. Click the Test Workflow button 3. Activate the workflow to run on a schedule |
| Sticky Note1                    | Sticky Note         | Workflow description                  | None                           | None                                   | # Nightly Discord Channel Cleanup ... (full description as in workflow overview)                       |
| Sticky Note2                    | Sticky Note         | Reminder to setup error workflow     | None                           | None                                   | **Note ☝️** Don’t forget to setup an error workflow to get notified if something goes wrong            |
| Sticky Note3                    | Sticky Note         | Tip about OAuth2 authentication       | None                           | None                                   | **Tip 👇** OAuth2 Authentication is very easy to setup                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Name: `Every day at 9pm`
   - Type: Schedule Trigger
   - Set to trigger daily at 21:00 (9 p.m.)

2. **Create a Discord node to get all channels:**
   - Name: `Get all Discord channels`
   - Type: Discord
   - Operation: `getAll` channels
   - Set `guildId` to your Discord server ID (must be configured)
   - Set `filter` to `[0,2]` to include text and voice channels
   - Enable `returnAll` to fetch all channels
   - Connect output from `Every day at 9pm` to this node

3. **Add a Wait node to respect API rate limits after fetching channels:**
   - Name: `Cool down Discord API rate limits`
   - Type: Wait
   - Set wait time to 2 seconds
   - Connect output from `Get all Discord channels` to this node

4. **Add a SplitInBatches node to loop over channels:**
   - Name: `Loop Over Channels`
   - Type: SplitInBatches
   - Use default batch size (1)
   - Connect output from `Cool down Discord API rate limits` to this node

5. **Add a Discord node to get all messages from the current channel:**
   - Name: `Get messages from Channel`
   - Type: Discord
   - Operation: `getAll` messages
   - Set `guildId` to your Discord server ID
   - Set `channelId` to expression: `={{ $json.id }}` (current channel ID from batch)
   - Enable `returnAll`
   - Set error handling to "Continue on Error"
   - Connect secondary output of `Loop Over Channels` to this node

6. **Add a Wait node after fetching messages to respect rate limits:**
   - Name: `Cool down Get messages API rate limits`
   - Type: Wait
   - Set wait time to 2 seconds
   - Connect output from `Get messages from Channel` to this node

7. **Connect output of `Cool down Get messages API rate limits` back to secondary output of `Loop Over Channels`**  
   (This creates a loop allowing message fetching for each channel)

8. **Add a Filter node to select messages older than 7 days:**
   - Name: `Filter Messages older than 7 days`
   - Type: Filter
   - Conditions:
     - `timestamp` exists
     - `timestamp` is before `{{$today.minus({days: 7})}}`
   - Connect main output of `Loop Over Channels` to this node

9. **Add a SplitInBatches node to loop over filtered old messages:**
   - Name: `Loop Over Messages`
   - Type: SplitInBatches
   - Use default batch size (1)
   - Connect output from `Filter Messages older than 7 days` to this node

10. **Add a Discord node to delete each old message:**
    - Name: `Delete Message`
    - Type: Discord
    - Operation: `deleteMessage`
    - Set `guildId` to your Discord server ID
    - Set `channelId` to expression: `={{ $json.channel_id }}`
    - Set `messageId` to expression: `={{ $json.id }}`
    - Set error handling to "Continue on Error"
    - Connect secondary output of `Loop Over Messages` to this node

11. **Add a Wait node after deleting messages to respect deletion rate limits:**
    - Name: `Cool down Message deletion API rate limits`
    - Type: Wait
    - Set wait time to 1 second
    - Connect output from `Delete Message` to this node

12. **Connect output of `Cool down Message deletion API rate limits` back to secondary output of `Loop Over Messages`**  
    (This creates a loop allowing deletion of each old message)

13. **Configure Discord credentials:**
    - Add your Discord OAuth2 credentials in n8n credentials manager.
    - Ensure the bot has permissions to read channels, read message history, and delete messages on the target server.

14. **Set the `guildId` parameter in all Discord nodes to your Discord server ID.**

15. **Test the workflow manually to verify correct operation.**

16. **Activate the workflow to run automatically every day at 9 p.m.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| **Note ☝️** Don’t forget to setup an error workflow to get notified if something goes wrong               | Sticky Note2 in workflow; recommended for production use                                           |
| ### 👨‍🎤 Setup 1. Add your **Discord** credentials 2. Change the server in each **Discord** node 3. Click the Test Workflow button 3. Activate the workflow to run on a schedule | Sticky Note with setup instructions                                                               |
| **Tip 👇** OAuth2 Authentication is very easy to setup                                                   | Sticky Note3; helpful for users configuring Discord credentials                                    |
| Workflow description and purpose as detailed in Sticky Note1                                            | Provides context and overview for users and maintainers                                            |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and maintaining the Nightly Discord Channel Cleanup workflow in n8n.