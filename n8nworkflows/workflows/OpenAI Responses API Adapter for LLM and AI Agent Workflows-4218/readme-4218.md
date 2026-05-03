OpenAI Responses API Adapter for LLM and AI Agent Workflows

https://n8nworkflows.xyz/workflows/openai-responses-api-adapter-for-llm-and-ai-agent-workflows-4218


# OpenAI Responses API Adapter for LLM and AI Agent Workflows

### 1. Workflow Overview

This workflow serves as an adapter layer to integrate OpenAI's new Responses API into existing Large Language Model (LLM) and AI Agent workflows in n8n, particularly those using Langchain nodes. It enables seamless use of OpenAI's Responses API, which has a fundamentally different response format and supports multimodal inputs, by intercepting and remapping requests and responses to maintain compatibility with Langchain-based AI nodes.

Logical blocks:

- **1.1 Input Reception & Triggering:** Handles incoming chat messages and webhooks, triggering the AI processing chain.
- **1.2 AI Processing & Agent Invocation:** Runs the AI agent with custom OpenAI credentials, forwarding requests appropriately.
- **1.3 Model Listing Endpoint:** Provides an endpoint that lists available OpenAI models, remapping API responses as needed.
- **1.4 Request Remapping for Responses API:** Transforms incoming chat completion requests into the schema expected by OpenAI’s Responses API.
- **1.5 Responses API Invocation:** Makes HTTP calls to the OpenAI Responses API with remapped request data.
- **1.6 Response Handling & Output Formatting:** Differentiates between streaming and non-streaming responses, formatting them to Langchain-compatible structures and sending proper webhook responses.
- **1.7 Supporting Webhooks & Credentials Setup:** Additional webhooks and notes to guide credential setup and usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Triggering

**Overview:**  
This block listens for incoming chat messages via Langchain's chat trigger node and standard HTTP webhooks, initiating the workflow execution.

**Nodes Involved:**  
- When chat message received  
- Webhook  

**Node Details:**  
- *When chat message received*  
  - Type: Langchain chatTrigger node  
  - Configuration: Public webhook with file uploads enabled (to support multimodal messages)  
  - Inputs: External chat messages from clients or services  
  - Outputs: Triggers AI Agent node  
  - Edge cases: File upload size limits, malformed messages, webhook availability  

- *Webhook*  
  - Type: n8n core webhook node  
  - Configuration: HTTP GET endpoint at `/n8n-responses-api/models` used to fetch model list  
  - Outputs: Triggers "OpenAI Models" HTTP Request node  
  - Edge cases: HTTP method restrictions, webhook path conflicts, slow response  

---

#### 2.2 AI Processing & Agent Invocation

**Overview:**  
Receives chat messages via the Langchain chat trigger node and runs the AI Agent node with a custom OpenAI credential configured to redirect requests to this adapter workflow.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- n8n Webhook (Langchain LLM node)  

**Node Details:**  
- *AI Agent*  
  - Type: Langchain agent node  
  - Configuration: Passthrough of binary images enabled, returns intermediate steps for debugging  
  - Inputs: Triggered from "When chat message received" and "n8n Webhook" (Langchain LLM node)  
  - Outputs: Sends requests to POST ChatCompletions webhook  
  - Edge cases: Authentication failures, network issues, malformed prompts  

- *n8n Webhook*  
  - Type: Langchain LM Chat OpenAI node  
  - Configuration: Model set to "gpt-4o-mini" (a small GPT-4 variant)  
  - Credential: Custom OpenAI credential named "n8n Document Understanding" with altered Base URL pointing to this workflow’s webhooks  
  - Inputs: Can be standalone entry point or used in sub-workflows  
  - Outputs: Connects to AI Agent node  
  - Edge cases: Credential misconfiguration, incorrect Base URL, API limits  

---

#### 2.3 Model Listing Endpoint

**Overview:**  
Provides a REST endpoint to list all OpenAI models by forwarding the request to OpenAI’s models API and remapping the response to a compatible format.

**Nodes Involved:**  
- Webhook  
- OpenAI Models (HTTP Request)  
- Models Response (RespondToWebhook)  

**Node Details:**  
- *OpenAI Models*  
  - Type: HTTP Request node  
  - Configuration: GET request to `https://api.openai.com/v1/models` with OpenAI API credential  
  - Credential: Standard OpenAI API credential  
  - Inputs: Triggered by Webhook node  
  - Outputs: Passes raw model list to Models Response node  
  - Edge cases: API authentication errors, rate limiting, network timeouts  

