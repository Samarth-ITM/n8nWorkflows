Scheduled Daily Affirmations via Email and Telegram using Cron

https://n8nworkflows.xyz/workflows/scheduled-daily-affirmations-via-email-and-telegram-using-cron-7421


# Scheduled Daily Affirmations via Email and Telegram using Cron

### 1. Workflow Overview

This n8n workflow automates the delivery of daily positive affirmations via email and optionally via Telegram. It is designed for users who want a gentle, automated daily boost of positivity sent directly to their inbox and/or Telegram chat every morning at 7 AM.

The workflow comprises the following logical blocks:

- **1.1 Scheduled Trigger:** A Cron node triggers the workflow daily at 7 AM.
- **1.2 Configuration Setup:** A Set node holds all user-configurable parameters such as email addresses, Telegram chat ID, subject line, and the list of affirmations.
- **1.3 Affirmation Selection:** A Code node randomly selects one affirmation from the configured list.
- **1.4 Notification Sending:** An Email node sends the chosen affirmation to the configured email address. A conditional IF node checks if Telegram sending is enabled, and if so, a Telegram node sends the affirmation to the specified chat.
- **1.5 Documentation & Setup Assistance:** Sticky Notes provide user instructions and setup tips embedded in the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Triggers the entire workflow once daily at 7 AM, initiating the affirmation sending process.

- **Nodes Involved:**  
  - Cron: Trigger Daily at 7 AM

- **Node Details:**  
  - **Cron: Trigger Daily at 7 AM**  
    - Type: Cron Trigger  
    - Configuration: Set to trigger the workflow at hour 7 (7 AM) every day. No minutes or seconds specified, so triggers exactly at 7:00 AM.  
    - Input: None (start node)  
    - Output: Emits a single execution event daily.  
    - Edge Cases: If the server running n8n is down or paused at trigger time, the event will be missed unless n8n is configured to run missed triggers.  
    - Version requirements: Standard Cron node available in n8n v1+.  

#### 2.2 Configuration Setup

- **Overview:**  
  Holds all user-customizable settings including email sender/recipient, email subject, Telegram chat ID, Telegram enable flag, and the list of affirmations. This centralized configuration allows easy editing without modifying code or nodes elsewhere.

- **Nodes Involved:**  
  - Set: Configuration (edit me)

- **Node Details:**  
  - **Set: Configuration (edit me)**  
    - Type: Set (data manipulation node)  
    - Configuration:  
      - `fromEmail`: Sender’s email address (string)  
      - `toEmail`: Recipient’s email address (string)  
      - `subject`: Email subject line (string), default "🌸 Morning Affirmation"  
      - `telegramChatId`: Telegram chat identifier (string). Leave empty to disable Telegram sending.  
      - `affirmations`: Multiline string with affirmations separated by new lines.  
      - `sendTelegram`: Boolean flag to enable or disable Telegram sending (default false if unset).  
    - Input: Trigger from Cron node  
    - Output: Passes all configuration data downstream for use by other nodes  
    - Edge Cases:  
      - Missing or invalid email addresses will cause email send failures.  
      - Empty affirmations list falls back to default affirmation in code node.  
      - If `sendTelegram` is false or missing, Telegram sending is skipped.  
    - Version requirements: Uses v2 typeVersion for Set node for enhanced features.  

#### 2.3 Affirmation Selection

- **Overview:**  
  Processes the multiline string of affirmations, splits them into an array, filters out empty lines, and randomly picks one affirmation per day. Provides a default fallback affirmation if the list is empty.

- **Nodes Involved:**  
  - Code: Pick Random Affirmation

- **Node Details:**  
  - **Code: Pick Random Affirmation**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Parses `$json.affirmations` string by splitting on newlines.  
      - Trims each line, removes empty strings.  
      - Randomly selects one affirmation.  
      - If no affirmations exist, defaults to "I am enough."  
      - Returns the chosen affirmation as a new JSON property `affirmation` in the output.  
    - Input: Configuration data from Set node  
    - Output: The same JSON object enriched with `affirmation` property  
    - Edge Cases:  
      - If affirmations string is empty or malformed, defaults to fallback.  
      - If expression fails, workflow will error; error handling is manual.  
    - Version requirements: Requires n8n supporting v2 JavaScript code nodes.  

