Telegram Bot with Menu-Driven Commands

https://n8nworkflows.xyz/workflows/telegram-bot-with-menu-driven-commands-3055


# Telegram Bot with Menu-Driven Commands

### 1. Workflow Overview

This workflow implements a **Telegram Bot with Menu-Driven Commands** designed for deterministic, command-based user interactions without relying on conversational AI. It enables users to select predefined commands from a Telegram bot menu and then provide specific content in response to prompts. The workflow manages conversation state internally to track which command is active and routes user input accordingly.

**Target Use Cases:**  
- Bots requiring structured command input followed by user content (e.g., forms, service requests, tutorials).  
- Scenarios where predictable, menu-driven interactions are preferred over free-text or AI-driven conversations.

**Logical Blocks:**

- **1.1 Initialization & State Management:** Setup and maintain conversation state per Telegram chat using static data.  
- **1.2 Telegram Input Reception:** Capture incoming Telegram messages and commands via webhook trigger.  
- **1.3 Command Routing:** Determine which command was triggered and update the conversation state accordingly.  
- **1.4 Content Requesting:** Prompt users to provide content after command selection.  
- **1.5 Content Processing:** Handle the user-provided content for each command, with placeholders for custom logic.  
- **1.6 State Clearing & Final Response:** Send final responses and reset conversation state after processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & State Management

**Overview:**  
This block initializes the static data object used to store conversation states for each Telegram chat and manages state transitions throughout the workflow.

**Nodes Involved:**  
- Temp to Initiate Static Data  
- Prepare IF Value  
- Check State  
- Clear State  
- Set waitingForContent1  
- Set waitingForContent2  
- Set waitingForContent3  

**Node Details:**

- **Temp to Initiate Static Data**  
  - *Type:* Code  
  - *Role:* One-time initialization of `telegramStates` object in workflow static data to track chat states.  
  - *Config:* Creates an empty object if not present.  
  - *Input:* None (manual trigger recommended once).  
  - *Output:* Static data initialized.  
  - *Failures:* None expected unless static data access fails.  
  - *Notes:* Must be run once before using the bot.

- **Prepare IF Value**  
  - *Type:* Code  
  - *Role:* Prepares a value used for conditional checks in the next node.  
  - *Config:* Extracts or formats data from incoming message to check state.  
  - *Input:* From Command Check node.  
  - *Output:* Passes prepared data to Check State.  
  - *Failures:* Expression errors if input data missing.

- **Check State**  
  - *Type:* Switch  
  - *Role:* Routes execution based on the current conversation state stored in static data for the chat.  
  - *Config:* Checks if the chat is waiting for content for Command1, Command2, Command3, or none.  
  - *Input:* From Prepare IF Value.  
  - *Output:* Routes to respective command processing or no command check.  
  - *Failures:* State lookup failure if static data corrupted.

- **Clear State**  
  - *Type:* Code  
  - *Role:* Resets the conversation state for the chat after command processing is complete.  
  - *Config:* Deletes or clears the chat's state in static data.  
  - *Input:* From Command result nodes.  
  - *Output:* Passes control downstream (usually ends flow).  
  - *Failures:* Static data write failure.

- **Set waitingForContent1 / Set waitingForContent2 / Set waitingForContent3**  
  - *Type:* Code  
  - *Role:* Sets the conversation state to indicate the bot is waiting for content input for the respective command.  
  - *Config:* Updates static data for the chat with a flag like `waitingForContent1 = true`.  
  - *Input:* From Switch (Command Routing).  
  - *Output:* Triggers content request nodes.  
  - *Failures:* Static data write failure.

---

#### 1.2 Telegram Input Reception

**Overview:**  
Receives incoming Telegram messages and commands via webhook and triggers the workflow.

**Nodes Involved:**  
- Telegram Trigger  
- Command Check  
- Send Typing action  

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Webhook node that listens for all incoming Telegram updates (messages, commands).  
  - *Config:* Uses bot API token credential; no filters applied to capture all messages.  
  - *Input:* External Telegram messages.  
  - *Output:* Passes message data to Command Check and Send Typing action nodes.  
  - *Failures:* Webhook misconfiguration, credential errors, Telegram API downtime.

