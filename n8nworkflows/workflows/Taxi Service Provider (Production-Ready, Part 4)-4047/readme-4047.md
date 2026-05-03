Taxi Service Provider (Production-Ready, Part 4)

https://n8nworkflows.xyz/workflows/taxi-service-provider--production-ready--part-4--4047


# Taxi Service Provider (Production-Ready, Part 4)

### 1. Workflow Overview

This n8n workflow, titled "🤖 Taxi Service Provider," is designed as a production-ready taxi service provider automation. It processes taxi booking requests received from a parent Taxi Service Workflow, fetches and caches provider data from external databases, manages provider usage counts, estimates fares using AI, creates booking records, and finally responds back with the AI-generated result. The workflow is optimized for scaling in a queue mode production environment and incorporates error handling.

Logical blocks of the workflow are:

- **1.1 Input Reception and Initialization:** Listens for incoming taxi service requests from sub-workflows or chat triggers and prepares initial data.
- **1.2 Provider Data Loading and Caching:** Checks Redis cache for provider info; if missing, loads from PostgreSQL and caches it.
- **1.3 Provider Validation:** Validates if the provider is active; if inactive, stops processing.
- **1.4 Provider Number Incrementing:** Uses Redis to increment and track provider usage numbers.
- **1.5 AI Fare Estimation and Booking Creation:** Uses an AI Agent with language models and tools (including a calculator and database write) to estimate fare and create a booking.
- **1.6 Output Decision and Response:** Depending on the scoring flag, formats the output with or without score and calls back the response.
- **1.7 Error Management:** Handles errors gracefully and routes to a callback with retry instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
  This block waits for external messages (likely from a parent workflow or chat interface) and sets up initial input fields to prepare for provider processing.

- **Nodes Involved:**  
  - Flow Trigger  
  - Input (Set node)  
  - Test Trigger (Chat Trigger)  
  - Test Fields (Set node)

- **Node Details:**

  - **Flow Trigger**  
    - Type: ExecuteWorkflowTrigger  
    - Role: Entry point to wait for messages from other sub-workflows.  
    - Config: No parameters; uses default waiting mode.  
    - Inputs: None  
    - Outputs: Connected to "Input" set node.  
    - Notes: None  
    - Edge Cases: Triggers only when a message is received; no timeout.

  - **Input**  
    - Type: Set  
    - Role: Initializes input variables (e.g., `provider_no` set to test).  
    - Config: Empty parameters; likely default or placeholder values for test.  
    - Inputs: From Flow Trigger  
    - Outputs: To "Provider Cache" node.  
    - Notes: Sticky note "provider_no: test" indicates sample input.  
    - Edge Cases: If input data is missing or malformed, downstream errors may occur.

  - **Test Trigger**  
    - Type: Chat Trigger (Langchain)  
    - Role: Alternative entry point for chat-based triggers (used for testing).  
    - Config: Webhook ID configured for external chat integration.  
    - Inputs: None  
    - Outputs: To "Test Fields" node.  
    - Edge Cases: Auth or webhook failures possible.

  - **Test Fields**  
    - Type: Set  
    - Role: Sets test input fields (e.g., `service_no: test`).  
    - Config: Empty parameters; placeholder values.  
    - Inputs: From Test Trigger  
    - Outputs: To "Input" node.  
    - Notes: Sticky note "service_no: test".  
    - Edge Cases: Invalid test data may propagate errors.

---

#### 1.2 Provider Data Loading and Caching

- **Overview:**  
  This block attempts to retrieve provider data from Redis cache. If the cache is empty or invalid, it loads fresh data from the PostgreSQL provider database and caches it for future use.

- **Nodes Involved:**  
  - Provider Cache (Redis)  
  - Provider Cache Switch  
  - Load Provider Data (Postgres)  
  - Parse Provider (Code)  
  - Save Provider Cache (Redis)

