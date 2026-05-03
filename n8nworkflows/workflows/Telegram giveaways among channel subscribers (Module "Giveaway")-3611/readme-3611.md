Telegram giveaways among channel subscribers (Module "Giveaway")

https://n8nworkflows.xyz/workflows/telegram-giveaways-among-channel-subscribers--module--giveaway---3611


# Telegram giveaways among channel subscribers (Module "Giveaway")

### 1. Workflow Overview

This workflow automates Telegram giveaways that require participants to subscribe to specified Telegram channels. It is designed for businesses or individuals running giveaways on Telegram, ensuring only eligible users who subscribe to required channels can participate. The workflow manages participant tracking, channel management, giveaway execution, and winner selection, all integrated with a Postgres database for persistence.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**: Receives Telegram messages via webhook, initializes variables, and determines the type of incoming interaction (start command, button press, referral).
- **1.2 Participant and Bot Status Management**: Handles participant registration, updates bot status, and manages referral logic.
- **1.3 Channel Management**: Allows adding, verifying, listing, and deleting Telegram channels required for the giveaway.
- **1.4 Giveaway Execution and Winner Selection**: Retrieves participants, randomly selects winners, verifies subscription statuses, and sends notifications.
- **1.5 Cleanup and Finalization**: Deletes temporary data after giveaway completion and sends closing messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
This block listens for incoming Telegram messages via webhook, initializes workflow variables, and routes the flow based on message content to handle start commands, button presses, or referral links.

**Nodes Involved:**  
- Telegram Trigger  
- Variables TG  
- Initialization  
- Start? (IF)  
- Button? (IF)  
- Referal? (IF)  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point, listens for Telegram messages sent to the bot.  
  - Configuration: Uses a webhook with a unique ID to receive updates.  
  - Inputs: Telegram user messages/events.  
  - Outputs: Passes data to "Variables TG".  
  - Edge cases: Telegram webhook failure, invalid updates, network issues.

- **Variables TG**  
  - Type: Set  
  - Role: Initializes or sets workflow variables for use downstream.  
  - Configuration: Likely sets default values or extracts key parameters from the Telegram message.  
  - Inputs: From Telegram Trigger.  
  - Outputs: To "Initialization".  
  - Edge cases: Missing expected data fields, expression errors.

- **Initialization**  
  - Type: Set  
  - Role: Prepares initial conditions for the workflow logic, possibly setting flags or cleaning previous state.  
  - Inputs: From Variables TG.  
  - Outputs: To Start? node.  
  - Edge cases: None significant; depends on variable correctness.

- **Start?**  
  - Type: IF  
  - Role: Checks if the incoming message is a start command (e.g., /start).  
  - Inputs: From Initialization.  
  - Outputs: Two branches: if true, to Referal? node; else, to Button? node.  
  - Edge cases: Unrecognized commands, empty messages.

- **Button?**  
  - Type: IF  
  - Role: Checks if the message contains button interactions (callback queries).  
  - Inputs: From Start? (false branch).  
  - Outputs: True branch to Buttons switch node, false branch to Get Bot Status.  
  - Edge cases: Missing button data, malformed callback data.

- **Referal?**  
  - Type: IF  
  - Role: Determines if the start command includes a referral parameter.  
  - Inputs: From Start? (true branch).  
  - Outputs: True branch to "Welcome message Referal", false to "Welcome message Manager".  
  - Edge cases: Missing referral data, invalid referral codes.

---

#### 2.2 Participant and Bot Status Management

**Overview:**  
Manages participant registration, updates bot status in the database, and sends welcome messages depending on referral presence.

**Nodes Involved:**  
- Welcome message Referal  
- Welcome message Manager  
- Update bot status and referal  
- Upsert bot status on START  
- Add participant in Giveaway  
- Upsert bot status on START (separate node)  

**Node Details:**

