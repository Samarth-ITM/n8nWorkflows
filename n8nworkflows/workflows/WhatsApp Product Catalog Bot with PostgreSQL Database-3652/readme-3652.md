WhatsApp Product Catalog Bot with PostgreSQL Database

https://n8nworkflows.xyz/workflows/whatsapp-product-catalog-bot-with-postgresql-database-3652


# WhatsApp Product Catalog Bot with PostgreSQL Database

### 1. Workflow Overview

This workflow, titled **WhatsApp Product Catalog Bot with PostgreSQL Database**, is designed for businesses or developers who want to integrate product information into a WhatsApp bot. It allows users to retrieve product details dynamically from a PostgreSQL database via WhatsApp messages.

The workflow automates product data management and retrieval, enabling seamless customer interaction without manual intervention. It is structured into the following logical blocks:

- **1.1 Input Reception:** Captures incoming WhatsApp messages and initializes the workflow.
- **1.2 Bot Status Management:** Retrieves and updates the bot's current status in the database to track conversation state.
- **1.3 Command Processing:** Determines user intent based on input and routes to appropriate actions.
- **1.4 Product Listing:** Queries the database for product lists and formats the response for WhatsApp.
- **1.5 Product Details Retrieval:** Fetches detailed information about a selected product and sends it to the user.
- **1.6 WhatsApp Messaging:** Handles sending messages back to users at various stages.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming WhatsApp messages and initializes the workflow by setting up necessary variables and fetching the bot's current status.

- **Nodes Involved:**  
  - WhatsApp Trigger  
  - Initialization  
  - Get Bot Status  
  - Start?

- **Node Details:**

  - **WhatsApp Trigger**  
    - *Type:* WhatsApp Trigger node  
    - *Role:* Entry point for incoming WhatsApp messages. Triggers workflow on message receipt.  
    - *Configuration:* Uses a webhook ID to listen for WhatsApp messages. No additional parameters.  
    - *Input:* External WhatsApp message event.  
    - *Output:* Message data including sender info and message content.  
    - *Edge Cases:* Webhook misconfiguration, message format variations, or connectivity issues may cause missed triggers.

  - **Initialization**  
    - *Type:* Set node  
    - *Role:* Prepares initial variables or context for the workflow.  
    - *Configuration:* Likely sets default values or extracts key data from the incoming message (e.g., user ID, message text).  
    - *Input:* Output from WhatsApp Trigger.  
    - *Output:* Structured data for downstream nodes.  
    - *Edge Cases:* Expression errors if expected fields are missing.

  - **Get Bot Status**  
    - *Type:* PostgreSQL node (Execute Query)  
    - *Role:* Retrieves the current status of the bot conversation for the user from the database.  
    - *Configuration:* Executes a SELECT query on the bot status table filtered by user ID.  
    - *Input:* User ID or session identifier from Initialization.  
    - *Output:* Bot status record or empty if none exists.  
    - *Edge Cases:* Database connection errors, empty results, or query timeouts.

  - **Start?**  
    - *Type:* If node  
    - *Role:* Checks if the conversation is new or ongoing based on bot status.  
    - *Configuration:* Conditional logic to branch workflow depending on whether bot status exists or indicates a start state.  
    - *Input:* Output from Get Bot Status.  
    - *Output:* Two branches: start new conversation or continue existing.  
    - *Edge Cases:* Incorrect status values or missing data.

---

#### 1.2 Bot Status Management

- **Overview:**  
  Updates or inserts the bot's conversation status in the PostgreSQL database to maintain state across messages.

- **Nodes Involved:**  
  - Starts  
  - Upsert Bot Status

- **Node Details:**

  - **Starts**  
    - *Type:* WhatsApp node (send message)  
    - *Role:* Sends an initial greeting or menu to the user when a new conversation starts.  
    - *Configuration:* Uses a webhook to send a predefined welcome message.  
    - *Input:* Triggered from Start? node's new conversation branch.  
    - *Output:* Confirmation of message sent.  
    - *Edge Cases:* WhatsApp API errors, message formatting issues.

  - **Upsert Bot Status**  
    - *Type:* PostgreSQL node (Execute Query)  
    - *Role:* Inserts or updates the bot status record for the user in the database.  
    - *Configuration:* Uses an UPSERT (INSERT ON CONFLICT) SQL query to maintain conversation state.  
    - *Input:* User ID and new status data from Starts or other nodes.  
    - *Output:* Confirmation of database write operation.  
    - *Edge Cases:* Database write failures, constraint violations.

---

#### 1.3 Command Processing

- **Overview:**  
  Processes user input commands and routes the workflow to appropriate actions such as showing the main menu or listing products.

- **Nodes Involved:**  
  - Commands  
  - Main Menu  
  - List?

