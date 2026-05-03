Automated Customer Reservations via Telegram and PostgreSQL (Module "Booking")

https://n8nworkflows.xyz/workflows/automated-customer-reservations-via-telegram-and-postgresql--module--booking---3651


# Automated Customer Reservations via Telegram and PostgreSQL (Module "Booking")

### 1. Workflow Overview

This workflow automates customer reservation management via a WhatsApp bot integrated with a PostgreSQL database. It targets businesses that need to handle appointment bookings efficiently by capturing customer inputs through WhatsApp messages and storing reservation details in a database. The workflow ensures seamless interaction with customers, validates inputs, updates booking statuses, and confirms successful reservations.

The workflow logic is organized into the following functional blocks:

- **1.1 Input Reception and Initialization**  
  Captures incoming WhatsApp messages, initializes workflow variables, and retrieves the current bot status.

- **1.2 Command and Flow Definition**  
  Routes user commands and defines the conversational flow, including menu display and booking initiation.

- **1.3 Booking Date and Time Selection**  
  Handles user input for selecting reservation dates and times, validating these inputs against working days and hours stored in the database.

- **1.4 Customer Information Collection**  
  Requests and validates customer phone number and name, ensuring data integrity before booking confirmation.

- **1.5 Database Operations and Status Updates**  
  Performs PostgreSQL operations to insert or update booking records and bot status, reflecting the current state of the reservation process.

- **1.6 Confirmation and Payment Handling**  
  Confirms successful bookings to customers and manages payment-related status updates.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block listens for incoming WhatsApp messages, initializes necessary variables, and retrieves the current bot status from the database to determine the next steps.

- **Nodes Involved:**  
  - WhatsApp Trigger  
  - Initialization  
  - Get Bot Status

- **Node Details:**

  - **WhatsApp Trigger**  
    - Type: WhatsApp Trigger  
    - Role: Entry point for incoming WhatsApp messages via webhook.  
    - Configuration: Uses a webhook ID to receive messages.  
    - Inputs: External WhatsApp messages.  
    - Outputs: Passes message data to Initialization node.  
    - Edge Cases: Webhook downtime, message format errors.

  - **Initialization**  
    - Type: Set  
    - Role: Initializes workflow variables and context for processing.  
    - Configuration: Sets default values or resets state variables.  
    - Inputs: Data from WhatsApp Trigger.  
    - Outputs: Passes initialized data to Get Bot Status.  
    - Edge Cases: Missing or malformed input data.

  - **Get Bot Status**  
    - Type: PostgreSQL  
    - Role: Queries the database to retrieve the current status of the bot session for the user.  
    - Configuration: Executes a SELECT query on the bot status table.  
    - Inputs: Initialized data.  
    - Outputs: Provides bot status to Define Flow node.  
    - Edge Cases: Database connection errors, empty or inconsistent status data.

#### 2.2 Command and Flow Definition

- **Overview:**  
  This block interprets user commands and defines the conversational flow, directing users to menus, booking processes, or payment options.

- **Nodes Involved:**  
  - Define Flow  
  - Commands  
  - Main Menu  
  - Payments  
  - Upsert Bot Status on START  
  - Update bot status on START