- **Welcome message Referal**  
  - Type: Telegram  
  - Role: Sends a customized welcome message to referred participants.  
  - Inputs: From Referal? (true branch).  
  - Outputs: To "Update bot status and referal".  
  - Edge cases: Telegram API errors, message formatting issues.

- **Welcome message Manager**  
  - Type: Telegram  
  - Role: Sends a welcome message to non-referred participants or the manager.  
  - Inputs: From Referal? (false branch).  
  - Outputs: To "Upsert bot status on START".  
  - Edge cases: Same as above.

- **Update bot status and referal**  
  - Type: Postgres  
  - Role: Inserts or updates participant status and referral info in the database.  
  - Inputs: From Welcome message Referal.  
  - Outputs: To "Add participant in Giveaway".  
  - Edge cases: DB connection errors, SQL failures.

- **Add participant in Giveaway**  
  - Type: Postgres  
  - Role: Adds participant data to the giveaway participants table.  
  - Inputs: From Update bot status and referal.  
  - Outputs: None (end of this branch).  
  - Edge cases: Duplicate entries, DB constraints.

- **Upsert bot status on START**  
  - Type: Postgres  
  - Role: Updates or inserts bot status when a participant starts interaction without referral.  
  - Inputs: From Welcome message Manager.  
  - Outputs: None (end of this branch).  
  - Edge cases: DB errors, concurrency issues.

---

#### 2.3 Channel Management

**Overview:**  
Handles adding new Telegram channels to the giveaway, verifying channel existence, listing current channels, and deleting channels after giveaway completion.

**Nodes Involved:**  
- Define flow (Switch)  
- Add channel (Postgres)  
- Request New Channel (Telegram)  
- Channel Exists (Telegram)  
- List Channels (Telegram)  
- Update bot status on GIVEAWAY REQUEST CHANNEL (Postgres)  
- Delete Channels (Postgres)  
- Delete Participants (Postgres)  

**Node Details:**

- **Define flow**  
  - Type: Switch  
  - Role: Routes commands related to channel management or giveaway steps.  
  - Inputs: From Get Bot Status.  
  - Outputs: Branches to Commands, Add channel, or Define Step Giveaway.  
  - Edge cases: Unknown commands, invalid input.

- **Add channel**  
  - Type: Postgres  
  - Role: Adds a new channel record to the database.  
  - Inputs: From Define flow (Add channel branch).  
  - Outputs: To Request New Channel and Channel Exists nodes.  
  - Edge cases: Duplicate channels, DB errors.

- **Request New Channel**  
  - Type: Telegram  
  - Role: Sends a message to request channel info from the user.  
  - Inputs: From Add channel.  
  - Outputs: None (awaits user input).  
  - Edge cases: Telegram API errors.

- **Channel Exists**  
  - Type: Telegram  
  - Role: Checks if the specified channel exists or is accessible.  
  - Inputs: From Add channel.  
  - Outputs: Not explicitly connected in JSON; likely used for validation.  
  - Edge cases: Channel not found, Telegram API permission errors.

- **List Channels**  
  - Type: Telegram  
  - Role: Sends a list of current giveaway channels to the user.  
  - Inputs: From Define Step Giveaway or Commands.  
  - Outputs: To Update bot status on GIVEAWAY REQUEST CHANNEL.  
  - Edge cases: Empty channel list, API errors.

- **Update bot status on GIVEAWAY REQUEST CHANNEL**  
  - Type: Postgres  
  - Role: Updates bot status reflecting the request to add or list channels.  
  - Inputs: From List Channels.  
  - Outputs: None.  
  - Edge cases: DB errors.

- **Delete Channels**  
  - Type: Postgres  
  - Role: Deletes channel records after giveaway completion.  
  - Inputs: From SMS for Manager node.  
  - Outputs: None.  
  - Edge cases: Referential integrity issues, DB errors.