#### 2.4 Notification Sending

- **Overview:**  
  Sends the selected affirmation via email (mandatory) and optionally via Telegram (conditional). The Telegram send is gated by an IF node checking the `sendTelegram` flag.

- **Nodes Involved:**  
  - Email: Send Affirmation  
  - IF: Telegram Enabled?  
  - Telegram: Send Affirmation

- **Node Details:**  

  - **Email: Send Affirmation**  
    - Type: Email Send  
    - Configuration:  
      - `fromEmail`, `toEmail`, `subject` and `text` are dynamically set using expressions from incoming JSON.  
      - Uses SMTP credentials configured in n8n (credential ID MX4srF0whebaDqig in this instance).  
    - Input: Affirmation JSON from Code node  
    - Output: Passes execution forward (not connected further here)  
    - Edge Cases:  
      - SMTP credential failure or misconfiguration causes errors.  
      - Invalid emails or network issues may cause delivery failures.  
    - Version requirements: Uses v2 node version.  

  - **IF: Telegram Enabled?**  
    - Type: IF (boolean condition)  
    - Configuration: Checks if `$json.sendTelegram` is true.  
    - Input: Affirmation data from Code node (via Email node output connection)  
    - Output: Main output if true, no output if false  
    - Edge Cases: If `sendTelegram` is missing or false, Telegram sending skipped.  

  - **Telegram: Send Affirmation**  
    - Type: Telegram node (send message)  
    - Configuration:  
      - Sends text message with prefix "🌸 " plus the affirmation text.  
      - Uses Telegram Bot credentials configured in n8n environment.  
      - `chatId` dynamically set from `$json.telegramChatId`.  
    - Input: Affirmation JSON from IF node’s true branch  
    - Output: None (end node)  
    - Edge Cases:  
      - Missing or invalid `telegramChatId` or credentials cause failures.  
      - Telegram API rate limits or downtime may cause errors.  
    - Version requirements: Node version 1.2+ recommended for Telegram node.  

#### 2.5 Documentation & Setup Assistance

- **Overview:**  
  Sticky Notes provide embedded documentation and setup checklists for users importing and configuring the workflow.

- **Nodes Involved:**  
  - README - Template Description  
  - Setup Tips

- **Node Details:**  
  - **README - Template Description**  
    - Type: Sticky Note  
    - Content explains the target audience, what the workflow does, how to set up, requirements, and customization tips.  
    - Positioned prominently for user reference.  

  - **Setup Tips**  
    - Type: Sticky Note  
    - Provides a quick checklist for setup steps including credential addition and testing suggestions.  

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                         | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                             |
|----------------------------|---------------------|---------------------------------------|----------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| README - Template Description | Sticky Note         | Documentation and user instructions   | None                       | None                          | Explains target users, workflow purpose, setup instructions, and customization options.                                  |
| Cron: Trigger Daily at 7 AM | Cron                | Starts workflow daily at 7 AM          | None                       | Set: Configuration (edit me)  |                                                                                                                         |
| Set: Configuration (edit me) | Set                 | Holds user-configurable parameters    | Cron                       | Code: Pick Random Affirmation |                                                                                                                         |
| Code: Pick Random Affirmation | Code                | Randomly selects daily affirmation    | Set: Configuration         | Email: Send Affirmation        |                                                                                                                         |
| Email: Send Affirmation      | Email Send          | Sends affirmation email                | Code: Pick Random Affirmation | IF: Telegram Enabled?          |                                                                                                                         |
| IF: Telegram Enabled?        | IF (Boolean)        | Checks if Telegram sending is enabled | Email: Send Affirmation     | Telegram: Send Affirmation     |                                                                                                                         |
| Telegram: Send Affirmation   | Telegram            | Sends affirmation via Telegram         | IF: Telegram Enabled?       | None                          |                                                                                                                         |
| Setup Tips                   | Sticky Note         | Setup checklist and testing tips      | None                       | None                          | Quick Setup Checklist: Add SMTP and (optional) Telegram credentials, edit configuration, test with temporary Cron change.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Node**  
   - Name: `Cron: Trigger Daily at 7 AM`  
   - Type: Cron Trigger  
   - Set trigger time to 7 AM daily (hour = 7, minutes and seconds default to 0).  

