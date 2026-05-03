Daily Affirmations & Weekly Gratitude Digest with Notion, Email & Telegram

https://n8nworkflows.xyz/workflows/daily-affirmations---weekly-gratitude-digest-with-notion--email---telegram-7555


# Daily Affirmations & Weekly Gratitude Digest with Notion, Email & Telegram

### 1. Workflow Overview

This workflow automates the sending of daily affirmations and a weekly gratitude digest using Notion as the data source and log, and delivers messages via Email and Telegram. It also includes error handling with alerts sent to Slack or Discord. The workflow is designed for wellness and mindfulness use cases, helping users receive daily motivational content and a weekly summary of gratitude entries.

Logical blocks in the workflow:

- **1.1 Scheduling Triggers**: Two cron nodes trigger daily affirmations and weekly gratitude digests.
- **1.2 Daily Affirmation Flow**: Retrieves user configuration, builds the affirmation message, sends it via Email and/or Telegram, and logs it to Notion.
- **1.3 Weekly Gratitude Digest Flow**: Defines a one-week time window, queries gratitude entries from Notion, processes and formats them, then sends a digest via Email and/or Telegram.
- **1.4 Error Handling and Alerts**: Captures errors from any node, assembles an alert message, waits briefly, and conditionally sends alerts to Slack and/or Discord.
- **1.5 Configuration Setups and Notes**: Nodes for setting static configuration variables and sticky notes for documentation or instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling Triggers

- **Overview:** This block contains the cron nodes that initiate the workflow runs for daily and weekly messages.

- **Nodes Involved:**  
  - Cron: Daily Affirmation  
  - Cron: Weekly Digest

- **Node Details:**

  - **Cron: Daily Affirmation**  
    - Type: Cron Trigger  
    - Configuration: Default schedule (specific timing not shown; to be configured by user)  
    - Inputs: None  
    - Outputs: Triggers "Set: User Config" node  
    - Edge Cases: Misconfiguration of cron schedule can cause missed or multiple triggers.

  - **Cron: Weekly Digest**  
    - Type: Cron Trigger  
    - Configuration: Default schedule (weekly, time unspecified)  
    - Inputs: None  
    - Outputs: Triggers "Set: Digest Config" node  
    - Edge Cases: Same as above regarding schedule misconfiguration.

#### 2.2 Daily Affirmation Flow

- **Overview:** Handles preparing and sending daily affirmations based on user configurations and logs them in Notion.

- **Nodes Involved:**  
  - Set: User Config  
  - Fn: Build Affirmation Message  
  - IF: Send Email?  
  - Email: Send Affirmation  
  - IF: Send Telegram?  
  - Telegram: Send Affirmation  
  - Notion: Log Affirmation

- **Node Details:**

  - **Set: User Config**  
    - Type: Set Node  
    - Role: Defines user-specific settings such as recipient emails, Telegram chat IDs, affirmation content parameters, and flags controlling message sending.  
    - Inputs: Triggered by "Cron: Daily Affirmation"  
    - Outputs: Passes data to "Fn: Build Affirmation Message"  
    - Edge Cases: Missing or incorrect config data could cause failures in downstream nodes.

  - **Fn: Build Affirmation Message**  
    - Type: Function  
    - Role: Constructs the affirmation message text using user config or other data.  
    - Inputs: From "Set: User Config"  
    - Outputs: To "IF: Send Email?", "IF: Send Telegram?", and "Notion: Log Affirmation"  
    - Key Expressions: Usually includes string concatenations and formatting to produce the final message.  
    - Edge Cases: JavaScript errors if variables are undefined or improperly formatted.

  - **IF: Send Email?**  
    - Type: If  
    - Role: Checks whether sending affirmation via Email is enabled by user config flag.  
    - Inputs: From "Fn: Build Affirmation Message"  
    - Outputs: If true, to "Email: Send Affirmation"; else no output.  
    - Edge Cases: Misconfigured flag could prevent sending or cause unnecessary attempts.

  - **Email: Send Affirmation**  
    - Type: Email Send  
    - Role: Sends the affirmation message via configured email service.  
    - Inputs: Affirmation message and recipient details from previous nodes.  
    - Outputs: None (end node)  
    - Requires: Proper email credentials configured (SMTP or other supported).  
    - Edge Cases: Auth errors, connectivity issues, invalid email addresses.

  - **IF: Send Telegram?**  
    - Type: If  
    - Role: Checks if Telegram sending is enabled.  
    - Inputs: From "Fn: Build Affirmation Message"  
    - Outputs: If true, to "Telegram: Send Affirmation"  
    - Edge Cases: Same as email regarding flag misconfiguration.

  - **Telegram: Send Affirmation**  
    - Type: Telegram  
    - Role: Sends affirmation message to Telegram chat/group specified in config.  
    - Inputs: Message from previous node.  
    - Requires: Telegram Bot credentials and chat ID setup.  
    - Edge Cases: Invalid bot token, chat ID errors, network issues.

  - **Notion: Log Affirmation**  
    - Type: Notion  
    - Role: Logs the sent affirmation into a Notion database for record-keeping.  
    - Inputs: Affirmation message and metadata.  
    - Requires: Notion API credentials and database ID configured.  
    - Edge Cases: API limits, permission errors, malformed data.

