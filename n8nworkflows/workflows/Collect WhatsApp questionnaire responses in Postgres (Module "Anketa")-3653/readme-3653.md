Collect WhatsApp questionnaire responses in Postgres (Module "Anketa")

https://n8nworkflows.xyz/workflows/collect-whatsapp-questionnaire-responses-in-postgres--module--anketa---3653


# Collect WhatsApp questionnaire responses in Postgres (Module "Anketa")

### 1. Workflow Overview

This workflow is designed to collect questionnaire responses via a WhatsApp bot and store them in a Postgres database. It targets businesses or organizations that want to automate the collection and management of user responses from WhatsApp interactions, centralizing data for analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming WhatsApp messages from users.
- **1.2 Initialization & Bot Status Management:** Checks and updates the bot’s session status in the database.
- **1.3 Command Handling:** Interprets user commands and directs flow accordingly.
- **1.4 Questionnaire Flow Control:** Retrieves questions, manages user answers, and navigates through the questionnaire.
- **1.5 Database Operations:** Reads and writes questionnaire questions, user answers, and bot status in Postgres.
- **1.6 WhatsApp Messaging:** Sends messages to users, including questions, menus, and completion notices.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming WhatsApp messages from users and triggers the workflow.

**Nodes Involved:**  
- WhatsApp Trigger

**Node Details:**  
- **WhatsApp Trigger**  
  - Type: Trigger node for WhatsApp messages.  
  - Configuration: Uses a webhook ID to receive WhatsApp messages. No parameters configured beyond default.  
  - Inputs: External WhatsApp message events.  
  - Outputs: Passes incoming message data to the Initialization block.  
  - Edge Cases: Webhook misconfiguration, message format errors, or WhatsApp API downtime.

---

#### 1.2 Initialization & Bot Status Management

**Overview:**  
Initializes workflow variables and retrieves the current bot status from the database to determine the user’s progress in the questionnaire.

**Nodes Involved:**  
- Initialization  
- Get Bot Status  
- Upsert Bot Status on START  
- Update Bot Status on START  
- Update Bot Status on ANKETA

**Node Details:**  
- **Initialization**  
  - Type: Set node to initialize variables or context.  
  - Configuration: No explicit parameters; likely sets default values or prepares data structure.  
  - Inputs: From WhatsApp Trigger.  
  - Outputs: To Get Bot Status.  
  - Edge Cases: Expression errors if variables expected are missing.

- **Get Bot Status**  
  - Type: Postgres node (read).  
  - Configuration: Queries the bot status table to retrieve current user session state.  
  - Inputs: From Initialization.  
  - Outputs: To Define Flow node.  
  - Edge Cases: Database connection errors, empty results if user not found.

- **Upsert Bot Status on START**  
  - Type: Postgres node (write).  
  - Configuration: Inserts or updates bot status when a new session starts.  
  - Inputs: From Starts WhatsApp node.  
  - Outputs: To next nodes managing flow.  
  - Edge Cases: DB write conflicts, transaction failures.

- **Update Bot Status on START**  
  - Type: Postgres node (write).  
  - Configuration: Updates bot status at the start of questionnaire or session.  
  - Inputs: From Finish Anketa node.  
  - Outputs: None (end of flow for that path).  
  - Edge Cases: DB write errors.

- **Update Bot Status on ANKETA**  
  - Type: Postgres node (write).  
  - Configuration: Updates bot status during questionnaire progress.  
  - Inputs: From First Question node.  
  - Outputs: To Add prev Answer node.  
  - Edge Cases: DB write errors.

---

#### 1.3 Command Handling

**Overview:**  
This block interprets user commands from WhatsApp messages and routes the workflow to appropriate actions such as starting the questionnaire or showing menus.

**Nodes Involved:**  
- Define Flow (Switch)  
- Commands (Switch)  
- Main Menu (WhatsApp)  
- Starts (WhatsApp)  
- Starts Test (WhatsApp)  

**Node Details:**  
- **Define Flow**  
  - Type: Switch node.  
  - Configuration: Routes flow based on message content or bot status.  
  - Inputs: From Get Bot Status.  
  - Outputs: To Starts, Commands, or Get prev Answer nodes.  
  - Edge Cases: Unmatched cases leading to no output.

- **Commands**  
  - Type: Switch node.  
  - Configuration: Further refines command routing (e.g., menu selection).  
  - Inputs: From Define Flow.  
  - Outputs: To Main Menu or Starts Test nodes.  
  - Edge Cases: Unknown commands.