- *Models Response*  
  - Type: RespondToWebhook node  
  - Configuration: Returns HTTP 200 with JSON content type, responds with raw JSON from OpenAI Models node  
  - Inputs: Receives data from OpenAI Models node  
  - Outputs: HTTP response to original caller  
  - Edge cases: Malformed JSON, webhook response failures  

---

#### 2.4 Request Remapping for Responses API

**Overview:**  
Transforms the incoming chat completion requests into the format required by OpenAI’s Responses API, especially converting message content types (text, image URLs, file URLs) appropriately.

**Nodes Involved:**  
- POST ChatCompletions (Webhook)  
- Remap to Response API Schema (Code)  

**Node Details:**  
- *POST ChatCompletions*  
  - Type: Webhook node  
  - Configuration: HTTP POST at `/n8n-responses-api/chat/completions`  
  - Inputs: Receives Langchain-like chat completion requests from AI Agent node  
  - Outputs: Passes body to remapping code node  
  - Edge cases: HTTP method enforcement, payload size limits  

- *Remap to Response API Schema*  
  - Type: Code node (JavaScript)  
  - Configuration: Runs once per incoming item, maps each message’s content array to a structured input array with explicit types (`input_text`, `input_image`, `input_file`)  
  - Key expressions:  
    - Uses helper functions `tranformContent` and `getInputType` to convert message content  
    - Maps `body.messages` array to new input format  
  - Outputs: JSON payload compatible with OpenAI Responses API format  
  - Edge cases: Unexpected content types, empty messages, malformed input arrays  

---

#### 2.5 Responses API Invocation

**Overview:**  
Sends the remapped request payload to OpenAI’s new Responses API endpoint, authenticating with the custom OpenAI credential.

**Nodes Involved:**  
- OpenAI Responses API (HTTP Request)  

**Node Details:**  
- *OpenAI Responses API*  
  - Type: HTTP Request node  
  - Configuration: POST request to `https://api.openai.com/v1/responses`  
  - Body: JSON containing model, stream flag, and transformed input from previous node  
  - Credential: Same OpenAI API credential as before, with custom base URL set to Responses API  
  - Inputs: From Remap to Response API Schema node  
  - Outputs: Raw responses from Responses API to conditional node  
  - Edge cases: Authentication errors, network issues, incorrect request structure, API rate limits  

---

#### 2.6 Response Handling & Output Formatting

**Overview:**  
Branch based on whether the Responses API returned a streaming response. Formats streaming and non-streaming responses to Langchain-compatible output and returns HTTP responses accordingly.

**Nodes Involved:**  
- Is Agent? (If node)  
- Format Stream Response (Code)  
- Format Completion Response (Code)  
- Text Response (RespondToWebhook)  
- JSON Response (RespondToWebhook)  

**Node Details:**  
- *Is Agent?*  
  - Type: If node  
  - Condition: Checks if `stream` property in the response JSON is true  
  - Inputs: From OpenAI Responses API node  
  - Outputs: Two branches — streaming and non-streaming  

- *Format Stream Response*  
  - Type: Code node (JavaScript)  
  - Configuration: Parses newline-delimited streaming data, extracts final completed response, constructs a Langchain streaming chunk object, and formats data for streaming output with `[DONE]` sentinel  
  - Outputs: Formatted streaming data to Text Response node  
  - Edge cases: Incomplete streaming data, JSON parse errors, missing events  

- *Text Response*  
  - Type: RespondToWebhook node  
  - Configuration: Returns streaming response as plain text to webhook caller  
  - Inputs: From Format Stream Response node  
  - Edge cases: Webhook response failures  

- *Format Completion Response*  
  - Type: Code node (JavaScript)  
  - Configuration: Maps non-streaming response fields to Langchain-compatible chat completion JSON structure, including id, model, choices array, usage tokens, and metadata  
  - Outputs: JSON-formatted response to JSON Response node  
  - Edge cases: Missing fields in response, malformed output arrays  

- *JSON Response*  
  - Type: RespondToWebhook node  
  - Configuration: Returns formatted JSON response with status 200 to webhook caller  
  - Inputs: From Format Completion Response node  
  - Edge cases: Response serialization failures  

---