#### 2.3 Weekly Gratitude Digest Flow

- **Overview:** Collects gratitude entries from Notion for the past week, filters and formats them, then sends a digest via Email and Telegram.

- **Nodes Involved:**  
  - Set: Digest Config  
  - Fn: Set Week Window  
  - Notion: Query Gratitude (7d)  
  - Fn: Filter & Format Gratitude (7d)  
  - Fn: Build Digest Message  
  - Email: Send Weekly Digest  
  - Telegram: Send Weekly Digest

- **Node Details:**

  - **Set: Digest Config**  
    - Type: Set  
    - Role: Sets configuration data for the weekly digest, including flags for sending methods and Notion database parameters.  
    - Inputs: Triggered by "Cron: Weekly Digest"  
    - Outputs: "Fn: Set Week Window"  
    - Edge Cases: Missing config leads to query failures or no messages sent.

  - **Fn: Set Week Window**  
    - Type: Function  
    - Role: Computes the ISO timestamp for one week ago to filter Notion entries.  
    - Logic: Uses JavaScript to get current date minus 7 days, outputs ISO string as `oneWeekAgoISO`.  
    - Inputs: From "Set: Digest Config"  
    - Outputs: "Notion: Query Gratitude (7d)"  
    - Edge Cases: Timezone inconsistencies could affect data range.

  - **Notion: Query Gratitude (7d)**  
    - Type: Notion  
    - Role: Queries Notion database for gratitude entries newer than `oneWeekAgoISO`.  
    - Requires: Notion API credentials and database configured with filter by date property.  
    - Inputs: From "Fn: Set Week Window"  
    - Outputs: "Fn: Filter & Format Gratitude (7d)"  
    - Edge Cases: API limit errors, bad query syntax, permission denied.

  - **Fn: Filter & Format Gratitude (7d)**  
    - Type: Function  
    - Role: Processes the array of gratitude entries, filters if needed, and formats them into a list or summary.  
    - Inputs: Query results from Notion  
    - Outputs: "Fn: Build Digest Message"  
    - Edge Cases: Empty results, malformed data, JavaScript errors.

  - **Fn: Build Digest Message**  
    - Type: Function  
    - Role: Constructs the weekly digest message text based on formatted gratitude data.  
    - Inputs: From filter/format function  
    - Outputs: Two outputs — to "Email: Send Weekly Digest" and "Telegram: Send Weekly Digest"  
    - Edge Cases: Empty messages, formatting errors.

  - **Email: Send Weekly Digest**  
    - Type: Email Send  
    - Role: Sends the weekly gratitude digest via email.  
    - Inputs: Digest message and recipient details.  
    - Requires: Email credentials.  
    - Edge Cases: Same as daily email send node.

  - **Telegram: Send Weekly Digest**  
    - Type: Telegram  
    - Role: Sends the weekly digest to Telegram chat/group.  
    - Inputs: Digest message.  
    - Requires: Telegram credentials.  
    - Edge Cases: Same as daily Telegram send node.

#### 2.4 Error Handling and Alerts

- **Overview:** Captures errors from any node, constructs an alert message, waits briefly, and sends alerts to Slack and/or Discord based on config.

