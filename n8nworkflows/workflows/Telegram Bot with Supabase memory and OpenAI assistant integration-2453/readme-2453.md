Telegram Bot with Supabase memory and OpenAI assistant integration

https://n8nworkflows.xyz/workflows/telegram-bot-with-supabase-memory-and-openai-assistant-integration-2453


# Telegram Bot with Supabase memory and OpenAI assistant integration

### 1. Workflow Overview

This workflow implements a Telegram chatbot integrated with OpenAI's assistant API and Supabase as a persistent memory store. It is designed to maintain conversational context and user-specific data, enabling human-like, context-aware interactions. The workflow architecture is divided into the following logical blocks:

- **1.1 Input Reception:** Listens for incoming Telegram messages.
- **1.2 User Validation & Memory Management:** Checks if the user exists in Supabase; if not, creates a new user record with a unique OpenAI conversation thread ID.
- **1.3 Conversation Handling with OpenAI:** Sends user messages to OpenAI's assistant, either continuing an existing conversation or creating a new thread.
- **1.4 OpenAI Assistant Run & Response Retrieval:** Executes the assistant run and retrieves the generated response.
- **1.5 Output Delivery:** Sends the OpenAI-generated reply back to the Telegram user.

This modular structure ensures the chatbot maintains user session states and conversation history, solving the common problem of stateless chatbot interactions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon receiving a new message from a Telegram user.

- **Nodes Involved:**  
  - `Get New Message`

- **Node Details:**  
  - **Node:** Get New Message  
    - Type: Telegram Trigger  
    - Role: Listens to Telegram for incoming messages (specifically "message" updates).  
    - Configuration: Uses Telegram API credentials to receive messages. No additional filters.  
    - Inputs: External webhook trigger from Telegram.  
    - Outputs: Emits the message JSON object containing user and chat details, and the message text.  
    - Edge Cases: Possible webhook connection failures or Telegram API downtime; malformed message payloads.  

---

#### 2.2 User Validation & Memory Management

- **Overview:**  
  This block verifies if the Telegram user already has a record in Supabase, representing an ongoing conversation thread. If not found, it creates a new user with a fresh OpenAI conversation thread.

- **Nodes Involved:**  
  - `Find User`  
  - `If User exists`  
  - `OPENAI - Create thread`  
  - `Create User`  
  - `Merge`

- **Node Details:**  
  - **Node:** Find User  
    - Type: Supabase  
    - Role: Queries the `telegram_users` table filtering by `telegram_id` to find existing user data.  
    - Configuration: Filter condition `telegram_id = <chat.id from incoming message>`. Retrieves all matching records.  
    - Inputs: Message JSON from `Get New Message`.  
    - Outputs: User data if exists, else empty.  
    - Edge Cases: Supabase connectivity issues, query errors, or empty results.  
  - **Node:** If User exists  
    - Type: If  
    - Role: Branches workflow based on whether `Find User` returned a user record (checks existence of `id` field).  
    - Configuration: Condition checks if `id` field exists in JSON.  
    - Inputs: Output from `Find User`.  
    - Outputs: True branch if user exists; False branch if user doesn't exist.  
    - Edge Cases: Expression evaluation errors if JSON structure changes.  
  - **Node:** OPENAI - Create thread  
    - Type: HTTP Request  
    - Role: Creates a new conversation thread in OpenAI for a new user.  
    - Configuration: POST to `https://api.openai.com/v1/threads` with header `OpenAI-Beta: assistants=v2`. Uses OpenAI credential.  
    - Inputs: None (triggered on new user branch).  
    - Outputs: Thread metadata including `id` (thread ID).  
    - Edge Cases: OpenAI API errors, authentication issues, API rate limits.  
  - **Node:** Create User  
    - Type: Supabase  
    - Role: Inserts a new user record into `telegram_users` with `telegram_id` and associated `openai_thread_id` from the created thread.  
    - Configuration: Fields set as `telegram_id` from incoming message, `openai_thread_id` from OpenAI thread creation node.  
    - Inputs: OpenAI thread creation output.  
    - Outputs: Confirmation of insertion.  
    - Edge Cases: Supabase write failures, invalid data, concurrency issues.  
  - **Node:** Merge  
    - Type: Merge  
    - Role: Combines user data from existing user branch or newly created user branch into a single data stream for downstream processing.  
    - Configuration: Default mode (pass through both inputs).  
    - Inputs: True branch from `If User exists` and output from `Create User`.  
    - Outputs: Unified user record JSON including `openai_thread_id`.  
    - Edge Cases: Data mismatch or missing fields if upstream nodes fail.