- **Main Menu**  
  - Type: WhatsApp message node.  
  - Configuration: Sends main menu options to user.  
  - Inputs: From Commands.  
  - Outputs: To next flow nodes as per user selection.  
  - Edge Cases: WhatsApp API errors.

- **Starts**  
  - Type: WhatsApp message node.  
  - Configuration: Sends start message or instructions.  
  - Inputs: From Define Flow.  
  - Outputs: To Upsert Bot Status on START.  
  - Edge Cases: Messaging failures.

- **Starts Test**  
  - Type: WhatsApp message node.  
  - Configuration: Sends test start message.  
  - Inputs: From Commands.  
  - Outputs: To Get First Question.  
  - Edge Cases: Messaging failures.

---

#### 1.4 Questionnaire Flow Control

**Overview:**  
Manages the retrieval of questions, user answers, and navigation through the questionnaire until completion.

**Nodes Involved:**  
- Get First Question (Postgres)  
- First Question (WhatsApp)  
- Get prev Answer (Postgres)  
- Is Question found? (If)  
- Get Available Questions (Postgres)  
- Is Questions available? (If)  
- Question (WhatsApp)  
- Add Answer (Postgres)  
- Add prev Answer (Postgres)  
- Add prev Answer (Postgres) [duplicate node name, different instance]  
- Finish Anketa (WhatsApp)

**Node Details:**  
- **Get First Question**  
  - Type: Postgres node (read).  
  - Configuration: Retrieves the first question from the questionnaire table.  
  - Inputs: From Starts Test.  
  - Outputs: To First Question.  
  - Edge Cases: No questions found, DB errors.

- **First Question**  
  - Type: WhatsApp message node.  
  - Configuration: Sends the first question to the user.  
  - Inputs: From Get First Question.  
  - Outputs: To Update Bot Status on ANKETA.  
  - Edge Cases: Messaging errors.

- **Get prev Answer**  
  - Type: Postgres node (read).  
  - Configuration: Retrieves the previous answer from the database to determine next steps.  
  - Inputs: From Define Flow.  
  - Outputs: To Is Question found? node.  
  - Edge Cases: No previous answer found.

- **Is Question found?**  
  - Type: If node.  
  - Configuration: Checks if a valid question exists for the user’s current state.  
  - Inputs: From Get prev Answer.  
  - Outputs: If true, to Get Available Questions; if false, to Add Answer.  
  - Edge Cases: Expression evaluation errors.

- **Get Available Questions**  
  - Type: Postgres node (read).  
  - Configuration: Retrieves the next set of available questions.  
  - Inputs: From Is Question found? (true branch).  
  - Outputs: To Is Questions available?  
  - Edge Cases: No questions available.

- **Is Questions available?**  
  - Type: If node.  
  - Configuration: Checks if there are any questions left to ask.  
  - Inputs: From Get Available Questions.  
  - Outputs: If true, to Question node; if false, to Finish Anketa.  
  - Edge Cases: Empty dataset.

- **Question**  
  - Type: WhatsApp message node.  
  - Configuration: Sends the current question to the user.  
  - Inputs: From Is Questions available? (true branch).  
  - Outputs: To Add prev Answer node.  
  - Edge Cases: Messaging failures.

- **Add Answer**  
  - Type: Postgres node (write).  
  - Configuration: Inserts the user’s answer into the database.  
  - Inputs: From Is Question found? (false branch).  
  - Outputs: To Get Available Questions.  
  - Edge Cases: DB write errors.

- **Add prev Answer** (two instances)  
  - Type: Postgres node (write).  
  - Configuration: Updates or inserts the previous answer to maintain state.  
  - Inputs: From Question and Update Bot Status on ANKETA.  
  - Outputs: To next flow nodes (e.g., Get Available Questions or others).  
  - Edge Cases: DB write errors.

- **Finish Anketa**  
  - Type: WhatsApp message node.  
  - Configuration: Sends a completion message to the user after questionnaire ends.  
  - Inputs: From Is Questions available? (false branch).  
  - Outputs: To Update Bot Status on START.  
  - Edge Cases: Messaging failures.

---

#### 1.5 Database Operations

**Overview:**  
Handles all interactions with the Postgres database, including reading questions, saving answers, and managing bot session states.

