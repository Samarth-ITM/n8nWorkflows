Telegram bot for item multi select and saving to Postgres (Module "Checkbox")

https://n8nworkflows.xyz/workflows/telegram-bot-for-item-multi-select-and-saving-to-postgres--module--checkbox---3648


# Telegram bot for item multi select and saving to Postgres (Module "Checkbox")

### 1. Workflow Overview

This workflow implements a Telegram bot module that allows users to select multiple items from a predefined list stored in a Postgres database and saves their selections back to the database. It is designed for developers and automation specialists who want to collect structured user input efficiently via Telegram and maintain a clean chat interface by deleting intermediate messages.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**: Receives Telegram updates, initializes variables, and manages the start of interaction.
- **1.2 User Interaction and Message Management**: Sends Telegram messages, edits messages, and deletes previous messages to keep the chat clean.
- **1.3 Data Retrieval and Storage**: Reads from and writes to the Postgres database, including fetching the shop list, user choices, and managing orders.
- **1.4 Data Processing and Conversion**: Converts raw database data and user input into formats suitable for Telegram display and database updates.
- **1.5 Decision Logic and Flow Control**: Uses conditional and switch nodes to route the workflow based on user input and state.
- **1.6 Finalization and Confirmation**: Sends success messages and performs cleanup actions after saving user selections.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
This block captures incoming Telegram updates via a webhook trigger, sets initial variables, and determines if the user is starting a new interaction.

**Nodes Involved:**  
- Telegram Trigger  
- Variables TG  
- Initialization  
- Start? (If)  
- Welcome Message  
- Upsert bot status on START (Postgres)  

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger (Webhook)  
  - Role: Entry point for Telegram updates (messages, callbacks)  
  - Configuration: Listens for updates from the configured Telegram bot token  
  - Inputs: Incoming Telegram webhook events  
  - Outputs: Passes data to "Variables TG"  
  - Failure modes: Webhook misconfiguration, invalid token, network issues  

- **Variables TG**  
  - Type: Set  
  - Role: Initializes or sets workflow variables based on incoming data  
  - Configuration: Sets variables like user ID, chat ID, message text, callback data  
  - Inputs: Telegram Trigger output  
  - Outputs: Passes to "Initialization"  
  - Failure modes: Expression errors if expected fields missing  

- **Initialization**  
  - Type: Set  
  - Role: Further variable initialization or resetting state variables  
  - Configuration: Sets flags or default values for the session  
  - Inputs: Variables TG output  
  - Outputs: Passes to "Start?" node  
  - Failure modes: Expression errors  

- **Start?**  
  - Type: If  
  - Role: Checks if the user input corresponds to a start command or initial interaction  
  - Configuration: Condition on message text or callback data (e.g., "/start")  
  - Inputs: Initialization output  
  - Outputs: True branch to "Welcome Message", False branch to "Define Type"  
  - Failure modes: Incorrect condition logic  

- **Welcome Message**  
  - Type: Telegram (Send Message)  
  - Role: Sends a welcome message to the user at the start of interaction  
  - Configuration: Predefined welcome text, uses Telegram credentials  
  - Inputs: Start? true branch  
  - Outputs: Passes to "Upsert bot status on START"  
  - Failure modes: Telegram API errors, invalid chat ID  

- **Upsert bot status on START**  
  - Type: Postgres  
  - Role: Inserts or updates the bot status record for the user in the database  
  - Configuration: SQL query to upsert user status on start  
  - Inputs: Welcome Message output  
  - Outputs: Passes to "Define Type"  
  - Failure modes: Database connection errors, SQL errors  

---

#### 1.2 User Interaction and Message Management

**Overview:**  
This block manages sending messages, editing messages, and deleting previous messages to keep the Telegram chat clean during the multi-select process.

