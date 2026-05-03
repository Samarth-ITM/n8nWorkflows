Trello Task Management with Telegram Notifications and Supabase Database

https://n8nworkflows.xyz/workflows/trello-task-management-with-telegram-notifications-and-supabase-database-9916


# Trello Task Management with Telegram Notifications and Supabase Database

### 1. Workflow Overview

This workflow automates Trello task management by synchronizing card and user data with a Supabase database and sending real-time notifications via Telegram. It is designed to:

- Track new Trello cards, update card details and membership changes
- Maintain corresponding records in Supabase tables (`cards`, `users`, `card_user`)
- Notify users on Telegram when they are added or removed from cards
- Remind users about tasks due on the current day through scheduled notifications

The workflow is logically divided into three main blocks:

**1.1 Trello Trigger Flow**  
Triggered by Trello board events (like new cards), this block creates card records in Supabase, sets up Trello webhooks for card updates, and manages user-card relationships.

**1.2 Webhook Event Flow**  
Handles incoming webhook events from Trello about card updates, member additions/removals, and syncs these changes into Supabase. It also generates personalized Telegram notifications for these actions.

**1.3 Due-Date Notification Flow**  
Runs on a schedule (twice daily) to find cards due today and sends Telegram reminders to assigned users.

---

### 2. Block-by-Block Analysis

#### 2.1 Trello Trigger Flow

- **Overview:**  
  Activated when a new Trello card is created on the specified board. This flow inserts the card’s details into Supabase, creates a Trello webhook for real-time updates on that card, and prepares for user-card membership management.

- **Nodes Involved:**  
  - Trello Trigger  
  - Create card (Supabase)  
  - Create Card Webhook (HTTP Request)

- **Node Details:**

  1. **Trello Trigger**  
     - Type: Trello Trigger node  
     - Role: Listens to new card creation events on a specified Trello board.  
     - Config: Linked to a Trello board ID and uses Trello API credentials.  
     - Input: Trello event webhook from the board  
     - Output: JSON payload including card data  
     - Edge Cases: Authentication failures with Trello API; invalid board ID; webhook misconfiguration.

  2. **Create card**  
     - Type: Supabase node  
     - Role: Inserts new card details into the `cards` table in Supabase.  
     - Config: Fields set to card ID, card name, and board name from Trello event. Uses Supabase credentials.  
     - Input: Output from Trello Trigger node  
     - Output: Confirmation of record creation  
     - Edge Cases: Database connectivity issues; duplicate card insertion errors; malformed data.

  3. **Create Card Webhook**  
     - Type: HTTP Request node  
     - Role: Registers a Trello webhook for the new card to listen for updates (e.g., member changes, due date updates).  
     - Config: POST request to Trello API with API key, token, webhook callback URL (n8n webhook node URL), and card ID as monitored model.  
     - Input: Output from Create card node (used to get card ID and name)  
     - Output: Webhook creation confirmation  
     - Edge Cases: Invalid Trello API key/token; incorrect webhook URL; Trello API rate limiting.

---

#### 2.2 Webhook Event Flow

- **Overview:**  
  Processes incoming Trello webhook events about card updates such as member additions/removals or due date changes. It updates Supabase accordingly, manages user existence and relationships, and sends formatted Telegram notifications to relevant users.

- **Nodes Involved:**  
  - Webhook  
  - Trello Action Type Check (If node)  
  - Update card due-date (Supabase)  
  - Add/Remove Member from Card Switch (Switch node)  
  - Get user row (Supabase)  
  - Does User Exist? (If node)  
  - Create user (Supabase)  
  - Code in JavaScript  
  - Create user card relation (Supabase)  
  - Delete from users-card (Supabase)  
  - username (Set node)  
  - Send a text message (Telegram)  

