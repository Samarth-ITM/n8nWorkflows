🍅 Custom Pomodoro Tracker using Telegram and Google Sheet

https://n8nworkflows.xyz/workflows/---custom-pomodoro-tracker-using-telegram-and-google-sheet-3307


# 🍅 Custom Pomodoro Tracker using Telegram and Google Sheet

### 1. Workflow Overview

This workflow implements a **Custom Pomodoro Tracker** using **Telegram** for user interaction and **Google Sheets** for logging session data. It targets creators, freelancers, students, and professionals who use the Pomodoro Technique and want to track their productivity sessions automatically.

The workflow is logically divided into the following blocks:

- **1.1 Initialization and Static Data Setup**: Prepares global static data to store user session states.
- **1.2 Telegram Message Reception and Command Routing**: Listens for Telegram messages, filters commands, and routes them accordingly.
- **1.3 Session Control (Start/Stop)**: Handles `/start` and `/stop` commands to begin or end Pomodoro sessions.
- **1.4 Pomodoro Cycle Management**: Manages the 25-minute deep work intervals and short breaks, including notifications.
- **1.5 Session Counting and State Management**: Tracks the number of Pomodoro cycles per user and manages session IDs.
- **1.6 Session Logging to Google Sheets**: Records deep work and long break sessions with detailed metadata.
- **1.7 Session Completion and Cleanup**: Handles long breaks after four cycles, notifies the user, logs the session, and clears stored state.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Static Data Setup

- **Overview:**  
  This block initializes the global static data object `telegramStates` to store per-user session states persistently across workflow executions.

- **Nodes Involved:**  
  - Initiate Static Data  
  - Sticky Note (Instruction)

- **Node Details:**  
  - **Initiate Static Data**  
    - Type: Code Node  
    - Role: Checks if `telegramStates` exists in global static data; if not, creates it as an empty object.  
    - Key Expressions: Uses `$getWorkflowStaticData('global')` to access global static data.  
    - Input: None (manual trigger recommended)  
    - Output: Returns the static data object for confirmation.  
    - Edge Cases: Should be run once before workflow activation; failure to run may cause undefined state errors.

  - **Sticky Note**  
    - Provides setup instructions for this block.

---

#### 2.2 Telegram Message Reception and Command Routing

- **Overview:**  
  Listens for incoming Telegram messages, filters those starting with `/` (commands), and routes them to either session control or an instruction message for invalid commands.

- **Nodes Involved:**  
  - Telegram Trigger  
  - If (Command Check)  
  - Instructions Message  
  - Sticky Note (Instruction)

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger Node  
    - Role: Listens for all incoming Telegram messages (`updates: ["message"]`).  
    - Configuration: Uses Telegram Bot Token credentials.  
    - Output: Emits message JSON including chat and message text.  
    - Edge Cases: Telegram API downtime, webhook misconfiguration.

  - **If (Command Check)**  
    - Type: If Node (v2)  
    - Role: Checks if message text starts with `/` to identify commands.  
    - Expression: `{{$json["message"]["text"].startsWith("/")}}`  
    - Routes:  
      - True: Forward to session control block.  
      - False: Forward to Instructions Message node.

  - **Instructions Message**  
    - Type: Telegram Node  
    - Role: Sends a default help message for unrecognized inputs.  
    - Configuration: Sends fixed text with valid commands `/start` and `/stop`.  
    - Input: Receives non-command messages.  
    - Edge Cases: Telegram API errors.

  - **Sticky Note1**  
    - Contains detailed instructions on setting up Telegram nodes and command routing.

---

#### 2.3 Session Control (Start/Stop)

- **Overview:**  
  Determines if the command is `/start` or `/stop`. `/start` initiates a Pomodoro cycle; `/stop` ends the session and clears user state.

- **Nodes Involved:**  
  - start or stop? (If Node)  
  - Start Cycle Notification  
  - End of Session Notification  
  - Clear Variables1  
  - Sticky Note (Instruction)