- **Command Check**  
  - *Type:* If  
  - *Role:* Checks if the incoming message contains a recognized command (`/command1`, `/command2`, `/command3`).  
  - *Config:* Uses expressions to detect command text in message.  
  - *Input:* From Telegram Trigger.  
  - *Output:* Routes to Switch (Command Routing) if command detected, else to Prepare IF Value.  
  - *Failures:* Expression errors if message format unexpected.

- **Send Typing action**  
  - *Type:* Telegram  
  - *Role:* Sends "typing..." action to Telegram chat to indicate bot is processing.  
  - *Config:* Uses chat ID from incoming message; no message text sent.  
  - *Input:* From Telegram Trigger (parallel to Command Check).  
  - *Output:* None (side effect only).  
  - *Failures:* Telegram API errors, credential issues.

---

#### 1.3 Command Routing

**Overview:**  
Routes execution based on the detected command and sets the conversation state to await user content.

**Nodes Involved:**  
- Switch (Command Routing)  
- Set waitingForContent1  
- Set waitingForContent2  
- Set waitingForContent3  

**Node Details:**

- **Switch (Command Routing)**  
  - *Type:* Switch  
  - *Role:* Routes based on the command text (`/command1`, `/command2`, `/command3`).  
  - *Config:* Checks incoming message text against command strings.  
  - *Input:* From Command Check node.  
  - *Output:* Routes to respective Set waitingForContent nodes.  
  - *Failures:* Expression errors if message text missing.

- **Set waitingForContent1 / Set waitingForContent2 / Set waitingForContent3**  
  - *Described above in 1.1.*

---

#### 1.4 Content Requesting

**Overview:**  
Sends a Telegram message prompting the user to provide the content required for the selected command.

**Nodes Involved:**  
- Command1 content request  
- Command2 content request  
- Command3 content request  

**Node Details:**

- **CommandX content request (Telegram nodes)**  
  - *Type:* Telegram  
  - *Role:* Sends a message asking the user to input the content for the respective command.  
  - *Config:* Uses chat ID from incoming message; message text is a prompt like "Please provide content for Command1."  
  - *Input:* From Set waitingForContentX nodes.  
  - *Output:* None (awaits user reply).  
  - *Failures:* Telegram API errors, credential issues.

---

#### 1.5 Content Processing

**Overview:**  
Processes the user-provided content after the bot has requested it, then sends a result message and clears the conversation state.

**Nodes Involved:**  
- Check State  
- Command1 processing  
- Command2 processing  
- Command3 processing  
- Command1 result  
- Command2 result  
- Command3 result  
- Clear State  

**Node Details:**

- **Check State**  
  - *Described above in 1.1.* Routes to the appropriate command processing node based on stored state.

- **CommandX processing (NoOp nodes)**  
  - *Type:* No Operation (placeholder)  
  - *Role:* Placeholder for custom logic to process the content received for each command.  
  - *Config:* No configuration; user should replace with actual processing nodes (e.g., validation, database storage).  
  - *Input:* From Check State node.  
  - *Output:* Routes to CommandX result node.  
  - *Failures:* None by default; user-added logic may introduce errors.

- **CommandX result (Telegram nodes)**  
  - *Type:* Telegram  
  - *Role:* Sends a confirmation or result message back to the user after processing.  
  - *Config:* Uses chat ID; message text can be customized to confirm receipt or show results.  
  - *Input:* From CommandX processing node.  
  - *Output:* Routes to Clear State node.  
  - *Failures:* Telegram API errors, credential issues.

- **Clear State**  
  - *Described above in 1.1.* Resets conversation state to allow new commands.

---

#### 1.6 No Command / Default Handling

**Overview:**  
Handles messages received when no command is active or recognized.

**Nodes Involved:**  
- No Command check  

**Node Details:**

- **No Command check**  
  - *Type:* Telegram  
  - *Role:* Sends a default message or prompt when the user sends input without an active command or after state is cleared.  
  - *Config:* Uses chat ID; message text can be a help message or menu prompt.  
  - *Input:* From Check State node (when no waiting state).  
  - *Output:* None.  
  - *Failures:* Telegram API errors.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                      |
