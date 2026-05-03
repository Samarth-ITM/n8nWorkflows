Create a Two-Way WhatsApp + Telegram Integration for 10k+ Customer Support Chats

https://n8nworkflows.xyz/workflows/create-a-two-way-whatsapp---telegram-integration-for-10k--customer-support-chats-8350


# Create a Two-Way WhatsApp + Telegram Integration for 10k+ Customer Support Chats

### 1. Workflow Overview

This workflow establishes a scalable two-way integration bridge between WhatsApp and Telegram to manage customer support conversations for 10,000+ customers. It routes incoming WhatsApp messages into dedicated Telegram forum topics (threads) within a Telegram supergroup, enabling support agents to reply from Telegram. Replies in Telegram topics are then sent back to the correct WhatsApp customer seamlessly, preserving conversation context.

**Target Use Cases:**  
- Customer support teams managing high volumes of WhatsApp chats.  
- Agents using Telegram as a unified interface for WhatsApp messaging.  
- Automatic thread creation and mapping for each unique customer.  
- Maintaining conversation context with message-thread mapping in a database.

**Logical Blocks:**

- **1.1 Incoming WhatsApp Message Handling**  
  Triggered by new WhatsApp messages, looks up or creates Telegram topics, and forwards messages to Telegram.

- **1.2 Outgoing Telegram Message Handling**  
  Triggered by Telegram messages in specific topics, resolves the mapping to WhatsApp customer, and sends replies back to WhatsApp.

- **1.3 Database Integration & Mapping Management**  
  Uses Supabase (Postgres) to map WhatsApp phone numbers with Telegram supergroup and topic IDs for message routing.

- **1.4 Auxiliary Setup and Control Nodes**  
  Includes nodes for setting constants (Telegram supergroup ID), merging datasets, and conditional routing.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Incoming WhatsApp Message Handling

- **Overview:**  
  Listens for incoming WhatsApp messages, checks if the customer already has a Telegram topic. If yes, forwards the message to the existing Telegram topic. If no, creates a new Telegram topic, stores the mapping, and then sends the message.

- **Nodes Involved:**  
  - WhatsApp Trigger  
  - Get existing customer details (Supabase)  
  - Merge  
  - Set Telegram SuperGroupID  
  - Get customer by phone (Supabase)  
  - Customer Details (Set)  
  - Merge1  
  - Set Customer Name  
  - Check existing conversation or not (If)  
  - Send Telegram Message to existing topic (HTTP Request)  
  - Create a Telegram Topic (HTTP Request)  
  - Create a row (Supabase)  
  - Send a Telegram message to topic (HTTP Request)  

