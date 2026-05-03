Bulk Delete Slack Messages with Smart Filtering and Confirmations

https://n8nworkflows.xyz/workflows/bulk-delete-slack-messages-with-smart-filtering-and-confirmations-4948


# Bulk Delete Slack Messages with Smart Filtering and Confirmations

### 1. Workflow Overview

This workflow, titled **"Bulk Delete Slack Messages with Smart Filtering and Confirmations"**, automates the process of deleting multiple Slack messages from a specified channel based on smart filtering criteria and user confirmations. It is designed for Slack workspace administrators or moderators who want to efficiently clean up channels by removing messages in bulk, with safeguards to avoid accidental deletions.

The workflow consists of the following logical blocks:

- **1.1 Input Reception and Initial Parsing:** Receive and parse user commands via a webhook to determine the intended deletion operation.
- **1.2 Command Validation and Message Retrieval:** Validate the parsed command, retrieve messages from Slack channels, and filter based on search terms.
- **1.3 User Confirmation and Pending Deletion Storage:** Confirm with the user before deletion, and store the pending deletion data.
- **1.4 Execution of Message Deletion:** Process batch deletion of the filtered messages, including pacing with wait nodes and error handling.
- **1.5 Post-Deletion Cleanup and Reporting:** Clean up workflow-related messages, send completion notifications, and handle error reports or cancellations.
- **1.6 Instruction Messaging and Interaction Handling:** Provide instructions to users and manage interaction flows, including cancelation and error message management.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Initial Parsing

**Overview:**  
This block receives incoming commands via a webhook, parses the command text, and prepares the data for validation.

**Nodes Involved:**  
- Webhook  
- Respond to Webhook  
- Switch  
- Instruction Message  
- Wait1  
- Delete Instruction  
- Parse Command  
- Prase Set  

**Node Details:**

- **Webhook**  
  - *Type:* Webhook (entry point)  
  - *Role:* Accept incoming HTTP requests with Slack command payloads.  
  - *Config:* Default webhook URL, no authentication.  
  - *Input/Output:* No input; outputs raw payload to Respond to Webhook.  
  - *Failure Modes:* Network issues, webhook misconfiguration.

- **Respond to Webhook**  
  - *Type:* Code  
  - *Role:* Sends an immediate HTTP response to acknowledge receipt of the webhook.  
  - *Config:* Custom JavaScript response to Slack to avoid timeout.  
  - *Input:* From Webhook node.  
  - *Output:* To Switch node.  
  - *Failure Modes:* Expression failures or output formatting errors.

- **Switch**  
  - *Type:* Switch  
  - *Role:* Routes the flow based on message content type or command type.  
  - *Config:* Conditions to detect instruction messages, user commands, or other interactions.  
  - *Input:* From Respond to Webhook.  
  - *Output:* To Check User Message, Instruction Message, or Parse Command.  
  - *Failure Modes:* Misrouted messages if conditions are not comprehensive.

- **Instruction Message**  
  - *Type:* Slack  
  - *Role:* Sends instructional messages to users when command is unclear or help is needed.  
  - *Config:* Uses Slack webhook credentials, posts predefined instructional text.  
  - *Input:* From Switch node.  
  - *Output:* To Wait1 node.  
  - *Failure Modes:* Slack API errors, credential issues.

- **Wait1**  
  - *Type:* Wait  
  - *Role:* Delays for a short period before deleting the instruction message.  
  - *Config:* Configured wait time (e.g., seconds).  
  - *Input:* From Instruction Message.  
  - *Output:* To Delete Instruction.  
  - *Failure Modes:* Timing misconfiguration.

- **Delete Instruction**  
  - *Type:* Slack  
  - *Role:* Deletes the instructional message from Slack to keep channels clean.  
  - *Config:* Uses Slack API with message timestamp from previous node.  
  - *Input:* From Wait1.  
  - *Output:* Ends flow for instruction path.  
  - *Failure Modes:* Message already deleted or permission errors.

