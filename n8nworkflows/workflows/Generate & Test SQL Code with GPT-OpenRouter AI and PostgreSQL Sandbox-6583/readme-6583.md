Generate & Test SQL Code with GPT/OpenRouter AI and PostgreSQL Sandbox

https://n8nworkflows.xyz/workflows/generate---test-sql-code-with-gpt-openrouter-ai-and-postgresql-sandbox-6583


# Generate & Test SQL Code with GPT/OpenRouter AI and PostgreSQL Sandbox

### 1. Workflow Overview

This n8n workflow titled **"Generate & Test SQL Code with GPT/OpenRouter AI and PostgreSQL Sandbox"** is designed to receive user prompts related to PostgreSQL database tasks, generate appropriate SQL code using AI models (OpenAI or OpenRouter), execute the generated SQL in a PostgreSQL sandbox, and handle any errors or debugging iteratively. It supports automatic error fixing loops and user-assisted error resolution, ensuring that the SQL code provided is executable and aligns with user requests.

**Target Use Cases:**  
- Interactive generation of PostgreSQL SQL code based on natural language instructions.  
- Automated testing and validation of generated SQL queries in a sandbox environment.  
- Iterative error detection and correction using AI, optionally with user intervention.  
- Managing AI chat memory and session states to support multi-turn conversations with context.

**Logical Blocks:**

- **1.1 Input Reception & Initialization:** Receiving user chat input, extracting parameters, initializing session variables and AI assistant existence checks.  
- **1.2 AI Model Selection & Prompting:** Determining AI provider (OpenAI vs OpenRouter), preparing prompts including error handling context, and invoking the appropriate AI model.  
- **1.3 SQL Execution & Result Processing:** Executing AI-generated SQL against PostgreSQL, processing output, and detecting execution errors.  
- **1.4 Error Handling & Auto-Fixing Loop:** Conditional logic for automatic or manual error fixing, including limits on retries and user prompts for instruction.  
- **1.5 Output Decision & Response:** Differentiating between executable SQL and user messages, formatting output accordingly, and managing memory context in chat models.  
- **1.6 Assistant Management:** Creating or retrieving AI assistants on OpenAI platform to handle chat history and session context.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

**Overview:**  
This block receives the initial chat message webhook, extracts necessary local variables and parameters, and checks if a dedicated AI assistant already exists or needs to be created for the session.

**Nodes Involved:**  
- When chat message received  
- localVariables  
- IfOpenAI  
- AgentName  
- getAssistantsList  
- isAssistantExistsCode  
- isAssistantExists  
- assistant  
- createOpenAiAssistant  
- Sticky Note, Sticky Note2, Sticky Note1 (for contextual guidance)

**Node Details:**

- *When chat message received*  
  - Type: Langchain chatTrigger (webhook)  
  - Role: Entry point; receives user chat input with parameters like sessionId, aiProvider, model, autoErrorFixing flag, and current DB schema.  
  - Config: Public webhook mode enabled.  
  - Outputs: passes JSON with all input parameters.  
  - Edge cases: Missing or malformed input parameters could block downstream processing.

- *localVariables*  
  - Type: Set node  
  - Role: Sets static instructions for AI, including detailed rules for generating executable PostgreSQL SQL code, data mocking, schema handling, and response formatting.  
  - Key variables: `instruction` string defining AI behavior and output format.  
  - Output connections: feeds into IfOpenAI for AI provider branching.

- *IfOpenAI*  
  - Type: If node  
  - Role: Checks if AI provider is OpenAI to branch logic accordingly.  
  - Condition: `$json.aiProvider === "openai"` (strict string equality).  
  - Outputs: True → AgentName; False → OpenRouterAgent (outside this block).

- *AgentName*  
  - Type: Set node  
  - Role: Constructs a unique AI assistant name combining a prefix with the model name (e.g., "AiDoubleCheck_gpt-4.1-mini") to identify or create an assistant on OpenAI.  
  - Uses expression: `'AiDoubleCheck_' + $('localVariables').last().json.model`  
  - Output feeds getAssistantsList.

- *getAssistantsList*  
  - Type: OpenAI assistant list operation  
  - Role: Retrieves existing assistants from OpenAI to check if the desired assistant exists.  
  - Credentials: OpenAI API key required.  
  - On error: continues without breaking workflow.  
  - Output feeds isAssistantExistsCode.