**Nodes Involved:**  
- Send (Telegram)  
- Edit (Telegram)  
- Delete Prev SMS (Telegram)  
- Delete Prev SMS (Telegram) [multiple instances]  
- Results (Telegram)  
- Success (Telegram)  

**Node Details:**  

- **Send** (multiple nodes named "Send")  
  - Type: Telegram (Send Message)  
  - Role: Sends messages with options or results to the user  
  - Configuration: Uses Telegram bot token, sends text or keyboard buttons  
  - Inputs: Various nodes depending on flow (e.g., after data conversion)  
  - Outputs: Passes to Postgres nodes for message_id update  
  - Failure modes: Telegram API rate limits, invalid chat ID  

- **Edit**  
  - Type: Telegram (Edit Message)  
  - Role: Edits existing Telegram messages to update options or status  
  - Configuration: Uses message_id stored in DB, edits text or inline keyboard  
  - Inputs: After data processing and conversion  
  - Outputs: Passes to Postgres node to update message_id  
  - Failure modes: Message not found, Telegram API errors  

- **Delete Prev SMS** (multiple nodes)  
  - Type: Telegram (Delete Message)  
  - Role: Deletes previous messages to keep chat clean  
  - Configuration: Uses message_id retrieved from DB  
  - Inputs: After fetching message_id from Postgres  
  - Outputs: Passes to next message sending or processing node  
  - Failure modes: Message already deleted, Telegram API errors  

- **Results**  
  - Type: Telegram (Send Message)  
  - Role: Sends the final list of selected items or summary to the user  
  - Configuration: Prepares summary message with user selections  
  - Inputs: After data aggregation and formatting  
  - Outputs: Passes to Postgres update for message_id  
  - Failure modes: Telegram API errors  

- **Success**  
  - Type: Telegram (Send Message)  
  - Role: Sends confirmation of successful save to the user  
  - Configuration: Static success message text  
  - Inputs: After saving user choices to DB  
  - Outputs: Passes to message deletion nodes  
  - Failure modes: Telegram API errors  

---

#### 1.3 Data Retrieval and Storage

**Overview:**  
This block handles all database interactions with Postgres, including fetching the shop list, user choices, orders, and updating message IDs for message management.

**Nodes Involved:**  
- Get Shop List (Postgres)  
- Add Shop List Choice (Postgres)  
- Get Shop List Choice (Postgres)  
- Update Shop List Choice (Postgres)  
- Get Shop List Choices (Postgres)  
- Get Shop List Choices (Postgres) [variant]  
- Add Order (Postgres)  
- Get Order (Postgres) [multiple nodes]  
- Update message_id (Postgres) [multiple nodes]  

**Node Details:**  

- **Get Shop List**  
  - Type: Postgres  
  - Role: Retrieves the list of available items from the `shop_list` table  
  - Configuration: SQL SELECT query on `shop_list` table  
  - Inputs: Triggered after order creation or initialization  
  - Outputs: Passes data to "Add Shop List Choice" or conversion nodes  
  - Failure modes: DB connection failure, empty result sets  

- **Add Shop List Choice**  
  - Type: Postgres  
  - Role: Inserts a new user choice record into the database  
  - Configuration: SQL INSERT with user ID and selected items  
  - Inputs: After fetching shop list and user input processing  
  - Outputs: Passes to data conversion nodes  
  - Failure modes: Duplicate entries, DB constraints  

- **Get Shop List Choice**  
  - Type: Postgres  
  - Role: Retrieves existing user choices for editing or validation  
  - Configuration: SQL SELECT query filtered by user ID  
  - Inputs: After updating shop list choice or on demand  
  - Outputs: Passes to conversion and editing nodes  
  - Failure modes: No existing choices found  

- **Update Shop List Choice**  
  - Type: Postgres  
  - Role: Updates existing user choice records with new selections  
  - Configuration: SQL UPDATE query based on user ID and message ID  
  - Inputs: After message ID retrieval and user input processing  
  - Outputs: Passes to "Get Shop List Choice" for confirmation  
  - Failure modes: Update conflicts, missing records  