#### 2.7 Supporting Webhooks & Credentials Setup

**Overview:**  
Includes sticky notes with instructions for credential creation and usage, plus explanations and documentation links.

**Nodes Involved:**  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note  

**Node Details:**  
- *Sticky Note1:* Instructions on creating a custom OpenAI credential with a changed Base URL to point webhook requests to this workflow.  
- *Sticky Note2:* Explains the model listing endpoint and usage of webhook trigger node.  
- *Sticky Note3:* Describes remapping of OpenAI Responses API output for Langchain compatibility.  
- *Sticky Note4:* Detailed usage instructions, workflow explanation, requirements, customization notes, and support links.  
- *Sticky Note:* Guidance on setting up LLM models via n8n webhooks, credential naming, and activation advice.  

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                                    | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                                                     |
|---------------------------|----------------------------------|---------------------------------------------------|----------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger | Entry point for chat message reception            |                            | AI Agent                      | See Sticky Note4 for usage instructions and setup details                                                                     |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent  | Runs AI agent with custom OpenAI credential       | When chat message received, n8n Webhook | POST ChatCompletions (via webhook) | See Sticky Note4 for usage instructions                                                                                        |
| Webhook                   | n8n-nodes-base.webhook           | Endpoint for listing available OpenAI models      |                            | OpenAI Models                 | See Sticky Note2 for model listing explanation                                                                                 |
| OpenAI Models             | n8n-nodes-base.httpRequest       | Calls OpenAI API to list models                    | Webhook                    | Models Response               |                                                                                                                                |
| Models Response           | n8n-nodes-base.respondToWebhook  | Returns model list JSON response                    | OpenAI Models              |                               |                                                                                                                                |
| POST ChatCompletions      | n8n-nodes-base.webhook           | Receives chat completion requests                  | AI Agent                   | Remap to Response API Schema  |                                                                                                                                |
| Remap to Response API Schema | n8n-nodes-base.code            | Transforms chat message content to Responses API schema | POST ChatCompletions       | OpenAI Responses API          | See Sticky Note3 for remapping explanation                                                                                     |
| OpenAI Responses API      | n8n-nodes-base.httpRequest       | Calls OpenAI Responses API with transformed input | Remap to Response API Schema | Is Agent?                    |                                                                                                                                |
| Is Agent?                 | n8n-nodes-base.if                | Branches based on streaming flag in response      | OpenAI Responses API       | Format Stream Response, Format Completion Response |                                                                                                                                |
| Format Stream Response    | n8n-nodes-base.code              | Parses and formats streaming responses             | Is Agent?                  | Text Response                |                                                                                                                                |
| Text Response             | n8n-nodes-base.respondToWebhook  | Returns streaming response as plain text           | Format Stream Response     |                               |                                                                                                                                |
| Format Completion Response| n8n-nodes-base.code              | Formats non-streaming response to Langchain JSON  | Is Agent?                  | JSON Response                |                                                                                                                                |
| JSON Response             | n8n-nodes-base.respondToWebhook  | Returns formatted JSON response                     | Format Completion Response |                               |                                                                                                                                |
| n8n Webhook               | @n8n/n8n-nodes-langchain.lmChatOpenAi | Langchain OpenAI model node with custom credential |                            | AI Agent                     | See Sticky Note4 and Sticky Note for credential setup guidance                                                                |
| Sticky Note1              | n8n-nodes-base.stickyNote        | Instructions to create custom OpenAI credential    |                            |                               | Contains link: https://docs.n8n.io/integrations/builtin/credentials/openai/                                                   |
| Sticky Note2              | n8n-nodes-base.stickyNote        | Explains model listing endpoint                     |                            |                               | Contains link: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/                                     |
| Sticky Note3              | n8n-nodes-base.stickyNote        | Explains remapping of Responses API output         |                            |                               | Contains link: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest                                 |
| Sticky Note4              | n8n-nodes-base.stickyNote        | Full workflow overview, usage, requirements, support |                            |                               | Discord: https://discord.com/invite/XPKeKXeB7d; Forum: https://community.n8n.io/                                               |
| Sticky Note               | n8n-nodes-base.stickyNote        | Tips on setting up LLM models via webhook and credentials |                            |                               |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Custom OpenAI Credential:**  
   - Name it (e.g., "n8n Document Understanding")  
   - Set API key to your OpenAI key  
   - Set Base URL to your n8n webhook URL for this workflow (e.g., `https://<your_n8n_url>/webhook/n8n-responses-api`)  
   - See Sticky Note1 for detailed instructions  