- *isAssistantExistsCode*  
  - Type: Code node  
  - Role: Iterates over returned assistants list and compares names to determine if the named assistant exists; outputs flags and assistant ID.  
  - Key expressions: trims and compares assistant names to AgentName.  
  - Outputs: `{output: 1 or 0, id: assistantId or null}` feeds isAssistantExists.

- *isAssistantExists*  
  - Type: If node  
  - Role: Branches based on assistant existence flag.  
  - True → assistant node (set assistant ID)  
  - False → createOpenAiAssistant node.

- *assistant*  
  - Type: Set node  
  - Role: Sets assistant ID for use in subsequent AI calls.

- *createOpenAiAssistant*  
  - Type: OpenAI assistant create operation  
  - Role: Creates a new OpenAI assistant with the designated name, model, and instructions.  
  - Credentials: OpenAI API.  
  - Inputs: name, modelId, description, and instruction from localVariables.

**Edge Cases & Failures:**  
- OpenAI API rate limits or auth errors when listing or creating assistants.  
- Missing or mismatched model names or parameters causing assistant creation to fail.  
- Empty or invalid input webhook requests.

---

#### 1.2 AI Model Selection & Prompting

**Overview:**  
This block selects the AI model (OpenAI or OpenRouter), prepares the prompt with or without error context, and sends it for SQL generation or error correction.

**Nodes Involved:**  
- IfOpenAI (from previous block, false branch)  
- OpenRouterAgent  
- OpenRouter Chat Model  
- OpenAIMainBrain  
- isOpenAI  
- Simple Memory  
- Sticky Note, Sticky Note1

**Node Details:**

- *IfOpenAI* (false branch)  
  - Sends flow to OpenRouterAgent for OpenRouter provider.

- *OpenRouterAgent*  
  - Type: Langchain agent node for OpenRouter AI  
  - Role: Sends prompt text to OpenRouter AI agent with system message instruction from localVariables.  
  - Input prompt: if error prompt generated, uses error handling mode prompt; else combines DB schema info + chat input + session prefix.  
  - Parameters: promptType "define", custom systemMessage.  
  - Output: AI-generated SQL or text response.  
  - Connected to Simple Memory node for chat context management.

- *OpenRouter Chat Model*  
  - Type: Langchain language model (AI model config)  
  - Role: Provides LM configuration for OpenRouterAgent node.  
  - Connected as ai_languageModel input to OpenRouterAgent.

- *Simple Memory*  
  - Type: Langchain memory buffer window  
  - Role: Maintains a sliding window chat memory buffer with length 7, keyed by sessionId from localVariables.  
  - Used only for OpenRouter to handle chat history client-side.  
  - Connected as ai_memory input to OpenRouterAgent.

- *isOpenAI*  
  - Type: If node  
  - Role: Checks if AI provider is OpenAI (true branch).  
  - Condition: localVariables.aiProvider == "openai".

- *OpenAIMainBrain*  
  - Type: Langchain OpenAI node  
  - Role: Sends prompt to OpenAI assistant (created or existing) with threadId and assistantId for chat context.  
  - Prompt and text built from localVariables instructions plus either error prompt or normal chat input with DB schema.  
  - Output: generated SQL or text response.

**Edge Cases & Failures:**  
- API key errors for OpenAI or OpenRouter.  
- Timeouts or rate limits.  
- Empty or malformed prompt generation.  
- Memory buffer overflow or session key mismatches.

---

#### 1.3 SQL Execution & Result Processing

**Overview:**  
Executes the SQL query generated by AI in the PostgreSQL sandbox, captures result or error message, and prepares output for next steps.

**Nodes Involved:**  
- setOutputByProvider  
- Execute_AI_result  
- executedSQLQuery  
- IfError  
- GenerateErrorPrompt  
- isOpenAI  
- Sticky Note3 (for output handling logic)

**Node Details:**

- *setOutputByProvider*  
  - Type: Set node  
  - Role: Selects the output from AI based on provider (OpenAI or OpenRouter) into a unified "output" field.  
  - Expression: conditional ternary based on aiProvider.

- *Execute_AI_result*  
  - Type: Postgres node (executeQuery)  
  - Role: Executes the SQL code contained in `output` against the PostgreSQL sandbox.  
  - Credentials: PostgreSQL credentials required.  
  - On error: configured with "continueErrorOutput" to catch errors without stopping workflow.  
  - Output: query execution result or error message.

- *executedSQLQuery*  
  - Type: Set node  
  - Role: Formats and stores execution status and query for downstream use.  
  - Sets fields: `query` (executed SQL), `type` (success), `executionResult` (full JSON from execution).

