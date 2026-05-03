Send memes via Discord

https://n8nworkflows.xyz/workflows/send-memes-via-discord-683


# Send memes via Discord

### 1. Workflow Overview

This workflow, titled **"Send memes via Discord"**, is designed to automatically send meme messages to a specified Discord channel via webhook at scheduled times. It targets users who want to receive entertaining meme content on specific days or intervals without manual intervention. The workflow consists of three main scheduling blocks, each triggering at different times or frequencies and sending a unique meme message through Discord.

Logical blocks:

- **1.1 Weekly Wednesday Meme Delivery**: Sends a specific meme every Wednesday at 9:00 AM.
- **1.2 Weekly Friday Meme Delivery**: Sends a different meme every Friday at 9:00 AM.
- **1.3 Repeating 30-minute Interval Meme Delivery**: Sends a third meme every 30 minutes continuously.

Each block contains a Cron node that triggers the flow at the configured time and a Discord node that posts the meme message to a Discord channel using a webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Weekly Wednesday Meme Delivery

- **Overview:**  
  This block triggers every Wednesday at 9:00 AM and sends a meme image along with a text message to a Discord channel via webhook.

- **Nodes Involved:**  
  - Cron  
  - Discord

- **Node Details:**

  - **Cron**  
    - Type: Trigger node to schedule workflows based on time.  
    - Configuration: Set to trigger every week on Wednesday at 9:00 (hour=9, weekday=6, mode=everyWeek).  
    - Inputs: None (trigger node).  
    - Outputs: Connected to Discord node.  
    - Potential Failures: Misconfigured time zone or cron settings may cause unexpected trigger times.

  - **Discord**  
    - Type: Action node to send messages to Discord channels via webhook.  
    - Configuration:  
      - Text message:  
        ```
        It's Wednesday, my dudes!
        https://i.kym-cdn.com/entries/icons/original/000/020/016/wednesdaymydudeswide.jpg
        ```  
      - Webhook URL: A Discord webhook URL provided to post messages.  
    - Inputs: Triggered by Cron node output.  
    - Outputs: None (terminal node).  
    - Potential Failures:  
      - Invalid webhook URL or revoked webhook leads to authentication or permission errors.  
      - Network issues or Discord API rate limiting.  
      - Message content too large or malformed could cause request failures.

---

#### 2.2 Weekly Friday Meme Delivery

- **Overview:**  
  This block triggers every Friday at 9:00 AM and sends a Friday-themed meme and message to the Discord channel.

- **Nodes Involved:**  
  - Cron1  
  - Discord1

- **Node Details:**

  - **Cron1**  
    - Type: Scheduled trigger node.  
    - Configuration: Set to trigger every week on Friday at 9:00 (hour=9, weekday=5, mode=everyWeek).  
    - Inputs: None.  
    - Outputs: Connected to Discord1 node.  
    - Potential Failures: Same as Cron node in 2.1.

  - **Discord1**  
    - Type: Discord message sender node.  
    - Configuration:  
      - Text message:  
        ```
        It's Friday, Friday
        Gotta get down on Friday!
        https://tenor.com/view/rebecca-black-friday-tgif-gif-4051598
        ```  
      - Same webhook URL as previous Discord node.  
    - Inputs: From Cron1.  
    - Outputs: None.  
    - Potential Failures: Same as Discord node in 2.1.

---

#### 2.3 Repeating 30-minute Interval Meme Delivery

- **Overview:**  
  This block triggers every 30 minutes continuously and sends a good night themed meme message to the Discord channel.

- **Nodes Involved:**  
  - Cron2  
  - Discord2

- **Node Details:**

  - **Cron2**  
    - Type: Interval trigger node.  
    - Configuration: Set to trigger every 30 minutes (mode=everyX, unit=minutes, value=30).  
    - Inputs: None.  
    - Outputs: Connected to Discord2 node.  
    - Potential Failures:  
      - If workflow is inactive, triggers won't execute.  
      - Cron node misconfiguration could cause irregular intervals.

  - **Discord2**  
    - Type: Discord message sender node.  
    - Configuration:  
      - Text message:  
        ```
        And with this, I sleep. Good night Pogger friends :)
        https://cdn.discordapp.com/attachments/756602216621539409/757054027518443600/93109046_836460460092895_6176715527851028509_n.jpg
        ```  
      - Same webhook URL as other Discord nodes.  
    - Inputs: From Cron2.  
    - Outputs: None.  
    - Potential Failures: Same as Discord nodes above.