- **Delete Participants**  
  - Type: Postgres  
  - Role: Deletes participant records after giveaway completion.  
  - Inputs: From SMS for Manager node.  
  - Outputs: None.  
  - Edge cases: DB errors.

---

#### 2.4 Giveaway Execution and Winner Selection

**Overview:**  
Manages the core giveaway logic: retrieving participants, selecting winners randomly, checking subscription statuses, and notifying winners and managers.

**Nodes Involved:**  
- Define Step Giveaway (Switch)  
- Get Participants (Postgres)  
- Random Participants (Sort)  
- Get Channels (Postgres)  
- Telegram (message sender)  
- Union Statuses (Aggregate)  
- Check Success? (IF)  
- Check Success (Telegram, disabled)  
- Check Failed (Telegram, disabled)  
- SMS for Winner (Telegram)  
- SMS for Manager (Telegram)  
- Update bot status on GIVEAWAY RUN (Postgres)  
- Create Giveaway (Telegram)  

**Node Details:**

- **Define Step Giveaway**  
  - Type: Switch  
  - Role: Determines which step of the giveaway process to execute (e.g., listing channels or selecting participants).  
  - Inputs: From Define flow.  
  - Outputs: Branches to List Channels or Get Participants.  
  - Edge cases: Invalid step values.

- **Get Participants**  
  - Type: Postgres  
  - Role: Queries the database for current giveaway participants.  
  - Inputs: From Define Step Giveaway.  
  - Outputs: To Random Participants.  
  - Edge cases: Empty participant list, DB errors.

- **Random Participants**  
  - Type: Sort  
  - Role: Randomizes participant order to select winners fairly.  
  - Inputs: From Get Participants.  
  - Outputs: To Get Channels.  
  - Edge cases: Sorting failures.

- **Get Channels**  
  - Type: Postgres  
  - Role: Retrieves the list of channels participants must be subscribed to.  
  - Inputs: From Random Participants.  
  - Outputs: To Telegram node for status checks.  
  - Edge cases: Empty channel list, DB errors.

- **Telegram**  
  - Type: Telegram  
  - Role: Likely used to check participant subscription status or send messages during the giveaway.  
  - Inputs: From Get Channels.  
  - Outputs: To Union Statuses.  
  - Edge cases: API rate limits, permission errors.

- **Union Statuses**  
  - Type: Aggregate  
  - Role: Aggregates subscription status results from Telegram checks.  
  - Inputs: From Telegram.  
  - Outputs: To Check Success?.  
  - Edge cases: Aggregation errors.

- **Check Success?**  
  - Type: IF  
  - Role: Determines if the subscription checks succeeded for all participants.  
  - Inputs: From Union Statuses.  
  - Outputs: True branch to Check Success and SMS for Winner nodes; false branch to Check Failed.  
  - Edge cases: False negatives, logic errors.

- **Check Success** (disabled)  
  - Type: Telegram  
  - Role: Sends success confirmation messages (disabled in current workflow).  
  - Inputs: From Check Success?.  
  - Outputs: None.  
  - Edge cases: N/A (disabled).

- **Check Failed** (disabled)  
  - Type: Telegram  
  - Role: Sends failure notifications (disabled).  
  - Inputs: From Check Success? false branch.  
  - Outputs: None.  
  - Edge cases: N/A.

- **SMS for Winner**  
  - Type: Telegram  
  - Role: Sends notification to the winner(s).  
  - Inputs: From Check Success? true branch.  
  - Outputs: To SMS for Manager.  
  - Edge cases: Message delivery failures.

- **SMS for Manager**  
  - Type: Telegram  
  - Role: Notifies the giveaway manager about the winner(s).  
  - Inputs: From SMS for Winner.  
  - Outputs: To Delete Channels, Delete Participants, and Update bot status on START.  
  - Edge cases: Message delivery failures.

- **Update bot status on GIVEAWAY RUN**  
  - Type: Postgres  
  - Role: Updates the bot status to reflect that the giveaway is running.  
  - Inputs: From Buttons switch node.  
  - Outputs: To Create Giveaway.  
  - Edge cases: DB errors.

