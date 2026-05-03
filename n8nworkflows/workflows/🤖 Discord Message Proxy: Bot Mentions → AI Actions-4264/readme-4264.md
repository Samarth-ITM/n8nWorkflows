🤖 Discord Message Proxy: Bot Mentions → AI Actions

https://n8nworkflows.xyz/workflows/---discord-message-proxy--bot-mentions---ai-actions-4264


# 🤖 Discord Message Proxy: Bot Mentions → AI Actions

### 1. Workflow Overview

This workflow is designed to automate processing of Discord messages where the bot is mentioned, acting as a proxy to trigger AI-related actions (e.g., responding or processing commands). It runs on a scheduled trigger (periodically) rather than using an event-driven Discord webhook trigger, which is acknowledged as a suboptimal method in the workflow name.

The main logical blocks are:

- **1.1 Scheduled Trigger and Channel Retrieval:** Periodically triggers workflow execution and fetches all Discord server channels accessible by the bot.
- **1.2 Message Retrieval and Filtering:** For each channel, retrieves the latest message, removes duplicates, filters out empty or old messages, and isolates those containing mentions.
- **1.3 Mention and Authorization Filtering:** Further filters messages to those that mention the bot specifically, then filters by authorized users only.
- **1.4 Message Cleaning and AI Processing:** Cleans up the message content before sending it to an external AI HTTP endpoint for processing.
- **1.5 Response Posting:** Posts the AI-generated response back to Discord.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Channel Retrieval

**Overview:**  
This block initiates the workflow on a schedule and retrieves all server channels the bot can access.

**Nodes Involved:**  
- Schedule Trigger  
- Set Values  
- Get All Servers' Channels  
- Filter (Remove empty)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* scheduleTrigger  
  - *Role:* Periodically starts the workflow on a fixed schedule (default unspecified, likely defaults)  
  - *Input:* None  
  - *Output:* Triggers Set Values node  
  - *Failure cases:* None typical; misconfiguration of schedule can cause no triggering.

- **Set Values**  
  - *Type:* set  
  - *Role:* Prepares any static or initial data for the workflow execution (empty parameters, so possibly a placeholder or to maintain flow)  
  - *Input:* From Schedule Trigger  
  - *Output:* To Get All Servers' Channels  
  - *Failure cases:* None expected, no parameters set.

- **Get All Servers' Channels**  
  - *Type:* discord  
  - *Role:* Retrieves all channels from all Discord servers the bot is connected to using Discord API  
  - *Input:* From Set Values  
  - *Output:* To Filter (Remove empty)  
  - *Failure cases:* Discord API errors (auth, rate limits), empty responses.

- **Filter (Remove empty)**  
  - *Type:* filter  
  - *Role:* Removes channels with no data or empty values to ensure only valid channels proceed  
  - *Input:* From Get All Servers' Channels  
  - *Output:* To Get last message  
  - *Failure cases:* No channels available.

---

#### 2.2 Message Retrieval and Filtering

**Overview:**  
This block fetches the latest messages from each channel, removes duplicates, filters out old messages, and isolates messages containing mentions.

**Nodes Involved:**  
- Get last message  
- Remove Duplicates  
- Filter (Remove Old)  
- Discord1  
- Remove Duplicates1  
- Filter (Messages with mentions)

**Node Details:**

- **Get last message**  
  - *Type:* discord  
  - *Role:* Retrieves the last message from each channel passed from previous filter  
  - *Input:* From Filter (Remove empty)  
  - *Output:* To Remove Duplicates  
  - *Failure cases:* Discord API errors, empty messages, timeout. Set to continue on error.

- **Remove Duplicates**  
  - *Type:* removeDuplicates  
  - *Role:* Eliminates duplicate messages to avoid redundant processing  
  - *Input:* From Get last message  
  - *Output:* To Filter (Remove Old)  
  - *Failure cases:* None typical.

- **Filter (Remove Old)**  
  - *Type:* filter  
  - *Role:* Filters out messages that are considered too old to process (time logic in filter)  
  - *Input:* From Remove Duplicates  
  - *Output:* To Discord1  
  - *Failure cases:* No recent messages found.

- **Discord1**  
  - *Type:* discord  
  - *Role:* Likely used here to re-fetch or further process messages or channels (exact usage ambiguous but likely to confirm message or channel data)  
  - *Input:* From Filter (Remove Old)  
  - *Output:* To Remove Duplicates1  
  - *Failure cases:* API errors.

- **Remove Duplicates1**  
  - *Type:* removeDuplicates  
  - *Role:* Removes duplicates post Discord1 node to ensure unique messages  
  - *Input:* From Discord1  
  - *Output:* To Filter (Messages with mentions)  
  - *Failure cases:* None typical.