- **Node Details:**

  - **Provider Cache**  
    - Type: Redis  
    - Role: Retrieve cached provider data with TTL 15 minutes (`service:{channel_no}:{service_no}:data`).  
    - Config: Uses Redis credentials; continues on error to avoid blocking.  
    - Inputs: From "Input" node.  
    - Outputs: To "Provider Cache Switch".  
    - Notes: Sticky note "TTL 15m service:{channel_no}:{service_no}:data".  
    - Edge Cases: Redis connection failure, cache miss.

  - **Provider Cache Switch**  
    - Type: Switch  
    - Role: Branch logic based on cache availability and validity.  
    - Config: Checks if cache is present or not; handles "NO CACHE FOR demo" case.  
    - Inputs: From "Provider Cache".  
    - Outputs: To "Parse Provider" or "Load Provider Data".  
    - Notes: Sticky note "NO CACHE FOR demo".  
    - Edge Cases: Misclassification of cache status.

  - **Load Provider Data**  
    - Type: Postgres  
    - Role: Query the `sys_provider` table or equivalent to load provider info from the external database.  
    - Config: Uses PostgreSQL credentials; set to execute once; continues on error.  
    - Inputs: From "Provider Cache Switch" (cache miss path).  
    - Outputs: To "If Active" node and also to "Provider Cache Switch" (fallback).  
    - Edge Cases: Database connection errors, query timeouts.

  - **Parse Provider**  
    - Type: Code  
    - Role: Parses provider data fetched either from cache or DB into usable format for downstream nodes.  
    - Config: Custom JavaScript code to transform data structure.  
    - Inputs: From "Provider Cache Switch".  
    - Outputs: To "Provider" node.  
    - Edge Cases: Parsing errors, unexpected data formats.

  - **Save Provider Cache**  
    - Type: Redis  
    - Role: Saves freshly loaded provider data back to Redis cache with TTL 15 minutes.  
    - Config: Uses Redis credentials; continues on error to avoid blocking.  
    - Inputs: From "If Active" node (active providers only).  
    - Outputs: To "Provider" node.  
    - Notes: Sticky note "TTL 15m".  
    - Edge Cases: Redis write failures.

---

#### 1.3 Provider Validation

- **Overview:**  
  Checks if the provider record is active. If inactive, the workflow halts further processing by routing to a test output node.

- **Nodes Involved:**  
  - If Active (If)  
  - Test Output (Set)  
  - Save Provider Cache

- **Node Details:**

  - **If Active**  
    - Type: If  
    - Role: Conditional check on provider active status flag.  
    - Config: Evaluates if provider is active (likely a boolean or status field).  
    - Inputs: From "Load Provider Data".  
    - Outputs: True → "Save Provider Cache", False → "Test Output".  
    - Edge Cases: Missing or malformed active flag.

  - **Test Output**  
    - Type: Set  
    - Role: Placeholder output for inactive providers, does no further processing.  
    - Config: Default or empty data set.  
    - Inputs: From "If Active" (false branch).  
    - Outputs: None or terminal.  
    - Edge Cases: None significant.

---

#### 1.4 Provider Number Incrementing

- **Overview:**  
  Uses Redis to increment a counter representing the number of service uses per provider, aiding in selection and load balancing.

- **Nodes Involved:**  
  - Provider (Set)  
  - Provider Number (Redis)

- **Node Details:**

  - **Provider**  
    - Type: Set  
    - Role: Prepares or reformats provider data before incrementing the usage number.  
    - Config: Custom values set for provider identification or keys.  
    - Inputs: From "Save Provider Cache".  
    - Outputs: To "Provider Number".  
    - Edge Cases: Invalid provider data.

  - **Provider Number**  
    - Type: Redis  
    - Role: Increments the provider usage number with TTL 5 minutes (`{session_id}:service:providers`).  
    - Config: Uses Redis credentials; continues on error.  
    - Inputs: From "Provider" set node.  
    - Outputs: To "AI Agent".  
    - Notes: Sticky note "{session_id}:service:providers TTL 5m".  
    - Edge Cases: Redis connection failure, race conditions.

---

#### 1.5 AI Fare Estimation and Booking Creation

- **Overview:**  
  This block triggers an AI agent to estimate the fare, potentially using an external language model and calculator tool, then creates a new booking record in PostgreSQL.