**Nodes Involved:**  
- Get Bot Status  
- Upsert Bot Status on START  
- Update Bot Status on START  
- Update Bot Status on ANKETA  
- Get First Question  
- Get prev Answer  
- Is Question found? (logic depends on DB output)  
- Get Available Questions  
- Is Questions available? (logic depends on DB output)  
- Add Answer  
- Add prev Answer (two nodes)

**Node Details:**  
- All Postgres nodes use SQL queries to interact with tables: `anketa_question`, user answers, and bot status tables.  
- Configured with Postgres credentials.  
- Inputs and outputs are chained to maintain state and flow.  
- Edge Cases: Connection failures, query errors, empty results, transaction conflicts.

---

#### 1.6 WhatsApp Messaging

**Overview:**  
Sends messages to users via WhatsApp, including start prompts, questions, menus, and completion notifications.

**Nodes Involved:**  
- Starts  
- Starts Test  
- Main Menu  
- First Question  
- Question  
- Finish Anketa

**Node Details:**  
- All are WhatsApp message nodes configured with webhook IDs.  
- Messages are dynamically constructed based on database queries and user interaction state.  
- Inputs come from flow control nodes.  
- Outputs proceed to database update nodes or next message nodes.  
- Edge Cases: WhatsApp API errors, message formatting issues.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                         | Input Node(s)                 | Output Node(s)                        | Sticky Note                         |
|---------------------------|-------------------------|---------------------------------------|------------------------------|-------------------------------------|-----------------------------------|
| WhatsApp Trigger          | WhatsApp Trigger        | Entry point for incoming WhatsApp messages | None                         | Initialization                      |                                   |
| Initialization           | Set                     | Initialize variables/context           | WhatsApp Trigger             | Get Bot Status                      |                                   |
| Get Bot Status           | Postgres                | Retrieve current bot session status    | Initialization              | Define Flow                        |                                   |
| Define Flow              | Switch                  | Route flow based on bot status/message | Get Bot Status              | Starts, Commands, Get prev Answer  |                                   |
| Starts                   | WhatsApp                | Send start message                      | Define Flow                 | Upsert Bot Status on START         |                                   |
| Upsert Bot Status on START| Postgres                | Insert/update bot status at session start | Starts                     |                                   |                                   |
| Commands                 | Switch                  | Interpret user commands                 | Define Flow                 | Main Menu, Starts Test             |                                   |
| Main Menu                | WhatsApp                | Send main menu options                  | Commands                   |                                   |                                   |
| Starts Test              | WhatsApp                | Send test start message                 | Commands                   | Get First Question                 |                                   |
| Get First Question       | Postgres                | Retrieve first questionnaire question  | Starts Test                | First Question                    |                                   |
| First Question           | WhatsApp                | Send first question to user             | Get First Question          | Update Bot Status on ANKETA        |                                   |
| Update Bot Status on ANKETA | Postgres             | Update bot status during questionnaire  | First Question             | Add prev Answer                   |                                   |
| Add prev Answer          | Postgres                | Save/update previous answer             | Update Bot Status on ANKETA |                                   |                                   |
| Get prev Answer          | Postgres                | Retrieve previous answer for user       | Define Flow                 | Is Question found?                |                                   |
| Is Question found?       | If                      | Check if question exists for user       | Get prev Answer             | Get Available Questions (true), Add Answer (false) |                                   |
| Get Available Questions  | Postgres                | Retrieve next available questions       | Is Question found? (true)   | Is Questions available?           |                                   |
| Is Questions available?  | If                      | Check if questions remain                | Get Available Questions     | Question (true), Finish Anketa (false) |                                   |
| Question                 | WhatsApp                | Send current question to user            | Is Questions available? (true) | Add prev Answer                  |                                   |
| Add prev Answer          | Postgres                | Save/update previous answer             | Question                   |                                   |                                   |
| Add Answer               | Postgres                | Save user answer                         | Is Question found? (false)  | Get Available Questions           |                                   |
| Finish Anketa            | WhatsApp                | Send completion message                  | Is Questions available? (false) | Update Bot Status on START       |                                   |
| Update Bot Status on START | Postgres              | Update bot status at questionnaire end  | Finish Anketa              |                                   |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure webhook ID for WhatsApp incoming messages.

2. **Create Initialization Node**  
   - Type: Set  
   - Purpose: Initialize variables or context for the session.

3. **Create Get Bot Status Node**  
   - Type: Postgres  
   - Configure with Postgres credentials.  
   - SQL: Query bot status table for current user session.  
   - Connect Initialization → Get Bot Status.

4. **Create Define Flow Node**  
   - Type: Switch  
   - Configure to route based on bot status or message content.  
   - Connect Get Bot Status → Define Flow.