- **Node Details:**  
  - **start or stop?**  
    - Type: If Node (v2)  
    - Role: Checks if the command equals `/start`.  
    - Routes:  
      - True: Start cycle notification and session start.  
      - False: End session notification and cleanup.

  - **Start Cycle Notification**  
    - Type: Telegram Node  
    - Role: Notifies user that a 25-minute deep work session has started.  
    - Configuration: Fixed message text, sends to user chat ID.

  - **End of Session Notification**  
    - Type: Telegram Node  
    - Role: Notifies user that the session was stopped early.  
    - Configuration: Fixed message text, sends to user chat ID.

  - **Clear Variables1**  
    - Type: Code Node  
    - Role: Deletes the user's entry from `telegramStates` in global static data to reset session state.  
    - Input: Receives from End of Session Notification.  
    - Edge Cases: If user state does not exist, no error but no effect.

  - **Sticky Note**  
    - Provides setup instructions for session control nodes.

---

#### 2.4 Pomodoro Cycle Management

- **Overview:**  
  Manages the timing of deep work sessions and breaks, sending notifications and looping through cycles until four are completed.

- **Nodes Involved:**  
  - Deep Work (Wait Node)  
  - Break (Wait Node)  
  - Short Break Notification  
  - Back to Work Notification  
  - < 4 Cycles (If Node)  
  - Long Break Notification  
  - Increment Count  
  - Sticky Note (Instruction)

- **Node Details:**  
  - **Deep Work**  
    - Type: Wait Node  
    - Role: Waits 25 minutes representing a deep work session.  
    - Configuration: 25 minutes fixed.  
    - Output: Triggers Short Break Notification.

  - **Short Break Notification**  
    - Type: Telegram Node  
    - Role: Notifies user to take a short break after deep work.  
    - Configuration: Fixed message text, sends to user chat ID.

  - **Break**  
    - Type: Wait Node  
    - Role: Waits for short break duration (default 5 minutes).  
    - Configuration: 5 minutes (default, configurable).  
    - Output: Triggers Increment Count node.

  - **Increment Count**  
    - Type: Code Node  
    - Role: Updates user's Pomodoro count and session ID in global static data.  
    - Logic:  
      - Initializes user state if missing.  
      - Generates unique session ID if missing.  
      - Increments Pomodoro count by 1.  
      - Returns count, sessionId, and startTime.  
    - Input: Receives from Break node.  
    - Output: Passes updated count to Back to Work Notification.

  - **Back to Work Notification**  
    - Type: Telegram Node  
    - Role: Notifies user that break is over and next cycle starts.  
    - Configuration: Message includes current cycle count.  
    - Output: Triggers < 4 Cycles node.

  - **< 4 Cycles**  
    - Type: If Node (v2)  
    - Role: Checks if Pomodoro count is greater than 4.  
    - Routes:  
      - False (count ≤ 4): Triggers Start Cycle Notification to begin next deep work cycle.  
      - True (count > 4): Triggers Long Break Notification.

  - **Long Break Notification**  
    - Type: Telegram Node  
    - Role: Notifies user to take a long break after 4 cycles.  
    - Configuration: Fixed message text, sends to user chat ID.  
    - Output: Triggers Clear Variables2 and Record Long Break nodes.

  - **Start Cycle Notification**  
    - Type: Telegram Node  
    - Role: Notifies user that a new deep work cycle is starting.  
    - Configuration: Fixed message text, sends to user chat ID.  
    - Output: Triggers Deep Work node.

  - **Sticky Note2**  
    - Contains detailed instructions on configuring timers, Telegram nodes, and Google Sheets logging for this block.

---

#### 2.5 Session Counting and State Management

- **Overview:**  
  Maintains per-user session state including Pomodoro count, session ID, and start time in global static data.

- **Nodes Involved:**  
  - Increment Count (Code Node)

- **Node Details:**  
  - **Increment Count**  
    - See details in 2.4 Pomodoro Cycle Management.

---

#### 2.6 Session Logging to Google Sheets

- **Overview:**  
  Logs each deep work session and long break to a Google Sheet with detailed metadata for tracking and analysis.

- **Nodes Involved:**  
  - Record Deep Work (Google Sheets Node)  
  - Record Long Break (Google Sheets Node)  
  - Sticky Note (Instruction)