- **Node Details:**

  1. **WhatsApp Trigger**  
     - Type: Trigger node for WhatsApp Cloud API messages  
     - Config: Listens for new message events  
     - Credentials: OAuth2 WhatsApp account  
     - Input: Incoming WhatsApp messages  
     - Output: Message JSON with phone number and text  
     - Edge cases: OAuth token expiration, network issues  

  2. **Get existing customer details**  
     - Type: Supabase DB query  
     - Config: Filters `wa_tg_threads` table by `phone_e164` = incoming phone and fixed supergroup ID `-1003071123489`  
     - Output: Existing Telegram topic mapping if any  
     - OnError: Continue (allows missing data)  
     - Edge cases: DB connection errors, empty results  

  3. **Merge**  
     - Type: Combine mode merge of incoming WhatsApp data and DB fetch  
     - Config: Joins on customer phone number fields  
     - Logic: Conditional join depending on presence of existing records  

  4. **Set Telegram SuperGroupID**  
     - Type: Set constant node  
     - Config: Sets Telegram supergroup ID string `-100307123389` (must be replaced by user)  

  5. **Get customer by phone**  
     - Type: Supabase DB query  
     - Config: Queries `orders` table by phone number to get customer name and details  
     - OnError: Continue  

  6. **Customer Details**  
     - Type: Set node  
     - Config: Sets `customerName` and `CustomerPhone` from previous workflow or inputs  

  7. **Merge1**  
     - Type: Combine mode merge  
     - Config: Joins customer details and order info by phone number  

  8. **Set Customer Name**  
     - Type: Set node  
     - Config: Sets `CustomerName` with formatted string containing customer name and phone number, depending on DB query results  

  9. **Check existing conversation or not (If)**  
     - Type: Conditional node  
     - Config: Checks if existing customer details records exist (>0 length)  
     - True branch: Send Telegram message to existing topic  
     - False branch: Create Telegram topic  

  10. **Send Telegram Message to existing topic (HTTP Request)**  
      - Type: HTTP POST  
      - URL: Telegram Bot API `sendMessage` endpoint  
      - Config: Uses supergroup ID and existing topic ID from DB to send message text  
      - Important: Replace `<your_bot_token>` with actual Telegram bot token  
      - Edge cases: API rate limits, invalid tokens  

  11. **Create a Telegram Topic (HTTP Request)**  
      - Type: HTTP POST  
      - URL: Telegram Bot API `createForumTopic` endpoint  
      - Config: Creates new forum topic named by customer name in Telegram supergroup  
      - Output: Topic creation result with `message_thread_id`  

  12. **Create a row (Supabase)**  
      - Type: DB insert  
      - Config: Stores mapping between phone number, new Telegram topic ID, and supergroup ID  

  13. **Send a Telegram message to topic (HTTP Request)**  
      - Type: HTTP POST  
      - URL: Telegram Bot API `sendMessage` endpoint  
      - Config: Sends initial customer message to newly created topic  

- **Edge Cases & Failure Points:**  
  - Telegram bot token missing or invalid  
  - Supabase DB connectivity issues or query failures  
  - Telegram API rate limits (429 errors)  
  - Missing or incorrect supergroup ID (must be valid supergroup with topics enabled)  
  - Phone number format mismatches  

---

#### Block 1.2: Outgoing Telegram Message Handling

- **Overview:**  
  Listens for Telegram messages in forum topics, filters to only replies inside topics, fetches mapping from `telegram_topic_id` to WhatsApp phone number, and sends the Telegram reply back to the corresponding WhatsApp customer.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Filter messages that are replies in topics (Set)  
  - If (to verify message is in correct supergroup and has text)  
  - Get a row (Supabase)  
  - Send message (WhatsApp)  

- **Node Details:**

  1. **Telegram Trigger**  
     - Type: Trigger node for Telegram Bot updates  
     - Config: Listens for new messages in the bot’s scope  
     - Credentials: Telegram API credentials  
     - Output: Telegram message JSON including `message_thread_id`  

  2. **Filter messages that are replies in topics**  
     - Type: Set node  
     - Config: Extracts `superGroupID` constant (-1003071123489) and message text from the Telegram message JSON  
     - Purpose: Prepare fields for filtering  

  3. **If**  
     - Type: Conditional node  
     - Config: Checks if message chat ID equals superGroupID AND message text exists (non-empty)  
     - True path continues to DB lookup  

  4. **Get a row**  
     - Type: Supabase DB query  
     - Config: Filters `wa_tg_threads` by `telegram_topic_id` matching Telegram message’s `message_thread_id`  
     - Output: Phone number associated with Telegram topic  

  5. **Send message (WhatsApp)**  
     - Type: WhatsApp Cloud API send message node  
     - Config: Sends text body from Telegram message to the `phone_e164` retrieved from DB  
     - Credentials: WhatsApp API token and phone number ID  
     - Edge cases: WhatsApp API quota limits, invalid phone number, expired token  

- **Edge Cases & Failure Points:**  
  - Telegram message not in expected supergroup or no topic id  
  - No mapping found in DB for Telegram topic (message lost)  
  - WhatsApp API send failures (network, auth)  
  - Message content empty or malformed  

---

#### Block 1.3: Database Integration & Mapping Management

- **Overview:**  
  Manages the relational mapping of WhatsApp phone numbers with Telegram supergroup and topic IDs in a Supabase/Postgres table `wa_tg_threads`. This mapping is used to route messages correctly both ways.