- **Nodes Involved:**  
  - On Error: Any Node  
  - Set: Alert Config  
  - Fn: Build Alert Message  
  - Wait 5s  
  - IF: Slack?  
  - IF: Discord?  
  - Slack: Post Alert  
  - Discord: Post Alert

- **Node Details:**

  - **On Error: Any Node**  
    - Type: Error Trigger  
    - Role: Listens for failures in any node in the workflow to initiate alert process.  
    - Outputs: "Set: Alert Config"  
    - Edge Cases: If error trigger is disabled, errors may go unnoticed.

  - **Set: Alert Config**  
    - Type: Set  
    - Role: Defines alert details including Slack and Discord webhook URLs or flags.  
    - Inputs: From error trigger  
    - Outputs: "Fn: Build Alert Message"  
    - Edge Cases: Missing webhook URLs disables alert sending.

  - **Fn: Build Alert Message**  
    - Type: Function  
    - Role: Creates a detailed alert message including error info, node name, and workflow context.  
    - Inputs: Alert config and error data  
    - Outputs: "Wait 5s"  
    - Edge Cases: Function errors prevent alert sending.

  - **Wait 5s**  
    - Type: Wait  
    - Role: Delays execution for 5 seconds, possibly to ensure error logs are fully processed.  
    - Inputs: Alert message  
    - Outputs: "IF: Slack?" and "IF: Discord?"  
    - Edge Cases: N/A

  - **IF: Slack?**  
    - Type: If  
    - Role: Checks if Slack alerts are enabled.  
    - Outputs: Sends to "Slack: Post Alert" if true  
    - Edge Cases: Misconfig flag prevents alert posting.

  - **Slack: Post Alert**  
    - Type: Slack  
    - Role: Posts alert message to Slack channel via webhook or API.  
    - Requires: Slack credentials / webhook URL  
    - Edge Cases: Auth errors, webhook URL errors.

  - **IF: Discord?**  
    - Type: If  
    - Role: Checks if Discord alerts are enabled.  
    - Outputs: Sends to "Discord: Post Alert" if true  
    - Edge Cases: Same as Slack IF node.

  - **Discord: Post Alert**  
    - Type: Discord  
    - Role: Posts alert message to Discord channel via webhook.  
    - Requires: Discord webhook URL  
    - Edge Cases: Invalid webhook or network errors.

#### 2.5 Configuration Setups and Sticky Notes

- **Overview:** Nodes setting static configurations or holding documentation notes within the workflow UI.

- **Nodes Involved:**  
  - README (Sticky Note)  
  - Sticky: Setup (Sticky Note)  
  - Set: User Config (also belongs to daily block)  
  - Set: Digest Config (also belongs to weekly block)  
  - Set: Alert Config (also belongs to error block)