- *IfError*  
  - Type: If node  
  - Role: Checks if there was an error in SQL execution by verifying if GenerateErrorPrompt node ran (indicating an error).  
  - Condition: GenerateErrorPrompt.isExecuted == true.

- *GenerateErrorPrompt*  
  - Type: Code node  
  - Role: Generates an AI prompt specifically for error fixing, including error message and description, asking AI to fix SQL.  
  - Input: error message and description from Execute_AI_result node.

- *isOpenAI* (downstream from GenerateErrorPrompt)  
  - Routes error fixing prompt to correct AI provider node (OpenAI or OpenRouter).

**Edge Cases & Failures:**  
- SQL syntax errors or runtime exceptions in PostgreSQL.  
- Credential or connectivity issues to PostgreSQL.  
- AI failure to produce valid SQL even after error prompt.  
- Infinite loops avoided by limiting retries (in later block).

---

#### 1.4 Error Handling & Auto-Fixing Loop

**Overview:**  
Handles the logic for automatic error fixing or user intervention when SQL execution fails. Limits the number of automatic retries to prevent infinite loops.

**Nodes Involved:**  
- AutoErrorFixing  
- IsMaxAutoErrorReached  
- maxAutoErrorLimitReached  
- askUserHowToHandleError  
- Sticky Note4, Sticky Note5

**Node Details:**

- *AutoErrorFixing*  
  - Type: If node  
  - Role: Determines if automatic error fixing is enabled (localVariables.autoErrorFixing == true).  
  - True → IsMaxAutoErrorReached node.  
  - False → askUserHowToHandleError node (request user instruction).

- *IsMaxAutoErrorReached*  
  - Type: If node  
  - Role: Checks if the number of attempts to fix the error has reached or exceeded 4 (limit).  
  - Uses `GenerateErrorPrompt.runIndex` to count retries.

- *maxAutoErrorLimitReached*  
  - Type: Set node  
  - Role: Sets output type to "maxAutoErrorLimitReached" and carries the last AI output for user notification.

- *askUserHowToHandleError*  
  - Type: Set node  
  - Role: Prepares message to user asking how to proceed with error fixing when auto fixing is disabled.

**Edge Cases & Failures:**  
- Infinite error fixing loops prevented by retry limit (4).  
- User opting out of auto fixing requires clear messaging.  
- Failures in error prompt generation or AI response during fixing attempts.

---

#### 1.5 Output Decision & Response

**Overview:**  
Determines whether the AI's response is executable SQL or a message for the user, formatting output accordingly.

**Nodes Involved:**  
- isExecutable  
- wordsForUser1  
- setOutputByProvider  
- executedSQLQuery  
- Sticky Note3

**Node Details:**

- *isExecutable*  
  - Type: If node  
  - Role: Checks if AI output contains the string "words_for_user" to differentiate between SQL code and user message.  
  - Condition: output does not contain "words_for_user".

- *wordsForUser1*  
  - Type: Set node  
  - Role: Sets output type to "wordsForUser" and message content for user-facing text responses.

- *setOutputByProvider* and *executedSQLQuery* (previously described) help unify output format.

**Edge Cases & Failures:**  
- AI might respond with natural language (no SQL) for out-of-scope questions.  
- Proper detection of "words_for_user" flag critical to prevent SQL execution errors.

---

#### 1.6 Assistant & Chat Memory Management

**Overview:**  
Manages the creation and reuse of OpenAI assistants to leverage built-in chat history and session management, while OpenRouter requires client-side memory handling.

**Nodes Involved:**  
- createOpenAiAssistant  
- assistant  
- getAssistantsList  
- isAssistantExistsCode  
- Simple Memory  
- Sticky Notes (explanations on chat history handling)

**Node Details:**

- *createOpenAiAssistant* and *assistant* nodes manage OpenAI assistant lifecycle.  
- *Simple Memory* maintains chat history for OpenRouter provider since OpenRouter lacks built-in assistant chat history.  
- Sticky Notes explain this distinction and provide contextual instructions.

**Edge Cases:**  
- Assistant creation failures impact chat context continuity.  
- Memory overflow or session key mismatch could lead to loss of context.

---

### 3. Summary Table