---

#### 2.3 Conversation Handling with OpenAI

- **Overview:**  
  This block sends the user's Telegram message to the OpenAI conversation thread, maintaining context.

- **Nodes Involved:**  
  - `OPENAI - Send message`

- **Node Details:**  
  - **Node:** OPENAI - Send message  
    - Type: HTTP Request  
    - Role: Posts the user's message text to the OpenAI conversation thread identified by `openai_thread_id`.  
    - Configuration:  
      - POST to `https://api.openai.com/v1/threads/{{openai_thread_id}}/messages`  
      - Body includes role `"user"` and content from Telegram message text.  
      - Header includes `OpenAI-Beta: assistants=v2` and uses OpenAI credentials.  
    - Inputs: Unified user data from `Merge`, Telegram message text.  
    - Outputs: Confirmation of message posting and message metadata.  
    - Edge Cases: Thread ID missing or invalid, API errors, malformed message content.

---

#### 2.4 OpenAI Assistant Run & Response Retrieval

- **Overview:**  
  Runs the assistant on the OpenAI thread to generate a reply, then fetches the latest messages from the thread.

- **Nodes Involved:**  
  - `OPENAI - Run assistant`  
  - `OPENAI - Get messages`

- **Node Details:**  
  - **Node:** OPENAI - Run assistant  
    - Type: HTTP Request  
    - Role: Triggers the assistant to process the conversation thread and generate a response.  
    - Configuration:  
      - POST to `https://api.openai.com/v1/threads/{{openai_thread_id}}/runs`  
      - Body includes `assistant_id` (custom assistant ID) and `stream` set to true.  
      - Header includes `OpenAI-Beta: assistants=v2`.  
      - Uses OpenAI API credentials.  
    - Inputs: Unified user data from `Merge`.  
    - Outputs: Run metadata and status updates.  
    - Edge Cases: Invalid assistant ID, API throttling, network issues.  
  - **Node:** OPENAI - Get messages  
    - Type: HTTP Request  
    - Role: Retrieves the latest messages from the OpenAI conversation thread after the assistant run completes.  
    - Configuration:  
      - GET `https://api.openai.com/v1/threads/{{openai_thread_id}}/messages`  
      - Headers include `OpenAI-Beta: assistants=v2`.  
      - Uses OpenAI credentials.  
    - Inputs: Output from `OPENAI - Run assistant`.  
    - Outputs: List of messages including the assistant's reply.  
    - Edge Cases: Empty message list, API errors, thread ID issues.

---

#### 2.5 Output Delivery

- **Overview:**  
  Sends the assistant's response text back to the Telegram user.

- **Nodes Involved:**  
  - `Send Message to User`

- **Node Details:**  
  - **Node:** Send Message to User  
    - Type: Telegram  
    - Role: Posts the assistant's response back to the Telegram chat.  
    - Configuration:  
      - Uses Telegram API credentials.  
      - Text set from the first message content's text value in the `OPENAI - Get messages` node output.  
      - Sends to the same `chat.id` from the incoming message.  
      - Attribution disabled for cleaner messages.  
    - Inputs: Assistant response messages and original Telegram message context.  
    - Outputs: Confirmation of message sent.  
    - Edge Cases: Telegram API rate limits, invalid chat ID, network failures.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                          |
