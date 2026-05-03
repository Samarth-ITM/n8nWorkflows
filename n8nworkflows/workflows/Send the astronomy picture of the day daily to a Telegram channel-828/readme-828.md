Send the astronomy picture of the day daily to a Telegram channel

https://n8nworkflows.xyz/workflows/send-the-astronomy-picture-of-the-day-daily-to-a-telegram-channel-828


# Send the astronomy picture of the day daily to a Telegram channel

### 1. Workflow Overview

This workflow automates the daily sharing of NASA’s Astronomy Picture of the Day (APOD) to a specified Telegram channel. It fetches the latest APOD image and posts it with a caption on Telegram. The workflow is designed to run once daily at a specified time, making it ideal for astronomy enthusiasts or informational channels that want to provide daily visual content from NASA.

**Logical blocks:**

- **1.1 Scheduled Trigger:** A Cron node schedules the workflow to run daily at 8 PM (configurable).
- **1.2 NASA Data Retrieval:** The NASA node fetches the Astronomy Picture of the Day metadata and optionally the image file.
- **1.3 Telegram Posting:** The Telegram node sends the fetched image and title as a photo post to a Telegram channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

**Overview:**  
This block initiates the workflow automatically every day at a set time (default 8 PM).

**Nodes Involved:**  
- Cron

**Node Details:**

- **Cron**  
  - Type: Trigger  
  - Role: Time-based trigger node that starts the workflow daily.  
  - Configuration: Set to trigger every day at hour 20 (8 PM). Minutes and seconds default to 0.  
  - Expressions/Variables: None used.  
  - Inputs: None (trigger node).  
  - Outputs: Connected to the NASA node.  
  - Edge Cases: Workflow will not trigger if the n8n instance is offline or Cron service is misconfigured. Time zone mismatches may cause unexpected trigger times.  
  - Version Requirements: Standard node, no specific version constraints.

#### 1.2 NASA Data Retrieval

**Overview:**  
This block fetches the Astronomy Picture of the Day from NASA's API. It retrieves metadata including the image URL and title. The image download option is set to false by default, so only the URL is fetched.

**Nodes Involved:**  
- NASA

**Node Details:**

- **NASA**  
  - Type: API Integration  
  - Role: Calls NASA’s Astronomy Picture of the Day API to fetch data.  
  - Configuration:  
    - Download Image: false (does not download the image file to n8n; sends URL only).  
    - Additional Fields: None specified.  
  - Credentials: Uses the "NASA" API credential to authenticate requests.  
  - Expressions/Variables: None inside node, but output JSON fields are accessed later.  
  - Inputs: Receives trigger from Cron node.  
  - Outputs: Passes output JSON with APOD data to Telegram node.  
  - Edge Cases:  
    - API limit exceeded or invalid credentials cause errors.  
    - Network issues may cause timeouts or failures.  
    - If download was enabled, large image files might slow workflow.  
  - Version Requirements: Standard node, no specific version constraints.

#### 1.3 Telegram Posting

**Overview:**  
This block sends the fetched image URL as a photo message to a Telegram channel, with the photo caption set to the APOD title.

**Nodes Involved:**  
- Telegram

**Node Details:**

- **Telegram**  
  - Type: Messaging Integration  
  - Role: Sends a photo message to a Telegram channel.  
  - Configuration:  
    - Operation: `sendPhoto` - posts an image.  
    - Chat ID: `-485365454` (Telegram channel identifier, must be a valid channel where the bot is admin).  
    - File: Dynamic expression `{{$node["NASA"].json["url"]}}` to use the image URL from NASA node output.  
    - Caption: Dynamic expression `{{$node["NASA"].json["title"]}}` to set the photo caption as the APOD title.  
  - Credentials: Uses "Telegram n8n bot" OAuth2 credentials with bot token and permissions.  
  - Inputs: Receives JSON output from NASA node.  
  - Outputs: None (end of workflow).  
  - Edge Cases:  
    - Invalid chat ID or bot permissions cause message sending failure.  
    - If the image URL is invalid or inaccessible, Telegram may show an error or empty image.  
    - Network or API outages can cause failure.  
  - Version Requirements: Standard node, no specific version constraints.

---

### 3. Summary Table

| Node Name | Node Type           | Functional Role         | Input Node(s) | Output Node(s) | Sticky Note                                                                                                         |
|-----------|---------------------|------------------------|---------------|----------------|---------------------------------------------------------------------------------------------------------------------|
| Cron      | Cron Trigger        | Schedule daily trigger | None          | NASA           | The Cron node triggers the workflow daily at 8 PM. You can update the time in the Cron node to your desired time.   |
| NASA      | NASA API            | Fetch APOD metadata     | Cron          | Telegram       | The NASA node fetches the Astronomy Picture of the Day from NASA API. Toggle Download Image to true to get the file.|
| Telegram  | Telegram Integration | Send photo to channel   | NASA          | None           | Sends the image to a Telegram channel. Replace with another platform node if needed. See NASA node docs: https://docs.n8n.io/nodes/n8n-nodes-base.nasa/#nasa |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it: "Send the Astronomy Picture of the day daily to a Telegram channel".

2. **Add a Cron node**:  
   - Type: `Cron`  
   - Set Trigger Times:  
     - Hour: 20 (8 PM)  
     - Minute: 0 (default)  
     - Seconds: 0 (default)  
   - No credentials needed.  
   - This node triggers the workflow daily.

3. **Add a NASA node**:  
   - Type: `NASA`  
   - Credentials: Select or create NASA API credentials (requires NASA API key).  
   - Parameters:  
     - Download Image: set to `false` (to fetch metadata and not download the image).  
     - Additional Fields: leave empty.  
   - Connect the Cron node’s main output to NASA node’s input.

4. **Add a Telegram node**:  
   - Type: `Telegram`  
   - Credentials: Set up Telegram Bot API credentials with your bot token.  
   - Parameters:  
     - Operation: `sendPhoto`  
     - Chat ID: use your Telegram channel ID (e.g., `-485365454`). Ensure the bot is an admin of the channel.  
     - File: Set expression to `{{$node["NASA"].json["url"]}}` — uses the image URL from NASA node.  
     - Caption: Set expression to `{{$node["NASA"].json["title"]}}` — uses the APOD title as caption.  
   - Connect NASA node’s main output to Telegram node’s input.

5. **Activate the workflow** or run manually to test.

6. **Optional adjustments:**  
   - To send the actual image file, set "Download Image" in NASA node to `true`. You may need to adjust Telegram node to send binary data instead of URL.  
   - To change posting time, modify the Cron node’s trigger hour/minute.  
   - To post on other platforms, replace the Telegram node accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow can be adapted to post daily images to other platforms by changing the last node.            | For example, replace Telegram node with Slack node or Twitter node as needed.                            |
| Detailed documentation for the NASA node is available here: https://docs.n8n.io/nodes/n8n-nodes-base.nasa/#nasa | Official n8n documentation for NASA node.                                                                |
| Ensure your Telegram bot has admin rights in the channel to send messages.                                 | Telegram API requirement for bot to post in channels.                                                    |
| The time zone for the Cron node depends on your n8n server settings; adjust accordingly if needed.        | Check your server’s time zone configuration to match desired posting time.                               |

---

This concludes the comprehensive reference documentation for the "Send the Astronomy Picture of the day daily to a Telegram channel" workflow.