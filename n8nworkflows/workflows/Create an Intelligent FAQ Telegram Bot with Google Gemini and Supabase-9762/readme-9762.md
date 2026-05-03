Create an Intelligent FAQ Telegram Bot with Google Gemini and Supabase

https://n8nworkflows.xyz/workflows/create-an-intelligent-faq-telegram-bot-with-google-gemini-and-supabase-9762


# Create an Intelligent FAQ Telegram Bot with Google Gemini and Supabase

### 1. Workflow Overview

This n8n workflow implements an intelligent FAQ Telegram bot using Google Gemini (an AI language model) and Supabase as a backend database. It is designed to interact with users on Telegram, identify whether they are new or existing users, and provide automated answers to frequently asked questions (FAQs) based on a predefined knowledge base.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Captures incoming messages from Telegram users via the Telegram Trigger node.
- **1.2 User Verification:** Checks if the user exists in the Supabase database and routes to either a new user onboarding path or an existing user path.
- **1.3 AI Processing:** For existing users, loads the FAQ context and uses Google Gemini to generate an AI-based answer to the user's question.
- **1.4 Response Delivery:** Sends the AI-generated answer or a welcome message (for new users) back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This initial block listens for incoming messages sent to the Telegram bot and triggers the workflow.
- **Nodes Involved:**  
  - Telegram Trigger  
  - Sticky Note (Step 1 explanation)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Receives message updates from Telegram users to start the workflow.  
    - Configuration: Listens to the `message` update type; no additional filters applied.  
    - Expressions: Reads message JSON payload, including `chat.id` and `message.text`.  
    - Input: Webhook event from Telegram.  
    - Output: JSON containing the user message and metadata.  
    - Edge Cases: Potential webhook delivery failure, malformed messages, or unsupported message types.  
    - Version: v1.2

  - **Sticky Note**  
    - Role: Documentation note explaining that this is the entry point receiving user messages.

---

#### 2.2 User Verification

- **Overview:** This block checks if the incoming Telegram user is already registered in the Supabase database and routes the workflow accordingly.
- **Nodes Involved:**  
  - Check if User Exists in Supabase  
  - Route: New or Existing User? (If node)  
  - Save New User to Supabase  
  - Send Welcome Message (New User)  
  - Sticky Notes (Step 2 and New User Path explanations)

- **Node Details:**

  - **Check if User Exists in Supabase**  
    - Type: Supabase node  
    - Role: Queries the `telegram_users` table filtering by the Telegram user’s `chat_id`.  
    - Configuration: Operation `get` with filter condition `chat_id = incoming Telegram chat id`.  
    - Credentials: Supabase API credentials required.  
    - Input: Telegram Trigger output.  
    - Output: Returns user record if found, empty if not.  
    - Edge Cases: Database connection failure, missing table, or incorrect permissions.  
    - Version: v1

  - **Route: New or Existing User?**  
    - Type: If node  
    - Role: Checks if the Supabase query returned a user or not by testing if the user id is empty.  
    - Configuration: Checks if `telegram_users` record is empty (user not found).  
    - Input: Output from Supabase node.  
    - Output: Two branches - True (new user), False (existing user).  
    - Edge Cases: Expression evaluation failures or unexpected data formats.  
    - Version: v2.2

  - **Save New User to Supabase**  
    - Type: Supabase node  
    - Role: Inserts a new record for the Telegram user into the `telegram_users` table.  
    - Configuration: Inserts the `chat_id` from Telegram message.  
    - Credentials: Supabase API credentials.  
    - Input: Triggered on "new user" branch.  
    - Output: Confirmation of insert operation.  
    - Edge Cases: Insert failures due to duplicate keys or permission issues.  
    - Version: v1

  - **Send Welcome Message (New User)**  
    - Type: Telegram node  
    - Role: Sends a one-time welcome message to the newly registered user on Telegram.  
    - Configuration: Static welcome text with FAQ link commands.  
    - Chat ID: Uses Telegram sender `from.id`.  
    - Credentials: Telegram API credentials required.  
    - Input: Output from Supabase insert node.  
    - Output: Confirmation of message sent.  
    - Edge Cases: Telegram API errors, invalid chat ID, network issues.  
    - Version: v1.2

  - **Sticky Notes**  
    - Explain the logic of checking user existence and onboarding new users.

---

#### 2.3 AI Processing

- **Overview:** For existing users, this block loads a predefined FAQ context and uses Google Gemini AI to process the user’s question and generate an appropriate answer.
- **Nodes Involved:**  
  - Load FAQ Context for AI (Set node)  
  - Google Gemini Chat Model  
  - Process Question with Gemini (Langchain Agent)  
  - Sticky Note (Step 3 explanation)