| Node Name                   | Node Type                              | Functional Role                                      | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                              |
|-----------------------------|--------------------------------------|-----------------------------------------------------|-------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received  | Langchain chatTrigger (webhook)      | Entry point, receives user chat input                | —                             | localVariables                |                                                                                                        |
| localVariables              | Set                                  | Defines AI instructions and session parameters       | When chat message received     | IfOpenAI                     | use this to get necessary local variables, like: instruction to AI, sessionId, and all input parameters |
| IfOpenAI                   | If                                   | Branches between OpenAI and OpenRouter AI providers  | localVariables                | AgentName / OpenRouterAgent  |                                                                                                        |
| AgentName                  | Set                                  | Constructs unique AI assistant name                   | IfOpenAI (true)               | getAssistantsList             |                                                                                                        |
| getAssistantsList          | OpenAI assistant list                 | Retrieves existing assistants                         | AgentName                    | isAssistantExistsCode         |                                                                                                        |
| isAssistantExistsCode      | Code                                 | Checks if assistant exists and extracts ID           | getAssistantsList             | isAssistantExists             |                                                                                                        |
| isAssistantExists          | If                                   | Branches to create or reuse assistant                 | isAssistantExistsCode         | assistant / createOpenAiAssistant |                                                                                                     |
| assistant                  | Set                                  | Sets assistant ID for AI calls                        | isAssistantExists             | OpenAIMainBrain              |                                                                                                        |
| createOpenAiAssistant      | OpenAI assistant create               | Creates new assistant with instructions               | isAssistantExists (false)     | assistant                   |                                                                                                        |
| OpenRouterAgent            | Langchain agent (OpenRouter AI)      | Sends prompt to OpenRouter AI and receives output    | IfOpenAI (false)              | isExecutable                 | OpenAI has built-in assistant that handles chat history on their side. For open-router we should handle chat history on our side. |
| OpenRouter Chat Model      | Langchain language model              | Provides LM config for OpenRouterAgent               | —                             | OpenRouterAgent              |                                                                                                        |
| Simple Memory              | Langchain memory buffer               | Maintains chat history for OpenRouter sessions       | localVariables                | OpenRouterAgent              |                                                                                                        |
| OpenAIMainBrain            | Langchain OpenAI                      | Sends prompt to OpenAI assistant                      | assistant / isOpenAI          | isExecutable                 |                                                                                                        |
| setOutputByProvider        | Set                                  | Normalizes AI output from OpenAI or OpenRouter       | OpenAIMainBrain / OpenRouterAgent | IfError                  |                                                                                                        |
| Execute_AI_result          | Postgres executeQuery                 | Executes generated SQL in PostgreSQL sandbox          | IfError                      | executedSQLQuery / GenerateErrorPrompt |                                                                                                    |
| executedSQLQuery           | Set                                  | Stores executed query, result and success type       | Execute_AI_result             | —                            |                                                                                                        |
| IfError                   | If                                   | Checks if SQL execution error occurred                | setOutputByProvider           | AutoErrorFixing / Execute_AI_result |                                                                                                     |
| GenerateErrorPrompt        | Code                                 | Generates prompt for AI to fix SQL errors             | Execute_AI_result             | isOpenAI                     |                                                                                                        |
| AutoErrorFixing            | If                                   | Determines if auto error fixing enabled               | IfError                      | IsMaxAutoErrorReached / askUserHowToHandleError |                                                                                               |
| IsMaxAutoErrorReached      | If                                   | Checks if max error fixing retries reached            | AutoErrorFixing               | maxAutoErrorLimitReached / Execute_AI_result | Error fixing loop will work only n times, defined in this node. It is done to prevent infinite loop |
| maxAutoErrorLimitReached   | Set                                  | Sets output indicating max retry limit reached        | IsMaxAutoErrorReached         | Execute_AI_result            |                                                                                                        |
| askUserHowToHandleError    | Set                                  | Prepares message for user when auto fixing disabled   | AutoErrorFixing               | —                            | If the user has selected automatic error fixing, debugging will be performed automatically, otherwise the system will ask the user for further instruction |
| isExecutable               | If                                   | Checks if AI output is executable SQL or user words  | OpenAIMainBrain / OpenRouterAgent | setOutputByProvider / wordsForUser1 | Sometimes we can't answer with just code. This node is responsible for separation. If it is not a code for sandbox, then it will go to user as words. |
| wordsForUser1              | Set                                  | Sets output as user message when no executable SQL   | isExecutable                 | —                            |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: `When chat message received`  
   - Mode: webhook, public enabled  
   - Purpose: receive user chat input with parameters including sessionId, aiProvider, model, chatInput, etc.

2. **Set Node for Instructions:**  
   - Type: `Set`  
   - Name: `localVariables`  
   - Assign a string variable `instruction` containing detailed AI rules for generating executable SQL code (as per the provided instructions).  
   - Output feeds into AI Provider selection.