- **Parse Command**  
  - *Type:* Code  
  - *Role:* Parses the user's command text to extract parameters like channel, search terms, or limits.  
  - *Config:* Custom JavaScript code parsing command syntax.  
  - *Input:* From Switch node.  
  - *Output:* To Prase Set.  
  - *Failure Modes:* Parsing errors if command format is incorrect.

- **Prase Set**  
  - *Type:* Set  
  - *Role:* Structures parsed command data into defined variables for downstream use.  
  - *Config:* Defines variables like channel ID, search text, delete limits.  
  - *Input:* From Parse Command.  
  - *Output:* To Check if Valid Command.  
  - *Failure Modes:* Missing or malformed parameters.

---

#### 1.2 Command Validation and Message Retrieval

**Overview:**  
This block validates the command parameters and retrieves Slack messages from the specified channel, filtering them based on search terms.

**Nodes Involved:**  
- Check if Valid Command  
- Send Error Message  
- Get Channel Messages  
- Filter Messages by Search Term  
- Messages Found?  
- No Messages Found  
- Error Report  
- Wait Error Report  
- Delete Report Error  

**Node Details:**

- **Check if Valid Command**  
  - *Type:* If  
  - *Role:* Verifies that command parameters such as channel and search terms are valid and present.  
  - *Input:* From Prase Set.  
  - *Output:* On true, proceeds to Get Channel Messages; on false, sends error message.  
  - *Failure Modes:* Incorrect command parameters causing false negatives.

- **Send Error Message**  
  - *Type:* Slack  
  - *Role:* Sends an error message to Slack if the command is invalid.  
  - *Config:* Posts to Slack channel or user with error details.  
  - *Input:* From Check if Valid Command (false branch).  
  - *Output:* To Wait Error Final.  
  - *Failure Modes:* Slack API failures.

- **Get Channel Messages**  
  - *Type:* Slack  
  - *Role:* Retrieves messages from the specified Slack channel.  
  - *Config:* Slack API call with channel ID, with error continuation enabled.  
  - *Input:* From Check if Valid Command (true branch).  
  - *Output:* To Filter Messages by Search Term or Error Report on failure.  
  - *Failure Modes:* API limits, permissions errors, timeouts.

- **Filter Messages by Search Term**  
  - *Type:* Code  
  - *Role:* Filters retrieved messages matching the search term(s) provided in the command.  
  - *Config:* JavaScript filtering logic, case insensitive, partial matches.  
  - *Input:* From Get Channel Messages.  
  - *Output:* To Messages Found?.  
  - *Failure Modes:* Logic errors, empty input array.

- **Messages Found?**  
  - *Type:* If  
  - *Role:* Checks whether any messages matched the filter criteria.  
  - *Input:* From Filter Messages by Search Term.  
  - *Output:* If true, proceeds to Store Pending Deletion and Send Confirmation Message; if false, to No Messages Found.  
  - *Failure Modes:* False negatives due to filter logic.

- **No Messages Found**  
  - *Type:* Slack  
  - *Role:* Sends a Slack message informing no messages matched the search.  
  - *Config:* Posts to user or channel indicating no messages found.  
  - *Input:* From Messages Found? (false branch).  
  - *Output:* To Wait No.M Final.  
  - *Failure Modes:* Slack API errors.

- **Error Report**  
  - *Type:* Slack  
  - *Role:* Posts error details if retrieving messages failed.  
  - *Input:* From Get Channel Messages (on error).  
  - *Output:* To Wait Error Report.  
  - *Failure Modes:* Slack API errors.

- **Wait Error Report**  
  - *Type:* Wait  
  - *Role:* Delay before deleting the error report message.  
  - *Config:* Configured wait period.  
  - *Input:* From Error Report.  
  - *Output:* To Delete Report Error.  
  - *Failure Modes:* Timing misconfiguration.

- **Delete Report Error**  
  - *Type:* Slack  
  - *Role:* Deletes the error report message to keep Slack tidy.  
  - *Input:* From Wait Error Report.  
  - *Output:* End of error reporting flow.  
  - *Failure Modes:* Message already deleted.

---

#### 1.3 User Confirmation and Pending Deletion Storage