- **Node Details:**

  - **Load FAQ Context for AI**  
    - Type: Set node  
    - Role: Defines a static FAQ context string containing common questions and answers.  
    - Configuration: Assigns `faq_context` variable with multi-question FAQ text.  
    - Input: Triggered for existing users.  
    - Output: JSON with `faq_context` field.  
    - Edge Cases: None substantial as data is static; potential expression errors if misconfigured.  
    - Version: v3.4

  - **Google Gemini Chat Model**  
    - Type: Google Gemini language model node (Langchain)  
    - Role: Provides AI language model capabilities used by the Agent node.  
    - Configuration: Default options; no custom parameters set.  
    - Input: Connected as AI language model for the Agent node.  
    - Output: Language model outputs for AI agent processing.  
    - Credentials: Requires valid Google Gemini API credentials (not shown explicitly).  
    - Edge Cases: API auth errors, rate limits, or latency.  
    - Version: v1

  - **Process Question with Gemini**  
    - Type: Langchain Agent node  
    - Role: Core AI logic that takes user question and FAQ context to generate an answer.  
    - Configuration:  
      - Prompt template includes instructions to answer based on FAQ context.  
      - Handles special user queries like listing all FAQ questions.  
      - Injects `faq_context` and user question dynamically via expressions.  
    - Input: Takes `faq_context` and user question text from Telegram Trigger.  
    - Output: AI-generated response text in `output` field.  
    - AI Language Model: Uses Google Gemini node as backend.  
    - Edge Cases: Model timeouts, malformed prompts, unexpected user inputs.  
    - Version: v2.2

  - **Sticky Note**  
    - Documents AI response generation step.

---

#### 2.4 Response Delivery

- **Overview:** Sends the AI-generated answer back to the user on Telegram.
- **Nodes Involved:**  
  - Send AI Answer to User (Telegram node)  
  - Sticky Note (Step 4 explanation)

