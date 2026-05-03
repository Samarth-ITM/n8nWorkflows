Route User Requests to Specialized Agents with GPT-4o Mini

https://n8nworkflows.xyz/workflows/route-user-requests-to-specialized-agents-with-gpt-4o-mini-4150


# Route User Requests to Specialized Agents with GPT-4o Mini

---

### 1. Workflow Overview

This workflow is designed to route user requests intelligently to specialized sub-agents using GPT-4o Mini via OpenRouter API. It acts as a conversational router that receives user input through a webhook, determines the user’s intent with an AI agent configured as a router, and directs the request to one of several dedicated sub-workflows (agents) specialized for reminders, emails, meetings, or document processing. The workflow then responds back with the agent’s output.

Logical blocks:

- **1.1 Input Reception:** Receives user requests via webhook.
- **1.2 AI Routing Agent:** Uses GPT-4o Mini and LangChain nodes to parse and classify user intent.
- **1.3 Agent Routing Switch:** Routes the request to one of four specialized sub-workflows according to the AI’s decision.
- **1.4 Specialized Agents Execution:** Executes sub-workflows for Reminder, Email, Meeting, or Document handling.
- **1.5 Response Delivery:** Sends the sub-agent’s response back to the user via webhook response nodes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives incoming POST requests from users containing their messages and triggers the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: HTTP Webhook Trigger  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook ID path  
      - Allowed Origins: All (*)  
      - Response Mode: Response Node (delays response until workflow completes)  
    - Input/Output: Receives JSON payload typically containing user message in `body.message`  
    - Failure Modes: Invalid payload, network errors, unauthorized origins (none restricted here)  
    - Version: 2

#### 2.2 AI Routing Agent

- **Overview:**  
  This block uses GPT-4o Mini as a routing agent to analyze the user message, decide which specialized sub-agent should handle the request, and parse its structured output using LangChain output parsers.

- **Nodes Involved:**  
  - Postgres Chat Memory  
  - GPT 4o Mini  
  - Auto-fixing Output Parser  
  - Structured Output Parser  
  - AI Agent

- **Node Details:**  
  - **Postgres Chat Memory**  
    - Type: LangChain Postgres Chat Memory  
    - Role: Stores and retrieves conversation history keyed by user message to provide context to AI  
    - Config: `sessionKey` set dynamically from webhook message content  
    - Credentials: Postgres database configured  
    - Edge Cases: DB connection failure, missing session keys  
    - Version: 1.3

  - **GPT 4o Mini**  
    - Type: LangChain LM Chat Open Router (GPT-4o Mini via OpenRouter API)  
    - Role: Provides the language model interface for routing agent  
    - Credentials: OpenRouter API credentials  
    - Version: 1

  - **Auto-fixing Output Parser**  
    - Type: LangChain Output Parser with Autofixing  
    - Role: Attempts to fix AI output if it doesn't conform to expected structured output constraints  
    - Config: Custom prompt instructing how to fix errors in AI completion  
    - Input: Output from Structured Output Parser  
    - Output: Corrected structured data for routing  
    - Edge Cases: Parser failure if output is severely malformed  
    - Version: 1

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI output into a JSON object with defined schema fields: Agent Name, sessionID, user input  
    - Config: JSON schema example provided  
    - Input: Raw AI output from GPT 4o Mini  
    - Output: Parsed structured object for routing decision  
    - Version: 1.2

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Orchestrates routing decision by calling GPT 4o Mini with a detailed system prompt describing sub-agent roles and instructions  
    - Config: Custom prompt clearly enumerating specialized agents, usage instructions, and routing rules; uses output parser and memory nodes  
    - Input: User message from webhook, enriched by chat memory  
    - Output: JSON with Agent Name and user input for routing  
    - Edge Cases: AI misclassification, timeout, malformed output  
    - Version: 1.8

#### 2.3 Agent Routing Switch

- **Overview:**  
  This block routes the parsed and classified user request to the correct specialized sub-workflow based on the agent name received from AI Agent.

- **Nodes Involved:**  
  - Agent Route (Switch)

- **Node Details:**  
  - **Agent Route**  
    - Type: Switch (conditional branching)  
    - Role: Routes workflow execution to one of four sub-agents: Reminder Agent, Email Agent, Meeting Agent, Document Agent  
    - Config: Exact matching rules on `output["Agent Name"]` field for cases: "Reminder Agent", "Email Agent", "Meeting Agent", "Document Agent"  
    - Input: AI Agent’s parsed output JSON  
    - Output: Branches to corresponding sub-workflow execute nodes  
    - Edge Cases: Unrecognized agent name leads to no branch triggered (workflow halts or error)  
    - Version: 3.2

#### 2.4 Specialized Agents Execution

- **Overview:**  
  This block executes the selected specialized sub-workflow passing the original user input for processing.

- **Nodes Involved:**  
  - Reminder Agent (Execute Workflow)  
  - Email Agent (Execute Workflow)  
  - Meeting Agent (Execute Workflow)  
  - Document Agent (Execute Workflow)