5. **Create Starts Node**  
   - Type: WhatsApp  
   - Configure webhook ID for sending start message.  
   - Connect Define Flow (start branch) → Starts.

6. **Create Upsert Bot Status on START Node**  
   - Type: Postgres  
   - Configure to insert/update bot status when session starts.  
   - Connect Starts → Upsert Bot Status on START.

7. **Create Commands Node**  
   - Type: Switch  
   - Configure to interpret user commands (e.g., menu, test start).  
   - Connect Define Flow (commands branch) → Commands.

8. **Create Main Menu Node**  
   - Type: WhatsApp  
   - Configure webhook ID to send main menu options.  
   - Connect Commands (menu branch) → Main Menu.

9. **Create Starts Test Node**  
   - Type: WhatsApp  
   - Configure webhook ID to send test start message.  
   - Connect Commands (test branch) → Starts Test.

10. **Create Get First Question Node**  
    - Type: Postgres  
    - Configure SQL to fetch first question from `anketa_question` table.  
    - Connect Starts Test → Get First Question.

11. **Create First Question Node**  
    - Type: WhatsApp  
    - Configure webhook ID to send first question message.  
    - Connect Get First Question → First Question.

12. **Create Update Bot Status on ANKETA Node**  
    - Type: Postgres  
    - Configure to update bot status during questionnaire progress.  
    - Connect First Question → Update Bot Status on ANKETA.

13. **Create Add prev Answer Node**  
    - Type: Postgres  
    - Configure to insert/update previous answer record.  
    - Connect Update Bot Status on ANKETA → Add prev Answer.

14. **Create Get prev Answer Node**  
    - Type: Postgres  
    - Configure SQL to fetch previous answer for current user.  
    - Connect Define Flow (prev answer branch) → Get prev Answer.

15. **Create Is Question found? Node**  
    - Type: If  
    - Configure condition to check if question exists based on previous answer.  
    - Connect Get prev Answer → Is Question found?.

16. **Create Get Available Questions Node**  
    - Type: Postgres  
    - Configure SQL to fetch next available questions.  
    - Connect Is Question found? (true branch) → Get Available Questions.

17. **Create Is Questions available? Node**  
    - Type: If  
    - Configure condition to check if questions remain.  
    - Connect Get Available Questions → Is Questions available?.

18. **Create Question Node**  
    - Type: WhatsApp  
    - Configure webhook ID to send current question.  
    - Connect Is Questions available? (true branch) → Question.

19. **Create Add prev Answer Node** (second instance)  
    - Type: Postgres  
    - Configure to save/update previous answer after sending question.  
    - Connect Question → Add prev Answer.

20. **Create Add Answer Node**  
    - Type: Postgres  
    - Configure to save user’s answer to database.  
    - Connect Is Question found? (false branch) → Add Answer.

21. **Create Finish Anketa Node**  
    - Type: WhatsApp  
    - Configure webhook ID to send questionnaire completion message.  
    - Connect Is Questions available? (false branch) → Finish Anketa.

22. **Create Update Bot Status on START Node**  
    - Type: Postgres  
    - Configure to update bot status at questionnaire end.  
    - Connect Finish Anketa → Update Bot Status on START.

23. **Connect WhatsApp Trigger → Initialization** to start the flow.

24. **Set up Credentials:**  
    - WhatsApp OAuth2 credentials for WhatsApp nodes.  
    - Postgres credentials for all Postgres nodes.

25. **Create required tables in Postgres:**  
    - Run provided SQL script to create `anketa_question` and related tables.  
    - Replace schema name as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Replace "n8n" in the SQL script with your database schema name before running in Postgres.      | Setup instructions in workflow description                                                        |
| Workflow tags include "module", "whatsapp", and "sell" indicating modular design and use case. | Workflow metadata                                                                                  |
| Ensure WhatsApp OAuth2 credentials and Postgres credentials are correctly configured in n8n.   | Credential setup                                                                                   |
| Customize `anketa_question` table to modify survey questions and data schema.                   | Customization guidance                                                                             |
| Workflow uses multiple WhatsApp webhook nodes for sending messages; ensure unique webhook IDs. | WhatsApp messaging nodes configuration                                                            |
| The workflow is versioned and inactive by default; activate after configuration.                | Workflow metadata                                                                                  |

---

This document fully describes the workflow structure, node configurations, and logic flow to enable reproduction, modification, and troubleshooting.