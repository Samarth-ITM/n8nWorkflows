Send automated daily reminders on Telegram

https://n8nworkflows.xyz/workflows/send-automated-daily-reminders-on-telegram-1387


# Send automated daily reminders on Telegram

### 1. Workflow Overview

This workflow, titled **"Daily Journal Reminder"**, is designed to automatically send a daily reminder message via Telegram every morning at 6 AM. Its primary use case is to encourage the user to write one thing they did the previous day, supporting a personal daily journaling habit. The workflow can be optionally extended by integrating a second workflow (linked in the description) that enables responding to the Telegram reminder and recording the reply in a Google Sheet.

The workflow consists of three main logical blocks:

- **1.1 Trigger Block:** Initiates the workflow on a daily schedule (6 AM).
- **1.2 Message Formatting Block:** Creates a custom reminder message referencing yesterday’s date.
- **1.3 Notification Block:** Sends the formatted message to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:**  
  This block triggers the workflow automatically every day at 6 AM, ensuring the reminder is sent at a consistent time.

- **Nodes Involved:**  
  - Morning reminder

- **Node Details:**  
  - **Morning reminder**  
    - Type: Cron Trigger  
    - Role: Schedules the workflow execution daily at a fixed hour.  
    - Configuration: Set to trigger at 6:00 AM every day (hour: 6).  
    - Key Parameters: `triggerTimes` with hour set to 6.  
    - Input: None (start node).  
    - Output: Triggers the next node `format reminder`.  
    - Version: n8n Cron node version 1 used.  
    - Edge Cases / Errors:  
      - Workflow will not trigger if n8n is offline at the scheduled time.  
      - Timezone considerations: The node uses n8n’s server timezone, so ensure it matches user expectations.  
    - Notes: Has a sticky note “Trigger very morning”.

#### 1.2 Message Formatting Block

- **Overview:**  
  This block generates the text content of the reminder message, dynamically inserting yesterday’s date in ISO format (YYYY-MM-DD).

- **Nodes Involved:**  
  - format reminder

- **Node Details:**  
  - **format reminder**  
    - Type: Function Item node  
    - Role: Runs JavaScript code to produce the message text.  
    - Configuration:  
      - The JavaScript code creates a Date object for today, then subtracts one day to get yesterday’s date.  
      - Constructs a string: `"What did you do: YYYY-MM-DD"` using yesterday’s date.  
    - Key Expression: Inline JavaScript code returning `{ message: "..." }`.  
    - Input: Triggered by `Morning reminder`.  
    - Output: Passes `{ message }` to the Telegram node.  
    - Version: n8n Function Item node version 1.  
    - Edge Cases / Errors:  
      - Date calculation assumes server timezone; may mismatch user timezone if server is elsewhere.  
      - JavaScript errors unlikely but if code is modified, might cause workflow failure.  
      - If date format needs localization, code must be adapted.  

#### 1.3 Notification Block

- **Overview:**  
  Sends the generated reminder message to a specific Telegram chat ID.

- **Nodes Involved:**  
  - Send journal reminder

- **Node Details:**  
  - **Send journal reminder**  
    - Type: Telegram node  
    - Role: Sends text message to Telegram chat.  
    - Configuration:  
      - Text message is dynamically set using an expression referencing `format reminder` output: `={{$node["format reminder"].json["message"]}}`.  
      - Chat ID is hardcoded as `666884239` (must be replaced with user’s Telegram chat ID).  
      - No additional fields are configured.  
    - Credentials: Requires Telegram Bot credentials to be configured in n8n (not included in workflow JSON).  
    - Input: Receives message from `format reminder`.  
    - Output: None further in this workflow.  
    - Version: Telegram node version 1.  
    - Edge Cases / Errors:  
      - Missing or invalid Telegram credentials will cause authentication errors.  
      - Incorrect chat ID will cause message delivery failure.  
      - Telegram API rate limits or downtime may cause temporary failures.  
      - Message text is limited by Telegram’s maximum message length (~4096 characters); no issue here as message is short.  
    - Notes: Sticky note advises to configure Telegram credentials.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                    | Input Node(s)      | Output Node(s)        | Sticky Note                              |
|---------------------|--------------------|----------------------------------|--------------------|-----------------------|------------------------------------------|
| Morning reminder    | Cron Trigger       | Daily scheduler at 6 AM           | -                  | format reminder       | Trigger very morning                     |
| format reminder     | Function Item      | Create reminder message with date | Morning reminder   | Send journal reminder |                                          |
| Send journal reminder | Telegram          | Send message to Telegram chat     | format reminder    | -                     | Make sure to configure your Telegram credentials! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Cron Trigger node**  
   - Name: `Morning reminder`  
   - Type: Cron Trigger  
   - Configuration: Set `Trigger Times` → `Hour` to `6` (to run daily at 6 AM).  
   - No credentials required.  
   - Position: Place it to the left.

2. **Add a Function Item node**  
   - Name: `format reminder`  
   - Type: Function Item  
   - Configuration: Insert the following JavaScript code:  
     ```javascript
     const today = new Date();
     const yesterday = new Date(today);
     yesterday.setDate(yesterday.getDate() - 1);
     const message = `What did you do: ${yesterday.toISOString().split('T')[0]}`;
     return { message };
     ```  
   - Connect `Morning reminder` main output to `format reminder` input.

3. **Add a Telegram node**  
   - Name: `Send journal reminder`  
   - Type: Telegram  
   - Configuration:  
     - Set Text to expression: `={{$node["format reminder"].json["message"]}}`  
     - Set Chat ID to your Telegram chat identifier (replace `666884239` with your own).  
   - Credentials: Configure Telegram Bot credentials with appropriate bot token and permissions.  
   - Connect `format reminder` output to this node input.

4. **Activate the workflow**  
   - Save and activate the workflow to enable daily execution.

**Optional:**  
- To enable replying to the message and recording inputs in Google Sheets, import and configure the linked [second workflow](https://n8n.io/workflows/1388) as described in the original workflow note.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                       |
|------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| The workflow is intended to support a 2022 goal of journaling one thing done each day.         | Workflow description                                                 |
| Optional integration with a second workflow allows replies to be recorded in Google Sheets.    | https://n8n.io/workflows/1388                                         |
| Ensure Telegram bot credentials are properly configured in n8n to send messages successfully. | Sticky note on Telegram node                                         |
| Timezone of the Cron node matches the n8n server’s timezone, which may affect message timing.  | General n8n Cron node behavior                                       |