|-------------------------|---------------------|----------------------------------------|--------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger    | Receives Telegram messages and commands | -                        | Command Check, Send Typing action |                                                                                                 |
| Command Check           | If                  | Detects which command was sent          | Telegram Trigger          | Switch (Command Routing), Prepare IF Value |                                                                                                 |
| Switch (Command Routing)| Switch              | Routes based on command text             | Command Check             | Set waitingForContent1/2/3      |                                                                                                 |
| Set waitingForContent1  | Code                | Sets state to wait for Command1 content | Switch (Command Routing)  | Command1 content request        |                                                                                                 |
| Set waitingForContent2  | Code                | Sets state to wait for Command2 content | Switch (Command Routing)  | Command2 content request        |                                                                                                 |
| Set waitingForContent3  | Code                | Sets state to wait for Command3 content | Switch (Command Routing)  | Command3 content request        |                                                                                                 |
| Command1 content request| Telegram            | Prompts user for Command1 content       | Set waitingForContent1    | -                              |                                                                                                 |
| Command2 content request| Telegram            | Prompts user for Command2 content       | Set waitingForContent2    | -                              |                                                                                                 |
| Command3 content request| Telegram            | Prompts user for Command3 content       | Set waitingForContent3    | -                              |                                                                                                 |
| Prepare IF Value        | Code                | Prepares data for state check            | Command Check             | Check State                    |                                                                                                 |
| Check State             | Switch              | Routes based on current conversation state | Prepare IF Value          | Command1/2/3 processing, No Command check |                                                                                                 |
| Command1 processing     | NoOp                | Placeholder for Command1 content logic   | Check State               | Command1 result                |                                                                                                 |
| Command2 processing     | NoOp                | Placeholder for Command2 content logic   | Check State               | Command2 result                |                                                                                                 |
| Command3 processing     | NoOp                | Placeholder for Command3 content logic   | Check State               | Command3 result                |                                                                                                 |
| Command1 result         | Telegram            | Sends confirmation after Command1 processing | Command1 processing       | Clear State                   |                                                                                                 |
| Command2 result         | Telegram            | Sends confirmation after Command2 processing | Command2 processing       | Clear State                   |                                                                                                 |
| Command3 result         | Telegram            | Sends confirmation after Command3 processing | Command3 processing       | Clear State                   |                                                                                                 |
| Clear State             | Code                | Clears conversation state after processing | Command1/2/3 result       | -                              |                                                                                                 |
| No Command check        | Telegram            | Sends message when no command active     | Check State               | -                              |                                                                                                 |
| Send Typing action      | Telegram            | Sends "typing..." action to user         | Telegram Trigger          | -                              |                                                                                                 |
| Temp to Initiate Static Data | Code           | One-time initialization of static data   | -                        | -                              | You only need to run the initialization step once per workflow, regardless of the number of Telegram chat IDs. The initialization creates the telegramStates object within the global static data of the workflow. Once that object exists, the workflow will use it to store the state for any chat ID. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API credentials.  
   - No filters; listens to all messages.  
   - Position: Start of workflow.

2. **Add Command Check Node (If)**  
   - Type: If  
   - Condition: Check if incoming message text equals `/command1`, `/command2`, or `/command3`.  
   - Input: Connect from Telegram Trigger.  
   - Output: True branch to Switch (Command Routing), False branch to Prepare IF Value.

3. **Add Switch (Command Routing) Node**  
   - Type: Switch  
   - Conditions: Route based on exact command text (`/command1`, `/command2`, `/command3`).  
   - Input: Connect from Command Check True branch.  
   - Output: Connect to Set waitingForContent1, Set waitingForContent2, Set waitingForContent3 respectively.

4. **Create Set waitingForContent1 Node (Code)**  
   - Type: Code  
   - Function: Update static data for current chat ID to set `waitingForContent1 = true`.  
   - Input: Connect from Switch (Command Routing) `/command1` output.  
   - Output: Connect to Command1 content request node.