- **Node Details:**  
  - **Reminder Agent**  
    - Type: Execute Workflow  
    - Role: Runs Reminder Agent sub-workflow with user input parameter mapped from AI Agent output  
    - Config: Workflow ID linked, input mapping on `Query` parameter from `output["user input"]`  
    - Edge Cases: Sub-workflow failure, input mapping errors  
    - Version: 1.2

  - **Email Agent**  
    - Type: Execute Workflow  
    - Role: Runs Email Agent sub-workflow with user input parameter  
    - Config: Workflow ID linked, input mapping on `User Input` from AI Agent output  
    - Edge Cases: Sub-workflow failure, invalid email addresses in input  
    - Version: 1.2

  - **Meeting Agent**  
    - Type: Execute Workflow  
    - Role: Runs Meeting Agent sub-workflow with user input parameter  
    - Config: Workflow ID linked, input mapping on `User Input` from AI Agent output  
    - Edge Cases: Sub-workflow failure, calendar API issues  
    - Version: 1.2

  - **Document Agent**  
    - Type: Execute Workflow  
    - Role: Runs Document Agent sub-workflow with user input parameter  
    - Config: Workflow ID linked, input mapping on `User Input` from AI Agent output  
    - Edge Cases: Sub-workflow failure, document API errors  
    - Version: 1.2

#### 2.5 Response Delivery

- **Overview:**  
  This block sends the response generated by the specialized agent back to the user via the webhook response nodes.

- **Nodes Involved:**  
  - Reminder Agent Response (Respond to Webhook)  
  - Email Agent Response (Respond to Webhook)  
  - Meeting Agent Response (Respond to Webhook)  
  - Document Agent2 (Respond to Webhook)

- **Node Details:**  
  - All **Respond to Webhook** nodes share:  
    - Type: Respond to Webhook  
    - Role: Return HTTP 200 with text-based response body from agent output  
    - Config: Response code 200, responseType text, responseBody mapped from sub-agent output JSON `output` field  
    - Input: Output from respective sub-workflow execution node  
    - Edge Cases: Response failures if output missing or malformed  
    - Versions: 1.1

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                         | Input Node(s)        | Output Node(s)               | Sticky Note                                                                                                    |
|-------------------------|-------------------------------------|---------------------------------------|----------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook                 | Webhook                             | Receives user input                   | -                    | AI Agent                    |                                                                                                               |
| GPT 4o Mini             | LangChain LM Chat Open Router       | Provides GPT-4o Mini model for agent | Postgres Chat Memory  | AI Agent                    |                                                                                                               |
| Postgres Chat Memory    | LangChain Memory Postgres Chat      | Stores conversation context           | Webhook               | GPT 4o Mini                 |                                                                                                               |
| AI Agent                | LangChain Agent                     | Routes user requests to agents        | Webhook, Auto-fixing Output Parser | Agent Route                 | Routing instructions include detailed agent descriptions and rules.                                           |
| Structured Output Parser| LangChain Structured Output Parser  | Parses AI output into structured JSON | GPT 4o Mini            | Auto-fixing Output Parser   |                                                                                                               |
| Auto-fixing Output Parser| LangChain Output Parser Autofixing  | Fixes structured output errors        | Structured Output Parser | AI Agent                    |                                                                                                               |
| Agent Route             | Switch                             | Routes to correct sub-agent workflow  | AI Agent               | Reminder Agent, Email Agent, Meeting Agent, Document Agent |                                                                                                               |
| Reminder Agent          | Execute Workflow                   | Handles reminder-related requests     | Agent Route            | Reminder Agent Response     |                                                                                                               |
| Reminder Agent Response | Respond to Webhook                 | Returns reminder agent output          | Reminder Agent         | -                           |                                                                                                               |
| Email Agent             | Execute Workflow                   | Handles email-related requests        | Agent Route            | Email Agent Response        |                                                                                                               |
| Email Agent Response    | Respond to Webhook                 | Returns email agent output             | Email Agent            | -                           |                                                                                                               |
| Meeting Agent           | Execute Workflow                   | Handles meeting scheduling requests   | Agent Route            | Meeting Agent Response      |                                                                                                               |
| Meeting Agent Response  | Respond to Webhook                 | Returns meeting agent output           | Meeting Agent          | -                           |                                                                                                               |
| Document Agent          | Execute Workflow                   | Handles document generation/editing   | Agent Route            | Document Agent2             |                                                                                                               |
| Document Agent2         | Respond to Webhook                 | Returns document agent output          | Document Agent         | -                           |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Method: POST  
   - Path: Unique identifier (e.g., `3576c6b9-11a2-4375-b7cb-f58e36557a7b`)  
   - Allowed Origins: `*`  
   - Response Mode: Response Node  
   - Position: (240, 800)

2. **Set up Postgres Chat Memory Node:**  
   - Type: LangChain Postgres Chat Memory  
   - Credentials: Configure Postgres DB  
   - Session Key: Expression `{{$node["Webhook"].json.body.message}}`  
   - Session ID Type: Custom Key  
   - Position: (580, 1020)