**Overview:**  
This block stores details of the messages pending deletion and sends a confirmation message to the user for approval.

**Nodes Involved:**  
- Store Pending Deletion  
- Send Confirmation Message  
- Wait Conf  
- Delete Conf  
- Delete or Cancel?  
- Send Cancel Message  
- Wait Cancel Message  
- Delete Cancel Message  

**Node Details:**

- **Store Pending Deletion**  
  - *Type:* Code  
  - *Role:* Saves the filtered message list and metadata awaiting user confirmation.  
  - *Input:* From Messages Found? (true branch).  
  - *Output:* Ends flow here; confirmation message sent in parallel.  
  - *Failure Modes:* Data storage or variable mishandling.

- **Send Confirmation Message**  
  - *Type:* Slack  
  - *Role:* Sends a message asking the user to confirm or cancel the deletion operation.  
  - *Config:* Interactive Slack message with approve/cancel buttons.  
  - *Input:* From Store Pending Deletion.  
  - *Output:* To Wait Conf.  
  - *Failure Modes:* Slack API and interaction response errors.

- **Wait Conf**  
  - *Type:* Wait  
  - *Role:* Waits for user interaction on confirmation message.  
  - *Input:* From Send Confirmation Message.  
  - *Output:* To Delete Conf.  
  - *Failure Modes:* Timeout or no response.

- **Delete Conf**  
  - *Type:* Slack  
  - *Role:* Deletes the confirmation message after response or timeout.  
  - *Input:* From Wait Conf.  
  - *Output:* Continues to Delete or Cancel? node via Switch node (not explicit in JSON).  
  - *Failure Modes:* Message deletion errors.

- **Delete or Cancel?**  
  - *Type:* If  
  - *Role:* Branches workflow depending on user confirmation (delete) or cancel action.  
  - *Input:* From confirmation response node (If1).  
  - *Output:* To Loop Over Messages if confirmed; to Send Progress Confirmation if canceled.  
  - *Failure Modes:* Incorrect interpretation of user response.

- **Send Cancel Message**  
  - *Type:* Slack  
  - *Role:* Notifies user that the deletion operation has been canceled.  
  - *Input:* From Delete or Cancel? (cancel branch).  
  - *Output:* To Wait Cancel Message.  
  - *Failure Modes:* Slack API errors.

- **Wait Cancel Message**  
  - *Type:* Wait  
  - *Role:* Allows time for user to read cancel message before deletion.  
  - *Input:* From Send Cancel Message.  
  - *Output:* To Delete Cancel Message.  
  - *Failure Modes:* Timing issues.

- **Delete Cancel Message**  
  - *Type:* Slack  
  - *Role:* Deletes the cancel notification message.  
  - *Input:* From Wait Cancel Message.  
  - *Output:* End of cancel flow.  
  - *Failure Modes:* Message already deleted.

---

#### 1.4 Execution of Message Deletion

**Overview:**  
After confirmation, this block deletes the filtered Slack messages in batches, includes pacing with wait nodes to avoid API rate limits, and handles errors during deletion.

**Nodes Involved:**  
- Loop Over Messages  
- Code Count  
- Wait Between Deletes  
- Delete Message  
- Post Error Final  
- Wait for Error Final  
- Delete Error Final  
- Send Progress Confirmation  
- Wait Progress M  
- Delete Progress M  

**Node Details:**

- **Loop Over Messages**  
  - *Type:* SplitInBatches  
  - *Role:* Processes the list of messages in manageable batches to avoid rate limits.  
  - *Config:* Batch size configured (e.g., 10 or 20 messages per batch).  
  - *Input:* From Delete or Cancel? (delete branch).  
  - *Output:* To Code Count or Wait Between Deletes.  
  - *Failure Modes:* Batch processing errors.

- **Code Count**  
  - *Type:* Code  
  - *Role:* Counts current batch messages and logs progress.  
  - *Input:* From Loop Over Messages (first batch).  
  - *Output:* To Send Completion Message.  
  - *Failure Modes:* Counting errors.