- **Get Shop List Choices** (two variants)  
  - Type: Postgres  
  - Role: Retrieves multiple user choices or aggregated data  
  - Configuration: SQL SELECT with possible joins or filters  
  - Inputs: After order retrieval or user input processing  
  - Outputs: Passes to "Active?" or conversion nodes  
  - Failure modes: Empty results, DB errors  

- **Add Order**  
  - Type: Postgres  
  - Role: Creates a new order record for the user session  
  - Configuration: SQL INSERT with user and session details  
  - Inputs: After user starts a new selection  
  - Outputs: Passes to "Get Shop List"  
  - Failure modes: DB insert errors  

- **Get Order** (multiple nodes)  
  - Type: Postgres  
  - Role: Retrieves order details for the current user session  
  - Configuration: SQL SELECT filtered by user ID and session  
  - Inputs: After user input or decision nodes  
  - Outputs: Passes to shop list choices or message ID retrieval  
  - Failure modes: Missing order records  

- **Update message_id** (multiple nodes)  
  - Type: Postgres  
  - Role: Updates stored Telegram message IDs for message management  
  - Configuration: SQL UPDATE queries to keep track of messages sent  
  - Inputs: After sending or editing Telegram messages  
  - Outputs: Passes to next workflow steps or message deletion  
  - Failure modes: DB update errors  

---

#### 1.4 Data Processing and Conversion

**Overview:**  
This block processes raw data from the database and user input, converting statuses and formatting lists for Telegram display and database updates.

**Nodes Involved:**  
- Convert statuses (Code)  
- Convert statuses and antistatuses (Code)  
- Convert statuses  (Code) [variant]  
- Union Number with Text (Set)  
- Union List (Summarize)  

**Node Details:**  

- **Convert statuses** (two nodes)  
  - Type: Code (JavaScript)  
  - Role: Transforms raw database status codes or user selections into human-readable text or structured data  
  - Configuration: Custom JavaScript logic, likely mapping IDs to labels  
  - Inputs: Postgres query outputs or user input data  
  - Outputs: Passes formatted data to Telegram send nodes  
  - Failure modes: Runtime errors, unexpected data formats  

- **Convert statuses and antistatuses**  
  - Type: Code (JavaScript)  
  - Role: Similar to "Convert statuses" but handles additional logic for inverse or complementary statuses  
  - Configuration: Custom JavaScript with conditional logic  
  - Inputs: User choices and shop list data  
  - Outputs: Passes to "Edit" Telegram node  
  - Failure modes: Logic errors, data inconsistencies  

- **Union Number with Text**  
  - Type: Set  
  - Role: Combines numeric IDs with descriptive text for display or storage  
  - Configuration: Sets new fields by concatenating or formatting data  
  - Inputs: After "Active?" node  
  - Outputs: Passes to "Union List"  
  - Failure modes: Expression errors  

- **Union List**  
  - Type: Summarize  
  - Role: Aggregates multiple records or selections into a single summarized string or object  
  - Configuration: Aggregation settings (e.g., join text fields)  
  - Inputs: After "Union Number with Text"  
  - Outputs: Passes to message ID retrieval and message sending  
  - Failure modes: Empty input data  

---

#### 1.5 Decision Logic and Flow Control

**Overview:**  
This block controls the workflow routing based on user input, session state, and database content using conditional and switch nodes.

**Nodes Involved:**  
- Define Type (Switch)  
- Switch (Switch)  
- New? (If)  
- Active? (If)  
- Answers (Switch)  
- Start? (If) [also in 1.1]  

**Node Details:**  

- **Define Type**  
  - Type: Switch  
  - Role: Routes the workflow based on the type of user input or command received  
  - Configuration: Conditions on message text or callback data types  
  - Inputs: From "Start?" false branch  
  - Outputs: Branches to "Switch" or "New?"  
  - Failure modes: Misrouted flows  

