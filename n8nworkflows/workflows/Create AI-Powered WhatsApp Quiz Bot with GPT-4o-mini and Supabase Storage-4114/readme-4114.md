Create AI-Powered WhatsApp Quiz Bot with GPT-4o-mini and Supabase Storage

https://n8nworkflows.xyz/workflows/create-ai-powered-whatsapp-quiz-bot-with-gpt-4o-mini-and-supabase-storage-4114


# Create AI-Powered WhatsApp Quiz Bot with GPT-4o-mini and Supabase Storage

### 1. Workflow Overview

This workflow implements an AI-powered WhatsApp Quiz Bot designed to enable users to study specific topics interactively via WhatsApp. It leverages GPT-4o-mini for quiz generation and Supabase as persistent storage for user data. The core purpose is to provide an engaging, personalized quiz experience by remembering user names and study topics, prompting for missing data, storing updates, and finally generating and sending quizzes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and User Data Retrieval**  
  Receives incoming WhatsApp messages, queries Supabase to fetch stored user information (name and study topic).

- **1.2 User Data Validation and Acquisition**  
  Checks if user name and quiz topic exist; if missing, prompts the user accordingly via WhatsApp and updates Supabase.

- **1.3 Data Merging and Preparation for AI Processing**  
  Merges collected user data (name, study topic) from initial fetch and any updates from prompts to form a complete context.

- **1.4 AI Quiz Generation**  
  Uses a LangChain AI Agent configured with a Portuguese Brazilian system prompt to generate a personalized quiz based on the user’s preferred topic and name.

- **1.5 Quiz Delivery**  
  Sends the generated quiz questions back to the user via WhatsApp.

- **1.6 Memory Management**  
  Maintains a short-term memory buffer window to provide context continuity for the AI agent during the conversation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and User Data Retrieval

- **Overview:**  
  Captures incoming WhatsApp messages via webhook and retrieves user data from Supabase to determine if name and study topic are already stored.

- **Nodes Involved:**  
  - Whatsapp Trigger  
  - Supabase: Fetch User Data

- **Node Details:**  
  - **Whatsapp Trigger**  
    - Type: Webhook Trigger  
    - Config: Listens for POST requests at a fixed path unique to this workflow instance.  
    - Inputs: External WhatsApp message POST  
    - Outputs: Forwarded webhook data to Supabase fetch node  
    - Edge Cases: Webhook misconfiguration, invalid payload format, HTTP errors.

  - **Supabase: Fetch User Data**  
    - Type: Supabase Database Select Operation  
    - Config: Queries user data table for the incoming user ID or phone number.  
    - Credentials: Supabase API key with read permissions.  
    - Inputs: WhatsApp Trigger output  
    - Outputs: User record data (including name and study topic)  
    - Edge Cases: Network or authentication errors, empty or multiple records returned, query timeout.

---

#### 1.2 User Data Validation and Acquisition

- **Overview:**  
  Checks for the presence of user name and quiz topic. If missing, prompts user via WhatsApp HTTP requests to provide this information and updates Supabase accordingly.

- **Nodes Involved:**  
  - User exist? (IF node)  
  - Quiz Topic Defined? (IF node)  
  - Ask For Name (HTTP Request)  
  - Supabase: Update User Name  
  - Ask For Study Topic (WhatsApp Message) (HTTP Request)  
  - Supabase: Update Study Topic

