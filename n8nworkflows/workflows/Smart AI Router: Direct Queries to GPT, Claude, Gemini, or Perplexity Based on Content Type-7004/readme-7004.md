Smart AI Router: Direct Queries to GPT, Claude, Gemini, or Perplexity Based on Content Type

https://n8nworkflows.xyz/workflows/smart-ai-router--direct-queries-to-gpt--claude--gemini--or-perplexity-based-on-content-type-7004


# Smart AI Router: Direct Queries to GPT, Claude, Gemini, or Perplexity Based on Content Type

### 1. Workflow Overview

This workflow, titled **Smart AI Router: Direct Queries to GPT, Claude, Gemini, or Perplexity Based on Content Type**, is designed to intelligently route user chat messages to the most appropriate large language model (LLM) based on the classification of the input request. It targets use cases where diverse query types (coding, reasoning, general questions, or search-related) need to be handled by specialized AI models to optimize response relevance, efficiency, and cost.

The logical flow is organized into the following blocks:

- **1.1 Input Reception:** Captures chat messages via a webhook trigger.
- **1.2 Request Classification:** Uses an LLM chain to classify the input into a request type.
- **1.3 Structured Parsing:** Parses the classification output into a structured format.
- **1.4 Model Selection:** Applies rules to select the appropriate AI model based on the request type.
- **1.5 AI Processing:** Routes the message through the selected AI model with conversational memory support.
- **1.6 Memory Handling:** Maintains conversation context to improve AI responses.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming chat messages from users, triggering the workflow.
- **Nodes Involved:** `When chat message received`
- **Node Details:**
  - **Type:** LangChain Chat Trigger node; acts as webhook listener.
  - **Configuration:** Default webhook setup with no additional options.
  - **Key Expressions:** Receives `chatInput` and `sessionId` from incoming JSON.
  - **Input:** External chat message via webhook.
  - **Output:** Forwards chat input for classification.
  - **Potential Failures:** Webhook connection issues, malformed input JSON.
  - **Version:** 1.1

#### 1.2 Request Classification

- **Overview:** Uses a prompt-based LLM chain to classify the type of user request.
- **Nodes Involved:** `Request Type`
- **Node Details:**
  - **Type:** LangChain Chain LLM node.
  - **Configuration:** 
    - Custom prompt instructs to classify request into one of four categories: general, reasoning, coding, google.
    - Output is expected to be parsed by a structured output parser downstream.
  - **Key Expressions:** Message content passed from `When chat message received` node.
  - **Input:** Raw user chat input.
  - **Output:** Text output indicating request type.
  - **Potential Failures:** LLM timeout, parsing errors if output format is incorrect.
  - **Version:** 1.7

#### 1.3 Structured Parsing

- **Overview:** Parses the raw classification result into a strict JSON object for downstream logic.
- **Nodes Involved:** `Structured Output Parser`
- **Node Details:**
  - **Type:** LangChain Structured Output Parser.
  - **Configuration:** Manual JSON schema defining a single property `request_type` of string type.
  - **Input:** Raw text output from `Request Type` node.
  - **Output:** JSON object with property `request_type`.
  - **Potential Failures:** Schema mismatch, malformed input from LLM.
  - **Version:** 1.3

#### 1.4 Model Selection

- **Overview:** Selects the target AI model index based on the parsed request type using predefined rules.
- **Nodes Involved:** `Model Selector`
- **Node Details:**
  - **Type:** LangChain Model Selector node.
  - **Configuration:** 
    - Rules match `request_type` to model indices:
      - `coding` → index 0 (Opus 4 / Claude)
      - `reasoning` → index 2 (GPT 4.1 mini)
      - `general` → index 3 (Perplexity)
      - `search` → index 4 (Perplexity)
    - Case-sensitive, strict string equality used.
  - **Input:** JSON from `Structured Output Parser`.
  - **Output:** Selected model index forwarded to AI Agent node.
  - **Potential Failures:** Unmatched request_type causing no model to be selected.
  - **Version:** 1

#### 1.5 AI Processing

- **Overview:** Processes the input query using the selected model with conversational memory.
- **Nodes Involved:** 
  - AI models: `Opus 4` (Claude), `Gemini Thinking Pro`, `GPT 4.1 mini`, `Perplexity`
  - `AI Agent`
  - `Simple Memory`