- **Switch**  
  - Type: Switch  
  - Role: Further routes based on specific user responses or commands  
  - Configuration: Multiple cases for different user actions (e.g., selecting items, requesting results)  
  - Inputs: From "Define Type"  
  - Outputs: Branches to "Answers", "Get Order", or other nodes  
  - Failure modes: Unhandled cases  

- **New?**  
  - Type: If  
  - Role: Checks if the current interaction is a new order or continuation  
  - Configuration: Condition on user session or order presence  
  - Inputs: From "Define Type"  
  - Outputs: True branch to "Add Order", False branch to other flows  
  - Failure modes: Incorrect session detection  

- **Active?**  
  - Type: If  
  - Role: Checks if the user choices or orders are active or valid  
  - Configuration: Condition on database flags or timestamps  
  - Inputs: From "Get Shop List Choices"  
  - Outputs: True branch to "Union Number with Text"  
  - Failure modes: Stale or invalid data  

- **Answers**  
  - Type: Switch  
  - Role: Routes based on user answers or selections in the multi-select process  
  - Configuration: Cases for success or further input required  
  - Inputs: From "Switch" node  
  - Outputs: Branches to "Success" or "Get Order"  
  - Failure modes: Unexpected user input  

---

#### 1.6 Finalization and Confirmation

**Overview:**  
This block completes the user interaction by confirming successful data saving and cleaning up messages.

**Nodes Involved:**  
- Success (Telegram)  
- Get message_id (Postgres) [multiple nodes]  
- Delete Prev SMS (Telegram) [multiple nodes]  
- Update message_id (Postgres) [multiple nodes]  

**Node Details:**  

- **Success**  
  - See 1.2 for details  

- **Get message_id** (multiple nodes)  
  - Type: Postgres  
  - Role: Retrieves Telegram message IDs for deletion or editing  
  - Configuration: SQL SELECT queries filtered by user and message context  
  - Inputs: After sending success or results messages  
  - Outputs: Passes to "Delete Prev SMS" or "Update message_id"  
  - Failure modes: Missing or outdated message IDs  

- **Delete Prev SMS** (multiple nodes)  
  - See 1.2 for details  