- **Node Details:**  
  - **Record Deep Work**  
    - Type: Google Sheets Node  
    - Role: Appends a row for each deep work session.  
    - Configuration:  
      - Document ID and Sheet Name configured to target the user’s Google Sheet.  
      - Columns mapped: Date, Time, User ID, Block Type ("Deep Work"), Pomodoro Count, Working Session ID, Focus Duration (25), Break Duration (5).  
    - Input: Receives from Back to Work Notification node.  
    - Edge Cases: Google API auth errors, sheet access issues.

  - **Record Long Break**  
    - Type: Google Sheets Node  
    - Role: Appends a row for the long break session after 4 cycles.  
    - Configuration:  
      - Similar to Record Deep Work but Block Type is "Long Break", Focus Duration is 0, Break Duration is 15.  
    - Input: Receives from Long Break Notification node.  
    - Edge Cases: Same as above.

  - **Sticky Note4**  
    - Provides setup instructions for Google Sheets nodes and credential configuration.

---

#### 2.7 Session Completion and Cleanup

- **Overview:**  
  After the long break notification and logging, clears the user’s session state to prepare for new sessions.

- **Nodes Involved:**  
  - Clear Variables2 (Code Node)

- **Node Details:**  
  - **Clear Variables2**  
    - Type: Code Node  
    - Role: Deletes the user’s session state from global static data.  
    - Input: Receives from Long Break Notification node.  
    - Edge Cases: No error if user state missing.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                | Input Node(s)             | Output Node(s)                     | Sticky Note                                                                                      |
|-------------------------|---------------------|-----------------------------------------------|---------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger    | Listens for Telegram messages                  | -                         | If                               | Sticky Note1: Setup Telegram trigger and command routing                                        |
| If                      | If (v2)             | Checks if message starts with `/`              | Telegram Trigger          | start or stop?, Instructions Message | Sticky Note1                                                                                     |
| Instructions Message    | Telegram            | Sends help message for invalid commands        | If                        | -                                | Sticky Note1                                                                                     |
| start or stop?          | If (v2)             | Checks if command is `/start`                   | If                        | Start Cycle Notification, End of Session Notification | Sticky Note1                                                                                     |
| Start Cycle Notification| Telegram            | Notifies start of deep work cycle               | start or stop?            | Deep Work                        | Sticky Note2: Setup deep work blocks and notifications                                         |
| Deep Work               | Wait                | Waits 25 minutes for deep work session          | Start Cycle Notification  | Short Break Notification          | Sticky Note2                                                                                     |
| Short Break Notification| Telegram            | Notifies user to take a short break             | Deep Work                 | Break                           | Sticky Note2                                                                                     |
| Break                   | Wait                | Waits 5 minutes for short break                  | Short Break Notification  | Increment Count                  | Sticky Note2                                                                                     |
| Increment Count         | Code                | Updates Pomodoro count and session state        | Break                     | Back to Work Notification        | Sticky Note2                                                                                     |
| Back to Work Notification| Telegram           | Notifies user break is over and next cycle starts | Increment Count           | < 4 Cycles                      | Sticky Note2                                                                                     |
| < 4 Cycles              | If (v2)             | Checks if Pomodoro count > 4                     | Back to Work Notification | Long Break Notification, Start Cycle Notification | Sticky Note2                                                                                     |
| Long Break Notification | Telegram            | Notifies user to take a long break               | < 4 Cycles                | Clear Variables2, Record Long Break | Sticky Note4: Setup long break logging and cleanup                                             |
| Record Deep Work        | Google Sheets       | Logs deep work session to Google Sheet           | Back to Work Notification | -                                | Sticky Note2                                                                                     |
| Record Long Break       | Google Sheets       | Logs long break session to Google Sheet          | Long Break Notification   | -                                | Sticky Note4                                                                                     |
| Clear Variables1        | Code                | Clears user session state on `/stop`             | End of Session Notification| -                                | Sticky Note1                                                                                     |
| End of Session Notification| Telegram          | Notifies user session stopped early              | start or stop? (false)    | Clear Variables1                | Sticky Note1                                                                                     |
| Clear Variables2        | Code                | Clears user session state after long break       | Long Break Notification   | -                                | Sticky Note4                                                                                     |
| Initiate Static Data    | Code                | Initializes global static data for session states | -                         | -                                | Sticky Note: Setup instructions for static data initialization                                 |
| Sticky Note             | Sticky Note         | Instructions for static data initialization      | -                         | -                                |                                                                                                |
| Sticky Note1            | Sticky Note         | Instructions for Telegram trigger and commands   | -                         | -                                |                                                                                                |
| Sticky Note2            | Sticky Note         | Instructions for deep work blocks and Google Sheets logging | -                         | -                                |                                                                                                |
| Sticky Note3            | Sticky Note         | Link to detailed tutorial video                   | -                         | -                                | Contains tutorial link: https://www.youtube.com/watch?v=ztMMrmbgGEo                            |
| Sticky Note4            | Sticky Note         | Instructions for long break logging and cleanup  | -                         | -                                |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot Token credentials.  
   - Set to listen for `message` updates.