- **Filter (Messages with mentions)**  
  - *Type:* filter  
  - *Role:* Keeps only messages that include mentions (any mentions)  
  - *Input:* From Remove Duplicates1  
  - *Output:* To Filter (Messages mentioning bot)  
  - *Failure cases:* No messages with mentions.

---

#### 2.3 Mention and Authorization Filtering

**Overview:**  
Filters messages specifically mentioning the bot and from authorized users only.

**Nodes Involved:**  
- Filter (Messages mentioning bot)  
- Filter (Authorized User)

**Node Details:**

- **Filter (Messages mentioning bot)**  
  - *Type:* filter  
  - *Role:* Extracts messages that mention the bot user ID or username specifically  
  - *Input:* From Filter (Messages with mentions)  
  - *Output:* To Filter (Authorized User)  
  - *Failure cases:* Messages that mention others only are excluded.

- **Filter (Authorized User)**  
  - *Type:* filter  
  - *Role:* Allows only messages sent by users authorized to interact with the bot (logic likely uses user ID whitelist or role check)  
  - *Input:* From Filter (Messages mentioning bot)  
  - *Output:* To Clean Message  
  - *Failure cases:* Unauthorized user messages discarded.

---

#### 2.4 Message Cleaning and AI Processing

**Overview:**  
Cleans the message content and sends it to an external HTTP endpoint (likely AI service) for processing.

**Nodes Involved:**  
- Clean Message  
- HTTP Request

**Node Details:**

- **Clean Message**  
  - *Type:* code  
  - *Role:* Executes custom JavaScript to sanitize or preprocess message text before forwarding (e.g., remove mentions, trim, normalize)  
  - *Input:* From Filter (Authorized User)  
  - *Output:* To HTTP Request  
  - *Failure cases:* Script errors, unexpected message structure.

- **HTTP Request**  
  - *Type:* httpRequest  
  - *Role:* Sends cleaned message content to external AI API for processing (e.g., OpenAI or custom AI endpoint)  
  - *Input:* From Clean Message  
  - *Output:* To It  
  - *Failure cases:* Network errors, API auth failures, rate limits, malformed requests or responses.

---

#### 2.5 Response Posting

**Overview:**  
Posts the processed AI response back to Discord, presumably into the same channel or thread.

**Nodes Involved:**  
- It

**Node Details:**

- **It**  
  - *Type:* discord  
  - *Role:* Sends the AI-generated response back to Discord as a message  
  - *Input:* From HTTP Request  
  - *Output:* Terminal (no further nodes)  
  - *Failure cases:* Discord API errors, message formatting issues.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                               | Input Node(s)                | Output Node(s)                | Sticky Note                                        |
|---------------------------|---------------------|-----------------------------------------------|------------------------------|------------------------------|---------------------------------------------------|
| Schedule Trigger          | scheduleTrigger     | Starts workflow on schedule                     | None                         | Set Values                   |                                                   |
| Set Values               | set                 | Initialize static or empty data                 | Schedule Trigger             | Get All Servers' Channels    |                                                   |
| Get All Servers' Channels | discord             | Fetch all accessible Discord server channels   | Set Values                  | Filter (Remove empty)        |                                                   |
| Filter (Remove empty)     | filter              | Remove empty or invalid channels                | Get All Servers' Channels    | Get last message             |                                                   |
| Get last message          | discord             | Retrieve last message from each channel         | Filter (Remove empty)        | Remove Duplicates            |                                                   |
| Remove Duplicates         | removeDuplicates    | Remove duplicate messages                        | Get last message             | Filter (Remove Old)          |                                                   |
| Filter (Remove Old)       | filter              | Filter out old messages                           | Remove Duplicates            | Discord1                    |                                                   |
| Discord1                 | discord             | Additional message/channel processing            | Filter (Remove Old)          | Remove Duplicates1           |                                                   |
| Remove Duplicates1        | removeDuplicates    | Remove duplicate messages post Discord1 node     | Discord1                    | Filter (Messages with mentions) |                                                   |
| Filter (Messages with mentions) | filter              | Keep messages containing mentions               | Remove Duplicates1           | Filter (Messages mentioning bot) |                                                   |
| Filter (Messages mentioning bot) | filter              | Keep messages specifically mentioning the bot  | Filter (Messages with mentions) | Filter (Authorized User)     |                                                   |
| Filter (Authorized User)  | filter              | Allow messages only from authorized users         | Filter (Messages mentioning bot) | Clean Message               |                                                   |
| Clean Message             | code                | Sanitize and preprocess message content           | Filter (Authorized User)     | HTTP Request                |                                                   |
| HTTP Request              | httpRequest         | Send message to AI API endpoint                   | Clean Message               | It                         |                                                   |
| It                       | discord             | Post AI response back to Discord                   | HTTP Request                | None                       |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: scheduleTrigger  
   - Set to a desired interval (e.g., every minute) to check messages regularly.