- **Node Details:**

  - **AI Agent**
    - **Type:** LangChain Agent node.
    - **Role:** Central orchestrator that receives input text, selected model, and memory to generate a response.
    - **Configuration:** Takes chat input from the initial trigger; returns intermediate steps.
    - **Input:** Receives the selected model from `Model Selector` and session memory.
    - **Output:** Final AI response.
    - **Potential Failures:** Model API errors, memory session issues.

  - **Opus 4**
    - **Type:** LangChain LM Chat node for Anthropic Claude.
    - **Configuration:** Uses `claude-sonnet-4-20250514` model.
    - **Credentials:** Anthropic API key.
    - **Input:** From `Model Selector` via AI Agent.
    - **Output:** AI-generated chat response.
    - **Potential Failures:** API auth errors, rate limits.

  - **Gemini Thinking Pro**
    - **Type:** LangChain LM Chat node for Google Gemini.
    - **Configuration:** Model `gemini-2.0-flash-thinking-exp`.
    - **Credentials:** Google Palm API.
    - **Potential Failures:** OAuth token expiration, API quota.

  - **GPT 4.1 mini**
    - **Type:** LangChain LM Chat node for OpenAI GPT.
    - **Configuration:** Model `gpt-4.1-mini`.
    - **Credentials:** OpenAI API key.
    - **Potential Failures:** API key issues, model availability.

  - **Perplexity**
    - **Type:** LangChain LM Chat node for OpenRouter Perplexity.
    - **Configuration:** Model `perplexity/sonar`.
    - **Credentials:** OpenRouter API key.
    - **Potential Failures:** Network errors, API limits.

  - **Simple Memory**
    - **Type:** LangChain Memory Buffer Window.
    - **Role:** Tracks session conversation using sessionId from chat trigger.
    - **Input:** Session key derived from chat input.
    - **Output:** Provides memory context to AI Agent.
    - **Potential Failures:** Memory overflow, session key mismatches.
    - **Version:** 1.3

#### 1.6 Sticky Note

- **Overview:** Documentation note embedded in the workflow for user reference.
- **Nodes Involved:** `Sticky Note`
- **Node Details:** Contains summary description of the workflow purpose and routing logic.
- **No inputs or outputs.**

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                       | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                       |
|----------------------------|-------------------------------------|------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received  | LangChain Chat Trigger               | Input Reception                    | External webhook              | Request Type                | AI Orchestrator: dynamically Selects Models Based on Input Type (workflow description)          |
| Request Type               | LangChain Chain LLM                  | Request Classification            | When chat message received   | Structured Output Parser    |                                                                                                 |
| Structured Output Parser   | LangChain Structured Output Parser  | Structured Parsing                | Request Type                 | Model Selector              |                                                                                                 |
| Model Selector             | LangChain Model Selector             | Model Selection                   | Structured Output Parser     | AI Agent                   |                                                                                                 |
| AI Agent                  | LangChain Agent                     | AI Processing Orchestration       | Model Selector, Simple Memory|                             |                                                                                                 |
| Simple Memory             | LangChain Memory Buffer Window       | Memory Handling                   | When chat message received   | AI Agent                   |                                                                                                 |
| Opus 4                    | LangChain LM Chat Anthropic          | Claude Model                      | Model Selector               | Model Selector (back to AI Agent) |                                                                                                 |
| Gemini Thinking Pro       | LangChain LM Chat Google Gemini      | Gemini Model                     | Model Selector               | Model Selector (back to AI Agent) |                                                                                                 |
| GPT 4.1 mini              | LangChain LM Chat OpenAI             | GPT Model                        | Model Selector               | Model Selector (back to AI Agent) |                                                                                                 |
| Perplexity                | LangChain LM Chat OpenRouter         | Perplexity Model                 | Model Selector               | Model Selector (back to AI Agent) |                                                                                                 |
| Sticky Note               | Sticky Note                         | Documentation                    | None                        | None                       | ## AI Orchestrator: dynamically Selects Models Based on Input Type. Optimizes response quality. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `When chat message received` node:**
   - Type: LangChain Chat Trigger.
   - Setup webhook for incoming chat messages.
   - No additional parameters.
   