2. **Add If Node to Check Command**  
   - Type: If (v2)  
   - Condition: Check if `{{$json["message"]["text"].startsWith("/")}}` is true.  
   - True path leads to session control; False path leads to instructions message.

3. **Add Instructions Message Node**  
   - Type: Telegram  
   - Text:  
     ```
     💡 Oops! That’s not a valid command.

     Here’s what you can do:
     ✅ /start – Kick off a Pomodoro session and get in the zone.
     ✅ /stop – Wrap up your session like a productivity pro.

     Now, let’s get some deep work done! 🔥💻
     ```  
   - Chat ID: `{{$json["message"]["from"]["id"]}}`

4. **Add If Node to Check `/start` Command**  
   - Type: If (v2)  
   - Condition: Check if `{{$json["message"]["text"]}} === "/start"`  
   - True path: Start Pomodoro cycle  
   - False path: Stop session

5. **Add Start Cycle Notification Node**  
   - Type: Telegram  
   - Text: `⏰ Time to focus! 25 minutes of deep work starts now.`  
   - Chat ID: `{{$json["message"]["from"]["id"]}}`

6. **Add End of Session Notification Node**  
   - Type: Telegram  
   - Text:  
     ```
     🛑 You decided to stop the session early.
     🚀 Use /start to relaunch a working session.
     ```  
   - Chat ID: `{{$json["message"]["from"]["id"]}}`

7. **Add Clear Variables1 Code Node**  
   - Type: Code  
   - Code:  
     ```javascript
     let workflowStaticData = $getWorkflowStaticData('global');
     if (workflowStaticData.telegramStates) {
         delete workflowStaticData.telegramStates[$('Telegram Trigger').first().json.message.chat.id.toString()];
     }
     return $input.all();
     ```

8. **Add Deep Work Wait Node**  
   - Type: Wait  
   - Duration: 25 minutes

9. **Add Short Break Notification Node**  
   - Type: Telegram  
   - Text: `🚰 Work session complete! Take a short break.`  
   - Chat ID: `{{$json["message"]["from"]["id"]}}`

10. **Add Break Wait Node**  
    - Type: Wait  
    - Duration: 5 minutes (default short break)

11. **Add Increment Count Code Node**  
    - Type: Code  
    - Code:  
      ```javascript
      let workflowStaticData = $getWorkflowStaticData('global');
      if (!workflowStaticData.telegramStates) {
          workflowStaticData.telegramStates = {};
      }
      let userId = $('Telegram Trigger').first().json.message.chat.id.toString();
      if (!workflowStaticData.telegramStates[userId]) {
          workflowStaticData.telegramStates[userId] = { count: 0, sessionId: "", startTime: "" };
      }
      if (!workflowStaticData.telegramStates[userId].sessionId) {
          workflowStaticData.telegramStates[userId].sessionId = Date.now().toString(36) + Math.random().toString(36).substring(2, 8);
          workflowStaticData.telegramStates[userId].startTime = new Date().toISOString();
      }
      workflowStaticData.telegramStates[userId].count += 1;
      return [{
          json: {
              count: workflowStaticData.telegramStates[userId].count,
              sessionId: workflowStaticData.telegramStates[userId].sessionId,
              startTime: workflowStaticData.telegramStates[userId].startTime
          }
      }];
      ```