- **Node Details:**

  - **Define Flow**  
    - Type: Switch  
    - Role: Routes the workflow based on bot status or user input to appropriate sub-flows (start, commands, booking, payments).  
    - Configuration: Switches on bot status or command keywords.  
    - Inputs: Bot status data.  
    - Outputs: Routes to Starts, Commands, Get book, Payments, or Edit Fields nodes.  
    - Edge Cases: Unrecognized commands, missing status.

  - **Commands**  
    - Type: Switch  
    - Role: Differentiates between user commands such as menu display or booking initiation.  
    - Configuration: Switch on command text.  
    - Inputs: User message data.  
    - Outputs: Routes to Main Menu or Get Work Days nodes.  
    - Edge Cases: Invalid commands.

  - **Main Menu**  
    - Type: WhatsApp  
    - Role: Sends the main menu options to the user.  
    - Configuration: Sends predefined menu text via WhatsApp webhook.  
    - Inputs: Routed from Commands node.  
    - Outputs: Updates bot status on START.  
    - Edge Cases: Message delivery failure.

  - **Payments**  
    - Type: WhatsApp  
    - Role: Handles payment-related interactions with the user.  
    - Configuration: Sends payment instructions or status.  
    - Inputs: Routed from Define Flow.  
    - Outputs: Updates bot status on START.  
    - Edge Cases: Payment status inconsistencies.

  - **Upsert Bot Status on START**  
    - Type: PostgreSQL  
    - Role: Inserts or updates the bot status record when the session starts.  
    - Configuration: Executes UPSERT query on bot status table.  
    - Inputs: Data from Starts node.  
    - Outputs: None (executeOnce).  
    - Edge Cases: Database write errors.

  - **Update bot status on START**  
    - Type: PostgreSQL  
    - Role: Updates bot status to reflect the start of a session or menu display.  
    - Configuration: Executes UPDATE query.  
    - Inputs: From Main Menu or Payments nodes.  
    - Outputs: None (executeOnce).  
    - Edge Cases: Database write errors.

#### 2.3 Booking Date and Time Selection

- **Overview:**  
  This block manages the selection and validation of booking dates and times, ensuring user inputs correspond to available working days and hours.

- **Nodes Involved:**  
  - Get Work Days  
  - Is Date correct?  
  - Union Number with Question  
  - Union list  
  - Request Date  
  - Update bot status on BOOKING  
  - Get Work Days (duplicate node with trailing space)  
  - Add Book  
  - Merge1  
  - Update Date AND Status  
  - Get Work Hours  
  - Union Number with Question1  
  - Union list1  
  - Request Time

- **Node Details:**

  - **Get Work Days**  
    - Type: PostgreSQL  
    - Role: Retrieves available working days from the database.  
    - Configuration: SELECT query on work days table.  
    - Inputs: Triggered by Commands or First Question? nodes.  
    - Outputs: Provides days data to Is Date correct? or Merge1.  
    - Edge Cases: Empty result sets, DB errors.

  - **Is Date correct?**  
    - Type: If  
    - Role: Validates if the user-selected date is within available work days.  
    - Configuration: Conditional check comparing input date to DB data.  
    - Inputs: User input and work days data.  
    - Outputs: Routes to Union Number with Question on success.  
    - Edge Cases: Invalid date formats, out-of-range dates.

  - **Union Number with Question**  
    - Type: Set  
    - Role: Combines user input (date) with a question prompt for confirmation or next step.  
    - Configuration: Sets variables merging date and question text.  
    - Inputs: From Is Date correct?  
    - Outputs: Passes to Union list.  
    - Edge Cases: Missing input data.

  - **Union list**  
    - Type: Summarize  
    - Role: Aggregates or formats the combined data for sending to the user.  
    - Configuration: Summarizes question and date info.  
    - Inputs: From Union Number with Question.  
    - Outputs: Routes to Request Date node.  
    - Edge Cases: Data formatting errors.

  - **Request Date**  
    - Type: WhatsApp  
    - Role: Sends a message to the user requesting confirmation or selection of the booking date.  
    - Configuration: Uses webhook to send message.  
    - Inputs: From Union list.  
    - Outputs: Updates bot status on BOOKING.  
    - Edge Cases: Message delivery failure.

  - **Update bot status on BOOKING**  
    - Type: PostgreSQL  
    - Role: Updates the bot status to reflect that the booking process is ongoing.  
    - Configuration: UPDATE query on bot status table.  
    - Inputs: From Request Date.  
    - Outputs: None (executeOnce).  
    - Edge Cases: DB write errors.

  - **Get Work Days (duplicate)**  
    - Same as above, used in a different branch to merge data.

  - **Add Book**  
    - Type: PostgreSQL  
    - Role: Inserts a new booking record into the database.  
    - Configuration: INSERT query into booking table.  
    - Inputs: From First Question? node.  
    - Outputs: Routes to Merge1.  
    - Edge Cases: Duplicate bookings, DB errors.

  - **Merge1**  
    - Type: Merge  
    - Role: Combines multiple data streams for further processing.  
    - Configuration: Merges data from Add Book, Get Work Days, and other nodes.  
    - Inputs: Multiple.  
    - Outputs: Routes to Update Date AND Status.  
    - Edge Cases: Data mismatch.

  - **Update Date AND Status**  
    - Type: PostgreSQL  
    - Role: Updates booking date and status in the database.  
    - Configuration: UPDATE query on booking table.  
    - Inputs: From Merge1.  
    - Outputs: Routes to Get Work Hours.  
    - Edge Cases: DB write errors.

  - **Get Work Hours**  
    - Type: PostgreSQL  
    - Role: Retrieves available working hours for the selected date.  
    - Configuration: SELECT query on work hours table.  
    - Inputs: From Update Date AND Status.  
    - Outputs: Routes to Union Number with Question1.  
    - Edge Cases: Empty results.

  - **Union Number with Question1**  
    - Type: Set  
    - Role: Combines available hours with a question prompt for user selection.  
    - Configuration: Sets variables.  
    - Inputs: From Get Work Hours.  
    - Outputs: Routes to Union list1.  
    - Edge Cases: Missing data.

  - **Union list1**  
    - Type: Summarize  
    - Role: Formats the question and available hours for user display.  
    - Configuration: Summarizes data.  
    - Inputs: From Union Number with Question1.  
    - Outputs: Routes to Request Time.  
    - Edge Cases: Formatting errors.

  - **Request Time**  
    - Type: WhatsApp  
    - Role: Sends a message requesting the user to select a booking time.  
    - Configuration: WhatsApp webhook message.  
    - Inputs: From Union list1.  
    - Outputs: Continues booking flow.  
    - Edge Cases: Message delivery failure.