- **Nodes Involved:**  
  - AI Agent (Langchain agent)  
  - Calculator (Langchain tool)  
  - xAI @grok-2-1212 (AI language model)  
  - Create Booking Data (Postgres tool)  
  - Code (for validation/processing)

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent  
    - Role: Orchestrates AI-driven fare estimation and booking creation.  
    - Config: Uses linked language model and tools; continues on error to allow fallback.  
    - Inputs: From "Provider Number" (Redis increment).  
    - Outputs: To "Code" node for further processing or "Error Output1" on error.  
    - Edge Cases: API errors, timeout, invalid AI response.

  - **Calculator**  
    - Type: Langchain Tool Calculator  
    - Role: Provides computational capabilities to AI Agent for fare calculation.  
    - Config: Linked as a tool to AI Agent.  
    - Inputs: Invoked by AI Agent.  
    - Outputs: To AI Agent.  
    - Edge Cases: Calculation errors, division by zero.

  - **xAI @grok-2-1212**  
    - Type: Langchain AI Language Model  
    - Role: Language model backend for AI Agent.  
    - Config: Custom model parameters (not detailed here).  
    - Inputs: Linked as the AI language model for AI Agent.  
    - Outputs: To AI Agent.  
    - Edge Cases: Model unavailability, rate limits.

  - **Create Booking Data**  
    - Type: Postgres Tool  
    - Role: Inserts new booking records into database based on AI output.  
    - Config: Uses Postgres credentials; linked as AI Agent tool.  
    - Inputs: Invoked by AI Agent.  
    - Outputs: To AI Agent.  
    - Edge Cases: DB errors, transaction failures.

  - **Code**  
    - Type: Code node  
    - Role: Validates AI agent output and prepares data for scoring decision.  
    - Config: Custom JavaScript code; onError set to continue error output.  
    - Inputs: From AI Agent.  
    - Outputs: True → "If Valid?", False → "Error Output1".  
    - Edge Cases: Expression errors, unexpected data format.

---

#### 1.6 Output Decision and Response

- **Overview:**  
  Decides whether to include a score in the output based on validation, formats the output accordingly, and executes a callback sub-workflow to send the response.

- **Nodes Involved:**  
  - If Valid? (If node)  
  - If Score (If node)  
  - Output w/ Score (Set)  
  - Output w/o Score (Set)  
  - Call Back (Execute Workflow)

- **Node Details:**

  - **If Valid?**  
    - Type: If  
    - Role: Checks if AI output is valid to proceed.  
    - Config: Conditional expression on AI output validity.  
    - Inputs: From "Code" node.  
    - Outputs: True → "If Score", False → "Test Output".  
    - Edge Cases: Invalid AI output format.

  - **If Score**  
    - Type: If  
    - Role: Determines if scoring information should be included in output.  
    - Config: Checks presence or flag for score.  
    - Inputs: From "If Valid?".  
    - Outputs: True → "Output w/ Score", False → "Output w/o Score".  
    - Edge Cases: Missing score flag.

  - **Output w/ Score**  
    - Type: Set  
    - Role: Formats output including score information.  
    - Inputs: From "If Score" (true branch).  
    - Outputs: To "Call Back".  
    - Edge Cases: Formatting errors.

  - **Output w/o Score**  
    - Type: Set  
    - Role: Formats output without score.  
    - Inputs: From "If Score" (false branch).  
    - Outputs: To "Call Back".  
    - Edge Cases: Formatting errors.

  - **Call Back**  
    - Type: Execute Workflow  
    - Role: Executes sub-workflow "Demo Call Back" to send the response back to caller.  
    - Inputs: From "Output w/ Score", "Output w/o Score" and "Error Output1".  
    - Outputs: None (terminal).  
    - Notes: Sticky note "Demo Call Back".  
    - Edge Cases: Sub-workflow unavailability, execution errors.

---

#### 1.7 Error Management

- **Overview:**  
  Provides a fallback path for error scenarios, setting an error message and calling back with retry instructions.

- **Nodes Involved:**  
  - Error Output1 (Set)  
  - Call Back (Execute Workflow)