- **Create Giveaway**  
  - Type: Telegram  
  - Role: Sends messages to start or announce the giveaway.  
  - Inputs: From Update bot status on GIVEAWAY RUN.  
  - Outputs: None.  
  - Edge cases: Telegram API errors.

---

#### 2.5 Cleanup and Finalization

**Overview:**  
After the giveaway ends, this block cleans up database tables and sends final messages to participants or managers.

**Nodes Involved:**  
- Delete Channels (Postgres)  
- Delete Participants (Postgres)  
- Update bot status on START (Postgres)  
- End Message (Telegram)  

**Node Details:**

- **Delete Channels**  
  - See above in Channel Management.

- **Delete Participants**  
  - See above in Channel Management.

- **Update bot status on START**  
  - Type: Postgres  
  - Role: Resets or updates bot status to initial state after giveaway completion.  
  - Inputs: From SMS for Manager.  
  - Outputs: To End Message.  
  - Edge cases: DB errors.

- **End Message**  
  - Type: Telegram  
  - Role: Sends a final message indicating the giveaway has ended.  
  - Inputs: From Update bot status on START.  
  - Outputs: None.  
  - Edge cases: Telegram API errors.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                              | Input Node(s)                     | Output Node(s)                                  | Sticky Note |
|----------------------------------|---------------------|----------------------------------------------|----------------------------------|------------------------------------------------|-------------|
| Telegram Trigger                 | Telegram Trigger    | Entry point for Telegram messages             | -                                | Variables TG                                    |             |
| Variables TG                    | Set                 | Initialize variables from Telegram data       | Telegram Trigger                 | Initialization                                  |             |
| Initialization                 | Set                 | Prepare initial workflow state                 | Variables TG                    | Start?                                          |             |
| Start?                        | IF                  | Check if message is /start command             | Initialization                 | Referal?, Button?                               |             |
| Button?                       | IF                  | Check if message is button interaction         | Start?                         | Buttons, Get Bot Status                         |             |
| Referal?                      | IF                  | Check if start command has referral parameter  | Start?                         | Welcome message Referal, Welcome message Manager|             |
| Welcome message Referal        | Telegram            | Send welcome message to referred users         | Referal?                       | Update bot status and referal                   |             |
| Welcome message Manager        | Telegram            | Send welcome message to non-referred users     | Referal?                       | Upsert bot status on START                       |             |
| Update bot status and referal  | Postgres            | Update participant and referral status         | Welcome message Referal        | Add participant in Giveaway                     |             |
| Add participant in Giveaway    | Postgres            | Add participant to giveaway table               | Update bot status and referal  | -                                              |             |
| Upsert bot status on START     | Postgres            | Update bot status on participant start          | Welcome message Manager        | -                                              |             |
| Define flow                   | Switch              | Route commands to channel management or giveaway steps | Get Bot Status                 | Commands, Add channel, Define Step Giveaway    |             |
| Add channel                   | Postgres            | Add new Telegram channel to DB                   | Define flow                   | Request New Channel, Channel Exists             |             |
| Request New Channel           | Telegram            | Request channel info from user                    | Add channel                   | -                                              |             |
| Channel Exists               | Telegram            | Verify if channel exists                          | Add channel                   | -                                              |             |
| List Channels               | Telegram            | List current giveaway channels                    | Commands, Define Step Giveaway | Update bot status on GIVEAWAY REQUEST CHANNEL  |             |
| Update bot status on GIVEAWAY REQUEST CHANNEL | Postgres  | Update status after channel request               | List Channels                 | -                                              |             |
| Delete Channels             | Postgres            | Delete giveaway channels after completion         | SMS for Manager               | -                                              |             |
| Delete Participants         | Postgres            | Delete giveaway participants after completion     | SMS for Manager               | -                                              |             |
| Define Step Giveaway         | Switch              | Determine giveaway step to execute                 | Define flow                   | List Channels, Get Participants                 |             |
| Get Participants             | Postgres            | Retrieve participants from DB                       | Define Step Giveaway          | Random Participants                             |             |
| Random Participants          | Sort                | Randomize participants for winner selection         | Get Participants             | Get Channels                                    |             |
| Get Channels                | Postgres            | Retrieve required channels for giveaway              | Random Participants          | Telegram                                        |             |
| Telegram                   | Telegram            | Check subscription status or send messages          | Get Channels                | Union Statuses                                  |             |
| Union Statuses             | Aggregate           | Aggregate subscription check results                  | Telegram                    | Check Success?                                  |             |
| Check Success?             | IF                  | Verify if all subscription checks succeeded          | Union Statuses              | Check Success, SMS for Winner, Check Failed    |             |
| Check Success              | Telegram (disabled) | Notify success (disabled)                              | Check Success?              | -                                              |             |
| Check Failed               | Telegram (disabled) | Notify failure (disabled)                              | Check Success?              | -                                              |             |
| SMS for Winner             | Telegram            | Notify winner(s)                                      | Check Success?              | SMS for Manager                                 |             |
| SMS for Manager            | Telegram            | Notify manager about winners                          | SMS for Winner              | Delete Channels, Delete Participants, Update bot status on START |             |
| Update bot status on GIVEAWAY RUN | Postgres       | Update status to giveaway running                      | Buttons                     | Create Giveaway                                 |             |
| Create Giveaway            | Telegram            | Announce or start giveaway                             | Update bot status on GIVEAWAY RUN | -                                              |             |
| Update bot status on START  | Postgres            | Reset bot status after giveaway                         | SMS for Manager             | End Message                                     |             |
| End Message                | Telegram            | Send final message after giveaway ends                  | Update bot status on START  | -                                              |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure webhook with your Telegram bot credentials.  
   - This is the entry point for all Telegram messages.