2. **Create `Request Type` node:**
   - Type: LangChain Chain LLM.
   - Configure prompt to classify request types into `general`, `reasoning`, `coding`, or `google`.
   - Use input expression: `{{$node["When chat message received"].json["chatInput"]}}`.
   - Enable output parser.

3. **Create `Structured Output Parser` node:**
   - Type: LangChain Structured Output Parser.
   - Set schema manually as:
     ```json
     {
       "type": "object",
       "properties": {
         "request_type": { "type": "string" }
       }
     }
     ```
   - Connect input from `Request Type` node.

4. **Create `Model Selector` node:**
   - Type: LangChain Model Selector.
   - Add four rules mapping `request_type` to model indices:
     - `"coding"` → model index 0
     - `"reasoning"` → model index 2
     - `"general"` → model index 3
     - `"search"` → model index 4
   - Use strict case-sensitive matching on `{{$json["request_type"]}}`.
   - Connect input from `Structured Output Parser`.

5. **Create AI Model Nodes:**

   - **Opus 4** (Claude)
     - Type: LangChain LM Chat Anthropic.
     - Model: `claude-sonnet-4-20250514`.
     - Set Anthropic API credentials.
   
   - **Gemini Thinking Pro**
     - Type: LangChain LM Chat Google Gemini.
     - Model name: `models/gemini-2.0-flash-thinking-exp`.
     - Set Google Palm API credentials.

   - **GPT 4.1 mini**
     - Type: LangChain LM Chat OpenAI.
     - Model: `gpt-4.1-mini`.
     - Set OpenAI API credentials.

   - **Perplexity**
     - Type: LangChain LM Chat OpenRouter.
     - Model: `perplexity/sonar`.
     - Set OpenRouter API credentials.

6. **Create `Simple Memory` node:**
   - Type: LangChain Memory Buffer Window.
   - Use session key expression: `{{$node["When chat message received"].json["sessionId"]}}`.
   - Session ID type: custom key.

7. **Create `AI Agent` node:**
   - Type: LangChain Agent.
   - Input text expression: `{{$node["When chat message received"].json["chatInput"]}}`.
   - Enable returning intermediate steps.
   - Connect AI memory input from `Simple Memory`.
   - Connect AI language model input from `Model Selector`.

8. **Connect Node Flows:**
   - `When chat message received` → `Request Type`.
   - `Request Type` → `Structured Output Parser`.
   - `Structured Output Parser` → `Model Selector`.
   - `Model Selector` → AI model nodes (`Opus 4`, `Gemini Thinking Pro`, `GPT 4.1 mini`, `Perplexity`) as language model options.
   - AI models → `Model Selector` (for routing back) → `AI Agent`.
   - `Simple Memory` → `AI Agent`.
   - `When chat message received` → `Simple Memory` (to get session key).

9. **Add a Sticky Note:**
   - Content summarizing the workflow purpose, e.g., dynamic routing to specialized AI models based on input type.

10. **Set Workflow Settings:**
    - Execution order: default (v1).
    - Activate workflow only after all credentials are tested.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| AI Orchestrator dynamically selects models based on input type to optimize response quality and efficiency.      | Workflow embedded sticky note for user reference.                                                  |
| For classification, request types are limited to: general, reasoning, coding, or google (search related).         | Classification prompt in `Request Type` node.                                                      |
| Supported AI models: Anthropic Claude, Google Gemini, OpenAI GPT-4.1 mini, and OpenRouter Perplexity.             | Credential requirements: Anthropic API, Google Palm API, OpenAI API, OpenRouter API.                |
| Memory management uses session key for conversational context to maintain coherent multi-turn dialogs.           | Simple Memory node uses sessionId from chat input.                                                 |
| Model Selector uses strict string matching and index-based routing; ensure input request types match exactly.     | Misclassification or unknown request_type values may cause routing failure.                         |
| OpenAI GPT-4.1 mini and other models require valid API keys and may have rate limits or usage quotas.             | Credential setup necessary before activation.                                                      |
| Google Gemini and Anthropic Claude require OAuth tokens or API keys; monitor token expiration for uninterrupted operation. | Credential maintenance critical for stable workflow execution.                                     |

---

*Disclaimer: The provided content is derived exclusively from an n8n automation workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.*