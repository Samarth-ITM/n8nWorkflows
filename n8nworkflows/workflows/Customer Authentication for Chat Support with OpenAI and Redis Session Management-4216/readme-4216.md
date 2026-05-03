Customer Authentication for Chat Support with OpenAI and Redis Session Management

https://n8nworkflows.xyz/workflows/customer-authentication-for-chat-support-with-openai-and-redis-session-management-4216


# Customer Authentication for Chat Support with OpenAI and Redis Session Management

### 1. Workflow Overview

This workflow implements a customer authentication mechanism for a chatbot powered by OpenAI (GPT-4.1-mini) with session management via Redis. It targets scenarios where users interact with a chat support agent anonymously (as guests) or as authenticated customers. Guests can request a login URL to authenticate, linking their chat session to their customer profile. The workflow is divided into three main logical blocks:

- **1.1 Chat Message Reception & Session Retrieval:** Listens for incoming chat messages, retrieves session data from Redis (if any), and passes it to the conversational agent.
- **1.2 Conversational Agent & AI Processing:** Processes messages through an AI agent that adapts responses based on whether the user is a guest or an authenticated customer. If authentication is needed, it uses a tool to generate a login URL.
- **1.3 Authentication Workflow (Login URL generation, Form Submission, Session Update):** Handles requests for authentication URLs via a sub-workflow, displays a login form, processes login submissions, updates Redis session with customer profile data, and confirms login success.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Message Reception & Session Retrieval

- **Overview:**  
  This block receives chat messages via a chat trigger node, retrieves the user’s session from Redis using their session ID, and prepares the context for the conversational AI.

- **Nodes Involved:**  
  - When chat message received  
  - Get Session

- **Node Details:**

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point for chat messages from users.  
    - Configuration: Default options, webhook enabled with unique webhook ID.  
    - Key Variables: Emits `sessionId` to identify user sessions.  
    - Input: External chat message webhook.  
    - Output: Passes sessionId downstream.  
    - Edge Cases: Missing or malformed sessionId might cause Redis retrieval to fail.

  - **Get Session**  
    - Type: `n8n-nodes-base.redis`  
    - Role: Retrieves the session data from Redis cache using the session ID from chat trigger.  
    - Configuration: Operation “get” on key `chat_{{ $json.sessionId }}`; looks for string key.  
    - Credentials: Redis account (localhost).  
    - Input: From chat trigger node.  
    - Output: Outputs session data if found (customer profile or empty for guest).  
    - Edge Cases: Redis connection errors, missing keys, or expired session data.

---

#### 2.2 Conversational Agent & AI Processing

- **Overview:**  
  This block runs the AI-powered customer support agent, using the session data as memory context. It handles guest vs. authenticated customer logic, including generating login URLs when needed.

- **Nodes Involved:**  
  - Memory  
  - LLM (OpenAI GPT-4.1-mini)  
  - Customer Support Agent  
  - Get Auth URL (Tool Workflow)

- **Node Details:**

  - **Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains conversational context per session for the AI agent.  
    - Configuration: Uses custom session key formatted as `chat_{{ sessionId }}` to group messages.  
    - Input: Receives sessionId from chat trigger node.  
    - Output: Provides memory context to the AI agent.  
    - Edge Cases: If session key is incorrect or missing, memory may be lost.

  - **LLM**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: OpenAI GPT-4.1-mini model to generate AI responses.  
    - Configuration: Model set to "gpt-4.1-mini", default options.  
    - Credentials: OpenAI API key configured.  
    - Input: Receives prompts and memory from agent node.  
    - Output: AI-generated response for chat.  
    - Edge Cases: API quota exceeded, network timeouts, invalid API key.

  - **Customer Support Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Orchestrates conversation logic, distinguishes guest vs. customer, prompts authentication when needed.  
    - Configuration:  
      - System message instructs agent on handling guests/customers, login URL usage, and session context.  
      - Uses passthrough for binary images.  
      - Input text is current chat message (`chatInput`).  
    - Input: Chat message, session data (customer profile or empty).  
    - Output: AI response, triggers tool to generate login URL when required.  
    - Edge Cases: Misidentification of user status, failure to generate URL tool call.

  - **Get Auth URL**  
    - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
    - Role: Sub-workflow tool to generate login URL for customer authentication form.  
    - Configuration: Calls the same workflow by ID, passing `eventType: "get_auth_url"` and current sessionId as inputs.  
    - Input: Triggered by agent when login URL needed.  
    - Output: URL string with appended sessionId query parameter.  
    - Edge Cases: Circular workflow calls, improper sessionId, failure in sub-workflow.