2. **Add a Set node ("Set Values")**  
   - Type: set  
   - Leave parameters empty (or configure as needed if static values are required).  
   - Connect Schedule Trigger → Set Values.

3. **Add a Discord node ("Get All Servers' Channels")**  
   - Type: discord  
   - Operation: List Channels (default)  
   - Use Discord OAuth2 credentials with bot permissions (read channels).  
   - Connect Set Values → Get All Servers' Channels.

4. **Add a Filter node ("Filter (Remove empty)")**  
   - Type: filter  
   - Condition: Check if channel data is not empty.  
   - Connect Get All Servers' Channels → Filter (Remove empty).

5. **Add a Discord node ("Get last message")**  
   - Type: discord  
   - Operation: Get Messages or Get Last Message in channel  
   - Use credentials with message read permissions.  
   - Configure to retrieve the most recent message for each channel.  
   - Set On Error to "Continue" to avoid halting on errors.  
   - Connect Filter (Remove empty) → Get last message.

6. **Add a Remove Duplicates node ("Remove Duplicates")**  
   - Type: removeDuplicates  
   - Configure to deduplicate messages by unique message ID.  
   - Connect Get last message → Remove Duplicates.

7. **Add a Filter node ("Filter (Remove Old)")**  
   - Type: filter  
   - Condition: Check message timestamp is within acceptable recent timeframe (e.g., last 5 minutes).  
   - Connect Remove Duplicates → Filter (Remove Old).

8. **Add a Discord node ("Discord1")**  
   - Type: discord  
   - Use for additional operations on message or channel if needed (e.g., fetch additional data).  
   - Connect Filter (Remove Old) → Discord1.

9. **Add a Remove Duplicates node ("Remove Duplicates1")**  
   - Type: removeDuplicates  
   - Deduplicate messages post Discord1 processing.  
   - Connect Discord1 → Remove Duplicates1.

10. **Add a Filter node ("Filter (Messages with mentions)")**  
    - Type: filter  
    - Condition: Check if message contains any mention entities.  
    - Connect Remove Duplicates1 → Filter (Messages with mentions).

11. **Add a Filter node ("Filter (Messages mentioning bot)")**  
    - Type: filter  
    - Condition: Check if mentions include the bot’s user ID.  
    - Connect Filter (Messages with mentions) → Filter (Messages mentioning bot).

12. **Add a Filter node ("Filter (Authorized User)")**  
    - Type: filter  
    - Condition: Check if the message author is in an authorized user list or has specific roles.  
    - Configure authorized user IDs or roles as expressions or static list.  
    - Connect Filter (Messages mentioning bot) → Filter (Authorized User).

13. **Add a Code node ("Clean Message")**  
    - Type: code  
    - JavaScript code to clean message content by removing bot mention strings, emojis, or unwanted characters.  
    - Connect Filter (Authorized User) → Clean Message.

14. **Add an HTTP Request node ("HTTP Request")**  
    - Type: httpRequest  
    - Method: POST (or as required by AI API)  
    - URL: AI service endpoint (e.g., OpenAI API URL or custom AI endpoint)  
    - Body: send cleaned message text, set headers (authorization tokens, content-type).  
    - Connect Clean Message → HTTP Request.

15. **Add a Discord node ("It")**  
    - Type: discord  
    - Operation: Send Message  
    - Use credentials with message sending permissions.  
    - Configure to send AI response to the appropriate channel or thread.  
    - Connect HTTP Request → It.

**Credentials to configure:**  
- Discord OAuth2 Bot credentials with permissions: Read Messages, Send Messages, Read Channels.  
- HTTP Request credentials or API Key for AI API endpoint.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow is named "Automatic [slow] Discord Task Worker with no community Trigger node (This is a bad way to do this and you should probably not)" indicating it uses polling rather than event-driven triggers, which is less efficient. | Workflow naming indicates best practice caution.                                                  |
| For Discord API details, refer to: https://discord.com/developers/docs/intro                                                  | Official Discord developer documentation.                                                         |
| When using AI HTTP API, ensure rate limits and auth tokens are handled properly to avoid request failures.                | AI API providers (OpenAI, etc.) documentation.                                                    |
| Mention filtering requires correct bot user ID; fetch dynamically to avoid hardcoding.                                    | Use Discord API Get Current User endpoint if needed.                                              |

---

**Disclaimer:** The text provided is exclusively generated from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.