2. **Add Set node "Variables TG"**  
   - Extract and initialize variables from incoming Telegram message data.  
   - Connect from Telegram Trigger.

3. **Add Set node "Initialization"**  
   - Prepare initial workflow state variables or flags.  
   - Connect from Variables TG.

4. **Add IF node "Start?"**  
   - Condition: Check if message text equals "/start".  
   - Connect from Initialization.

5. **Add IF node "Button?"**  
   - Condition: Check if update contains button callback data.  
   - Connect from Start? false branch.

6. **Add IF node "Referal?"**  
   - Condition: Check if /start command includes referral parameter.  
   - Connect from Start? true branch.

7. **Add Telegram node "Welcome message Referal"**  
   - Send welcome message tailored for referred users.  
   - Connect from Referal? true branch.

8. **Add Postgres node "Update bot status and referal"**  
   - Upsert participant and referral info into database.  
   - Connect from Welcome message Referal.

9. **Add Postgres node "Add participant in Giveaway"**  
   - Insert participant record into giveaway table.  
   - Connect from Update bot status and referal.

10. **Add Telegram node "Welcome message Manager"**  
    - Send welcome message for non-referred users.  
    - Connect from Referal? false branch.

11. **Add Postgres node "Upsert bot status on START"**  
    - Update bot status for participants starting without referral.  
    - Connect from Welcome message Manager.

12. **Add IF node "Buttons"**  
    - Check for button interactions.  
    - Connect from Button? true branch.

13. **Add Postgres node "Update bot status on GIVEAWAY RUN"**  
    - Update bot status to running giveaway.  
    - Connect from Buttons.

14. **Add Telegram node "Create Giveaway"**  
    - Announce or start giveaway messages.  
    - Connect from Update bot status on GIVEAWAY RUN.

15. **Add Postgres node "Get Bot Status"**  
    - Retrieve current bot status from DB.  
    - Connect from Button? false branch.