---

#### 2.3 Authentication Workflow (Login URL Generation, Form Submission, Session Update)

- **Overview:**  
  This block handles generation of login URLs, user login form submission, customer profile assignment to the session, and confirmation of successful login.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Switch  
  - Get Auth Url Response  
  - Login Form  
  - Replace Me!  
  - Get Customer Profile  
  - Update Session  
  - Confirm Login Success

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: `n8n-nodes-base.executeWorkflowTrigger`  
    - Role: Entry point for sub-workflow calls (e.g., login URL generation).  
    - Configuration: Accepts inputs `eventType` (string) and `data` (object).  
    - Input: Sub-workflow invocations from tool node.  
    - Output: Routes flow based on eventType.  
    - Edge Cases: Missing inputs or unexpected eventType values.

  - **Switch**  
    - Type: `n8n-nodes-base.switch`  
    - Role: Routes workflow based on `eventType`.  
    - Configuration: Checks if `eventType` equals `"get_auth_url"`.  
    - Input: From Execute Workflow Trigger node.  
    - Output: To `Get Auth Url Response` if true.  
    - Edge Cases: Unknown eventType leads to no output.

  - **Get Auth Url Response**  
    - Type: `n8n-nodes-base.set`  
    - Role: Constructs the authentication URL string with sessionId query parameter.  
    - Configuration: Sets `auth_url` to `https://<your-n8n-url>/form/auth?sessionId={{ $json.data.sessionId }}`.  
    - Input: From Switch node.  
    - Output: Returns the URL to the calling tool.  
    - Edge Cases: Placeholder URL must be replaced before production.

  - **Login Form**  
    - Type: `n8n-nodes-base.formTrigger`  
    - Role: Presents a login form for users to submit username, password, and hidden sessionId.  
    - Configuration:  
      - Webhook path: `/form/auth`  
      - Fields: Username (required), Password (required), sessionId (hidden)  
      - Button label: Submit  
      - Response shows last node output.  
    - Input: HTTP form submission.  
    - Output: Form data for authentication.  
    - Edge Cases: Missing fields, bots ignored, no authentication logic implemented.

  - **Replace Me!**  
    - Type: `n8n-nodes-base.noOp`  
    - Role: Placeholder where authentication logic should be implemented (e.g., database check).  
    - Configuration: No operation, passes data through.  
    - Input: Login form submission.  
    - Output: Forwarded input for demonstration.  
    - Edge Cases: No actual authentication; must be replaced.

  - **Get Customer Profile**  
    - Type: `n8n-nodes-base.set`  
    - Role: Simulates retrieval of authenticated customer profile data after successful login.  
    - Configuration: Outputs a hardcoded JSON object with customer info and basket items.  
    - Input: From Replace Me! node (post-authentication).  
    - Output: Customer profile JSON.  
    - Edge Cases: Hardcoded data; in real deployments, replace with actual data source.

  - **Update Session**  
    - Type: `n8n-nodes-base.redis`  
    - Role: Updates Redis session key with authenticated customer profile JSON and sets TTL to 3600 seconds (1 hour).  
    - Configuration:  
      - Key: `chat_{{ sessionId from Login Form submission }}`  
      - Operation: set with expiration.  
    - Credentials: Redis account (localhost).  
    - Input: Customer profile JSON.  
    - Output: Stores updated session and triggers confirmation.  
    - Edge Cases: Redis write failures, session expiration.

  - **Confirm Login Success**  
    - Type: `n8n-nodes-base.form`  
    - Role: Displays a success page after login with a personalized welcome message.  
    - Configuration:  
      - Title: "Login Successful!"  
      - Message: "Welcome back {{ $json.name }}, Your chat session is now authenticated. You may now close this window and chat as a customer."  
    - Input: From Update Session node.  
    - Output: HTTP response to user.  
    - Edge Cases: Missing user name in JSON would affect message.