- **Node Details:**

  - **Send AI Answer to User**  
    - Type: Telegram node  
    - Role: Sends the cleaned AI response text back to the Telegram user.  
    - Configuration:  
      - Message text cleans special markdown characters from AI output using a regex replace expression: `{{ $json.output.replace(/[*_`[\]()]/g, "") }}`  
      - Uses Telegram sender `from.id` as chat ID.  
    - Credentials: Telegram API credentials required.  
    - Input: Output from AI processing node.  
    - Output: Confirmation of message delivery.  
    - Edge Cases: Telegram API errors, invalid chat ID, network timeouts.  
    - Version: v1.2

  - **Sticky Note**  
    - Explains final step of sending AI answer.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                         | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                                      |
|------------------------------|----------------------------------|---------------------------------------|----------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger                 | Entry point receiving user messages   | Webhook                    | Check if User Exists in Supabase | ## STEP 1: RECEIVE USER MESSAGE This workflow starts when a user sends a message to the Telegram bot.           |
| Check if User Exists in Supabase | Supabase                       | Check if user is registered            | Telegram Trigger           | Route: New or Existing User?     | ## STEP 2: CHECK IF USER IS NEW Looks up the user's Telegram ID in the database (Supabase).                     |
| Route: New or Existing User? | If                              | Route workflow based on user existence | Check if User Exists in Supabase | Save New User to Supabase, Load FAQ Context for AI | ## STEP 2: CHECK IF USER IS NEW Looks up the user's Telegram ID in the database (Supabase).                     |
| Save New User to Supabase    | Supabase                       | Insert new user record                 | Route: New or Existing User? | Send Welcome Message (New User) | ## PATH FOR NEW USERS If the user is not found in the database, this path runs. It creates a new user record and sends a one-time welcome message. |
| Send Welcome Message (New User) | Telegram                      | Send welcome message to new user      | Save New User to Supabase  | -                               | ## PATH FOR NEW USERS If the user is not found in the database, this path runs. It creates a new user record and sends a one-time welcome message. |
| Load FAQ Context for AI      | Set                             | Define static FAQ context for AI      | Route: New or Existing User? | Process Question with Gemini    | -                                                                                                               |
| Google Gemini Chat Model     | Google Gemini Language Model     | AI language model backend              | -                          | Process Question with Gemini     | ## PATH FOR EXISTING USERS If the user already exists, their question is processed by the AI to find an answer. |
| Process Question with Gemini | Langchain Agent                 | Generate AI answer based on FAQ       | Load FAQ Context for AI, Google Gemini Chat Model | Send AI Answer to User          | ## STEP 3: GENERATE AI ANSWER The user's question is sent to the Google Gemini model. The AI Agent uses its knowledge base to generate a relevant answer for the FAQ. |
| Send AI Answer to User       | Telegram                        | Send AI-generated answer to Telegram  | Process Question with Gemini | -                               | ## STEP 4: SEND AI RESPONSE The final answer generated by the AI is sent back to the user via Telegram.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen for `message` updates only.  
   - Credential: Connect to your Telegram bot API credentials.  
   - Position: Entry node.

2. **Add Supabase node to check user existence:**  
   - Type: Supabase  
   - Operation: `get`  
   - Table: `telegram_users`  
   - Filters: Condition where `chat_id` equals `{{$json.message.chat.id}}` from Telegram Trigger.  
   - Credential: Use valid Supabase API credentials.

3. **Add If node to route based on user existence:**  
   - Type: If (version 2.2 recommended)  
   - Condition: Check if the Supabase query returned empty (no user found).  
   - Expression: Use expression to detect if user record is empty or missing.

4. **Create Supabase node to save new user:**  
   - Type: Supabase  
   - Operation: `insert`  
   - Table: `telegram_users`  
   - Fields: `chat_id` with value from Telegram message `{{$json.message.chat.id}}`.  
   - Credential: Supabase API credentials.

5. **Add Telegram node to send welcome message:**  
   - Type: Telegram  
   - Text: Custom welcome message with FAQ commands included:  
     `"Thank you for trying my project. You can also check out the FAQ or ask any questions you may have. If you want to see the FAQ here, just click this link |/Who_are_you?/What_can_you_do?/Who_created_you?/Are_you_free?"`  
   - Chat ID: Use `{{$json.message.from.id}}` from Telegram Trigger.  
   - Credential: Telegram API credentials.

6. **Add Set node to define FAQ context:**  
   - Type: Set  
   - Variable: `faq_context`  
   - Value: Multi-question FAQ string:  
     ```
     Q: Who are you? A: I am an automated assistant bot created to learn n8n and AI.  
     Q: What can you do? A: Currently, I can answer basic questions about myself. In the future, I will be developed to do other things.  
     Q: Who created you? A: I was created by the owner of this n8n server using Google Gemini and n8n.  
     Q: Are you free? A: Yes, I am a learning project and free to use.
     ```
   - Triggered on existing user path.

7. **Add Google Gemini Chat Model node:**  
   - Type: Google Gemini (Langchain)  
   - Configuration: Default options.  
   - Credential: Provide valid Google Gemini API credentials.

8. **Add Langchain Agent node (Process Question with Gemini):**  
   - Type: Langchain Agent  
   - Text prompt:  
     ```
     Anda adalah asisten bot yang ramah dan membantu. Tugas Anda adalah menjawab pertanyaan pengguna berdasarkan Konteks FAQ yang diberikan. Jika pertanyaan pengguna tidak bisa dijawab dari konteks, jawab dengan benar pertanyaan tersebut
     Jika pengguna bertanya terkait "apa saja faq nya" kamu bisa memberi semua pertanyaan yang tertera di faq dengan mengawali pertanyaan dengan / dan hilangkan spasi yang terkandung dalam pertanyaan ganti lah dengan underscore"_"
     --- KONTEKS FAQ ---
     {{ $json.faq_context }}
     --- AKHIR KONTEKS ---

     Pertanyaan Pengguna: {{ $('Telegram Trigger').item.json.message.text }}
     ```
   - AI language model: Connect Google Gemini node here.

9. **Add Telegram node to send AI answer back:**  
   - Type: Telegram  
   - Text: Use expression to clean markdown from AI output:  
     `{{ $json.output.replace(/[*_`[\]()]/g, "") }}`  
   - Chat ID: `{{$json.message.from.id}}` from Telegram Trigger.  
   - Credential: Telegram API credentials.

10. **Connect nodes in order:**  
    - Telegram Trigger → Check if User Exists in Supabase → Route: New or Existing User?  
    - Route True (new user): Save New User to Supabase → Send Welcome Message (New User)  
    - Route False (existing user): Load FAQ Context for AI → Process Question with Gemini → Send AI Answer to User  
    - Google Gemini Chat Model connected as AI Language Model for Process Question with Gemini.

11. **Set up credentials:**  
    - Telegram API credentials for all Telegram nodes.  
    - Supabase API credentials for Supabase nodes.  
    - Google Gemini API credentials for AI nodes.

12. **Validate and test:**  
    - Test sending messages from Telegram to confirm user detection, welcome messages, and AI-generated answers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow is designed as a learning project, leveraging Google Gemini and n8n capabilities to create an intelligent Telegram FAQ bot. | Project description and context from workflow metadata.                                            |
| Welcome message includes clickable FAQ commands using Telegram command format (e.g., `/Who_are_you?`).                                 | Telegram command formatting to facilitate FAQ navigation.                                          |
| The AI prompt is in Indonesian language, providing friendly and context-aware answers based on the static FAQ context.                | Language and prompt design tailored for target users.                                              |
| Supabase is used as a lightweight user database to track Telegram chat IDs and enable conditional routing for onboarding.             | Supabase usage for user record management.                                                         |
| Google Gemini integration requires valid credentials and connection to Langchain nodes in n8n.                                         | See Google Gemini API documentation for credential setup and limits.                               |
| Regex expression in final Telegram send node cleans AI output to avoid markdown formatting issues in Telegram messages.               | `{{ $json.output.replace(/[*_`[\]()]/g, "") }}`                                                    |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It complies with all current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.