- **Node Details:**  

  - **User exist?**  
    - Type: IF node  
    - Config: Checks if user name exists (non-empty string).  
    - Inputs: Supabase fetch output  
    - Outputs: If yes, continues to check quiz topic; if no, sends prompt for name.  
    - Edge Cases: Misinterpretation of empty string, bad data format.

  - **Quiz Topic Defined?**  
    - Type: IF node  
    - Config: Checks if study topic exists (non-empty string).  
    - Inputs: Output from User exist? (true branch)  
    - Outputs: If yes, proceeds; if no, sends prompt for study topic.  
    - Edge Cases: Same as User exist?.

  - **Ask For Name**  
    - Type: HTTP Request (POST)  
    - Config: Sends a WhatsApp message asking the user their name. Message body: "Bem-vindo(a) ao nosso espaço de estudos! Para que eu possa te ajudar melhor, me conta, qual é o seu nome?"  
    - Inputs: User exist? false branch  
    - Outputs: Triggers Supabase update upon user reply (via next run)  
    - Edge Cases: WhatsApp API failures, message formatting errors.

  - **Supabase: Update User Name**  
    - Type: Supabase Update Operation  
    - Config: Updates the user name field in Supabase with the provided answer.  
    - Inputs: Output from Ask For Name response (implied)  
    - Edge Cases: Update conflicts, permission errors.

  - **Ask For Study Topic (WhatsApp Message)**  
    - Type: HTTP Request (POST)  
    - Config: Sends WhatsApp message prompting for study topic using the user’s name placeholder: "{{NOME_DO_USUARIO_AQUI}}, pensando em te ajudar nos estudos, qual matéria ou tópico específico prefere para o nosso quiz?"  
    - Inputs: Quiz Topic Defined? false branch  
    - Outputs: Triggers Supabase update on reply  
    - Edge Cases: Similar to Ask For Name.

  - **Supabase: Update Study Topic**  
    - Type: Supabase Update Operation  
    - Config: Updates study topic field with user's answer.  
    - Inputs: Output from Ask For Study Topic  
    - Outputs: Sends merged data downstream  
    - Edge Cases: Same as Update User Name.

---

#### 1.3 Data Merging and Preparation for AI Processing

- **Overview:**  
  Merges the original fetched user data and any updates from prompts into a consolidated data set for AI processing.

- **Nodes Involved:**  
  - Merge

- **Node Details:**  
  - **Merge**  
    - Type: Merge node  
    - Config: Merges inputs from either freshly updated user info or initially fetched data.  
    - Inputs:  
      - From Quiz Topic Defined? true branch or from Supabase: Update Study Topic  
      - From User exist? true branch or Supabase: Update User Name  
    - Outputs: Consolidated data object for AI agent  
    - Edge Cases: Conflicting data inputs, empty merge, ordering issues.

---

#### 1.4 AI Quiz Generation

- **Overview:**  
  Uses a LangChain AI Agent with a detailed Portuguese BR system prompt to generate an educational quiz on the user-defined topic, personalized with the user’s name. The AI agent maintains context via a memory buffer and uses GPT-4o-mini model.

- **Nodes Involved:**  
  - AI Agent - Portuguese BR System Msg  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**  

  - **AI Agent - Portuguese BR System Msg**  
    - Type: LangChain Agent node  
    - Config:  
      - System message sets AI personality as a friendly, expert quiz tutor in Brazilian Portuguese.  
      - Instructions include addressing user by name, focusing on provided quiz topic, generating 10 multiple-choice questions with answers, and providing empathetic feedback.  
      - Uses placeholders to dynamically inject user name and topic.  
    - Inputs: Merged user data  
    - Outputs: Generated quiz content  
    - Edge Cases: Expression errors in placeholders, incomplete context, API quota limits.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat LLM node  
    - Config: Uses GPT-4o-mini model for generation with default options.  
    - Credentials: OpenAI API key  
    - Inputs: AI Agent requests  
    - Outputs: AI agent responses  
    - Edge Cases: OpenAI service outages, authentication errors, rate limits.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Config: Maintains recent conversation context to provide continuity for AI agent responses.  
    - Inputs: AI Agent memory input  
    - Outputs: Updated context memory  
    - Edge Cases: Memory overflow, context loss on restarts.

---

#### 1.5 Quiz Delivery

- **Overview:**  
  Sends the AI-generated quiz questions back to the user through WhatsApp.

- **Nodes Involved:**  
  - Send Message to User (WhatsApp Message)

- **Node Details:**  
  - **Send Message to User (WhatsApp Message)**  
    - Type: HTTP Request (POST)  
    - Config: Sends the AI agent’s quiz response text to the user’s WhatsApp number.  
    - Inputs: AI Agent output  
    - Edge Cases: WhatsApp API limits, message formatting errors, delivery failures.

---

#### 1.6 Memory Management

- **Overview:**  
  Supports conversational context by buffering recent interactions for the AI agent.

- **Nodes Involved:**  
  - Simple Memory (also referenced in 1.4)

- **Details:**  
  See above in 1.4.

---

### 3. Summary Table