- **Node Details:**

  1. **Webhook**  
     - Type: Webhook node  
     - Role: Receives Trello webhook event payloads at a specified path with POST method.  
     - Config: Accepts POST requests with CORS open (`*`), responds with "ok" text.  
     - Input: External HTTP POST from Trello webhook  
     - Output: JSON with Trello action details  
     - Edge Cases: Unauthenticated requests; webhook URL mismatch; malformed payload.

  2. **Trello Action Type Check**  
     - Type: If node  
     - Role: Filters webhook events to only process member addition/removal actions.  
     - Config: Checks if `$json.body.action.type` equals `addMemberToCard` or `removeMemberFromCard`.  
     - Input: Output from Webhook node  
     - Output: True branch if condition met, else false  
     - Edge Cases: Unexpected action types; null or missing `action.type`.

  3. **Update card due-date**  
     - Type: Supabase node  
     - Role: Updates due date, completion status, and URL of the card in the `cards` table.  
     - Config: Filters by card ID; updates `due_date`, `completed`, and `url` fields using values from webhook payload.  
     - Input: Output from Webhook node (via Trello Action Type Check)  
     - Edge Cases: Database update conflicts; invalid date formatting.

  4. **Add/Remove Member from Card Switch**  
     - Type: Switch node  
     - Role: Routes actions based on whether a member was added or removed from a card.  
     - Config: Matches `addMemberToCard` or `removeMemberFromCard` action types.  
     - Input: Output from Trello Action Type Check  
     - Output: Two branches - one for add, one for remove  
     - Edge Cases: Unexpected action types; missing action type.

  5. **Get user row**  
     - Type: Supabase node  
     - Role: Retrieves user record from `users` table by Trello member ID.  
     - Config: Filter by member ID from webhook payload.  
     - Input: Output from Add/Remove Member switch (both branches)  
     - Edge Cases: No user found; database connectivity issues.

  6. **Does User Exist?**  
     - Type: If node  
     - Role: Checks if user record exists based on presence of an `id` field.  
     - Config: Condition checks if `$json.id` is not empty.  
     - Input: Output from Get user row  
     - Output: True (user exists), False (create user)  
     - Edge Cases: False positives from malformed data.

  7. **Create user**  
     - Type: Supabase node  
     - Role: Inserts new user record into `users` table with member ID and name.  
     - Config: Uses member ID and name from webhook payload.  
     - Input: False output from Does User Exist?  
     - Edge Cases: Duplicate insertion attempts; database write failures.

  8. **Code in JavaScript**  
     - Type: Code node  
     - Role: Generates a personalized notification message based on action type (add/remove member) with randomized friendly messages. Escapes Markdown for Telegram formatting.  
     - Uses data such as username mappings, card name, action date, description, URL, and due date.  
     - Input: Output from username node (which sets Telegram username aliases)  
     - Output: JSON with finalMessage and other card/user details  
     - Edge Cases: Expression errors; missing fields; escaping errors.

  9. **username**  
     - Type: Set node  
     - Role: Maps Trello member names to Telegram usernames using a static dictionary.  
     - Config: Uses expression to map member names to Telegram handles or fallback to member name.  
     - Input: Output from Trello Action Type Check node  
     - Edge Cases: Unmapped usernames; case sensitivity issues.

  10. **Create user card relation**  
      - Type: Supabase node  
      - Role: Creates a record linking users to cards in `card_user` table.  
      - Config: Uses card ID and user ID from the output of the JavaScript code node which maps multiple users.  
      - Input: Output from Code in JavaScript1 (explained below)  
      - Edge Cases: Duplicate entries; foreign key constraint failures.

  11. **Delete from users-card**  
      - Type: Supabase node  
      - Role: Removes the user-card link from `card_user` table on member removal.  
      - Config: Filters on user ID and card ID from webhook payload.  
      - Input: Remove member branch from Add/Remove Member switch  
      - Edge Cases: Record not found; database deletion errors.

  12. **Send a text message**  
      - Type: Telegram node  
      - Role: Sends formatted notification messages to Telegram chat, informing users about their card membership changes.  
      - Config: Uses chat ID, message text from JavaScript code output, supports Markdown V2 formatting.  
      - Input: Output from Code in JavaScript node  
      - Edge Cases: Telegram API errors; chat ID invalid or bot blocked.

---

#### 2.3 Due-Date Notification Flow

- **Overview:**  
  Periodically triggers a schedule (every 12 hours) that queries Supabase for cards due today, retrieves assigned users, and sends Telegram reminders about imminent due dates.

- **Nodes Involved:**  
  - Due-Date Notification Schedule Trigger  
  - Get all cards due today (Supabase)  
  - Get user-card (Supabase)  
  - Get users in card (Supabase)  
  - usernames 1 (Set node)  
  - Code in JavaScript2  
  - Send a text message1 (Telegram)