16. **Add Switch node "Define flow"**  
    - Route commands to channel management or giveaway steps.  
    - Connect from Get Bot Status.

17. **Add Switch node "Commands"**  
    - Handle commands like listing channels.  
    - Connect from Define flow.

18. **Add Telegram node "List Channels"**  
    - Send list of channels to user.  
    - Connect from Commands.

19. **Add Postgres node "Update bot status on GIVEAWAY REQUEST CHANNEL"**  
    - Update status after channel list request.  
    - Connect from List Channels.

20. **Add Postgres node "Add channel"**  
    - Insert new channel into DB.  
    - Connect from Define flow.

21. **Add Telegram node "Request New Channel"**  
    - Ask user to provide new channel info.  
    - Connect from Add channel.

22. **Add Telegram node "Channel Exists"**  
    - Verify if channel exists on Telegram.  
    - Connect from Add channel.

23. **Add Switch node "Define Step Giveaway"**  
    - Determine giveaway step (list channels or get participants).  
    - Connect from Define flow.

24. **Add Postgres node "Get Participants"**  
    - Query participants from DB.  
    - Connect from Define Step Giveaway.

25. **Add Sort node "Random Participants"**  
    - Randomize participants for winner selection.  
    - Connect from Get Participants.

26. **Add Postgres node "Get Channels"**  
    - Retrieve required channels from DB.  
    - Connect from Random Participants.

27. **Add Telegram node "Telegram"**  
    - Check subscription status or send messages.  
    - Connect from Get Channels.

28. **Add Aggregate node "Union Statuses"**  
    - Aggregate subscription check results.  
    - Connect from Telegram.

29. **Add IF node "Check Success?"**  
    - Check if all subscription checks succeeded.  
    - Connect from Union Statuses.

30. **Add Telegram node "SMS for Winner"**  
    - Notify winners.  
    - Connect from Check Success? true branch.

31. **Add Telegram node "SMS for Manager"**  
    - Notify manager of winners.  
    - Connect from SMS for Winner.

32. **Add Postgres node "Delete Channels"**  
    - Delete channels after giveaway.  
    - Connect from SMS for Manager.

33. **Add Postgres node "Delete Participants"**  
    - Delete participants after giveaway.  
    - Connect from SMS for Manager.

34. **Add Postgres node "Update bot status on START"**  
    - Reset bot status after giveaway ends.  
    - Connect from SMS for Manager.

35. **Add Telegram node "End Message"**  
    - Send final message indicating giveaway end.  
    - Connect from Update bot status on START.

36. **Connect all nodes according to the described flow, ensuring error handling where appropriate.**

37. **Set up credentials:**  
    - Telegram Bot credentials for all Telegram nodes and triggers.  
    - Postgres credentials for all database nodes.

38. **Deploy and test the workflow with sample Telegram messages and database setup.**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow requires a Postgres database with tables created via a provided SQL script. Replace `"n8n"` schema name as needed. | Setup instructions in workflow description.                                                        |
| Telegram nodes use bot token credentials; ensure the bot has required permissions to send messages and read channel info. | Telegram Bot API documentation: https://core.telegram.org/bots/api                                   |
| The workflow manages referral tracking and participant eligibility to ensure fair giveaways.                         | Useful for marketing campaigns requiring subscriber verification.                                   |
| To customize messages and logic, edit Telegram node message templates and add nodes as needed.                       | Messages can be localized or enhanced with rich formatting.                                         |
| Disabled nodes "Check Success" and "Check Failed" can be enabled for additional notifications if desired.            | Can be used to notify participants/managers about subscription check results.                       |
| Sticky notes in the workflow are currently empty but can be used to add documentation or instructions in n8n UI.     | Use sticky notes for internal documentation or team communication.                                  |

---

This structured documentation enables advanced users and automation agents to understand, reproduce, and modify the Telegram giveaway workflow effectively.