- **Nodes Involved:**  
  - Get existing customer details (Supabase)  
  - Get a row (Supabase)  
  - Get customer by phone (Supabase)  
  - Create a row (Supabase)  

- **Node Details:**  
  - Queries and inserts are filtered by phone number and supergroup ID.  
  - The table columns include `phone_e164`, `supergroup_id`, `telegram_topic_id`, with timestamps.  
  - OnError continues on select queries to avoid workflow failures on empty results.  
  - Insertions happen when a new Telegram topic is created for a new customer.

- **Edge Cases & Failure Points:**  
  - DB connection timeouts or failures  
  - Data inconsistency due to race conditions (multiple messages from same number simultaneously)  
  - Missing indices on `phone_e164` or `telegram_topic_id` could slow queries  

---

#### Block 1.4: Auxiliary Setup and Control Nodes

- **Overview:**  
  Support nodes to set fixed parameters, merge multiple data sources, and provide conditionals for flow control.

- **Nodes Involved:**  
  - Set Telegram SuperGroupID  
  - Merge  
  - Merge1  
  - Customer Details (Set)  
  - Check existing conversation or not (If)  

- **Node Details:**  
  - Use of conditional merges with `enrichInput1` or `enrichInput2` join mode depending on presence of existing data.  
  - Fixed Telegram supergroup ID is set here and used globally.  
  - Logical branching based on DB query results to route messages properly.

- **Edge Cases & Failure Points:**  
  - Incorrect or outdated supergroup ID can cause routing failures.  
  - Merge logic failures if input data is missing or malformed.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                     | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                                  |
|-----------------------------------|---------------------|----------------------------------------------------|----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger                  | WhatsApp Trigger    | Triggers on incoming WhatsApp messages             | None                             | Get existing customer details        | ## Incoming WhatsApp message to Telegram                                                                     |
| Get existing customer details     | Supabase            | Queries mapping table for existing Telegram topic  | WhatsApp Trigger                | Merge                               |                                                                                                              |
| Merge                            | Merge               | Combines incoming message with existing DB data    | Get existing customer details    | Set Telegram SuperGroupID             |                                                                                                              |
| Set Telegram SuperGroupID         | Set                 | Sets Telegram supergroup ID constant                | Merge                           | Get customer by phone, Customer Details | ## Change this to your Telegram Supergroup ID. The given -1003071123489 is a sample non working supergroupID |
| Get customer by phone             | Supabase            | Retrieves customer info from orders table           | Set Telegram SuperGroupID       | Merge1                              |                                                                                                              |
| Customer Details                 | Set                 | Sets customer name and phone from previous steps    | Set Telegram SuperGroupID       | Merge1                              |                                                                                                              |
| Merge1                           | Merge               | Combines customer details and order info            | Get customer by phone, Customer Details | Set Customer Name                   |                                                                                                              |
| Set Customer Name                | Set                 | Formats customer display name with phone            | Merge1                          | Check existing conversation or not   |                                                                                                              |
| Check existing conversation or not | If                  | Branches based on existing conversation presence    | Set Customer Name               | Send Telegram Message to existing topic, Create a Telegram Topic |                                                                                                              |
| Send Telegram Message to existing topic | HTTP Request        | Sends WhatsApp message text to existing Telegram topic | Check existing conversation or not (True) | None                                | ### Send another WhatsApp message from the same number. Confirm that the message goes into the same Telegram topic, not a new one. |
| Create a Telegram Topic           | HTTP Request        | Creates new Telegram forum topic for new customer   | Check existing conversation or not (False) | Create a row                        | ### Send a WhatsApp message from a new number. Check if a new Telegram topic is automatically created for that customer. |
| Create a row                     | Supabase            | Stores mapping between WhatsApp number and Telegram topic | Create a Telegram Topic        | Send a Telegram message to topic      | ### Create a table in supabase which maps phone number to telegram supergroup and message threadId           |
| Send a Telegram message to topic | HTTP Request        | Sends initial WhatsApp message to newly created Telegram topic | Create a row                  | None                                | ### Add Your Bot Token in the Telegram request URL                                                           |
| Telegram Trigger                | Telegram Trigger   | Triggers on incoming Telegram messages in topics   | None                             | Filter messages that are replies in topics | ## Outgoing Telegram Messages to WhatsApp                                                                     |
| Filter messages that are replies in topics | Set                 | Extracts superGroupID and message text for filtering | Telegram Trigger               | If                                  |                                                                                                              |
| If                              | If                  | Checks message belongs to supergroup and has text  | Filter messages that are replies in topics | Get a row                        |                                                                                                              |
| Get a row                       | Supabase            | Retrieves WhatsApp phone number by Telegram topic ID | If                             | Send message                        |                                                                                                              |
| Send message                   | WhatsApp             | Sends Telegram reply text back to WhatsApp customer | Get a row                      | None                                | ### Customer will receive Telegram message in customer's Whatsapp                                            |
| Sticky Note (various)           | Sticky Note          | Provides instructions, notes, and visuals          | None                            | None                                | Various notes including setup, usage tips, and testing instructions                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node:**  
   - Type: WhatsApp Trigger  
   - Set webhook to listen for message updates  
   - Configure OAuth2 credentials for WhatsApp Cloud API  