- **Wait Between Deletes**  
  - *Type:* Wait  
  - *Role:* Adds delay between delete API calls to respect Slack rate limits.  
  - *Input:* From Loop Over Messages (subsequent batches).  
  - *Output:* To Delete Message.  
  - *Failure Modes:* Timing misconfiguration.

- **Delete Message**  
  - *Type:* Slack  
  - *Role:* Deletes individual messages using Slack API.  
  - *Config:* Uses message timestamp and channel ID.  
  - *Input:* From Wait Between Deletes.  
  - *Output:* On success, loops back to Loop Over Messages; on error, to Post Error Final.  
  - *Failure Modes:* Permission errors, deleted messages, API rate limits.

- **Post Error Final**  
  - *Type:* Slack  
  - *Role:* Posts an error message when deletion fails.  
  - *Input:* From Delete Message (on error).  
  - *Output:* To Wait for Error Final.  
  - *Failure Modes:* Slack API issues.

- **Wait for Error Final**  
  - *Type:* Wait  
  - *Role:* Waits before deleting error message for user visibility.  
  - *Input:* From Post Error Final.  
  - *Output:* To Delete Error Final.  
  - *Failure Modes:* Timing issues.

- **Delete Error Final**  
  - *Type:* Slack  
  - *Role:* Deletes the error notification message.  
  - *Input:* From Wait for Error Final.  
  - *Output:* End of error flow.  
  - *Failure Modes:* Message already deleted.

- **Send Progress Confirmation**  
  - *Type:* Slack  
  - *Role:* Sends a progress update message during deletion batches.  
  - *Input:* From Delete or Cancel? (cancel branch).  
  - *Output:* To Wait Progress M.  
  - *Failure Modes:* Slack API errors.

- **Wait Progress M**  
  - *Type:* Wait  
  - *Role:* Delay before deleting progress message.  
  - *Input:* From Send Progress Confirmation.  
  - *Output:* To Delete Progress M.  
  - *Failure Modes:* Timing issues.

- **Delete Progress M**  
  - *Type:* Slack  
  - *Role:* Deletes progress confirmation message to keep Slack tidy.  
  - *Input:* From Wait Progress M.  
  - *Output:* Ends progress update flow.  
  - *Failure Modes:* Message already deleted.

---

#### 1.5 Post-Deletion Cleanup and Reporting

**Overview:**  
This block handles cleanup of workflow messages, sends completion notifications, and manages error or cancellation messaging post-operation.

**Nodes Involved:**  
- Send Completion Message  
- Wait  
- Clean Up Workflow Messages  
- Loop Over Items  
- Delete Workflow Messages  
- Code Count (also involved here)  
- Error Report (overlaps with retrieval errors)  
- Wait Error Report  
- Delete Report Error  
- Delete Error Message  
- Wait Error Final  
- Delete Error Final  

**Node Details:**

- **Send Completion Message**  
  - *Type:* Slack  
  - *Role:* Notifies users that the bulk deletion operation completed successfully.  
  - *Input:* From Code Count node after last batch.  
  - *Output:* To Wait node for deletion of completion message.  
  - *Failure Modes:* Slack API failures.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Waits before deleting completion message for user to acknowledge.  
  - *Input:* From Send Completion Message.  
  - *Output:* To Clean Up Workflow Messages.  
  - *Failure Modes:* Timing issues.

- **Clean Up Workflow Messages**  
  - *Type:* Code  
  - *Role:* Prepares list of workflow-generated messages for deletion to keep Slack clean.  
  - *Input:* From Wait.  
  - *Output:* To Loop Over Items.  
  - *Failure Modes:* Data preparation errors.

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes cleanup messages in batches.  
  - *Input:* From Clean Up Workflow Messages or Delete Workflow Messages.  
  - *Output:* To Delete Workflow Messages or ends cleanup.  
  - *Failure Modes:* Batch processing errors.

- **Delete Workflow Messages**  
  - *Type:* Slack  
  - *Role:* Deletes workflow-generated messages from Slack channels.  
  - *Input:* From Loop Over Items.  
  - *Output:* Back to Loop Over Items or ends flow.  
  - *Failure Modes:* Permission errors, messages already deleted.

