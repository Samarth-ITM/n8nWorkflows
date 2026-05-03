Generate & Explain Excel/Google Sheets Formulas with DeepSeek AI

https://n8nworkflows.xyz/workflows/generate---explain-excel-google-sheets-formulas-with-deepseek-ai-6396


# Generate & Explain Excel/Google Sheets Formulas with DeepSeek AI

### 1. Workflow Overview

This workflow automates the generation and explanation of Excel or Google Sheets formulas using AI, specifically leveraging DeepSeek AI via OpenRouter for language model integration. It is designed to receive input requests through a webhook, detect the user's intent (either to generate a formula or to explain one), process the request using AI agents, and then respond with the AI-generated formula or explanation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming HTTP requests via a webhook and preprocesses them for intent detection.
- **1.2 Intent Detection & Routing:** Determines whether the user wants to generate a new formula or get an explanation for an existing formula, then routes the flow accordingly.
- **1.3 AI Processing:** Uses AI agents powered by OpenRouter chat models to generate or explain formulas.
- **1.4 Post-processing and Response:** Processes AI output and sends the final response back to the requester via the webhook response node.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures incoming HTTP requests containing user input and initiates workflow execution.

- **Nodes Involved:**  
  - Webhook  
  - Detect Intent (Set node)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (HTTP trigger)  
    - Configuration: Default settings, receives POST requests at a unique webhook URL. No additional parameters configured.  
    - Inputs: None (entry point)  
    - Outputs: Data payload from the incoming HTTP request.  
    - Edge cases: Missing or malformed request body, unsupported HTTP methods.  
    - Notes: This node is the primary entry point for external triggers.

  - **Detect Intent**  
    - Type: Set node  
    - Role: Prepares or extracts relevant data fields from the incoming webhook data to detect user intent.  
    - Configuration: No explicit parameters shown; likely used to set or transform input data for downstream logic.  
    - Inputs: Data from Webhook node.  
    - Outputs: Data with intent-related fields set or transformed for the Intent Switch node.  
    - Edge cases: If expected data fields are missing or improperly formatted, intent detection may fail or misroute.

#### 1.2 Intent Detection & Routing

- **Overview:**  
  This block evaluates the user's intent and routes the workflow to the appropriate AI agent for formula generation or explanation.

- **Nodes Involved:**  
  - Intent Switch

- **Node Details:**

  - **Intent Switch**  
    - Type: Switch node  
    - Role: Branches the workflow based on the detected intent in the data (e.g., "generate formula" vs. "explain formula").  
    - Configuration: Uses conditions (not explicitly shown) to direct flow to either "Generate Formula" or "Explain Formula" nodes.  
    - Inputs: Data from Detect Intent.  
    - Outputs: Two main branches - one to Generate Formula, one to Explain Formula.  
    - Edge cases: Unrecognized or missing intent values can cause no path to be taken, resulting in workflow dead-end or errors.

#### 1.3 AI Processing

- **Overview:**  
  This block sends the user input to AI agents using OpenRouter chat models to either generate a new formula or explain an existing one. It integrates two AI agents specialized for these tasks.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - OpenRouter Chat Model1  
  - Generate Formula  
  - Explain Formula

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: Langchain OpenRouter Chat Model node  
    - Role: Provides a language model interface for the Generate Formula agent node.  
    - Configuration: Default parameters; connects as the AI language model for "Generate Formula".  
    - Inputs: Not direct input from workflow nodes; rather, it acts as a language model provider.  
    - Outputs: Supplies AI-generated content to "Generate Formula".  
    - Edge cases: API authentication failures, rate limits, model unavailability.

  - **OpenRouter Chat Model1**  
    - Type: Langchain OpenRouter Chat Model node  
    - Role: Provides a language model interface for the Explain Formula agent node.  
    - Configuration: Default parameters; connected as AI language model for "Explain Formula".  
    - Inputs/Outputs: Same as above but linked to "Explain Formula".  
    - Edge cases: Same as above.

  - **Generate Formula**  
    - Type: Langchain Agent node  
    - Role: Implements an AI agent designed to generate Excel or Google Sheets formulas based on user input.  
    - Configuration: Uses OpenRouter Chat Model for AI responses.  
    - Inputs: Routed from Intent Switch when intent is to generate formulas.  
    - Outputs: AI-generated formula text.  
    - Edge cases: Ambiguous or incomplete user input may cause irrelevant or incorrect formulas.

  - **Explain Formula**  
    - Type: Langchain Agent node  
    - Role: AI agent that explains an existing Excel or Google Sheets formula provided by the user.  
    - Configuration: Uses OpenRouter Chat Model1 for AI responses.  
    - Inputs: Routed from Intent Switch when intent is to explain formulas.  
    - Outputs: AI-generated formula explanations.  
    - Edge cases: Incorrect or malformed formulas as input may cause failed or inaccurate explanations.

#### 1.4 Post-processing and Response

- **Overview:**  
  Processes AI output and sends the response back to the user via the webhook response node.

- **Nodes Involved:**  
  - Code  
  - Respond to Webhook