---

### 3. Summary Table

| Node Name                      | Node Type                                  | Functional Role                                    | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                   |
|--------------------------------|--------------------------------------------|--------------------------------------------------|--------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------|
| When chat message received      | @n8n/n8n-nodes-langchain.chatTrigger       | Entry point for incoming chat messages           | (Webhook)                      | Get Session                    |                                                                                               |
| Get Session                    | n8n-nodes-base.redis                        | Retrieve user session data from Redis             | When chat message received     | Customer Support Agent          |                                                                                               |
| Memory                        | @n8n/n8n-nodes-langchain.memoryBufferWindow| Maintain conversation memory per session          | -                             | Customer Support Agent          |                                                                                               |
| LLM                           | @n8n/n8n-nodes-langchain.lmChatOpenAi      | Generate AI response using OpenAI GPT-4.1-mini    | Customer Support Agent         | Customer Support Agent          |                                                                                               |
| Customer Support Agent         | @n8n/n8n-nodes-langchain.agent              | Conversational AI agent handling guest/customer   | Get Session, Memory, LLM       | Get Auth URL (tool), LLM       | ## 1. Create a Conversational Agent with Session Access<br>[Learn more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br>Our support agent handles both guests and customers, with the latter identified by available session data coming in from Redis. For guests who wish to authenticate as customers, the agent is instructed to generate a login URL to share with the user - this is accomplished using a tool. |
| Get Auth URL                  | @n8n/n8n-nodes-langchain.toolWorkflow       | Sub-workflow tool to generate login URL           | Customer Support Agent         | Customer Support Agent          | ## 2. Generate a Login URL via Tool<br>[Learn more about Subworkflow Triggers](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflowtrigger)<br>The login URL is a simple redirect to a form which handles the authentication. For obvious reasons, we've deliberately avoided doing this through the chat as it could have exposed sensitive credentials. The login URL appends the SessionID as a query parameter because this is currently the only way to pre-fill fields in a n8n form. |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger         | Sub-workflow entry for handling event types       | Get Auth URL (tool)            | Switch                        |                                                                                               |
| Switch                       | n8n-nodes-base.switch                       | Branch workflow based on eventType                 | When Executed by Another Workflow | Get Auth Url Response          |                                                                                               |
| Get Auth Url Response          | n8n-nodes-base.set                          | Construct login URL with sessionId query parameter | Switch                        | (returns to tool)               |                                                                                               |
| Login Form                   | n8n-nodes-base.formTrigger                   | User login form for authentication                 | (HTTP Form Submission)         | Replace Me!                    | ## 3. A Login Form to Link Session to Customer Profile<br>[Learn more about the Form Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger)<br>This login form is pretty standard and takes 3 fields: username, password and sessionId. The idea being that once authenticated successfully, the user's customer profile can be used to associate with the sessionID. Note that I haven't implemented the actual authentication logic - you'll need to do this yourself! |
| Replace Me!                  | n8n-nodes-base.noOp                         | Placeholder for authentication logic              | Login Form                    | Get Customer Profile            | ### Replace with Authentication Logic!<br>This could be a query to a PostgreSQL database or otherwise. |
| Get Customer Profile          | n8n-nodes-base.set                          | Provide customer profile data (hardcoded demo)     | Replace Me!                   | Update Session                 |                                                                                               |
| Update Session               | n8n-nodes-base.redis                        | Update Redis session with authenticated profile    | Get Customer Profile          | Confirm Login Success           |                                                                                               |
| Confirm Login Success         | n8n-nodes-base.form                         | Show login success message to user                  | Update Session                |                                |                                                                                               |
| Sticky Note                  | n8n-nodes-base.stickyNote                    | Documentation notes                                 | -                             | -                              | ## 1. Create a Conversational Agent with Session Access<br>[Learn more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br>Our support agent handles both guests and customers, with the latter identified by available session data coming in from Redis. For guests who wish to authenticate as customers, the agent is instructed to generate a login URL to share with the user - this is accomplished using a tool. |
| Sticky Note1                 | n8n-nodes-base.stickyNote                    | Documentation notes                                 | -                             | -                              | ## 2. Generate a Login URL via Tool<br>[Learn more about Subworkflow Triggers](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflowtrigger)<br>The login URL is a simple redirect to a form which handles the authentication. For obvious reasons, we've deliberately avoided doing this through the chat as it could have exposed sensitive credentials. The login URL appends the SessionID as a query parameter because this is currently the only way to pre-fill fields in a n8n form. |
| Sticky Note2                 | n8n-nodes-base.stickyNote                    | Documentation notes                                 | -                             | -                              | ## 3. A Login Form to Link Session to Customer Profile<br>[Learn more about the Form Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger)<br>This login form is pretty standard and takes 3 fields: username, password and sessionId. The idea being that once authenticated successfully, the user's customer profile can be used to associate with the sessionID. Note that I haven't implemented the actual authentication logic - you'll need to do this yourself! |
| Sticky Note3                 | n8n-nodes-base.stickyNote                    | Documentation notes                                 | -                             | -                              | ### Replace with Authentication Logic!<br>This could be a query to a PostgreSQL database or otherwise. |
| Sticky Note4                 | n8n-nodes-base.stickyNote                    | Detailed project overview and instructions         | -                             | -                              | ## Try It Out!<br>### This n8n template demonstrates one approach to customer authentication via chat agents.<br>*Detailed instructions and usage notes included with links to Discord and Forum.* |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: `When chat message received`  
   - Configure webhook to receive chat messages.  
   - No special parameters needed.