- **Error Report, Wait Error Report, Delete Report Error, Delete Error Message, Wait Error Final, Delete Error Final**  
  - *Roles:* Handle errors at different stages by posting error notifications, waiting, then deleting error messages to avoid clutter.  
  - *Failure Modes:* Slack API errors, timing configuration issues.

---

#### 1.6 Instruction Messaging and Interaction Handling

**Overview:**  
Provides user instructions when commands are unclear or missing, manages wait times and deletion of instruction messages, and handles user message checks.

**Nodes Involved:**  
- Check User Message  
- If1  
- Instruction Message  
- Wait1  
- Delete Instruction  
- Switch (also part of Input Reception)  

**Node Details:**

- **Check User Message**  
  - *Type:* Code  
  - *Role:* Validates user messages for content and format before processing.  
  - *Input:* From Switch node.  
  - *Output:* To If1 node.  
  - *Failure Modes:* Logic errors in validation.

- **If1**  
  - *Type:* If  
  - *Role:* Branches flow based on user message validity.  
  - *Input:* From Check User Message.  
  - *Output:* To Delete or Cancel? or Send Cancel Message.  
  - *Failure Modes:* Incorrect branching.

- **Instruction Message, Wait1, Delete Instruction**  
  - *Previously detailed in 1.1 block.*  
  - *Role:* Provide instructions and clean up after displaying them.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                                | Input Node(s)                          | Output Node(s)                         | Sticky Note                             |