| Node Name                         | Node Type                                   | Functional Role                          | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                  |
|----------------------------------|---------------------------------------------|----------------------------------------|----------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------|
| Whatsapp Trigger                 | Webhook                                     | Receives incoming WhatsApp messages    | -                                | Supabase: Fetch User Data             | ## WhatsApp Quiz Bot<br>Let users study a specific topic via WhatsApp.<br>🧠 Fetch user name & topic<br>📚 Ask for missing info<br>📥 Save to Supabase<br>🤖 Generate quiz with AI<br>📲 Send back questions |
| Supabase: Fetch User Data        | Supabase Select                             | Retrieves user info from Supabase      | Whatsapp Trigger                 | User exist?                          | ## Flow Overview<br>Trigger: Incoming WhatsApp msg<br>Fetch user data (Supabase)<br>Check if name & topic exist<br>Ask missing info via WhatsApp<br>Update Supabase with answers<br>Merge inputs<br>AI Agent generates quiz<br>Send response to user |
| User exist?                     | IF Node                                    | Checks if user name exists              | Supabase: Fetch User Data        | Quiz Topic Defined?, Ask For Name    |                                                                                              |
| Quiz Topic Defined?              | IF Node                                    | Checks if quiz topic exists             | User exist?                     | Merge, Ask For Study Topic (WhatsApp Message) |                                                                                              |
| Ask For Name                    | HTTP Request                               | Sends WhatsApp message to ask name     | User exist? (false)             | Supabase: Update User Name            |                                                                                              |
| Supabase: Update User Name      | Supabase Update                            | Updates user name in Supabase           | Ask For Name                   | (none)                              |                                                                                              |
| Ask For Study Topic (WhatsApp Message) | HTTP Request                               | Sends WhatsApp message to ask topic    | Quiz Topic Defined? (false)    | Supabase: Update Study Topic          |                                                                                              |
| Supabase: Update Study Topic    | Supabase Update                            | Updates quiz topic in Supabase          | Ask For Study Topic             | Merge                               |                                                                                              |
| Merge                          | Merge Node                                | Merges user data inputs                  | Quiz Topic Defined? (true), Supabase: Update Study Topic | AI Agent - Portuguese BR System Msg |                                                                                              |
| AI Agent - Portuguese BR System Msg | LangChain Agent                           | Generates personalized quiz via AI      | Merge                          | Send Message to User (WhatsApp Message) |                                                                                              |
| OpenAI Chat Model               | LangChain OpenAI LLM                      | Underlying AI model for text generation | AI Agent - Portuguese BR System Msg | AI Agent - Portuguese BR System Msg (ai_languageModel) |                                                                                              |
| Simple Memory                  | LangChain Memory Buffer Window             | Maintains conversation context          | AI Agent - Portuguese BR System Msg | AI Agent - Portuguese BR System Msg (ai_memory) |                                                                                              |
| Send Message to User (WhatsApp Message) | HTTP Request                               | Sends AI-generated quiz to WhatsApp user | AI Agent - Portuguese BR System Msg | (none)                              |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: "Whatsapp Trigger"  
   - HTTP Method: POST  
   - Path: Generate unique path (e.g., UUID)  
   - Purpose: Receive WhatsApp incoming messages  

2. **Create Supabase Select Node**  
   - Type: Supabase  
   - Name: "Supabase: Fetch User Data"  
   - Operation: Select  
   - Configure Supabase credentials (API key with read permissions)  
   - Query: Fetch user by WhatsApp ID or phone number from incoming webhook data  

3. **Create IF Node for User Name Check**  
   - Type: IF  
   - Name: "User exist?"  
   - Condition: Check if user name field is non-empty in Supabase result  

4. **Create IF Node for Study Topic Check**  
   - Type: IF  
   - Name: "Quiz Topic Defined?"  
   - Condition: Check if study topic field is non-empty  

5. **Create HTTP Request Node to Ask for Name**  
   - Type: HTTP Request  
   - Name: "Ask For Name"  
   - Method: POST  
   - Body (JSON): "Bem-vindo(a) ao nosso espaço de estudos! Para que eu possa te ajudar melhor, me conta, qual é o seu nome?"  
   - Connect from "User exist?" false branch  