2. **Create Redis Node to Get Session**  
   - Type: `n8n-nodes-base.redis`  
   - Name: `Get Session`  
   - Operation: `Get`  
   - Key: `=chat_{{ $json.sessionId }}`  
   - Key Type: `string`  
   - Credentials: Set up Redis credentials (localhost or your Redis server).  
   - Connect input from `When chat message received`.

3. **Create Memory Node for Conversation Context**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Name: `Memory`  
   - Session Key: `=chat_{{ $('When chat message received').first().json.sessionId }}`  
   - Session ID Type: `customKey`

4. **Create OpenAI LLM Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: `LLM`  
   - Model: `gpt-4.1-mini`  
   - Credentials: Set up OpenAI API credentials.  

5. **Create Customer Support Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `Customer Support Agent`  
   - Text: `={{ $('When chat message received').item.json.chatInput }}`  
   - System Message: Instructions distinguishing guests and customers, how to generate login URL, session context usage, and instructions for guest authentication.  
   - Passthrough Binary Images: Enabled.  
   - Connect inputs:  
     - Session data from `Get Session`  
     - Memory from `Memory` node  
     - LLM from `LLM` node  

6. **Create Tool Workflow Node for Login URL**  
   - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Name: `Get Auth URL`  
   - Workflow ID: Use this workflow’s own ID (self-reference).  
   - Workflow Inputs:  
     - `eventType`: `"get_auth_url"`  
     - `data`: `{ "sessionId": $('When chat message received').first().json.sessionId }`  
   - Connect Tool input to `Customer Support Agent` (ai_tool).