|-------------------------------|---------------------|-----------------------------------------------|--------------------------------------|---------------------------------------|---------------------------------------|
| Webhook                       | Webhook             | Receives Slack command payload                 | None                                 | Respond to Webhook                    |                                       |
| Respond to Webhook            | Code                | Sends immediate HTTP response                   | Webhook                              | Switch                               |                                       |
| Switch                       | Switch              | Routes flow based on message type               | Respond to Webhook                   | Check User Message, Instruction Message, Parse Command |                                       |
| Instruction Message          | Slack               | Sends instructional message                      | Switch                              | Wait1                                |                                       |
| Wait1                        | Wait                | Waits before deleting instructional message     | Instruction Message                 | Delete Instruction                   |                                       |
| Delete Instruction           | Slack               | Deletes instructional message                    | Wait1                               | None                                |                                       |
| Parse Command                | Code                | Parses user command                              | Switch                              | Prase Set                           |                                       |
| Prase Set                   | Set                 | Structures parsed command data                   | Parse Command                      | Check if Valid Command              |                                       |
| Check if Valid Command       | If                  | Validates command parameters                      | Prase Set                         | Get Channel Messages, Send Error Message |                                       |
| Send Error Message           | Slack               | Sends error message for invalid command          | Check if Valid Command             | Wait Error Final                   |                                       |
| Get Channel Messages         | Slack               | Retrieves messages from Slack channel             | Check if Valid Command             | Filter Messages by Search Term, Error Report |                                       |
| Filter Messages by Search Term| Code                | Filters messages by search term                    | Get Channel Messages               | Messages Found?                    |                                       |
| Messages Found?              | If                  | Checks if filtered messages exist                   | Filter Messages by Search Term    | Store Pending Deletion, Send Confirmation Message, No Messages Found |                                       |
| No Messages Found            | Slack               | Notifies no messages matched                        | Messages Found?                   | Wait No.M Final                   |                                       |
| Error Report                | Slack               | Posts error if message retrieval failed             | Get Channel Messages (error)      | Wait Error Report                |                                       |
| Wait Error Report           | Wait                | Waits before deleting error report message          | Error Report                    | Delete Report Error             |                                       |
| Delete Report Error         | Slack               | Deletes error report message                          | Wait Error Report               | None                           |                                       |
| Store Pending Deletion      | Code                | Stores messages pending deletion                     | Messages Found?                  | None                           |                                       |
| Send Confirmation Message  | Slack               | Sends confirmation request for deletion             | Store Pending Deletion           | Wait Conf                     |                                       |
| Wait Conf                  | Wait                | Waits for user response on confirmation message      | Send Confirmation Message       | Delete Conf                   |                                       |
| Delete Conf                | Slack               | Deletes confirmation message                          | Wait Conf                      | Delete or Cancel?             |                                       |
| Delete or Cancel?          | If                  | Branches based on user's confirmation or cancellation| If1                            | Loop Over Messages, Send Progress Confirmation |                                       |
| Send Cancel Message        | Slack               | Notifies user of cancellation                         | Delete or Cancel?               | Wait Cancel Message          |                                       |
| Wait Cancel Message        | Wait                | Waits before deleting cancel message                  | Send Cancel Message             | Delete Cancel Message        |                                       |
| Delete Cancel Message      | Slack               | Deletes cancel notification                            | Wait Cancel Message             | None                        |                                       |
| Loop Over Messages         | SplitInBatches      | Processes deletion in batches                          | Delete or Cancel? (confirmed)   | Code Count, Wait Between Deletes |                                       |
| Code Count                 | Code                | Counts messages in current batch                         | Loop Over Messages             | Send Completion Message       |                                       |
| Wait Between Deletes       | Wait                | Waits between individual message deletions               | Loop Over Messages             | Delete Message              |                                       |
| Delete Message             | Slack               | Deletes individual Slack message                          | Wait Between Deletes           | Loop Over Messages, Post Error Final |                                       |
| Post Error Final           | Slack               | Posts error message if deletion fails                    | Delete Message (error)          | Wait for Error Final        |                                       |
| Wait for Error Final       | Wait                | Waits before deleting error message                       | Post Error Final               | Delete Error Final          |                                       |
| Delete Error Final         | Slack               | Deletes error message after wait                           | Wait for Error Final           | None                       |                                       |
| Send Progress Confirmation | Slack               | Sends progress update message                              | Delete or Cancel? (cancel)     | Wait Progress M             |                                       |
| Wait Progress M            | Wait                | Waits before deleting progress message                       | Send Progress Confirmation    | Delete Progress M           |                                       |
| Delete Progress M          | Slack               | Deletes progress notification message                         | Wait Progress M               | None                       |                                       |
| Send Completion Message    | Slack               | Notifies completion of deletion operation                   | Code Count                   | Wait                      |                                       |
| Wait                      | Wait                | Waits before deleting completion message                      | Send Completion Message       | Clean Up Workflow Messages   |                                       |
| Clean Up Workflow Messages | Code                | Prepares workflow-generated messages for cleanup              | Wait                        | Loop Over Items             |                                       |
| Loop Over Items            | SplitInBatches      | Batches messages for deletion                                 | Clean Up Workflow Messages, Delete Workflow Messages | Delete Workflow Messages, None |                                       |
| Delete Workflow Messages   | Slack               | Deletes messages generated by workflow                         | Loop Over Items              | Loop Over Items or None     |                                       |
| Check User Message         | Code                | Validates user message content                                | Switch                      | If1                      |                                       |
| If1                       | If                  | Branches based on user message validity                        | Check User Message           | Delete or Cancel?, Send Cancel Message |                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Purpose: Receive Slack slash command or interactive payloads.  
   - Configuration: Default path, no authentication, listens for POST requests.

2. **Add Code Node: Respond to Webhook**  
   - Purpose: Immediately respond to Slack webhook to prevent timeout.  
   - Configuration: JavaScript code returning 200 OK JSON response.  
   - Connect Webhook → Respond to Webhook.

3. **Add Switch Node**  
   - Purpose: Route based on message type (commands, instructions).  
   - Configure conditions to detect instruction messages, commands, or others.  
   - Connect Respond to Webhook → Switch.

4. **Add Slack Node: Instruction Message**  
   - Purpose: Send help/instruction messages to user.  
   - Configure Slack credentials and message content with instructions.  
   - Connect Switch (instruction branch) → Instruction Message.