---

### 3. Summary Table

| Node Name | Node Type         | Functional Role                    | Input Node(s) | Output Node(s) | Sticky Note                                                     |
|-----------|-------------------|----------------------------------|---------------|----------------|----------------------------------------------------------------|
| Cron      | Cron Trigger      | Weekly Wednesday schedule trigger| None          | Discord        |                                                                |
| Discord   | Discord           | Sends Wednesday meme message     | Cron          | None           |                                                                |
| Cron1     | Cron Trigger      | Weekly Friday schedule trigger   | None          | Discord1       |                                                                |
| Discord1  | Discord           | Sends Friday meme message        | Cron1         | None           |                                                                |
| Cron2     | Cron Trigger      | Every 30 minutes interval trigger| None          | Discord2       |                                                                |
| Discord2  | Discord           | Sends good night meme message    | Cron2         | None           |                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it "Send memes via Discord".**

2. **Add the first scheduling block (Wednesday meme):**  
   - Add a **Cron** node.  
     - Set mode to **"Every Week"**.  
     - Set weekday to **Wednesday (6)**.  
     - Set hour to **9** (9:00 AM).  
   - Add a **Discord** node.  
     - Paste the Discord webhook URL in the webhook URI field.  
     - Set the message text to:  
       ```
       It's Wednesday, my dudes!
       https://i.kym-cdn.com/entries/icons/original/000/020/016/wednesdaymydudeswide.jpg
       ```  
   - Connect the Cron node output to the Discord node input.

3. **Add the second scheduling block (Friday meme):**  
   - Add a second **Cron** node (name it Cron1).  
     - Set mode to **"Every Week"**.  
     - Set weekday to **Friday (5)**.  
     - Set hour to **9** (9:00 AM).  
   - Add a second **Discord** node (name it Discord1).  
     - Use the same webhook URL as above.  
     - Set message text to:  
       ```
       It's Friday, Friday
       Gotta get down on Friday!
       https://tenor.com/view/rebecca-black-friday-tgif-gif-4051598
       ```  
   - Connect Cron1 output to Discord1 input.

4. **Add the third scheduling block (30-minute repeating meme):**  
   - Add a third **Cron** node (name it Cron2).  
     - Set mode to **"Every X"**.  
     - Set unit to **"minutes"**.  
     - Set value to **30**.  
   - Add a third **Discord** node (name it Discord2).  
     - Use the same webhook URL as above.  
     - Set message text to:  
       ```
       And with this, I sleep. Good night Pogger friends :)
       https://cdn.discordapp.com/attachments/756602216621539409/757054027518443600/93109046_836460460092895_6176715527851028509_n.jpg
       ```  
   - Connect Cron2 output to Discord2 input.

5. **Activate the workflow** if you want it to start sending messages as scheduled. Note the original workflow’s "active" flag is false, so it will not run unless activated.

6. **Credential Configuration:**  
   - No special credentials for Discord nodes are required if using the webhook URL directly.  
   - Ensure the webhook URL is valid and has the necessary permissions to post messages to the target Discord channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                              |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow is inactive by default; activate it to enable scheduled meme posting.                                            | n8n workflow setting                                         |
| Discord webhook URLs used here must be kept secure to prevent unauthorized posting.                                            | Discord developer documentation on webhooks                  |
| Memes are sent as plain text messages containing URLs to images/GIFs hosted externally (e.g., tenor.com, discordcdn.com).      | Ensure external links remain valid to avoid broken images.   |
| For more advanced Discord integrations, consider Discord bots with OAuth2 instead of webhooks.                                | https://discord.com/developers/docs/intro                    |

---

This document provides a complete, structured reference to understand, reproduce, and modify the "Send memes via Discord" workflow safely and efficiently.