#### 2.4 Customer Information Collection

- **Overview:**  
  This block collects and validates customer phone number and name, ensuring the booking record is complete and accurate.

- **Nodes Involved:**  
  - Phone validation1  
  - Request Phone  
  - Update Phone  
  - Request Name  
  - Update Name  
  - Phone Error  
  - page booking success  
  - Update status on BOOKED

- **Node Details:**

  - **Phone validation1**  
    - Type: If  
    - Role: Validates the format and correctness of the phone number provided by the user.  
    - Configuration: Conditional checks on phone number format.  
    - Inputs: User input from Request Phone.  
    - Outputs: Routes to Update Phone on success or Phone Error on failure.  
    - Edge Cases: Invalid phone formats, missing input.

  - **Request Phone**  
    - Type: WhatsApp  
    - Role: Requests the user to provide their phone number.  
    - Configuration: Sends prompt message via WhatsApp webhook.  
    - Inputs: From Update Hour node.  
    - Outputs: Passes user response to Phone validation1.  
    - Edge Cases: Message delivery failure.

  - **Update Phone**  
    - Type: PostgreSQL  
    - Role: Updates the booking record with the validated phone number.  
    - Configuration: UPDATE query on booking table.  
    - Inputs: From Phone validation1 success path.  
    - Outputs: Routes to Request Name.  
    - Edge Cases: DB write errors.

  - **Request Name**  
    - Type: WhatsApp  
    - Role: Requests the user to provide their name.  
    - Configuration: Sends prompt message.  
    - Inputs: From Update Phone.  
    - Outputs: Passes user response to Update Name.  
    - Edge Cases: Message delivery failure.

  - **Update Name**  
    - Type: PostgreSQL  
    - Role: Updates the booking record with the customer's name.  
    - Configuration: UPDATE query on booking table.  
    - Inputs: From Request Name.  
    - Outputs: Routes to page booking success.  
    - Edge Cases: DB write errors.

  - **Phone Error**  
    - Type: WhatsApp  
    - Role: Sends an error message to the user if the phone number validation fails.  
    - Configuration: Sends error prompt.  
    - Inputs: From Phone validation1 failure path.  
    - Outputs: None (ends or loops).  
    - Edge Cases: Message delivery failure.

  - **page booking success**  
    - Type: WhatsApp  
    - Role: Sends a confirmation message to the user indicating successful booking.  
    - Configuration: Sends success message.  
    - Inputs: From Update Name.  
    - Outputs: Routes to Update status on BOOKED.  
    - Edge Cases: Message delivery failure.

  - **Update status on BOOKED**  
    - Type: PostgreSQL  
    - Role: Updates the booking status to "BOOKED" in the database.  
    - Configuration: UPDATE query on booking table.  
    - Inputs: From page booking success.  
    - Outputs: Routes to Update bot status on PAYMENTS.  
    - Edge Cases: DB write errors.