- **Update message_id** (multiple nodes)  
  - See 1.3 for details  

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                                | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                 |
|-------------------------------|-------------------------|-----------------------------------------------|----------------------------------|----------------------------------|-----------------------------------------------------------------------------|
| Telegram Trigger              | Telegram Trigger        | Entry point for Telegram updates               | -                                | Variables TG                     |                                                                             |
| Variables TG                 | Set                     | Initialize workflow variables                   | Telegram Trigger                 | Initialization                  |                                                                             |
| Initialization               | Set                     | Set/reset session variables                     | Variables TG                    | Start?                         |                                                                             |
| Start?                      | If                      | Check if user starts interaction                | Initialization                  | Welcome Message, Define Type    |                                                                             |
| Welcome Message              | Telegram                | Send welcome message                            | Start?                         | Upsert bot status on START      |                                                                             |
| Upsert bot status on START   | Postgres                | Insert/update bot status in DB                   | Welcome Message                | Define Type                    |                                                                             |
| Define Type                 | Switch                  | Route based on user input type                   | Start?                         | Switch, New?                   |                                                                             |
| Switch                      | Switch                  | Route based on user commands                      | Define Type                    | Answers, Get Order, Get Order   |                                                                             |
| New?                        | If                      | Check if new order                               | Define Type                    | Add Order                     |                                                                             |
| Add Order                   | Postgres                | Insert new order record                           | New?                           | Get Shop List                 |                                                                             |
| Get Shop List               | Postgres                | Retrieve shop list options                        | Add Order                     | Add Shop List Choice           |                                                                             |
| Add Shop List Choice        | Postgres                | Insert user choice                                | Get Shop List                 | Convert statuses               |                                                                             |
| Convert statuses            | Code                    | Convert DB statuses to readable format           | Add Shop List Choice           | Send                         |                                                                             |
| Send                       | Telegram                | Send message with options                         | Convert statuses              | Update message_id             |                                                                             |
| Update message_id           | Postgres                | Update Telegram message ID in DB                  | Send                         | -                            |                                                                             |
| Get message_id              | Postgres                | Retrieve message ID for deletion/editing          | Get Order                    | Update Shop List Choice       |                                                                             |
| Update Shop List Choice     | Postgres                | Update user choices                               | Get message_id                | Get Shop List Choice          |                                                                             |
| Get Shop List Choice        | Postgres                | Retrieve user choices                             | Update Shop List Choice       | Convert statuses and antistatuses |                                                                             |
| Convert statuses and antistatuses | Code              | Convert user choices and complementary statuses   | Get Shop List Choice          | Edit                         |                                                                             |
| Edit                       | Telegram                | Edit Telegram message with updated options        | Convert statuses and antistatuses | Update message_id           |                                                                             |
| Update message_id           | Postgres                | Update message ID after editing                    | Edit                         | -                            |                                                                             |
| Get Shop List Choices       | Postgres                | Retrieve multiple user choices                     | Get Order                    | Active?                      |                                                                             |
| Active?                    | If                      | Check if choices/orders are active                  | Get Shop List Choices         | Union Number with Text        |                                                                             |
| Union Number with Text      | Set                     | Combine numeric IDs with text                       | Active?                      | Union List                   |                                                                             |
| Union List                 | Summarize               | Aggregate selections into summary                    | Union Number with Text        | Get message_id               |                                                                             |
| Get message_id              | Postgres                | Retrieve message ID for deletion                     | Union List                   | Delete Prev SMS              |                                                                             |
| Delete Prev SMS             | Telegram                | Delete previous Telegram message                      | Get message_id               | Results                      |                                                                             |
| Results                    | Telegram                | Send final results message                           | Delete Prev SMS              | Update message_id            |                                                                             |
| Update message_id           | Postgres                | Update message ID after sending results               | Results                     | -                           |                                                                             |
| Answers                    | Switch                  | Route based on user answers                            | Switch                      | Success, Get Order           |                                                                             |
| Success                    | Telegram                | Send success confirmation message                      | Answers                     | Get message_id               |                                                                             |
| Get message_id              | Postgres                | Retrieve message ID for cleanup                         | Success                     | Delete Prev SMS              |                                                                             |
| Delete Prev SMS             | Telegram                | Delete previous message after success                   | Get message_id               | -                           |                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook with your Telegram bot token credentials.  
   - Set to listen for messages and callback queries.  

2. **Create Set Node "Variables TG"**  
   - Extract user ID, chat ID, message text, callback data from Telegram Trigger output.  
   - Store these as variables for downstream use.  

3. **Create Set Node "Initialization"**  
   - Initialize or reset session variables such as flags for new session, message IDs, etc.  

4. **Create If Node "Start?"**  
   - Condition: Check if message text equals "/start" or equivalent start command.  
   - True branch: Proceed to send welcome message.  
   - False branch: Proceed to define input type.  

5. **Create Telegram Node "Welcome Message"**  
   - Send a welcome message text to the user.  
   - Use Telegram credentials.  

6. **Create Postgres Node "Upsert bot status on START"**  
   - Configure SQL to insert or update bot status record for the user.  
   - Use user ID from variables.  

7. **Create Switch Node "Define Type"**  
   - Route based on message type or callback data.  
   - Branch to "Switch" or "New?" nodes.  

8. **Create Switch Node "Switch"**  
   - Define cases for different user commands or selections.  
   - Branch to "Answers", "Get Order", or other nodes accordingly.  