3. **If Node to Check AI Provider:**  
   - Type: `If`  
   - Name: `IfOpenAI`  
   - Condition: `{{$json.aiProvider}} === "openai"` (strict match)  
   - True branch for OpenAI flow, False for OpenRouter.

4. **OpenAI Assistant Preparation (True Branch):**  
   - Set node `AgentName` to create unique assistant name: `'AiDoubleCheck_' + model` from localVariables.  
   - OpenAI node `getAssistantsList` to list assistants.  
   - Code node `isAssistantExistsCode` to check if assistant with that name exists, output flag and id.  
   - If node `isAssistantExists` to branch:  
     - True → Set node `assistant` to set assistant ID.  
     - False → OpenAI node `createOpenAiAssistant` to create new assistant with name, model, and instructions; then set assistant ID.  
   - Output assistant ID to next OpenAI chat node.

5. **OpenAI Chat Node:**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Name: `OpenAIMainBrain`  
   - Parameters:  
     - `text`: Conditional text based on error prompt or normal chat input with DB schema.  
     - `threadId` and `assistantId` from previous nodes.  
   - Credentials: OpenAI API key configured.

6. **OpenRouter Chat Node (False Branch):**  
   - Langchain language model node `OpenRouter Chat Model` configured for OpenRouter.  
   - Langchain agent node `OpenRouterAgent` with:  
     - `text` parameter as conditional prompt (error or normal).  
     - `systemMessage` from localVariables instruction.  
     - Memory input from `Simple Memory` node maintaining context buffer keyed by sessionId.

7. **Simple Memory Node:**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Session key: sessionId from localVariables  
   - Context window length: 7.

8. **Set Node `setOutputByProvider`:**  
   - Normalizes output from OpenAI or OpenRouter AI nodes into a unified `output` field.

9. **PostgreSQL Execution Node:**  
   - Type: Postgres  
   - Name: `Execute_AI_result`  
   - Operation: executeQuery with query from `output`.  
   - Credentials: PostgreSQL sandbox credentials.  
   - On error: continue workflow with error output.

10. **Set Node `executedSQLQuery`:**  
    - Stores query, sets type "success", and execution result JSON.

11. **If Node `IfError`:**  
    - Checks if error occurred by verifying if `GenerateErrorPrompt` node executed.

12. **Error Prompt Code Node:**  
    - `GenerateErrorPrompt` creates new AI prompt embedding error message and description, requesting SQL fix.

13. **AI Provider Check for Error Prompt:**  
    - `isOpenAI` routes error prompt to correct AI node.

14. **Auto Error Fixing If Node:**  
    - Checks if autoErrorFixing enabled.  
    - True → Check max retries with `IsMaxAutoErrorReached`.  
      - If max reached → set output with `maxAutoErrorLimitReached` message.  
      - Else → retry AI prompt execution.  
    - False → Set node `askUserHowToHandleError` to send message asking user for instructions.

15. **If Node `isExecutable`:**  
    - Checks if AI output contains "words_for_user" string.  
    - If true → route to `wordsForUser1` node to send user message.  
    - Else → proceed with SQL execution.

16. **Set Node `wordsForUser1`:**  
    - Sets output type "wordsForUser" and message content for user display.

17. **Sticky Notes:**  
    - Add sticky notes at relevant points explaining:  
      - Input parameters expected at webhook.  
      - AI provider differences in chat memory handling.  
      - Purpose of error handling and loop limits.  
      - Differentiation between SQL code and user messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Input parameters expected at webhook: sessionId (uuidv4), threadId (nullable), apiKey, aiProvider, model, autoErrorFixing (boolean), chatInput (user prompt), currentDbSchemaWithData (JSON schema) | Sticky Note2 near webhook node                                                                              |
| OpenAI has built-in assistant chat history management; OpenRouter requires explicit memory buffer node for chat history | Sticky Note1 near AI model selection nodes                                                                  |
| Error fixing loop capped at 4 retries to prevent infinite loops                                                      | Sticky Note4 near AutoErrorFixing and IsMaxAutoErrorReached nodes                                           |
| If automatic error fixing disabled, workflow asks user how to proceed with error correction                          | Sticky Note5 near askUserHowToHandleError node                                                             |
| Differentiate between executable SQL and user messages with "words_for_user" flag to avoid SQL execution errors      | Sticky Note3 near isExecutable node                                                                          |
| PostgreSQL sandbox credentials must be configured securely with proper permissions                                    | —                                                                                                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.