#### 2.5 Database Operations and Status Updates

- **Overview:**  
  This block handles all database interactions related to booking creation, updates, and bot status management to maintain accurate session and booking states.

- **Nodes Involved:**  
  - Add Book  
  - Update Date AND Status  
  - Update Hour  
  - Update Phone  
  - Update Name  
  - Update status on BOOKED  
  - Update bot status on START  
  - Update bot status on BOOKING  
  - Update bot status on PAYMENTS  
  - Upsert Bot Status on START  
  - Get Bot Status  
  - Get Work Days  
  - Get Work Hours

- **Node Details:**  
  Each PostgreSQL node executes specific SQL queries to insert or update records in the respective tables (booking, bot status, work days/hours). They are configured with credentials for the PostgreSQL database and are set to execute once or per message as appropriate. Potential failure points include database connectivity issues, query syntax errors, and data integrity conflicts.

#### 2.6 Confirmation and Payment Handling

- **Overview:**  
  This block confirms bookings to customers and manages payment-related interactions and status updates.

- **Nodes Involved:**  
  - Payments  
  - Update bot status on PAYMENTS

- **Node Details:**

  - **Payments**  
    - Type: WhatsApp  
    - Role: Sends payment instructions or status updates to the user.  
    - Configuration: Uses WhatsApp webhook to communicate.  
    - Inputs: Routed from Update bot status on PAYMENTS.  
    - Outputs: Updates bot status on START.  
    - Edge Cases: Message delivery failure.

  - **Update bot status on PAYMENTS**  
    - Type: PostgreSQL  
    - Role: Updates the bot status to reflect payment processing or completion.  
    - Configuration: UPDATE query on bot status table.  
    - Inputs: From Update status on BOOKED.  
    - Outputs: Routes to Payments node.  
    - Edge Cases: DB write errors.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                          | Input Node(s)                      | Output Node(s)                      | Sticky Note |