- **Node Details:**

  1. **Due-Date Notification Schedule Trigger**  
     - Type: Schedule Trigger node  
     - Role: Runs workflow every 12 hours to check for due cards.  
     - Config: Interval set to 12 hours.  
     - Edge Cases: Trigger downtime; missed schedules.

  2. **Get all cards due today**  
     - Type: Supabase node  
     - Role: Queries `cards` table for entries where `due_date` equals today’s date.  
     - Config: Uses dynamic filter with `new Date().toLocaleDateString('en-US')`.  
     - Edge Cases: Timezone mismatches; date format inconsistencies.

  3. **Get user-card**  
     - Type: Supabase node  
     - Role: Retrieves card-user relation records for the due cards (filters by card ID).  
     - Edge Cases: Missing relations; large data sets.

  4. **Get users in card**  
     - Type: Supabase node  
     - Role: Fetches user details for each user ID found in card-user relations.  
     - Edge Cases: Missing user records.

  5. **usernames 1**  
     - Type: Set node  
     - Role: Maps user names to Telegram usernames for messaging.  
     - Config: Similar mapping dictionary as the `username` node.  
     - Edge Cases: Unmapped usernames.

  6. **Code in JavaScript2**  
     - Type: Code node  
     - Role: Constructs the final message payload with username, card name, board name, due date, and URL for Telegram notification.  
     - Input: Data from usernames 1 and Get all cards due today nodes.  
     - Edge Cases: Missing data; malformed data.

  7. **Send a text message1**  
     - Type: Telegram node  
     - Role: Sends due date reminder messages to users’ Telegram chat.  
     - Config: Similar markdown formatted message including warning emojis and card details.  
     - Edge Cases: Telegram API failures; invalid chat IDs.

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                                      | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                              |
|--------------------------------|------------------------|-----------------------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------|
| Trello Trigger                 | Trello Trigger         | Triggers workflow on new Trello card creation       | —                                | Create card                     | Replace `[BOARD_ID]` with your Trello board Id. See Sticky Note2 for details                           |
| Create card                   | Supabase               | Inserts new card record in database                  | Trello Trigger                   | Create Card Webhook             | See Sticky Note5 for API key/token replacement and webhook URL setting                                  |
| Create Card Webhook           | HTTP Request           | Creates Trello webhook for card updates              | Create card                     | —                               | See Sticky Note5                                                                                        |
| Webhook                      | Webhook                | Receives Trello webhook events                        | External Trello webhook          | Trello Action Type Check         | Replace `[WEBHOOK_PATH]` (Sticky Note6) and ensure webhook URL matches Trello settings                   |
| Trello Action Type Check      | If                     | Filters webhook events for member add/remove actions | Webhook                        | Update card due-date, Add/Remove Member from Card Switch | —                                                                                                      |
| Update card due-date          | Supabase               | Updates card due date and status                      | Trello Action Type Check         | —                               | —                                                                                                      |
| Add/Remove Member from Card Switch | Switch             | Routes member add or remove actions                   | Trello Action Type Check         | Get user row, Delete from users-card | —                                                                                                      |
| Get user row                 | Supabase               | Retrieves user record by member ID                    | Add/Remove Member from Card Switch | Does User Exist?                | —                                                                                                      |
| Does User Exist?             | If                     | Checks if user exists in database                     | Get user row                   | Code in JavaScript1 (if exists), Create user (if not) | —                                                                                                      |
| Create user                 | Supabase               | Inserts new user record                               | Does User Exist? (false branch)  | Code in JavaScript1             | —                                                                                                      |
| Code in JavaScript1           | Code                   | Maps multiple members to user-card relation payload  | Does User Exist? (true), Create user | Create user card relation       | —                                                                                                      |
| Create user card relation     | Supabase               | Inserts card-user relation record                     | Code in JavaScript1             | —                               | —                                                                                                      |
| Delete from users-card       | Supabase               | Deletes user-card relation on member removal          | Add/Remove Member from Card Switch (remove branch) | —                               | —                                                                                                      |
| username                    | Set                    | Maps Trello member names to Telegram usernames       | Trello Action Type Check         | Code in JavaScript              | See Sticky Note4 for username mapping                                                                  |
| Code in JavaScript           | Code                   | Constructs Telegram notification text for member changes | username                       | Send a text message             | —                                                                                                      |
| Send a text message          | Telegram                | Sends Telegram message about membership changes      | Code in JavaScript              | —                               | Replace `[TELEGRAM_CHAT_ID]` (Sticky Note3)                                                            |
| Due-Date Notification Schedule Trigger | Schedule Trigger   | Triggers twice daily to check due tasks               | —                                | Get all cards due today          | —                                                                                                      |
| Get all cards due today      | Supabase               | Queries cards with due date equal to today            | Due-Date Notification Schedule Trigger | Get user-card                 | —                                                                                                      |
| Get user-card               | Supabase               | Retrieves user-card relations for due cards           | Get all cards due today          | Get users in card               | —                                                                                                      |
| Get users in card           | Supabase               | Retrieves user info for each user linked to card      | Get user-card                  | usernames 1                    | —                                                                                                      |
| usernames 1                 | Set                    | Maps user names to Telegram usernames                  | Get users in card              | Code in JavaScript2             | See Sticky Note4                                                                                        |
| Code in JavaScript2          | Code                   | Builds Telegram message payload for due date reminders | usernames 1                    | Send a text message1            | —                                                                                                      |
| Send a text message1         | Telegram                | Sends Telegram due date reminder message              | Code in JavaScript2             | —                               | Replace `[TELEGRAM_CHAT_ID]` (Sticky Note3)                                                            |
| Sticky Note, Sticky Note1,...| Sticky Note            | Various instructions and workflow overview notes      | —                                | —                               | See notes below                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trello Trigger node:**  
   - Type: Trello Trigger  
   - Set board ID to your Trello board's ID  
   - Use Trello API credentials  
   - Purpose: Trigger on new cards created on the board