5. **Create Set waitingForContent2 Node (Code)**  
   - Same as above but sets `waitingForContent2 = true`.  
   - Connect from Switch `/command2`.  
   - Output to Command2 content request.

6. **Create Set waitingForContent3 Node (Code)**  
   - Same as above but sets `waitingForContent3 = true`.  
   - Connect from Switch `/command3`.  
   - Output to Command3 content request.

7. **Create Command1 content request Node (Telegram)**  
   - Type: Telegram  
   - Sends message: "Please provide content for Command1."  
   - Input: Connect from Set waitingForContent1.  
   - Use same Telegram credentials.

8. **Create Command2 content request Node (Telegram)**  
   - Similar to above with message for Command2.  
   - Connect from Set waitingForContent2.

9. **Create Command3 content request Node (Telegram)**  
   - Similar to above with message for Command3.  
   - Connect from Set waitingForContent3.

10. **Create Prepare IF Value Node (Code)**  
    - Type: Code  
    - Extracts chat ID and message text for state checking.  
    - Input: Connect from Command Check False branch.  
    - Output: Connect to Check State node.

11. **Create Check State Node (Switch)**  
    - Type: Switch  
    - Checks static data for current chat ID flags: `waitingForContent1`, `waitingForContent2`, `waitingForContent3`.  
    - Input: Connect from Prepare IF Value.  
    - Output: Route to Command1 processing, Command2 processing, Command3 processing, or No Command check.

12. **Create Command1 processing Node (NoOp)**  
    - Placeholder node for user to add processing logic for Command1 content.  
    - Input: Connect from Check State.  
    - Output: Connect to Command1 result node.

13. **Create Command2 processing Node (NoOp)**  
    - Same as above for Command2.

14. **Create Command3 processing Node (NoOp)**  
    - Same as above for Command3.

15. **Create Command1 result Node (Telegram)**  
    - Sends confirmation message after processing Command1 content.  
    - Input: Connect from Command1 processing.  
    - Output: Connect to Clear State node.

16. **Create Command2 result Node (Telegram)**  
    - Same as above for Command2.

17. **Create Command3 result Node (Telegram)**  
    - Same as above for Command3.

18. **Create Clear State Node (Code)**  
    - Clears the conversation state for the current chat ID in static data.  
    - Input: Connect from all CommandX result nodes.  
    - Output: Ends flow or optionally loops back.

19. **Create No Command check Node (Telegram)**  
    - Sends a default message when no command is active or recognized.  
    - Input: Connect from Check State no-match output.

20. **Create Send Typing action Node (Telegram)**  
    - Sends "typing..." action to user to indicate processing.  
    - Input: Connect from Telegram Trigger (parallel to Command Check).  
    - No output needed.

21. **Create Temp to Initiate Static Data Node (Code)**  
    - One-time initialization node to create `telegramStates` object in static data.  
    - Run manually once before starting bot.  
    - No input or output connections.

22. **Configure all Telegram nodes with your Telegram Bot API credentials.**

23. **Test the workflow:**  
    - Run the initialization node once.  
    - Start the workflow active.  
    - Use Telegram bot to send `/command1`, `/command2`, or `/command3`.  
    - Verify prompts and responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This template provides a deterministic, menu-driven command handling approach for Telegram bots, avoiding conversational AI. | Workflow description and unique value proposition.                                              |
| Initialization of static data (`telegramStates`) is required once per workflow to enable state tracking.                     | See "Temp to Initiate Static Data" node description.                                           |
| Replace the NoOp nodes with your own processing logic and add input validation for robustness.                               | Customization instructions in workflow description.                                           |
| Telegram credentials must be configured in all Telegram nodes (Trigger and message nodes).                                   | Telegram Bot API token setup.                                                                   |
| This approach is ideal for bots requiring structured input after command selection, such as forms or service requests.       | Use case examples in workflow description.                                                     |
| For more info on Telegram node usage in n8n, refer to: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.telegram/ | Official n8n Telegram node documentation.                                                      |

---

This structured documentation enables developers and AI agents to fully understand, reproduce, and customize the Telegram bot workflow for menu-driven commands with clear state management and extensible processing logic.