- **Node Details:**

  - **Error Output1**  
    - Type: Set  
    - Role: Sets error message "Please retry." for output.  
    - Config: Static message.  
    - Inputs: From AI Agent or Code node on error paths.  
    - Outputs: To "Call Back".  
    - Notes: Sticky note "Please retry."  
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name            | Node Type                      | Functional Role                             | Input Node(s)             | Output Node(s)          | Sticky Note                      |
|----------------------|--------------------------------|---------------------------------------------|----------------------------|--------------------------|---------------------------------|
| Flow Trigger         | ExecuteWorkflowTrigger         | Entry point for sub-workflow messages       | None                       | Input                    |                                 |
| Input                | Set                           | Initializes input fields                      | Flow Trigger, Test Fields  | Provider Cache           | provider_no: test               |
| Test Trigger         | Chat Trigger                  | Alternative chat-based trigger                | None                       | Test Fields              |                                 |
| Test Fields          | Set                           | Sets test input fields                        | Test Trigger               | Input                    | service_no: test                |
| Provider Cache       | Redis                         | Retrieve provider data from cache             | Input                      | Provider Cache Switch     | TTL 15m service:{channel_no}:{service_no}:data |
| Provider Cache Switch| Switch                        | Branch on cache availability                   | Provider Cache             | Parse Provider, Load Provider Data | NO CACHE FOR demo              |
| Load Provider Data   | Postgres                      | Load provider data from external DB           | Provider Cache Switch       | If Active                |                                 |
| If Active            | If                            | Check if provider is active                    | Load Provider Data          | Save Provider Cache, Test Output |                                 |
| Save Provider Cache  | Redis                         | Save provider data in cache                    | If Active                  | Provider                 | TTL 15m                        |
| Parse Provider       | Code                          | Parse provider data into usable format         | Provider Cache Switch       | Provider                 |                                 |
| Provider             | Set                           | Prepare provider data for usage increment      | Save Provider Cache, Parse Provider | Provider Number          |                                 |
| Provider Number      | Redis                         | Increment provider usage number                 | Provider                   | AI Agent                 | {session_id}:service:providers TTL 5m |
| AI Agent             | Langchain Agent               | AI estimation and booking creation              | Provider Number            | Code, Error Output1      |                                 |
| Calculator           | Langchain Tool Calculator     | Provides calculation tool for AI Agent         | Invoked by AI Agent        | AI Agent                 |                                 |
| xAI @grok-2-1212     | Langchain AI Language Model   | Language model backend for AI Agent             | Invoked by AI Agent        | AI Agent                 |                                 |
| Create Booking Data  | Postgres Tool                 | Insert booking data into database                | Invoked by AI Agent        | AI Agent                 |                                 |
| Code                 | Code                          | Validate AI output and prepare for scoring      | AI Agent                   | If Valid?, Error Output1 |                                 |
| If Valid?            | If                            | Check validity of AI output                      | Code                       | If Score, Test Output    |                                 |
| If Score             | If                            | Decide if output includes score                  | If Valid?                  | Output w/ Score, Output w/o Score |                                 |
| Output w/ Score      | Set                           | Format output including score                     | If Score                   | Call Back                |                                 |
| Output w/o Score     | Set                           | Format output without score                       | If Score                   | Call Back                |                                 |
| Call Back            | Execute Workflow              | Send response back via sub-workflow               | Output w/ Score, Output w/o Score, Error Output1 | None                    | Demo Call Back                 |
| Error Output1        | Set                           | Sets error message "Please retry."                 | AI Agent, Code             | Call Back                | Please retry.                  |
| Test Output          | Set                           | Placeholder output for inactive or invalid cases  | If Active, If Valid? (false) | None                    |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Flow Trigger" node**  
   - Type: ExecuteWorkflowTrigger  
   - No parameters.  
   - Position: Entry point for receiving taxi request messages.

2. **Create "Input" node**  
   - Type: Set  
   - Purpose: Initialize input parameters such as `provider_no`.  
   - Connect output of "Flow Trigger" to this node.

3. **Create "Test Trigger" node** (optional, for chat testing)  
   - Type: Chat Trigger (Langchain)  
   - Configure webhook ID for external chat integration.  
   - Connect no input (standalone trigger).

4. **Create "Test Fields" node**  
   - Type: Set  
   - Purpose: Set test input fields like `service_no`.  
   - Connect output of "Test Trigger" to this node.  
   - Connect output of "Test Fields" into "Input" node to unify the path.

5. **Create "Provider Cache" node**  
   - Type: Redis  
   - Configure Redis credentials.  
   - Set TTL to 15 minutes with key pattern `service:{channel_no}:{service_no}:data`.  
   - Connect output of "Input" to this node.

6. **Create "Provider Cache Switch" node**  
   - Type: Switch  
   - Configure conditions to detect cache availability.  
   - Connect output of "Provider Cache" to this switch.  
   - One output branch leads to "Parse Provider" (cache hit), other to "Load Provider Data" (cache miss).