2. **Create Supabase node "Create card":**  
   - Type: Supabase  
   - Operation: Insert into `cards` table  
   - Fields:  
     - `id`: `{{$json.action.data.card.id}}`  
     - `card_name`: `{{$json.action.data.card.name}}`  
     - `board_name`: `{{$json.action.data.board.name}}`  
   - Credentials: Supabase API credentials  
   - Connect input from Trello Trigger

3. **Create HTTP Request node "Create Card Webhook":**  
   - Method: POST  
   - URL: `https://api.trello.com/1/tokens/[TRELLO_TOKEN]/webhooks/`  
   - Body (JSON):  
     - `key`: `[TRELLO_API_KEY]`  
     - `callbackURL`: Your n8n webhook node URL  
     - `idModel`: Card ID from previous node  
     - `description`: Description string with card name  
   - Headers: `Content-Type: application/json`  
   - Connect input from "Create card" node

4. **Create Webhook node:**  
   - Path: `[WEBHOOK_PATH]` (your chosen path)  
   - Method: POST  
   - Response Data: "ok" with content-type text/plain  
   - Allow all origins  
   - This node will receive Trello webhook calls for card updates

5. **Create If node "Trello Action Type Check":**  
   - Condition: `$json.body.action.type` is "addMemberToCard" OR "removeMemberFromCard"  
   - Input from Webhook node's main output

6. **Create Supabase node "Update card due-date":**  
   - Operation: Update `cards` table  
   - Filter: `id` equals `$json.body.action.data.card.id`  
   - Fields to update:  
     - `due_date`: Format date from `$json.body.model.badges.due`  
     - `completed`: `$json.body.model.badges.dueComplete`  
     - `url`: `$json.body.model.url`  
   - Connect from True branch of Trello Action Type Check node

7. **Create Switch node "Add/Remove Member from Card Switch":**  
   - Conditions:  
     - Case 1: `$json.body.action.type` equals "addMemberToCard"  
     - Case 2: `$json.body.action.type` equals "removeMemberFromCard"  
   - Connect input from Trello Action Type Check node

8. **Create Supabase node "Get user row":**  
   - Operation: Get from `users` table  
   - Filter: `id` equals `$json.body.action.data.member.id`  
   - Connect input from both Switch node outputs (add and remove member)

9. **Create If node "Does User Exist?":**  
   - Condition: Check if `$json.id` is not empty  
   - Input from "Get user row"

10. **Create Supabase node "Create user":**  
    - Operation: Insert into `users` table  
    - Fields:  
      - `id`: `$json.body.action.data.member.id`  
      - `name`: `$json.body.action.data.member.name`  
    - Connect input from False branch of "Does User Exist?"

11. **Create Code node "Code in JavaScript1":**  
    - JavaScript code:  
      ```js
      const cardId = $('Webhook').first().json.body.action.data.card.id;
      const userIds = $('Webhook').first().json.body.model.idMembers;
      return userIds.map(userId => ({ json: { cardId, userId } }));
      ```
    - Connect input from True branch of "Does User Exist?" and from "Create user"