|------------------------------|---------------------|----------------------------------------|----------------------------------|-----------------------------------|-------------|
| WhatsApp Trigger             | WhatsApp Trigger    | Entry point for WhatsApp messages      | -                                | Initialization                    |             |
| Initialization              | Set                 | Initialize variables                    | WhatsApp Trigger                 | Get Bot Status                   |             |
| Get Bot Status              | PostgreSQL          | Retrieve current bot status             | Initialization                  | Define Flow                     |             |
| Define Flow                 | Switch              | Route flow based on status/commands    | Get Bot Status                  | Starts, Commands, Get book, Payments, Edit Fields |             |
| Starts                      | WhatsApp            | Start session interaction               | Define Flow                    | Upsert Bot Status on START       |             |
| Upsert Bot Status on START  | PostgreSQL          | Insert/update bot status on start       | Starts                        | -                               |             |
| Commands                    | Switch              | Interpret user commands                  | Define Flow                    | Main Menu, Get Work Days         |             |
| Main Menu                   | WhatsApp            | Send main menu options                   | Commands                      | Update bot status on START       |             |
| Update bot status on START  | PostgreSQL          | Update bot status on session start      | Main Menu, Payments            | -                               |             |
| Payments                   | WhatsApp            | Handle payment interactions              | Update bot status on PAYMENTS  | Update bot status on START       |             |
| Update bot status on PAYMENTS| PostgreSQL          | Update bot status for payments           | Update status on BOOKED        | Payments                       |             |
| Get Work Days               | PostgreSQL          | Retrieve available working days          | Commands                      | Is Date correct?                 |             |
| Is Date correct?            | If                  | Validate selected booking date            | Get Work Days                 | Union Number with Question       |             |
| Union Number with Question  | Set                 | Combine date with question prompt         | Is Date correct?              | Union list                      |             |
| Union list                  | Summarize           | Format question and date for user         | Union Number with Question    | Request Date                   |             |
| Request Date                | WhatsApp            | Request booking date confirmation         | Union list                    | Update bot status on BOOKING     |             |
| Update bot status on BOOKING| PostgreSQL          | Update bot status during booking          | Request Date                  | -                               |             |
| Add Book                   | PostgreSQL          | Insert new booking record                  | First Question?               | Merge1                         |             |
| Merge1                     | Merge               | Combine booking and work day data          | Add Book, Get Work Days        | Update Date AND Status           |             |
| Update Date AND Status      | PostgreSQL          | Update booking date and status             | Merge1                       | Get Work Hours                 |             |
| Get Work Hours             | PostgreSQL          | Retrieve available booking hours            | Update Date AND Status        | Union Number with Question1      |             |
| Union Number with Question1 | Set                 | Combine hours with question prompt          | Get Work Hours               | Union list1                    |             |
| Union list1                | Summarize           | Format question and hours for user           | Union Number with Question1   | Request Time                   |             |
| Request Time               | WhatsApp            | Request booking time selection               | Union list1                  | -                               |             |
| Phone validation1          | If                  | Validate phone number format                  | Request Phone                | Update Phone, Phone Error        |             |
| Request Phone              | WhatsApp            | Request customer phone number                  | Update Hour                 | Phone validation1               |             |
| Update Phone               | PostgreSQL          | Update booking with phone number               | Phone validation1 (success)  | Request Name                   |             |
| Phone Error                | WhatsApp            | Notify user of invalid phone number             | Phone validation1 (fail)     | -                               |             |
| Request Name               | WhatsApp            | Request customer name                          | Update Phone                | Update Name                   |             |
| Update Name                | PostgreSQL          | Update booking with customer name                | Request Name                | page booking success           |             |
| page booking success       | WhatsApp            | Confirm successful booking                       | Update Name                 | Update status on BOOKED         |             |
| Update status on BOOKED    | PostgreSQL          | Update booking status to "BOOKED"                 | page booking success        | Update bot status on PAYMENTS   |             |
| Update Hour                | PostgreSQL          | Update booking hour                               | Switch                      | Request Phone                  |             |
| Get book                  | PostgreSQL          | Retrieve booking data                             | Define Flow                 | Merge                         |             |
| Merge                      | Merge               | Combine booking data streams                       | Get book, Edit Fields       | First Question?                |             |
| Edit Fields                | Set                 | Prepare booking data for merging                    | Define Flow                 | Merge                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create PostgreSQL Tables**  
   - Create tables for bookings, bot status, work days, and work hours as per your schema. Replace `"n8n"` with your schema name.

2. **Add Credentials**  
   - Add Telegram/WhatsApp bot credentials with webhook access.  
   - Add PostgreSQL database credentials.

3. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure webhook ID to receive messages.

4. **Create Initialization Node**  
   - Type: Set  
   - Initialize variables for session and message context.

5. **Create Get Bot Status Node**  
   - Type: PostgreSQL  
   - Query bot status table for current user session.

6. **Create Define Flow Node**  
   - Type: Switch  
   - Route based on bot status or command input to Starts, Commands, Get book, Payments, or Edit Fields.

7. **Create Starts Node**  
   - Type: WhatsApp  
   - Send welcome/start message.

8. **Create Upsert Bot Status on START Node**  
   - Type: PostgreSQL  
   - Insert or update bot status when session starts.