- **Node Details:**

  - **Sticky Notes**  
    - Type: Sticky Note  
    - Role: For documentation or reminders; currently empty in this workflow.  
    - No input or output connections.

  - **Set Nodes**  
    - Role: Define configuration parameters for the respective blocks, such as sending flags, recipient info, and API credentials references.  
    - These nodes are essential for controlling workflow behavior without hardcoding in functions.

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                        | Input Node(s)             | Output Node(s)                         | Sticky Note                      |
|----------------------------|---------------------------|-------------------------------------|---------------------------|---------------------------------------|---------------------------------|
| README                     | Sticky Note               | Documentation placeholder            | —                         | —                                     |                                 |
| Sticky: Setup              | Sticky Note               | Documentation placeholder            | —                         | —                                     |                                 |
| Cron: Daily Affirmation    | Cron Trigger              | Triggers daily affirmation flow      | —                         | Set: User Config                      |                                 |
| Cron: Weekly Digest        | Cron Trigger              | Triggers weekly digest flow          | —                         | Set: Digest Config                    |                                 |
| Set: User Config           | Set                       | Defines daily affirmation config     | Cron: Daily Affirmation    | Fn: Build Affirmation Message         |                                 |
| Fn: Build Affirmation Message | Function                | Builds daily affirmation text        | Set: User Config          | IF: Send Email?, IF: Send Telegram?, Notion: Log Affirmation |                                 |
| IF: Send Email?            | If                        | Checks if email sending enabled      | Fn: Build Affirmation Message | Email: Send Affirmation              |                                 |
| Email: Send Affirmation    | Email Send                | Sends affirmation via email          | IF: Send Email?            | —                                     |                                 |
| IF: Send Telegram?         | If                        | Checks if Telegram sending enabled   | Fn: Build Affirmation Message | Telegram: Send Affirmation           |                                 |
| Telegram: Send Affirmation | Telegram                  | Sends affirmation via Telegram       | IF: Send Telegram?         | —                                     |                                 |
| Notion: Log Affirmation    | Notion                    | Logs affirmation to Notion           | Fn: Build Affirmation Message | —                                   |                                 |
| Set: Digest Config         | Set                       | Defines weekly digest config         | Cron: Weekly Digest        | Fn: Set Week Window                   |                                 |
| Fn: Set Week Window        | Function                  | Calculates one week ago timestamp    | Set: Digest Config         | Notion: Query Gratitude (7d)          |                                 |
| Notion: Query Gratitude (7d) | Notion                  | Queries gratitude entries from Notion | Fn: Set Week Window        | Fn: Filter & Format Gratitude (7d)    |                                 |
| Fn: Filter & Format Gratitude (7d) | Function           | Filters and formats gratitude data  | Notion: Query Gratitude (7d) | Fn: Build Digest Message            |                                 |
| Fn: Build Digest Message   | Function                  | Builds weekly digest message         | Fn: Filter & Format Gratitude (7d) | Email: Send Weekly Digest, Telegram: Send Weekly Digest |                                 |
| Email: Send Weekly Digest  | Email Send                | Sends weekly digest via email        | Fn: Build Digest Message   | —                                     |                                 |
| Telegram: Send Weekly Digest | Telegram                | Sends weekly digest via Telegram     | Fn: Build Digest Message   | —                                     |                                 |
| On Error: Any Node         | Error Trigger             | Captures errors in any node          | —                         | Set: Alert Config                     |                                 |
| Set: Alert Config          | Set                       | Defines alert sending config         | On Error: Any Node         | Fn: Build Alert Message               |                                 |
| Fn: Build Alert Message    | Function                  | Builds error alert message           | Set: Alert Config          | Wait 5s                              |                                 |
| Wait 5s                   | Wait                      | Delays for 5 seconds before alerts   | Fn: Build Alert Message    | IF: Slack?, IF: Discord?              |                                 |
| IF: Slack?                 | If                        | Checks if Slack alerts enabled       | Wait 5s                   | Slack: Post Alert (true), none (false) |                               |
| Slack: Post Alert          | Slack                     | Sends alert message to Slack         | IF: Slack?                | —                                     |                                 |
| IF: Discord?               | If                        | Checks if Discord alerts enabled     | Wait 5s                   | Discord: Post Alert (true), none (false) |                              |
| Discord: Post Alert        | Discord                   | Sends alert message to Discord       | IF: Discord?              | —                                     |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Nodes**  
   - Create two Cron nodes:  
     - "Cron: Daily Affirmation" — set to trigger every day at desired time.  
     - "Cron: Weekly Digest" — set to trigger weekly at desired time.

2. **Set Up User Configuration for Daily Affirmation**  
   - Add a Set node named "Set: User Config".  
   - Define fields such as:  
     - Email recipients, Telegram chat IDs, flags to enable/disable email and Telegram sending, affirmation message templates or parameters.  
   - Connect "Cron: Daily Affirmation" to "Set: User Config".

3. **Build Affirmation Message**  
   - Add a Function node "Fn: Build Affirmation Message".  
   - Write code to construct the affirmation message from user config and any other data.  
   - Connect "Set: User Config" to this node.

4. **Conditional Sending for Email and Telegram**  
   - Add two If nodes: "IF: Send Email?" and "IF: Send Telegram?".  
   - Each checks respective boolean flags in data to decide whether to send.  
   - Connect "Fn: Build Affirmation Message" output to both If nodes.

5. **Send Affirmation via Email**  
   - Add Email Send node "Email: Send Affirmation".  
   - Configure with SMTP or appropriate email credentials.  
   - Use message and recipient data from previous nodes.  
   - Connect "IF: Send Email?" true output to this node.