- **Node Details:**

  - **Commands**  
    - *Type:* Switch node  
    - *Role:* Evaluates the user's message text to determine which command was issued.  
    - *Configuration:* Multiple cases based on message content (e.g., "1" for product list).  
    - *Input:* User message text from previous nodes.  
    - *Output:* Branches for each recognized command.  
    - *Edge Cases:* Unrecognized commands, case sensitivity, partial matches.

  - **Main Menu**  
    - *Type:* WhatsApp node (send message)  
    - *Role:* Sends the main menu options to the user.  
    - *Configuration:* Predefined menu text with options for user selection.  
    - *Input:* Triggered from Commands node.  
    - *Output:* Confirmation of message sent.  
    - *Edge Cases:* WhatsApp API errors.

  - **List?**  
    - *Type:* If node  
    - *Role:* Checks if the user requested a product list.  
    - *Configuration:* Conditional check on command input.  
    - *Input:* Output from Commands node.  
    - *Output:* Branches to product listing or other actions.  
    - *Edge Cases:* Incorrect command parsing.

---

#### 1.4 Product Listing

- **Overview:**  
  Queries the PostgreSQL database for available products and prepares a formatted list to send to the user.

- **Nodes Involved:**  
  - Get Card products  
  - Get Card products (duplicate node with trailing space)  
  - Union Number with Question  
  - Union list  
  - List Cards

- **Node Details:**

  - **Get Card products**  
    - *Type:* PostgreSQL node (Execute Query)  
    - *Role:* Retrieves a list of products from the database.  
    - *Configuration:* SELECT query fetching product IDs, names, and possibly brief descriptions.  
    - *Input:* Triggered from List? node.  
    - *Output:* Array of product records.  
    - *Edge Cases:* Empty product list, database errors.

  - **Get Card products (with trailing space)**  
    - *Type:* PostgreSQL node (Execute Query)  
    - *Role:* Likely retrieves detailed product data or a subset for further processing.  
    - *Configuration:* Similar to Get Card products but used in a different branch.  
    - *Input:* Branch from List? node.  
    - *Output:* Product data for detailed retrieval.  
    - *Edge Cases:* Same as above.

  - **Union Number with Question**  
    - *Type:* Set node  
    - *Role:* Combines product numbers with corresponding questions or prompts for user selection.  
    - *Configuration:* Formats product list into numbered options with questions.  
    - *Input:* Output from Get Card products.  
    - *Output:* Formatted list ready for summarization.  
    - *Edge Cases:* Formatting errors.

  - **Union list**  
    - *Type:* Summarize node  
    - *Role:* Aggregates the formatted product list into a single message string.  
    - *Configuration:* Joins array elements into a text block suitable for WhatsApp.  
    - *Input:* Output from Union Number with Question.  
    - *Output:* Single string message.  
    - *Edge Cases:* Large lists causing message length issues.

  - **List Cards**  
    - *Type:* WhatsApp node (send message)  
    - *Role:* Sends the product list message to the user.  
    - *Configuration:* Uses webhook to send the aggregated product list.  
    - *Input:* Output from Union list.  
    - *Output:* Confirmation of message sent.  
    - *Edge Cases:* WhatsApp API limits, message formatting.

---

#### 1.5 Product Details Retrieval

- **Overview:**  
  Retrieves detailed information about a selected product from the database and sends it to the user.

- **Nodes Involved:**  
  - Get Card product  
  - Card

- **Node Details:**

  - **Get Card product**  
    - *Type:* PostgreSQL node (Execute Query)  
    - *Role:* Fetches detailed information for a single product based on user selection.  
    - *Configuration:* SELECT query filtered by product ID or user input.  
    - *Input:* Product selection from user message.  
    - *Output:* Detailed product record.  
    - *Edge Cases:* Invalid product ID, no matching record, database errors.

  - **Card**  
    - *Type:* WhatsApp node (send message)  
    - *Role:* Sends detailed product information to the user.  
    - *Configuration:* Formats product details into a message.  
    - *Input:* Output from Get Card product.  
    - *Output:* Confirmation of message sent.  
    - *Edge Cases:* Message formatting, API errors.

---

#### 1.6 WhatsApp Messaging

- **Overview:**  
  Handles all outgoing WhatsApp messages at various stages: greetings, menus, product lists, and product details.

- **Nodes Involved:**  
  - Starts  
  - Main Menu  
  - List Cards  
  - Card

- **Node Details:**  
  These nodes are WhatsApp nodes configured with webhooks to send messages. Each node corresponds to a specific message type or stage in the conversation. They require proper WhatsApp credentials and webhook setup.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                          | Input Node(s)          | Output Node(s)            | Sticky Note                  |