- **Node Details:**

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Processes or reformats AI output before responding. May handle data shaping or error handling.  
    - Configuration: Script code not disclosed, but likely processes input from either Generate Formula or Explain Formula.  
    - Inputs: AI agent outputs from either Generate Formula or Explain Formula.  
    - Outputs: Processed data for final response.  
    - Edge cases: Script errors, unexpected AI output format, null or empty data.

  - **Respond to Webhook**  
    - Type: Respond to Webhook node  
    - Role: Sends the final processed data back as HTTP response to the initial webhook request.  
    - Configuration: Default parameters, responds to the originating request.  
    - Inputs: Output from Code node.  
    - Outputs: HTTP response to client.  
    - Edge cases: Response timeouts if AI processing is slow, malformed response data.

---

### 3. Summary Table

| Node Name            | Node Type                                   | Functional Role                      | Input Node(s)         | Output Node(s)         | Sticky Note                         |
|----------------------|---------------------------------------------|------------------------------------|-----------------------|------------------------|-----------------------------------|
| Webhook              | Webhook                                     | Entry point; receives HTTP requests| None                  | Detect Intent          |                                   |
| Detect Intent        | Set                                         | Prepares data for intent routing   | Webhook               | Intent Switch          |                                   |
| Intent Switch        | Switch                                      | Routes workflow based on intent    | Detect Intent          | Generate Formula, Explain Formula |                                   |
| OpenRouter Chat Model| Langchain OpenRouter Chat Model              | AI language model for formula generation | Connected internally to Generate Formula | Generate Formula (as AI model) |                                   |
| OpenRouter Chat Model1| Langchain OpenRouter Chat Model             | AI language model for formula explanation | Connected internally to Explain Formula | Explain Formula (as AI model) |                                   |
| Generate Formula     | Langchain Agent                             | Generates Excel/Sheets formulas    | Intent Switch          | Code                   |                                   |
| Explain Formula      | Langchain Agent                             | Explains formulas                  | Intent Switch          | Code                   |                                   |
| Code                 | Code                                        | Processes AI output before response | Generate Formula, Explain Formula | Respond to Webhook      |                                   |
| Respond to Webhook   | Respond to Webhook                          | Sends HTTP response back to client | Code                  | None                   |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node:**  
   - Type: Webhook  
   - Configure to accept POST requests.  
   - Note down the webhook URL for external calls.

2. **Add a Set Node (Detect Intent):**  
   - Connect Webhook node output to this node.  
   - Configure to extract or set fields necessary for intent detection (e.g., input text, action keywords).

3. **Add a Switch Node (Intent Switch):**  
   - Connect Detect Intent node output to this node.  
   - Configure conditions to detect intent values such as:  
     - Condition 1: If intent equals "generate formula" → route to Generate Formula node.  
     - Condition 2: If intent equals "explain formula" → route to Explain Formula node.

4. **Add Two Langchain OpenRouter Chat Model Nodes:**  
   - Create one node named "OpenRouter Chat Model" for formula generation.  
   - Create another node named "OpenRouter Chat Model1" for formula explanation.  
   - Set up credentials for OpenRouter API access (requires API keys).  
   - Ensure both nodes are configured with default or custom AI model parameters suitable for chat completion.

5. **Add Two Langchain Agent Nodes:**  
   - "Generate Formula": Connect input from the appropriate Intent Switch output branch.  
     - Set AI language model to "OpenRouter Chat Model".  
     - Configure prompt templates or agent behavior to generate Excel/Sheets formulas based on input text.  
   - "Explain Formula": Connect input from the other Intent Switch output branch.  
     - Set AI language model to "OpenRouter Chat Model1".  
     - Configure prompt templates or agent behavior to explain formulas passed as input.

6. **Add a Code Node:**  
   - Connect outputs from both "Generate Formula" and "Explain Formula" nodes into this Code node (merge inputs if needed).  
   - Write JavaScript code to process AI responses, handle formatting, and prepare the final output JSON.  
   - Include error handling for empty or malformed AI outputs.

7. **Add a Respond to Webhook Node:**  
   - Connect the Code node output to this node.  
   - Configure it to respond to the original webhook request with the processed AI output.

8. **Verify Credentials:**  
   - Set up OpenRouter API credentials in n8n credentials manager.  
   - Test API connectivity.

9. **Activate Workflow:**  
   - Save and activate the workflow.  
   - Test by sending HTTP requests with payloads indicating either formula generation or explanation requests.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                              |
|-----------------------------------------------------------------------------------------------|----------------------------------------------|
| The workflow uses OpenRouter as a chat model provider for DeepSeek AI integration.            | OpenRouter API documentation                  |
| Ensure that webhook endpoint is secured if exposed publicly to avoid unauthorized usage.      | n8n Webhook security best practices           |
| Workflow assumes input payloads contain clear intent indicators for routing (e.g., 'generate formula' or 'explain formula'). | Workflow input design principle               |
| For advanced use, consider adding validation nodes before AI agents to sanitize user input.   | General input validation recommendation       |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.