2. **Create Supabase Node "Get existing customer details":**  
   - Type: Supabase (Get)  
   - Table: `wa_tg_threads`  
   - Filter conditions: `phone_e164` equals incoming WhatsApp phone number, `supergroup_id` fixed to your Telegram supergroup ID (e.g., `-1003071123489`)  
   - On error: Continue  

3. **Create Merge Node:**  
   - Mode: Combine  
   - Join Mode: Conditional expression to enrich inputs based on if DB returned results  
   - Merge by fields: `customerPhone` and `phone_e164`  

4. **Create Set Node "Set Telegram SuperGroupID":**  
   - Assign variable `superGroupID` with your Telegram supergroup ID (string, e.g., `-100307123389`)  

5. **Create Supabase Node "Get customer by phone":**  
   - Type: Supabase (Get)  
   - Table: `orders`  
   - Filter: `phone_number` equals customer phone from previous step  
   - On error: Continue  

6. **Create Set Node "Customer Details":**  
   - Assign `customerName` and `CustomerPhone` from inputs or previous node outputs  

7. **Create Merge Node "Merge1":**  
   - Mode: Combine  
   - Join Mode: Conditional join based on whether customer by phone results exist  
   - Merge by fields: `CustomerPhone` and `phone_number`  

8. **Create Set Node "Set Customer Name":**  
   - Assign `CustomerName` with conditional formatting:  
     - If customer found, use DB name and phone  
     - Else, use fallback from customer details  

9. **Create If Node "Check existing conversation or not":**  
   - Condition: Check if length of items from "Get existing customer details" > 0  
   - True branch: connect to "Send Telegram Message to existing topic"  
   - False branch: connect to "Create a Telegram Topic"  

10. **Create HTTP Request Node "Send Telegram Message to existing topic":**  
    - Method: POST  
    - URL: `https://api.telegram.org/bot<your_bot_token>/sendMessage` (replace token)  
    - Body parameters:  
      - `chat_id`: superGroupID  
      - `message_thread_id`: from existing DB mapping  
      - `text`: customer message text  
    - Send body as JSON  

11. **Create HTTP Request Node "Create a Telegram Topic":**  
    - Method: POST  
    - URL: `https://api.telegram.org/bot<your_bot_token>/createForumTopic`  
    - Body parameters:  
      - `chat_id`: superGroupID  
      - `name`: customer name  
    - Output contains new `message_thread_id`  

12. **Create Supabase Node "Create a row":**  
    - Table: `wa_tg_threads`  
    - Fields:  
      - `phone_e164`: customer phone  
      - `telegram_topic_id`: new topic ID  
      - `supergroup_id`: superGroupID  

13. **Create HTTP Request Node "Send a Telegram message to topic":**  
    - Method: POST  
    - URL: `https://api.telegram.org/bot<your_bot_token>/sendMessage`  
    - Body parameters:  
      - `chat_id`: superGroupID  
      - `message_thread_id`: new topic ID  
      - `text`: customer message  