7. **Create "Load Provider Data" node**  
   - Type: Postgres  
   - Configure Postgres credentials pointing to provider database.  
   - Write SQL query to fetch provider info from `sys_provider` or your own table.  
   - Set to execute once.  
   - Connect output of "Provider Cache Switch" cache miss branch to this node.

8. **Create "If Active" node**  
   - Type: If  
   - Condition: Check if provider active flag is true.  
   - Connect output of "Load Provider Data" to this node.

9. **Create "Save Provider Cache" node**  
   - Type: Redis  
   - Configure Redis credentials.  
   - Set TTL 15 minutes.  
   - Connect true output of "If Active" to this node.

10. **Create "Parse Provider" node**  
    - Type: Code  
    - Write JavaScript code to parse provider data into usable format.  
    - Connect cache hit output of "Provider Cache Switch" and output of "Save Provider Cache" to this node.

11. **Create "Provider" node**  
    - Type: Set  
    - Prepare provider data for usage count increment.  
    - Connect output of "Parse Provider" to this node.

12. **Create "Provider Number" node**  
    - Type: Redis  
    - Configure Redis credentials.  
    - Key pattern: `{session_id}:service:providers`, TTL 5 minutes.  
    - Configure to increment a counter.  
    - Connect output of "Provider" node to this node.

13. **Create "AI Agent" node**  
    - Type: Langchain Agent  
    - Configure AI model, tools, and prompt for fare estimation and booking creation.  
    - Link the Calculator and Create Booking Data nodes as tools.  
    - Connect output of "Provider Number" to this node.  
    - Set error mode to continue on error output.

14. **Create "Calculator" node**  
    - Type: Langchain Tool Calculator  
    - Connect to AI Agent as a tool.

15. **Create "Create Booking Data" node**  
    - Type: Postgres Tool  
    - Configure Postgres credentials for booking database.  
    - Connect as a tool to AI Agent.

16. **Create "xAI @grok-2-1212" node**  
    - Type: Langchain AI Language Model  
    - Configure with desired AI model parameters.  
    - Connect to AI Agent as language model.

17. **Create "Code" node**  
    - Type: Code  
    - Write JavaScript to validate AI output and prepare for scoring decision.  
    - Connect output of "AI Agent" to this node.  
    - On error, continue to error output.

18. **Create "If Valid?" node**  
    - Type: If  
    - Condition: AI output validity check.  
    - Connect output of "Code" node.

19. **Create "If Score" node**  
    - Type: If  
    - Condition: Check if output includes score.  
    - Connect true output of "If Valid?" to this node.

20. **Create "Output w/ Score" node**  
    - Type: Set  
    - Format output including score.  
    - Connect true output of "If Score" to this node.

21. **Create "Output w/o Score" node**  
    - Type: Set  
    - Format output without score.  
    - Connect false output of "If Score" to this node.

22. **Create "Call Back" node**  
    - Type: Execute Workflow  
    - Configure to execute "Demo Call Back" sub-workflow or your own callback logic.  
    - Connect outputs of "Output w/ Score", "Output w/o Score", and "Error Output1" to this node.

23. **Create "Error Output1" node**  
    - Type: Set  
    - Set error message: "Please retry."  
    - Connect error outputs from AI Agent and Code node to this node.

24. **Create "Test Output" node**  
    - Type: Set  
    - Placeholder output for inactive or invalid cases.  
    - Connect false outputs of "If Active" and "If Valid?" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                        | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Pull and Set up required SQL from GitHub repository                                                | https://github.com/ChatPayLabs/n8n-chatbot-core                                                                    |
| Create Redis credentials as per n8n integration documentation                                     | https://docs.n8n.io/integrations/builtin/credentials/slack/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal |
| Create Postgres credentials as per n8n integration documentation                                  | https://docs.n8n.io/integrations/builtin/credentials/slack/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal |
| The workflow uses a scaling design to support n8n queue mode in production                        |                                                                                                                     |
| This template uses a flexible prompt design to customize taxi providers using cached provider data |                                                                                                                     |
| Sub-workflow "Demo Call Back" must be created or replaced with your own callback workflow          |                                                                                                                     |

---

**Disclaimer:** The provided text is exclusively sourced from an automated workflow built with n8n, a workflow automation tool. It complies strictly with in-force content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.