12. **Add Back to Work Notification Node**  
    - Type: Telegram  
    - Text: `🏢 Break over! Back to work for the cycle: {{$json.count}}`  
    - Chat ID: `{{$json["message"]["from"]["id"]}}`

13. **Add If Node to Check Cycle Count**  
    - Type: If (v2)  
    - Condition: Check if `{{$json.count}} > 4`  
    - True path: Long break  
    - False path: Start next deep work cycle

14. **Add Long Break Notification Node**  
    - Type: Telegram  
    - Text: `🍴 Time for a long break. Great job!`  
    - Chat ID: `{{$json["message"]["from"]["id"]}}`

15. **Add Clear Variables2 Code Node**  
    - Type: Code  
    - Code:  
      ```javascript
      let workflowStaticData = $getWorkflowStaticData('global');
      if (workflowStaticData.telegramStates) {
          delete workflowStaticData.telegramStates[$('Telegram Trigger').first().json.message.chat.id.toString()];
      }
      return $input.all();
      ```

16. **Add Google Sheets Node to Record Deep Work**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: Your target sheet (e.g., "Sheet1")  
    - Columns Mapping:  
      - Date: `{{$now.format('dd-LL-yyyy')}}`  
      - Time: `{{$now.hour.toString().padStart(2, '0')}}:{{$now.minute.toString().padStart(2, '0')}}`  
      - User ID: `{{$json.result.chat.id}}`  
      - Block Type: `"Deep Work"`  
      - Pomodoro Count: `{{$json.count}}`  
      - Working Session ID: `{{$json.sessionId}}`  
      - Focus Duration (min): `"25"`  
      - Break Duration (min): `"5"`

17. **Add Google Sheets Node to Record Long Break**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID and Sheet Name: Same as above  
    - Columns Mapping:  
      - Date, Time, User ID, Working Session ID: same as above  
      - Block Type: `"Long Break"`  
      - Pomodoro Count: `{{$json.count}}`  
      - Focus Duration (min): `"0"`  
      - Break Duration (min): `"15"`

18. **Add Initiate Static Data Code Node**  
    - Type: Code  
    - Code:  
      ```javascript
      let workflowStaticData = $getWorkflowStaticData('global');
      if (!workflowStaticData.telegramStates) {
          workflowStaticData.telegramStates = {};
      }
      return workflowStaticData;
      ```  
    - Run this node once before activating the workflow.

19. **Connect Nodes According to Logical Flow:**  
    - Telegram Trigger → If (Command Check)  
    - If True → start or stop?  
    - start or stop? True → Start Cycle Notification → Deep Work → Short Break Notification → Break → Increment Count → Back to Work Notification → < 4 Cycles → (False) Start Cycle Notification (loop) or (True) Long Break Notification → Clear Variables2 & Record Long Break  
    - start or stop? False → End of Session Notification → Clear Variables1  
    - If False (Command Check) → Instructions Message

20. **Set Credentials:**  
    - Telegram Trigger and Telegram Nodes: Use your Telegram Bot Token credentials.  
    - Google Sheets Nodes: Use Google Sheets OAuth2 credentials with access to your target spreadsheet.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow created by Samir, Supply Chain Engineer and Data Scientist, founder of LogiGreen Consulting | https://logi-green.com                                                                              |
| Pomodoro Technique explanation and productivity benefits                                            | https://runmefit.com/wp-content/uploads/2024/01/How-the-Pomodoro-Technique-Works.jpg                |
| Video tutorial for detailed step-by-step guide                                                      | https://www.youtube.com/watch?v=ztMMrmbgGEo                                                        |
| Workflow template page with screenshots and instructions                                            | https://www.samirsaci.com/content/images/2025/03/image-15.png                                       |
| LinkedIn profile for networking and productivity tips                                               | https://www.linkedin.com/in/samir-saci                                                            |
| Google Sheets Node documentation                                                                    | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets                       |
| Telegram Trigger Node documentation                                                                  | https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger/               |
| Code Node documentation                                                                             | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code                             |

---

This comprehensive documentation enables users and developers to understand, reproduce, and extend the Pomodoro Tracker workflow with confidence.