|-----------------------|---------------------|----------------------------------------|-------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------|
| Get New Message       | Telegram Trigger     | Receive incoming Telegram messages     | External webhook         | Find User                  | Create own Telegram bot in [Botfather bot](https://t.me/botfather)                                                   |
| Find User             | Supabase            | Query user by telegram_id               | Get New Message          | If User exists             | Create table in [Supabase](https://supabase.com) with SQL query                                                      |
| If User exists        | If                  | Branch on user existence                 | Find User                | Merge (true), OPENAI - Create thread (false) |                                                                                                                      |
| OPENAI - Create thread| HTTP Request        | Create new OpenAI conversation thread  | If User exists (false)   | Create User                | Create assistant in [OpenAI](https://platform.openai.com/assistants). Specify own assistant id here                  |
| Create User           | Supabase            | Insert new user record with thread ID  | OPENAI - Create thread   | Merge                      | Create table in [Supabase](https://supabase.com) with SQL query                                                      |
| Merge                 | Merge               | Combine user data for conversation     | If User exists (true), Create User | OPENAI - Send message      |                                                                                                                      |
| OPENAI - Send message | HTTP Request        | Send user message to OpenAI thread     | Merge                    | OPENAI - Run assistant     | Create assistant in [OpenAI](https://platform.openai.com/assistants). Specify own assistant id here                  |
| OPENAI - Run assistant| HTTP Request        | Run OpenAI assistant to generate reply | OPENAI - Send message    | OPENAI - Get messages      | Create assistant in [OpenAI](https://platform.openai.com/assistants). Specify own assistant id here                  |
| OPENAI - Get messages | HTTP Request        | Retrieve OpenAI assistant's reply      | OPENAI - Run assistant   | Send Message to User        | Create assistant in [OpenAI](https://platform.openai.com/assistants). Specify own assistant id here                  |
| Send Message to User  | Telegram             | Send AI-generated reply to Telegram    | OPENAI - Get messages, Get New Message | None                      |                                                                                                                      |
| Sticky Note1          | Sticky Note         | SQL table creation query for Supabase  | None                     | None                       | SQL query to create table in Supabase: `create table public.telegram_users...`                                      |
| Sticky Note2          | Sticky Note         | Assistant creation instructions         | None                     | None                       | Create assistant in [OpenAI](https://platform.openai.com/assistants). Specify own assistant id here                  |
| Sticky Note3          | Sticky Note         | Telegram bot creation instructions      | None                     | None                       | Create own Telegram bot in [Botfather bot](https://t.me/botfather)                                                   |
| Sticky Note4          | Sticky Note         | Supabase table creation note             | None                     | None                       | Create table in [Supabase](https://supabase.com) with SQL query                                                      |
| Sticky Note5          | Sticky Note         | Setup steps summary                      | None                     | None                       | Setup steps for Telegram bot, Supabase, OpenAI, and n8n environment configuration                                     |
| Sticky Note6          | Sticky Note         | Branding and overview                    | None                     | None                       | AI Telegram Bot with Supabase memory, made by Mark Shcherbakov, with community link [5minAI](https://www.skool.com/5minai-2861) |
| Sticky Note7          | Sticky Note         | Video setup guide link                   | None                     | None                       | ... or watch setup video [5 min]: https://www.youtube.com/watch?v=kS41gut8l0g                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot:**  
   - Use [Botfather](https://t.me/botfather) to create a new bot and obtain the token.

2. **Set up Supabase:**  
   - Create a new Supabase project and get `SUPABASE_URL` and `SUPABASE_KEY`.  
   - Run this SQL query to create the `telegram_users` table:
   ```sql
   create table public.telegram_users (
     id uuid not null default gen_random_uuid (),
     date_created timestamp with time zone not null default (now() at time zone 'utc'::text),
     telegram_id bigint null,
     openai_thread_id text null,
     constraint telegram_users_pkey primary key (id)
   ) tablespace pg_default;
   ```

3. **Create OpenAI Assistant:**  
   - Create a new assistant via [OpenAI Assistants](https://platform.openai.com/assistants).  
   - Note the `assistant_id` for use in the workflow nodes.

4. **In n8n, create the following nodes and configure credentials:**

   - **Node 1: Get New Message**  
     - Type: Telegram Trigger  
     - Parameters: Listen to "message" updates only.  
     - Credentials: Use Telegram Bot credentials.  
   
   - **Node 2: Find User**  
     - Type: Supabase  
     - Parameters:  
       - Table: `telegram_users`  
       - Operation: Get all records  
       - Filter: `telegram_id = {{$json["message"]["chat"]["id"]}}`  
     - Credentials: Supabase credentials with `SUPABASE_URL` and `SUPABASE_KEY`.  
   
   - **Node 3: If User exists**  
     - Type: If  
     - Condition: Check if `id` exists in the output JSON from `Find User`.  
   
   - **Node 4: OPENAI - Create thread**  
     - Type: HTTP Request  
     - Parameters:  
       - URL: `https://api.openai.com/v1/threads`  
       - Method: POST  
       - Headers: Include `OpenAI-Beta: assistants=v2`  
       - Authentication: OpenAI API credentials  
     - Trigger: Connected from false branch of `If User exists`.  
   
   - **Node 5: Create User**  
     - Type: Supabase  
     - Parameters:  
       - Table: `telegram_users`  
       - Operation: Insert  
       - Fields:  
         - `telegram_id`: `={{ $json["message"]["chat"]["id"] }}`  
         - `openai_thread_id`: `={{ $json["id"] }}` (from thread creation node)  
     - Credentials: Supabase credentials.  
   
   - **Node 6: Merge**  
     - Type: Merge  
     - Parameters: Default (pass through multiple inputs)  
     - Inputs: True branch from `If User exists` and output of `Create User`.  
   
   - **Node 7: OPENAI - Send message**  
     - Type: HTTP Request  
     - Parameters:  
       - URL: `https://api.openai.com/v1/threads/{{$json["openai_thread_id"]}}/messages`  
       - Method: POST  
       - Body: JSON with `"role": "user"`, `"content": {{$json["message"]["text"]}}`  
       - Headers: `OpenAI-Beta: assistants=v2`  
       - Authentication: OpenAI API credentials.  
     - Input: Output from `Merge`.  
   
   - **Node 8: OPENAI - Run assistant**  
     - Type: HTTP Request  
     - Parameters:  
       - URL: `https://api.openai.com/v1/threads/{{$json["openai_thread_id"]}}/runs`  
       - Method: POST  
       - Body: JSON with `"assistant_id": "YOUR_ASSISTANT_ID"`, `"stream": true`  
       - Headers: `OpenAI-Beta: assistants=v2`  
       - Authentication: OpenAI API credentials.  
     - Input: Output from `OPENAI - Send message`.  
   
   - **Node 9: OPENAI - Get messages**  
     - Type: HTTP Request  
     - Parameters:  
       - URL: `https://api.openai.com/v1/threads/{{$json["openai_thread_id"]}}/messages`  
       - Method: GET  
       - Headers: `OpenAI-Beta: assistants=v2`  
       - Authentication: OpenAI API credentials.  
     - Input: Output from `OPENAI - Run assistant`.  
   
   - **Node 10: Send Message to User**  
     - Type: Telegram  
     - Parameters:  
       - Chat ID: `={{ $json["message"]["chat"]["id"] }}`  
       - Text: `={{ $json["data"][0]["content"][0]["text"]["value"] }}` (extract the assistant reply)  
       - Disable Attribution.  
     - Credentials: Telegram Bot credentials.  
     - Input: Output from `OPENAI - Get messages`.  

5. **Connect nodes in this order:**

   - `Get New Message` → `Find User` → `If User exists`  
     - True branch → `Merge` (input 1) → `OPENAI - Send message` → `OPENAI - Run assistant` → `OPENAI - Get messages` → `Send Message to User`  
     - False branch → `OPENAI - Create thread` → `Create User` → `Merge` (input 2)  

6. **Environment variables and credentials setup:**  
   - Telegram API credentials (bot token).  
   - Supabase API credentials (`SUPABASE_URL` and `SUPABASE_KEY`).  
   - OpenAI API credentials with access to assistant API.  
   - Replace `"YOUR_ASSISTANT_ID"` in `OPENAI - Run assistant` node with your actual assistant ID.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Detailed video guide for building the AI Telegram bot from scratch, including setup and walkthrough.                                                        | [YouTube Video](https://www.youtube.com/watch?v=QrZxuWgFqBI)                                        |
| Create your Telegram bot using Botfather.                                                                                                                   | [Botfather Telegram Bot](https://t.me/botfather)                                                    |
| Create and configure your Supabase project and database table to store user info and conversation thread IDs.                                              | [Supabase](https://supabase.com)                                                                     |
| Create OpenAI assistant in the OpenAI platform and obtain your assistant ID for use in the workflow.                                                        | [OpenAI Assistants](https://platform.openai.com/assistants)                                        |
| Author and community credit: Mark Shcherbakov from 5minAI community.                                                                                        | [Mark Shcherbakov LinkedIn](https://www.linkedin.com/in/marklowcoding/), [5minAI Community](https://www.skool.com/5minai-2861) |
| Setup summary for Telegram Bot, Supabase, OpenAI API keys, and n8n environment with credentials and triggers.                                                | See Sticky Note5 content in workflow                                                               |
| Video setup alternative: short 5-minute video walkthrough available at:                                                                                     | [YouTube Setup Video](https://www.youtube.com/watch?v=kS41gut8l0g)                                  |

---

This document provides a detailed, node-by-node breakdown and procedural instructions for reproducing and maintaining the Telegram bot workflow with OpenAI and Supabase integrations. It anticipates common failure points such as API connectivity, credential misconfiguration, and data consistency issues, enabling efficient troubleshooting and customization.