6. **Create Supabase Update Node to Save User Name**  
   - Type: Supabase  
   - Name: "Supabase: Update User Name"  
   - Operation: Update  
   - Credentials: Supabase with write permission  
   - Configure to update user name with data from user reply  
   - Connect from "Ask For Name" output  

7. **Create HTTP Request Node to Ask for Study Topic**  
   - Type: HTTP Request  
   - Name: "Ask For Study Topic (WhatsApp Message)"  
   - Method: POST  
   - Body (JSON): Use placeholder for user name: "{{NOME_DO_USUARIO_AQUI}}, pensando em te ajudar nos estudos, qual matéria ou tópico específico prefere para o nosso quiz?"  
   - Connect from "Quiz Topic Defined?" false branch  

8. **Create Supabase Update Node to Save Study Topic**  
   - Type: Supabase  
   - Name: "Supabase: Update Study Topic"  
   - Operation: Update  
   - Configure to update study topic with user reply  
   - Connect from "Ask For Study Topic (WhatsApp Message)" output  

9. **Create Merge Node**  
   - Type: Merge  
   - Name: "Merge"  
   - Merge Mode: Wait for all inputs  
   - Inputs:  
     - True branch from "Quiz Topic Defined?" (existing data)  
     - Output from "Supabase: Update Study Topic" (new topic)  
     - True branch from "User exist?" (existing user)  
     - Output from "Supabase: Update User Name" (new name)  

10. **Create LangChain Memory Buffer Node**  
    - Type: LangChain Memory Buffer Window  
    - Name: "Simple Memory"  
    - Default config (window size as desired)  

11. **Create LangChain OpenAI Chat Model Node**  
    - Type: LangChain OpenAI LLM  
    - Name: "OpenAI Chat Model"  
    - Model: gpt-4o-mini  
    - Credentials: OpenAI API key  

12. **Create LangChain Agent Node**  
    - Type: LangChain Agent  
    - Name: "AI Agent - Portuguese BR System Msg"  
    - System Message: Use the detailed prompt defining the AI as a friendly Brazilian Portuguese quiz expert with instructions for quiz generation and user engagement.  
    - Connect to "Merge" node output  
    - Link OpenAI Chat Model node as AI language model  
    - Link Simple Memory node as memory input  

13. **Create HTTP Request Node to Send Quiz Back**  
    - Type: HTTP Request  
    - Name: "Send Message to User (WhatsApp Message)"  
    - Method: POST  
    - Body: Pass AI Agent output message  
    - Connect from "AI Agent - Portuguese BR System Msg" output  

14. **Connect Flow**  
    - Whatsapp Trigger → Supabase: Fetch User Data → User exist?  
    - User exist? true → Quiz Topic Defined?  
    - User exist? false → Ask For Name → Supabase: Update User Name  
    - Quiz Topic Defined? true → Merge  
    - Quiz Topic Defined? false → Ask For Study Topic → Supabase: Update Study Topic  
    - Supabase: Update User Name and Supabase: Update Study Topic → Merge  
    - Merge → AI Agent - Portuguese BR System Msg → Send Message to User  

15. **Credential Setup**  
    - Setup and assign Supabase API credentials with read/write permissions  
    - Setup and assign OpenAI API credentials  
    - Configure WhatsApp API credentials or endpoint as required by HTTP Request nodes  

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| WhatsApp Quiz Bot lets users study a specific topic via WhatsApp by fetching user name and topic, asking for missing info, saving data, generating quizzes with AI, and sending questions back. | Workflow description and sticky notes in workflow.                                                         |
| AI Agent’s system prompt is carefully crafted for Brazilian Portuguese, focusing on empathy, education, quiz expertise, and personalized interactions. | Embedded in "AI Agent - Portuguese BR System Msg" node configuration.                                       |
| Supabase stores user data persistently, enabling personalized quizzes and remembering user preferences across sessions.                 | Supabase nodes configuration; requires proper database schema and API permissions.                         |
| GPT-4o-mini is used as the underlying large language model for quiz generation, balancing capability and cost.                            | OpenAI Chat Model node configuration.                                                                       |
| Workflow uses LangChain integrations in n8n for AI model and memory management, enabling conversational context retention.                | Nodes: Simple Memory, OpenAI Chat Model, AI Agent.                                                          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.