|-------------------------|-------------------------|----------------------------------------|------------------------|---------------------------|------------------------------|
| WhatsApp Trigger        | WhatsApp Trigger         | Entry point for incoming WhatsApp msgs | (Webhook event)        | Initialization            |                              |
| Initialization          | Set                     | Prepares initial variables              | WhatsApp Trigger       | Get Bot Status            |                              |
| Get Bot Status          | PostgreSQL              | Retrieves bot conversation status       | Initialization         | Start?                    |                              |
| Start?                  | If                      | Checks if conversation is new or ongoing| Get Bot Status         | Starts, Commands          |                              |
| Starts                  | WhatsApp                | Sends greeting/start message             | Start? (new conversation) | Upsert Bot Status         |                              |
| Upsert Bot Status       | PostgreSQL              | Inserts/updates bot status in DB         | Starts                 |                           |                              |
| Commands                | Switch                  | Routes user commands                     | Start? (existing conversation) | Main Menu, List?          |                              |
| Main Menu               | WhatsApp                | Sends main menu options                   | Commands               |                           |                              |
| List?                   | If                      | Checks if user requested product list    | Commands               | Get Card products, Get Card products (duplicate) |                              |
| Get Card products       | PostgreSQL              | Retrieves list of products                | List?                  | Union Number with Question |                              |
| Get Card products (dup) | PostgreSQL              | Retrieves product data for details       | List?                  | Get Card product          |                              |
| Union Number with Question | Set                   | Formats product list with numbers/questions | Get Card products       | Union list                |                              |
| Union list              | Summarize               | Aggregates product list into message     | Union Number with Question | List Cards               |                              |
| List Cards              | WhatsApp                | Sends product list message                | Union list             |                           |                              |
| Get Card product        | PostgreSQL              | Retrieves detailed product info           | Get Card products (dup) | Card                      |                              |
| Card                    | WhatsApp                | Sends detailed product info message       | Get Card product       |                           |                              |
| Sticky Note3            | Sticky Note             | (Empty content)                           |                        |                           |                              |
| Sticky Note6            | Sticky Note             | (Empty content)                           |                        |                           |                              |
| Sticky Note7            | Sticky Note             | (Empty content)                           |                        |                           |                              |
| Sticky Note8            | Sticky Note             | (Empty content)                           |                        |                           |                              |
| Sticky Note9            | Sticky Note             | (Empty content)                           |                        |                           |                              |
| Sticky Note10           | Sticky Note             | (Empty content)                           |                        |                           |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create PostgreSQL Tables**  
   - Prepare your PostgreSQL database with the required tables for bot status and product catalog.  
   - Modify the provided SQL script to replace the schema name "n8n" with your own schema.  
   - Execute the script to create tables.

2. **Add Credentials in n8n**  
   - Add WhatsApp credentials (OAuth2 or Account credentials) to enable WhatsApp Trigger and WhatsApp nodes.  
   - Add PostgreSQL credentials with access to your product catalog database.

3. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure webhook ID to listen for incoming WhatsApp messages.  
   - No additional parameters needed.

4. **Create Initialization Node**  
   - Type: Set  
   - Configure to extract and set variables such as user ID, message text, and any other context needed.

5. **Create Get Bot Status Node**  
   - Type: PostgreSQL  
   - Configure with a SELECT query to fetch bot status by user ID.  
   - Use credentials set earlier.

6. **Create Start? Node**  
   - Type: If  
   - Configure condition to check if bot status exists or indicates a new conversation.

7. **Create Starts Node**  
   - Type: WhatsApp  
   - Configure webhook ID for sending messages.  
   - Set message content as a greeting or welcome message.

8. **Create Upsert Bot Status Node**  
   - Type: PostgreSQL  
   - Configure an UPSERT query to insert or update bot status for the user.

9. **Create Commands Node**  
   - Type: Switch  
   - Configure cases for recognized commands (e.g., "1" for product list).  
   - Route to Main Menu or List? nodes accordingly.

10. **Create Main Menu Node**  
    - Type: WhatsApp  
    - Configure webhook ID and message content listing available commands.

11. **Create List? Node**  
    - Type: If  
    - Configure condition to check if user requested product list.

12. **Create Get Card products Node**  
    - Type: PostgreSQL  
    - Configure SELECT query to retrieve product list (IDs, names).  
    - Connect from List? node.

13. **Create Get Card products (duplicate) Node**  
    - Type: PostgreSQL  
    - Configure SELECT query to retrieve product details for selected product.

14. **Create Union Number with Question Node**  
    - Type: Set  
    - Configure to format product list into numbered options with prompts.

15. **Create Union list Node**  
    - Type: Summarize  
    - Configure to join formatted product options into a single message string.

16. **Create List Cards Node**  
    - Type: WhatsApp  
    - Configure webhook ID to send the product list message.

17. **Create Get Card product Node**  
    - Type: PostgreSQL  
    - Configure SELECT query to fetch detailed info for a selected product.

18. **Create Card Node**  
    - Type: WhatsApp  
    - Configure webhook ID and message content to send detailed product info.

19. **Connect Nodes According to Logic:**  
    - WhatsApp Trigger → Initialization → Get Bot Status → Start?  
    - Start? (new) → Starts → Upsert Bot Status  
    - Start? (existing) → Commands → Main Menu or List?  
    - List? → Get Card products → Union Number with Question → Union list → List Cards  
    - List? → Get Card products (dup) → Get Card product → Card

20. **Test Workflow:**  
    - Send WhatsApp messages to trigger the bot.  
    - Verify product list retrieval and detailed product info responses.  
    - Monitor database for bot status updates.

---

This documentation provides a complete and structured reference to understand, reproduce, and modify the WhatsApp Product Catalog Bot workflow integrated with a PostgreSQL database.