7. **Create Execute Workflow Trigger Node**  
   - Type: `n8n-nodes-base.executeWorkflowTrigger`  
   - Name: `When Executed by Another Workflow`  
   - Workflow Inputs: `eventType` (string), `data` (object)  

8. **Create Switch Node**  
   - Type: `n8n-nodes-base.switch`  
   - Name: `Switch`  
   - Rules: If `eventType` equals `"get_auth_url"`, route accordingly.

9. **Create Set Node for Auth URL Response**  
   - Type: `n8n-nodes-base.set`  
   - Name: `Get Auth Url Response`  
   - Set Field: `auth_url` = `https://<your-n8n-url>/form/auth?sessionId={{ $json.data.sessionId }}`  
   - Replace `<your-n8n-url>` with your actual n8n public URL.

10. **Connect Execute Workflow Trigger → Switch → Get Auth Url Response**  
    - Outputs from Switch node to `Get Auth Url Response`.

11. **Create Form Trigger Node for Login Form**  
    - Type: `n8n-nodes-base.formTrigger`  
    - Name: `Login Form`  
    - Path: `/form/auth`  
    - Form Title: `Authenticate Chat Session`  
    - Form Description: `Please login to validate your current chat session.`  
    - Fields:  
      - Username (required)  
      - Password (required)  
      - sessionId (hidden field)  
    - Button Label: `Submit`

12. **Create No-Op Node as Placeholder for Authentication Logic**  
    - Type: `n8n-nodes-base.noOp`  
    - Name: `Replace Me!`  
    - Connect input from `Login Form`.  
    - Replace with actual authentication logic later.

13. **Create Set Node for Customer Profile**  
    - Type: `n8n-nodes-base.set`  
    - Name: `Get Customer Profile`  
    - Output hardcoded JSON with user details (id, name, email, basket items) for demo purposes.  
    - Connect input from `Replace Me!`.

14. **Create Redis Node to Update Session**  
    - Type: `n8n-nodes-base.redis`  
    - Name: `Update Session`  
    - Operation: `Set` with expiration (TTL 3600 seconds)  
    - Key: `=chat_{{ $('Login Form').first().json.formQueryParameters.sessionId }}`  
    - Value: `={{ $json.toJsonString() }}` (customer profile JSON)  
    - Credentials: Redis account.  
    - Connect input from `Get Customer Profile`.

15. **Create Form Node to Confirm Login Success**  
    - Type: `n8n-nodes-base.form`  
    - Name: `Confirm Login Success`  
    - Completion Title: `Login Successful!`  
    - Completion Message:  
      ```
      Welcome back {{ $json.name }},
      Your chat session is now authenticated.

      You may now close this window and chat as a customer.
      ```  
    - Connect input from `Update Session`.

16. **Finalize Connections for Authentication Workflow**  
    - Connect `Login Form` → `Replace Me!` → `Get Customer Profile` → `Update Session` → `Confirm Login Success`.

17. **Connect Main Workflow to Sub-Workflow**  
    - `Customer Support Agent` → `Get Auth URL` (tool)  
    - `When Executed by Another Workflow` receives calls from `Get Auth URL`.

18. **Activate Workflow**  
    - Ensure all credentials are configured (OpenAI, Redis).  
    - Replace placeholder URLs and authentication logic.  
    - Deploy and test.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow demonstrates an approach to customer authentication via chat agents, allowing guests to authenticate at any time during a chat session rather than upfront. | Sticky Note4 in workflow |
| Learn more about AI Agents used in this workflow: [n8n AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) | Sticky Note |
| Learn more about Subworkflow Triggers used to generate login URLs: [Execute Workflow Trigger Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflowtrigger) | Sticky Note1 |
| Learn more about Form Trigger used for login form: [Form Trigger Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger) | Sticky Note2 |
| Join community support channels for help: [Discord](https://discord.com/invite/XPKeKXeB7d), [n8n Forum](https://community.n8n.io/) | Sticky Note4 |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.