5. **Add Wait Node: Wait1**  
   - Purpose: Wait before deleting instructional message.  
   - Set appropriate wait time (e.g., 60 seconds).  
   - Connect Instruction Message → Wait1.

6. **Add Slack Node: Delete Instruction**  
   - Purpose: Delete instructional messages after wait.  
   - Configure to delete message by timestamp from Instruction Message.  
   - Connect Wait1 → Delete Instruction.

7. **Add Code Node: Parse Command**  
   - Purpose: Extract parameters from user command text (channel, search terms).  
   - Write JavaScript code parsing Slack command payload.  
   - Connect Switch (command branch) → Parse Command.

8. **Add Set Node: Prase Set**  
   - Purpose: Format and store parsed command parameters into variables.  
   - Define variables: channel ID, search term, limit, etc.  
   - Connect Parse Command → Prase Set.

9. **Add If Node: Check if Valid Command**  
   - Purpose: Validate presence and correctness of command parameters.  
   - Condition: Check necessary variables are defined and non-empty.  
   - Connect Prase Set → Check if Valid Command.

10. **Add Slack Node: Send Error Message**  
    - Purpose: Notify user of invalid command.  
    - Configure error message content.  
    - Connect Check if Valid Command (false branch) → Send Error Message.

11. **Add Slack Node: Get Channel Messages**  
    - Purpose: Retrieve messages from specified Slack channel.  
    - Configure Slack API method to fetch channel history.  
    - Connect Check if Valid Command (true branch) → Get Channel Messages.

12. **Add Code Node: Filter Messages by Search Term**  
    - Purpose: Filter retrieved messages by provided search term.  
    - Write JavaScript to filter messages array.  
    - Connect Get Channel Messages → Filter Messages by Search Term.

13. **Add If Node: Messages Found?**  
    - Purpose: Check if any messages matched filter.  
    - Condition: Filtered messages array length > 0.  
    - Connect Filter Messages by Search Term → Messages Found?.

14. **Add Slack Node: No Messages Found**  
    - Purpose: Notify user no messages matched criteria.  
    - Connect Messages Found? (false branch) → No Messages Found.

15. **Add Wait Node: Wait No.M Final**  
    - Purpose: Wait before deleting no messages found notification.  
    - Connect No Messages Found → Wait No.M Final.

16. **Add Slack Node: Delete No Messages**  
    - Purpose: Delete no messages notification after wait.  
    - Connect Wait No.M Final → Delete No Messages.

17. **Add Slack Node: Error Report**  
    - Purpose: Notify of errors during message retrieval.  
    - Connect Get Channel Messages (on error) → Error Report.

18. **Add Wait Node: Wait Error Report**  
    - Connect Error Report → Wait Error Report.

19. **Add Slack Node: Delete Report Error**  
    - Connect Wait Error Report → Delete Report Error.

20. **Add Code Node: Store Pending Deletion**  
    - Purpose: Store filtered messages for deletion upon confirmation.  
    - Connect Messages Found? (true branch) → Store Pending Deletion.

21. **Add Slack Node: Send Confirmation Message**  
    - Purpose: Ask user to confirm or cancel deletion.  
    - Configure interactive Slack message with buttons.  
    - Connect Store Pending Deletion → Send Confirmation Message.

22. **Add Wait Node: Wait Conf**  
    - Purpose: Wait for user response to confirmation.  
    - Connect Send Confirmation Message → Wait Conf.

23. **Add Slack Node: Delete Conf**  
    - Purpose: Delete confirmation message after response or timeout.  
    - Connect Wait Conf → Delete Conf.

24. **Add Code Node: Check User Message**  
    - Purpose: Validate user response (confirm or cancel).  
    - Connect Switch → Check User Message.

25. **Add If Node: If1**  
    - Purpose: Branch based on user confirmation or cancellation.  
    - Connect Check User Message → If1.

26. **Add If Node: Delete or Cancel?**  
    - Purpose: Branch to deletion or cancellation flows.  
    - Connect If1 → Delete or Cancel?.

27. **Add Slack Node: Send Cancel Message**  
    - Purpose: Notify user deletion was canceled.  
    - Connect Delete or Cancel? (cancel branch) → Send Cancel Message.