2. **Create Webhook Node for Model Listing:**  
   - Path: `n8n-responses-api/models`  
   - HTTP Method: GET (default)  
   - Response Mode: "Respond Node"  

3. **Create HTTP Request Node to OpenAI Models API:**  
   - URL: `https://api.openai.com/v1/models`  
   - Method: GET  
   - Authentication: Use your OpenAI API credential (standard)  

4. **Create RespondToWebhook Node for Models Response:**  
   - Respond with: JSON  
   - HTTP Status: 200  
   - Content-Type: application/json  
   - Response Body: `{{ $json }}` (pass-through)  

5. **Connect Model Listing Flow:**  
   - Webhook → OpenAI Models → Models Response  

6. **Create Webhook Node for Chat Completions:**  
   - Path: `n8n-responses-api/chat/completions`  
   - HTTP Method: POST  
   - Response Mode: "Respond Node"  

7. **Create Code Node to Remap Incoming Requests:**  
   - Mode: Run once per item  
   - JavaScript code to transform incoming messages content into OpenAI Responses API input schema (see code in Remap to Response API Schema node)  

8. **Create HTTP Request Node to OpenAI Responses API:**  
   - URL: `https://api.openai.com/v1/responses`  
   - Method: POST  
   - Authentication: Use the custom OpenAI credential created in step 1 (with modified Base URL)  
   - Body: JSON containing model, stream flag, and remapped input from code node  

9. **Create If Node to Check if Response is Streaming:**  
   - Condition: Check if `stream` flag in response JSON is `true`  

10. **Create Code Node to Format Streaming Response:**  
    - Parse streaming text, extract final response, and format according to Langchain streaming chunk structure  
    - Output plain text data with `data:` lines and `[DONE]` sentinel  

11. **Create RespondToWebhook Node for Streaming Text Response:**  
    - Respond with: plain text  
    - Response Body: mapped streaming data from previous node  

12. **Create Code Node to Format Non-Streaming Completion Response:**  
    - Map response fields to Langchain-compatible JSON, including choices, usage tokens, and metadata  

13. **Create RespondToWebhook Node for JSON Response:**  
    - Respond with: JSON  
    - Response Body: mapped JSON data from previous node  

14. **Connect Chat Completion Flow:**  
    - POST ChatCompletions → Remap to Response API Schema → OpenAI Responses API → Is Agent?  
    - Is Agent? true → Format Stream Response → Text Response  
    - Is Agent? false → Format Completion Response → JSON Response  

15. **Create Langchain Chat Trigger Node:**  
    - Public webhook with file uploads enabled  
    - Triggers AI Agent node  

16. **Create Langchain AI Agent Node:**  
    - Enable passthroughBinaryImages and returnIntermediateSteps options  
    - Connects to POST ChatCompletions webhook node  

17. **Create Langchain OpenAI Chat Node:**  
    - Model: gpt-4o-mini  
    - Credential: Custom OpenAI credential (same as step 1)  
    - Connects to AI Agent node  

18. **Add Sticky Notes:**  
    - Add notes with instructions and links as per Sticky Note nodes content for user guidance  

19. **Activate Workflow:**  
    - Ensure all webhooks are active and publicly reachable  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| To enable OpenAI Github Models with existing n8n nodes, create a new OpenAI credential with a modified Base URL pointing to this workflow's webhooks.                                                                        | https://docs.n8n.io/integrations/builtin/credentials/openai/           |
| The workflow demonstrates listing OpenAI models by proxying and remapping the OpenAI Models API response.                                                                                                                     | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/ |
| OpenAI Responses API output format is incompatible with Langchain; this workflow remaps it for compatibility but may lose some new API features.                                                                              | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest |
| Join n8n Discord or Forum for support and updates on the Responses API integration.                                                                                                                                           | Discord: https://discord.com/invite/XPKeKXeB7d ; Forum: https://community.n8n.io/ |
| Activate this workflow using production webhook URLs to ensure proper routing and operation of the custom OpenAI credential.                                                                                                | N/A                                                                    |

---

*Disclaimer:* The text provided derives exclusively from an automated workflow created with n8n, respecting content policies strictly and containing no illegal or protected content. All data processed is legal and publicly available.