12. **Create Supabase node "Create user card relation":**  
    - Operation: Insert into `card_user` table  
    - Fields:  
      - `cardId`: `{{$json.cardId}}`  
      - `userId`: `{{$json.userId}}`  
    - Connect input from "Code in JavaScript1"

13. **Create Supabase node "Delete from users-card":**  
    - Operation: Delete from `card_user` table  
    - Filters:  
      - `userId` equals `$json.body.action.data.member.id`  
      - `cardId` equals `$json.body.action.data.card.id`  
    - Connect input from removeMemberFromCard branch of Switch node

14. **Create Set node "username":**  
    - Set JSON output mode  
    - Map Trello member names to Telegram usernames using a lookup dictionary with fallback  
    - Connect input from Trello Action Type Check node

15. **Create Code node "Code in JavaScript":**  
    - JavaScript code to build personalized Telegram message based on action type, escapes Markdown V2, includes card info and usernames  
    - Connect input from "username" node

16. **Create Telegram node "Send a text message":**  
    - Use your Telegram bot credentials  
    - Chat ID: Your Telegram chat ID  
    - Text: Use `{{$json.finalMessage}}` plus card details with Markdown V2 enabled  
    - Connect input from "Code in JavaScript"

17. **Create Schedule Trigger node "Due-Date Notification Schedule Trigger":**  
    - Set interval to 12 hours (twice daily)

18. **Create Supabase node "Get all cards due today":**  
    - Operation: Get all from `cards` table  
    - Filter: `due_date` equals today's date formatted `new Date().toLocaleDateString('en-US')`  
    - Connect input from schedule trigger

19. **Create Supabase node "Get user-card":**  
    - Operation: Get from `card_user` table  
    - Filter: `cardId` equals card ID from previous node  
    - Connect input from "Get all cards due today"

20. **Create Supabase node "Get users in card":**  
    - Operation: Get from `users` table  
    - Filter: `id` equals userId from previous node  
    - Connect input from "Get user-card"

21. **Create Set node "usernames 1":**  
    - Map Trello usernames to Telegram usernames, similar to the earlier "username" node  
    - Connect input from "Get users in card"

22. **Create Code node "Code in JavaScript2":**  
    - Build Telegram due date reminder message with card and user details  
    - Connect input from "usernames 1"

23. **Create Telegram node "Send a text message1":**  
    - Send due date reminder message  
    - Use Telegram credentials and chat ID  
    - Connect input from "Code in JavaScript2"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Replace `[BOARD_ID]` with your Trello board Id. To find it, open a card, append `.json` to the card URL, and extract the `"idList"` field.                                                                                                                                                                            | Sticky Note2                                                                                                                     |
| Replace `[TELEGRAM_CHAT_ID]` with your Telegram chat id where the bot has access. If your chat has multiple topics, optionally specify `[THREAD_ID]`.                                                                                                                                                                  | Sticky Note3                                                                                                                     |
| Map Trello usernames to Telegram usernames in the "username" and "usernames 1" Set nodes to address users properly in Telegram messages.                                                                                                                                                                              | Sticky Note4                                                                                                                     |
| Replace `[TRELLO_TOKEN]` and `[TRELLO_API_KEY]` with your Trello API credentials from Atlassian Developer Dashboard. Replace `[WEBHOOK_URL]` with the URL of the Webhook node in n8n for Trello to send updates.                                                                                                          | Sticky Note5                                                                                                                     |
| Replace `[WEBHOOK_PATH]` with your chosen webhook path in the Webhook node configuration.                                                                                                                                                                                                                               | Sticky Note6                                                                                                                     |
| Workflow automates Trello → Supabase → Telegram integration for card and user sync, plus notifications.                                                                                                                                                                                                                 | Sticky Note (Workflow Overview)                                                                                                 |
| For webhook setup instructions, refer to Trello official documentation on webhooks: https://developer.atlassian.com/cloud/trello/guides/rest-api/webhooks                                                                                                                                                             | Sticky Note1                                                                                                                    |
| Testing tip: Create a test card in Trello, add a member, set due date, and verify Supabase records and Telegram notifications are correctly created and sent.                                                                                                                                                            | Sticky Note1                                                                                                                    |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow built with n8n, respecting all applicable content policies and containing no illegal or protected data. All processed data is public and lawful.