28. **Add Wait Node: Wait Cancel Message**  
    - Connect Send Cancel Message → Wait Cancel Message.

29. **Add Slack Node: Delete Cancel Message**  
    - Connect Wait Cancel Message → Delete Cancel Message.

30. **Add SplitInBatches Node: Loop Over Messages**  
    - Purpose: Batch process message deletions.  
    - Connect Delete or Cancel? (delete branch) → Loop Over Messages.

31. **Add Code Node: Code Count**  
    - Purpose: Count and log messages in batch.  
    - Connect Loop Over Messages → Code Count.

32. **Add Slack Node: Send Completion Message**  
    - Purpose: Notify user of completion.  
    - Connect Code Count → Send Completion Message.

33. **Add Wait Node: Wait**  
    - Purpose: Wait before deleting completion notification.  
    - Connect Send Completion Message → Wait.

34. **Add Code Node: Clean Up Workflow Messages**  
    - Purpose: Prepare messages for workflow cleanup.  
    - Connect Wait → Clean Up Workflow Messages.

35. **Add SplitInBatches Node: Loop Over Items**  
    - Purpose: Batch process cleanup messages.  
    - Connect Clean Up Workflow Messages → Loop Over Items.

36. **Add Slack Node: Delete Workflow Messages**  
    - Purpose: Delete workflow-generated messages.  
    - Connect Loop Over Items → Delete Workflow Messages.

37. **Add Wait Node: Wait Between Deletes**  
    - Purpose: Wait between deleting messages to avoid rate limits.  
    - Connect Loop Over Messages → Wait Between Deletes.

38. **Add Slack Node: Delete Message**  
    - Purpose: Delete each Slack message.  
    - Connect Wait Between Deletes → Delete Message.

39. **Add Slack Node: Post Error Final**  
    - Purpose: Notify user of deletion errors.  
    - Connect Delete Message (on error) → Post Error Final.

40. **Add Wait Node: Wait for Error Final**  
    - Connect Post Error Final → Wait for Error Final.

41. **Add Slack Node: Delete Error Final**  
    - Connect Wait for Error Final → Delete Error Final.

42. **Add Slack Node: Send Progress Confirmation**  
    - Purpose: Notify user of cancellation progress.  
    - Connect Delete or Cancel? (cancel branch) → Send Progress Confirmation.

43. **Add Wait Node: Wait Progress M**  
    - Connect Send Progress Confirmation → Wait Progress M.

44. **Add Slack Node: Delete Progress M**  
    - Connect Wait Progress M → Delete Progress M.

#### Credential Setup:

- Slack OAuth2 credentials must be configured with permissions for reading channel history, posting messages, deleting messages, and interacting with Slack apps.
- Ensure webhook URLs in Slack app match the Webhook node URLs in n8n.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow requires Slack OAuth2 credentials with the following scopes: channels:history, chat:write, chat:write.public, chat:delete, commands, and interactions. | Slack API Documentation: https://api.slack.com/scopes                                           |
| Interactive Slack messages are used for user confirmation, requiring Slack app setup with interactivity enabled. | Slack Interactivity Guide: https://api.slack.com/interactivity                                  |
| Rate limiting is mitigated using Wait nodes between deletion calls; recommended batch size is 10-20 messages. | Slack API Rate Limits: https://api.slack.com/docs/rate-limits                                   |
| The workflow cleans up its own messages (confirmation, progress, error) to avoid cluttering Slack channels. | Important for maintaining Slack channel hygiene.                                                |
| The workflow uses extensive error handling and message deletion to ensure clean user experience.        |                                                                                                |
| For detailed Slack API error codes, consult Slack's error documentation.                                | Slack Error Codes: https://api.slack.com/methods/chat.delete#errors                             |

---

**Disclaimer:** The provided documentation is based exclusively on an automated n8n workflow designed for Slack message bulk deletion with advanced filtering and confirmation features. It adheres strictly to current content policies and does not include any illegal or protected content. All processed data is publicly available or authorized for use.