6. **Send Affirmation via Telegram**  
   - Add Telegram node "Telegram: Send Affirmation".  
   - Configure Telegram Bot credentials and chat ID.  
   - Connect "IF: Send Telegram?" true output to this node.

7. **Log Affirmation in Notion**  
   - Add Notion node "Notion: Log Affirmation".  
   - Configure with Notion API credentials and target database.  
   - Connect "Fn: Build Affirmation Message" output to this node.

8. **Set Up Weekly Digest Configuration**  
   - Add Set node "Set: Digest Config" defining flags for sending methods and Notion database info.  
   - Connect "Cron: Weekly Digest" to this node.

9. **Calculate One Week Ago Timestamp**  
   - Add Function node "Fn: Set Week Window".  
   - Code: `const iso = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString(); return [{ json: { oneWeekAgoISO: iso } }];`  
   - Connect "Set: Digest Config" to this node.

10. **Query Gratitude Entries in Notion**  
    - Add Notion node "Notion: Query Gratitude (7d)".  
    - Configure with Notion credentials and database; filter entries with date >= `oneWeekAgoISO` from input.  
    - Connect "Fn: Set Week Window" to this node.

11. **Filter and Format Gratitude Entries**  
    - Add Function node "Fn: Filter & Format Gratitude (7d)".  
    - Write code to process Notion query results and format them into digest content.  
    - Connect "Notion: Query Gratitude (7d)" to this node.

12. **Build Digest Message**  
    - Add Function node "Fn: Build Digest Message".  
    - Construct the final digest message text.  
    - Connect "Fn: Filter & Format Gratitude (7d)" to this node.

13. **Send Weekly Digest via Email**  
    - Add Email Send node "Email: Send Weekly Digest".  
    - Configure with email credentials and recipient info.  
    - Connect first output of "Fn: Build Digest Message" to this node.

14. **Send Weekly Digest via Telegram**  
    - Add Telegram node "Telegram: Send Weekly Digest".  
    - Configure Telegram Bot credentials and chat ID.  
    - Connect second output of "Fn: Build Digest Message" to this node.

15. **Configure Error Handling**  
    - Add Error Trigger node "On Error: Any Node" to catch errors globally.  
    - Add Set node "Set: Alert Config" to define alert sending parameters (Slack, Discord webhook URLs, flags).  
    - Connect Error Trigger to "Set: Alert Config".

16. **Build Alert Message**  
    - Add Function node "Fn: Build Alert Message" to create alert content from error info.  
    - Connect "Set: Alert Config" to this function.

17. **Add Wait Node**  
    - Add Wait node "Wait 5s" to delay execution briefly before sending alerts.  
    - Connect "Fn: Build Alert Message" to "Wait 5s".

18. **Conditional Alert Sending**  
    - Add two If nodes: "IF: Slack?" and "IF: Discord?" checking respective flags.  
    - Connect "Wait 5s" to both.

19. **Send Alerts to Slack and Discord**  
    - Add Slack node "Slack: Post Alert" configured with Slack credentials/webhook.  
    - Connect "IF: Slack?" true output to this node.  
    - Add Discord node "Discord: Post Alert" configured with Discord webhook.  
    - Connect "IF: Discord?" true output to this node.

20. **Optional Sticky Notes**  
    - Add Sticky Note nodes "README" and "Sticky: Setup" for workflow documentation and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                |
|---------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow requires valid credentials for Email (SMTP or service), Telegram Bot API, Notion API, Slack webhook, and Discord webhook. | Setup instructions depend on user environment and API keys.  |
| Workflow timezone is set to use the workflow's timezone setting dynamically.                                                          | Workflow settings > Timezone                                   |
| The error handling uses a 5-second wait before sending alerts to ensure any asynchronous logs or error data are fully captured.       | Best practice for error alerting                               |
| For Notion integration, ensure database properties and filters match the expected query parameters.                                   | Notion API documentation: https://developers.notion.com/     |
| Telegram Bot requires adding the bot to the target chat or channel with required permissions.                                          | Telegram Bot API documentation: https://core.telegram.org/bots/api |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.