14. **Create Telegram Trigger Node:**  
    - Type: Telegram Trigger  
    - Listen for message updates  
    - Credentials: Telegram bot token with admin rights in supergroup  

15. **Create Set Node "Filter messages that are replies in topics":**  
    - Assign `superGroupID` constant (same as earlier)  
    - Extract Telegram message text from input JSON  

16. **Create If Node:**  
    - Condition: Telegram message chat ID equals superGroupID AND message text exists  

17. **Create Supabase Node "Get a row":**  
    - Filter: `telegram_topic_id` equals Telegram message's `message_thread_id`  
    - Table: `wa_tg_threads`  

18. **Create WhatsApp Node "Send message":**  
    - Operation: Send message  
    - PhoneNumberId: your WhatsApp phone number ID  
    - RecipientPhoneNumber: phone number fetched from DB  
    - TextBody: message text from Telegram message  

19. **Connect nodes as per logical flow:**  
    - WhatsApp Trigger → Get existing customer details → Merge → Set Telegram SuperGroupID → Get customer by phone & Customer Details → Merge1 → Set Customer Name → Check existing conversation or not → (True) Send Telegram Message to existing topic OR (False) Create a Telegram Topic → Create a row → Send a Telegram message to topic  
    - Telegram Trigger → Filter messages → If → Get a row → Send message  

20. **Credentials Setup:**  
    - WhatsApp Cloud API OAuth2 for WhatsApp nodes  
    - Telegram Bot API token with admin rights on supergroup for Telegram nodes  
    - Supabase API key and URL for Supabase nodes  

21. **Notes:**  
    - Replace all placeholder tokens and IDs (`<your_bot_token>`, supergroup ID, phone number IDs) with actual deployment values  
    - Ensure Telegram bot is admin with manage topics permission in supergroup  
    - Create `wa_tg_threads` table in Supabase with columns for phone_e164, telegram_topic_id, supergroup_id, timestamps  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This n8n template builds a WhatsApp ↔ Telegram Customer Support Bridge that creates one Telegram topic per customer and syncs replies both ways, no custom app required. Use cases include routing WhatsApp conversations into Telegram forum topics, letting agents reply from Telegram, and scaling cleanly to 10k+ customers.                                                                                                                                                                                             | Overview from embedded Sticky Note11                                                                         |
| The Telegram Bot must be added as admin to a supergroup with Topics enabled. The supergroup ID must be set correctly in the workflow nodes (Set Telegram SuperGroupID).                                                                                                                                                                                                                                                                                                                                                   | Sticky Note8                                                                                                |
| Add your actual Telegram Bot token in all Telegram HTTP Request URLs replacing `<your_bot_token>`.                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note9, Sticky Note10                                                                                   |
| Create a Supabase/Postgres table `wa_tg_threads` with columns: `id` (UUID), `phone_e164` (string), `supergroup_id` (string), `telegram_topic_id` (integer), and timestamps to map phone numbers to Telegram topics.                                                                                                                                                                                                                                                                                                             | Sticky Note7                                                                                                |
| Test scripts for Telegram API: create forums and send messages using `curl` commands are provided in the documentation. Use these to verify token and permissions.                                                                                                                                                                                                                                                                                                                                                          | From embedded notes in Sticky Note11                                                                         |
| For scale and robustness: use Docker secrets or n8n credentials environment variables for tokens, handle Telegram 429 rate limits with retry/backoff, and consider queuing WhatsApp sends.                                                                                                                                                                                                                                                                                                                                    | Notes from Sticky Note11                                                                                      |
| If messages land in the Telegram supergroup’s General chat instead of topics, verify that the `message_thread_id` parameter is correctly included in Telegram API requests.                                                                                                                                                                                                                                                                                                                                                | Notes from Sticky Note11                                                                                      |
| Email for feedback and support: [info@zenithworks.ai](mailto:info@zenithworks.ai)                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note11                                                                                                |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.