9. **Create If Node "New?"**  
   - Check if current session is a new order.  
   - True branch: Add new order.  
   - False branch: Continue with existing order.  

10. **Create Postgres Node "Add Order"**  
    - Insert new order record with user/session details.  

11. **Create Postgres Node "Get Shop List"**  
    - Select all items from `shop_list` table.  

12. **Create Postgres Node "Add Shop List Choice"**  
    - Insert user selection into choices table.  

13. **Create Code Node "Convert statuses"**  
    - Write JavaScript to map database status codes to readable text or buttons.  

14. **Create Telegram Node "Send"**  
    - Send message with inline keyboard showing selectable options.  

15. **Create Postgres Node "Update message_id"**  
    - Update DB with Telegram message ID for tracking.  

16. **Create Postgres Node "Get message_id"**  
    - Retrieve message ID for editing or deletion.  

17. **Create Postgres Node "Update Shop List Choice"**  
    - Update user choices with new selections.  

18. **Create Postgres Node "Get Shop List Choice"**  
    - Retrieve current user choices.  

19. **Create Code Node "Convert statuses and antistatuses"**  
    - Convert user choices and complementary statuses for display.  

20. **Create Telegram Node "Edit"**  
    - Edit existing Telegram message with updated options.  

21. **Create Postgres Node "Update message_id"** (for edit)  
    - Update message ID after editing.  

22. **Create Postgres Node "Get Shop List Choices"**  
    - Retrieve multiple user choices for active orders.  

23. **Create If Node "Active?"**  
    - Check if user choices/orders are active.  

24. **Create Set Node "Union Number with Text"**  
    - Combine numeric IDs with descriptive text for display.  

25. **Create Summarize Node "Union List"**  
    - Aggregate selections into a summary string.  

26. **Create Postgres Node "Get message_id"** (for deletion)  
    - Retrieve message ID for deleting previous messages.  

27. **Create Telegram Node "Delete Prev SMS"**  
    - Delete previous Telegram message to keep chat clean.  

28. **Create Telegram Node "Results"**  
    - Send final summary message to user.  

29. **Create Postgres Node "Update message_id"** (after results)  
    - Update message ID after sending results.  

30. **Create Switch Node "Answers"**  
    - Route based on user answers after selection.  

31. **Create Telegram Node "Success"**  
    - Send confirmation message after saving selections.  

32. **Create Postgres Node "Get message_id"** (for success cleanup)  
    - Retrieve message ID for deleting success message.  

33. **Create Telegram Node "Delete Prev SMS"** (after success)  
    - Delete success message to clean chat.  

34. **Connect all nodes according to the logical flow described in the connections section.**  

35. **Add and configure credentials:**  
    - Telegram Bot Token credential for all Telegram nodes.  
    - Postgres credential with access to the database containing `shop_list` and related tables.  

36. **Create required Postgres tables:**  
    - `shop_list` table with item options.  
    - Tables for user choices, orders, and message ID tracking as per SQL scripts provided in setup.  

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Replace `"n8n"` schema in SQL scripts with your actual database schema before running.                          | Setup instructions                                                                                              |
| Telegram Bot Token and Postgres credentials must be added in n8n credentials manager before running workflow.    | Setup instructions                                                                                              |
| The workflow automatically deletes previous Telegram messages to keep the chat clean and user-friendly.          | Workflow description                                                                                            |
| Modify the `shop_list` table to customize selectable options according to your use case.                         | Customization guidance                                                                                           |
| Useful for inventory selection, order systems, or preference tracking via Telegram.                              | Use case description                                                                                            |
| Tags associated: telegram, module, sell                                                                            | Workflow metadata                                                                                               |

---

This document provides a complete and detailed reference for understanding, reproducing, and modifying the Telegram multi-select bot workflow integrated with Postgres. It covers all nodes, logic blocks, and configuration essentials to ensure smooth operation and extensibility.