3. **Create GPT 4o Mini Node:**  
   - Type: LangChain LM Chat Open Router  
   - Credentials: OpenRouter API (GPT-4o Mini)  
   - Position: (460, 1020)

4. **Create Structured Output Parser Node:**  
   - Type: LangChain Structured Output Parser  
   - JSON Schema Example:  
     ```json
     {
       "Agent Name": "Agent Name",
       "sessionID": "Session ID",
       "user input": "user input"
     }
     ```  
   - Position: (840, 1220)

5. **Create Auto-fixing Output Parser Node:**  
   - Type: LangChain Output Parser Autofixing  
   - Prompt:  
     ```
     Instructions:
     --------------
     {instructions}
     --------------
     Completion:
     --------------
     {completion}
     --------------
     
     Above, the Completion did not satisfy the constraints given in the Instructions.
     Error:
     --------------
     {error}
     --------------
     
     Please try again. Please only respond with an answer that satisfies the constraints laid out in the Instructions:
     ```  
   - Position: (700, 1040)

6. **Create AI Agent Node:**  
   - Type: LangChain Agent  
   - Prompt: Use the detailed routing instructions provided, including descriptions of Reminder Agent, Email Agent, Meeting Agent, Document Agent, and ATS Agent.  
   - Enable Output Parser; connect to Auto-fixing Output Parser as output parser node.  
   - Connect Postgres Chat Memory as memory input.  
   - Use GPT 4o Mini as LM model input.  
   - Position: (500, 800)

7. **Connect Nodes for AI Routing:**  
   - Webhook → AI Agent  
   - AI Agent → Agent Route  
   - GPT 4o Mini → Structured Output Parser  
   - Structured Output Parser → Auto-fixing Output Parser  
   - Auto-fixing Output Parser → AI Agent (output parser)

8. **Create Agent Route Node:**  
   - Type: Switch  
   - Conditions:  
     - If `output["Agent Name"]` equals "Reminder Agent" → Reminder Agent node  
     - If equals "Email Agent" → Email Agent node  
     - If equals "Meeting Agent" → Meeting Agent node  
     - If equals "Document Agent" → Document Agent node  
   - Position: (1080, 780)

9. **Create Sub-Workflow Execute Nodes:**  
   - Reminder Agent Execute Workflow:  
     - Workflow ID: Reminder Agent sub-workflow  
     - Input: Map `Query` to `output["user input"]`  
     - Position: (1280, 500)

   - Email Agent Execute Workflow:  
     - Workflow ID: Email Agent sub-workflow  
     - Input: Map `User Input` to `output["user input"]`  
     - Position: (1280, 700)

   - Meeting Agent Execute Workflow:  
     - Workflow ID: Meeting Agent sub-workflow  
     - Input: Map `User Input` to `output["user input"]`  
     - Position: (1280, 900)

   - Document Agent Execute Workflow:  
     - Workflow ID: Document Agent sub-workflow  
     - Input: Map `User Input` to `output["user input"]`  
     - Position: (1280, 1100)

10. **Create Respond to Webhook Nodes:**  
    - Reminder Agent Response:  
      - Response code: 200  
      - Respond with: Text  
      - Body: `={{ $json.output }}`  
      - Position: (1500, 500)

    - Email Agent Response:  
      - Same config as above  
      - Position: (1500, 700)

    - Meeting Agent Response:  
      - Same config as above  
      - Position: (1500, 900)

    - Document Agent Response (Document Agent2):  
      - Same config as above  
      - Position: (1520, 1100)

11. **Connect Sub-Workflow Outputs to Response Nodes:**  
    - Reminder Agent → Reminder Agent Response  
    - Email Agent → Email Agent Response  
    - Meeting Agent → Meeting Agent Response  
    - Document Agent → Document Agent2

12. **Credential Setup:**  
    - Configure OpenRouter API credentials for GPT 4o Mini and Output Parser Model nodes.  
    - Configure Postgres credentials for Postgres Chat Memory node.  
    - Ensure all sub-workflows are properly deployed and accessible by their workflow IDs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| The AI Agent prompt includes precise instructions to avoid follow-up questions and to route only one sub-workflow per request.                                                             | See AI Agent node prompt                                                                                          |
| OpenRouter API is used for GPT-4o Mini model integration, requiring appropriate API key credentials.                                                                                        | `OpenRouter account - sentiimenta.ai` credential                                                                 |
| Postgres Chat Memory enables session-based context handling, improving AI routing accuracy.                                                                                                  | Requires Postgres DB connection                                                                                   |
| Sub-workflows for Reminder, Email, Meeting, and Document agents must be developed separately and linked via workflow IDs.                                                                     | Workflow IDs referenced in Execute Workflow nodes                                                                 |
| The workflow uses LangChain nodes for advanced AI output parsing and error correction, improving structured data reliability.                                                                | Auto-fixing Output Parser node                                                                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n integration and automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---