9. **Create Commands Node**  
   - Type: Switch  
   - Route commands to Main Menu or Get Work Days.

10. **Create Main Menu Node**  
    - Type: WhatsApp  
    - Send main menu options.

11. **Create Update bot status on START Node**  
    - Type: PostgreSQL  
    - Update bot status for session start/menu display.

12. **Create Payments Node**  
    - Type: WhatsApp  
    - Handle payment interactions.

13. **Create Update bot status on PAYMENTS Node**  
    - Type: PostgreSQL  
    - Update bot status for payment phase.

14. **Create Get Work Days Node**  
    - Type: PostgreSQL  
    - Query available working days.

15. **Create Is Date correct? Node**  
    - Type: If  
    - Validate user-selected date against work days.

16. **Create Union Number with Question Node**  
    - Type: Set  
    - Combine date with prompt question.

17. **Create Union list Node**  
    - Type: Summarize  
    - Format message for user.

18. **Create Request Date Node**  
    - Type: WhatsApp  
    - Request date confirmation.

19. **Create Update bot status on BOOKING Node**  
    - Type: PostgreSQL  
    - Update bot status to booking phase.

20. **Create Add Book Node**  
    - Type: PostgreSQL  
    - Insert new booking record.

21. **Create Merge1 Node**  
    - Type: Merge  
    - Combine booking and work day data.

22. **Create Update Date AND Status Node**  
    - Type: PostgreSQL  
    - Update booking date and status.

23. **Create Get Work Hours Node**  
    - Type: PostgreSQL  
    - Query available booking hours.

24. **Create Union Number with Question1 Node**  
    - Type: Set  
    - Combine hours with prompt.

25. **Create Union list1 Node**  
    - Type: Summarize  
    - Format hours message.

26. **Create Request Time Node**  
    - Type: WhatsApp  
    - Request time selection.

27. **Create Update Hour Node**  
    - Type: PostgreSQL  
    - Update booking hour.

28. **Create Request Phone Node**  
    - Type: WhatsApp  
    - Request customer phone number.

29. **Create Phone validation1 Node**  
    - Type: If  
    - Validate phone number format.

30. **Create Update Phone Node**  
    - Type: PostgreSQL  
    - Update booking with phone number.

31. **Create Phone Error Node**  
    - Type: WhatsApp  
    - Notify user of invalid phone number.

32. **Create Request Name Node**  
    - Type: WhatsApp  
    - Request customer name.

33. **Create Update Name Node**  
    - Type: PostgreSQL  
    - Update booking with customer name.

34. **Create page booking success Node**  
    - Type: WhatsApp  
    - Confirm successful booking.

35. **Create Update status on BOOKED Node**  
    - Type: PostgreSQL  
    - Update booking status to "BOOKED".

36. **Create Update bot status on PAYMENTS Node**  
    - Type: PostgreSQL  
    - Update bot status for payment phase.

37. **Connect Nodes According to Workflow Logic**  
    - Follow the connection map: WhatsApp Trigger → Initialization → Get Bot Status → Define Flow → respective branches as detailed above.

38. **Test Workflow**  
    - Validate each step with test messages and database entries.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Replace `"n8n"` in SQL scripts with your actual PostgreSQL schema name before execution.     | Setup instructions in workflow description.                                                     |
| Add your Telegram/WhatsApp bot credentials and PostgreSQL credentials for integration.       | Setup instructions in workflow description.                                                     |
| Customize available days and times in the database to match your business hours.             | Workflow customization advice.                                                                  |
| Adjust database schema to include additional customer preferences or payment info if needed. | Workflow customization advice.                                                                  |
| Workflow timezone is set to Asia/Almaty; adjust if necessary in workflow settings.           | Workflow metadata.                                                                              |
| For WhatsApp integration, ensure webhook URLs are properly configured and accessible.         | WhatsApp Trigger and WhatsApp nodes configuration.                                              |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the Automated Customer Reservations workflow using WhatsApp and PostgreSQL in n8n. It covers all nodes, their roles, configurations, and interconnections to facilitate efficient management and customization.