2. **Create a Set Node for Configuration**  
   - Name: `Set: Configuration (edit me)`  
   - Type: Set (v2)  
   - Add string fields:  
     - `fromEmail`: e.g., "sender@example.com"  
     - `toEmail`: e.g., "recipient@example.com"  
     - `subject`: e.g., "🌸 Morning Affirmation"  
     - `telegramChatId`: leave empty or fill with Telegram chat ID  
     - `affirmations`: multiline string of affirmations separated by new lines (edit as desired)  
   - Add boolean field:  
     - `sendTelegram`: false by default or true to enable Telegram sending  
   - Set to keep only these fields (remove all others).  

3. **Connect Cron node output to the Set node input.**

4. **Create a Code Node to Pick a Random Affirmation**  
   - Name: `Code: Pick Random Affirmation`  
   - Type: Code (JavaScript, v2)  
   - Paste this code:  
     ```js
     const lines = $json.affirmations
       .split('\n')
       .map(s => s.trim())
       .filter(Boolean);

     const pick = lines.length
       ? lines[Math.floor(Math.random() * lines.length)]
       : 'I am enough.';

     return [{
       json: {
         ...$json,
         affirmation: pick
       }
     }];
     ```  
   - Connect the Set node output to this Code node input.  

5. **Create an Email Send Node**  
   - Name: `Email: Send Affirmation`  
   - Type: Email Send (v2)  
   - Set parameters using expressions:  
     - From Email: `{{$json.fromEmail}}`  
     - To Email: `{{$json.toEmail}}`  
     - Subject: `{{$json.subject}}`  
     - Text: `{{$json.affirmation}}`  
   - Select SMTP credentials (create or import an SMTP credential beforehand).  
   - Connect Code node output to Email node input.  

6. **Create an IF Node to Check Telegram Enablement**  
   - Name: `IF: Telegram Enabled?`  
   - Type: IF (Boolean)  
   - Condition: Check if `{{$json.sendTelegram}}` is true  
   - Connect Email node output to IF node input.  

7. **Create a Telegram Node to Send Affirmation**  
   - Name: `Telegram: Send Affirmation`  
   - Type: Telegram node (send message) version 1.2+  
   - Set parameters using expressions:  
     - Text: `🌸 {{$json.affirmation}}`  
     - Chat ID: `{{$json.telegramChatId}}`  
   - Select or add Telegram Bot credentials.  
   - Connect IF node’s "true" output to Telegram node input.  

8. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - One describing the workflow purpose, setup instructions, and usage.  
   - Another providing a quick setup checklist including credential additions and testing tips.  

9. **Activate the Workflow**  
   - Test by temporarily setting the Cron node to trigger every minute.  
   - Verify email and optional Telegram messages are sent.  
   - Revert Cron node to daily 7 AM trigger once confirmed.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow is tailored for busy women, creators, and homemakers seeking a gentle, automated daily affirmation.                      | README - Template Description sticky note content                                                                |
| To customize, edit the affirmation list, change Cron schedule, or add logging branches such as Notion integration.                     | README - Template Description sticky note content                                                                |
| Ensure SMTP credentials are valid and capable of sending emails from the configured `fromEmail`.                                        | Setup Tips sticky note                                                                                             |
| Telegram sending is optional; ensure Telegram Bot credentials and chat ID are correctly configured if using this feature.               | Setup Tips sticky note                                                                                             |
| For testing, temporarily change the Cron node to trigger every minute, then revert to daily schedule after confirmation.               | Setup Tips sticky note                                                                                             |
| No secrets are stored directly in nodes; all sensitive info is handled via n8n credentials for security.                                | README - Template Description sticky note content                                                                |
| Workflow tested with n8n versions supporting Set node v2 and Code node v2 syntax. Telegram node version 1.2+ recommended for best use. | README - Template Description sticky note content                                                                |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, following all applicable content policies. No illegal, offensive, or protected content is included. All processed data is legal and public.