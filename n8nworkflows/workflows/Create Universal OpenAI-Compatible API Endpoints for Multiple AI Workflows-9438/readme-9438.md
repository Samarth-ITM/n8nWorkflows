Create Universal OpenAI-Compatible API Endpoints for Multiple AI Workflows

https://n8nworkflows.xyz/workflows/create-universal-openai-compatible-api-endpoints-for-multiple-ai-workflows-9438


# Create Universal OpenAI-Compatible API Endpoints for Multiple AI Workflows

### 1. Workflow Overview

This n8n workflow provides a universal, OpenAI-compatible API interface to multiple AI workflows or agents managed within n8n. It is designed to:

- List all available AI models/workflows tagged as `aimodel` in the n8n environment, exposing them in a format compatible with OpenAI’s `/models` endpoint.
- Accept chat completion requests through a standard OpenAI-like API endpoint, route these requests to the appropriate internal n8n AI workflow (agent), and return responses formatted in OpenAI’s chat completion schema.
- Support both streaming and non-streaming response modes, adapting the response type accordingly.
- Facilitate seamless integration of multiple AI agents/workflows behind a universal API interface, enabling clients expecting OpenAI-compatible endpoints to interact with custom AI workflows.

The workflow’s logic is grouped into three main blocks:

**1.1 Listing Available Models:**  
Handles GET requests to `/youragents/models` and returns a JSON list of available AI workflows tagged `aimodel`, formatted to match OpenAI’s `/models` response schema.

**1.2 Processing Chat Completion Requests:**  
Handles POST requests to `/youragents/chat/completions` with OpenAI-compatible chat completion payloads. It remaps incoming messages, calls the target AI workflow via webhook, and formats the responses accordingly.

**1.3 Internal AI Agent Invocation:**  
Invokes specific AI workflows configured as agents via HTTP requests, either streaming or standard, and manages response formatting for both scenarios.

Additional nodes and sticky notes provide guidance on credential setup and usage of specialized n8n Langchain AI nodes.

---

### 2. Block-by-Block Analysis

#### 2.1 Listing Available Models

**Overview:**  
This block listens for GET requests at `/youragents/models`, retrieves all n8n workflows tagged with `aimodel`, extracts relevant fields, aggregates the data, and formats the response to conform to OpenAI’s `/models` API.

**Nodes Involved:**  
- GET models (Webhook Trigger)  
- Get many workflows (n8n API node)  
- Edit Fields (Set node)  
- Aggregate (Aggregate node)  
- Models Response (Respond to Webhook node)  
- Sticky Note2 (documentation)

**Node Details:**

- **GET models**  
  - Type: Webhook Trigger  
  - Role: Entry point listening on path `/youragents/models` for GET requests.  
  - Config: Path set to `youragents/models`, responds using response node mode.  
  - Inputs: External HTTP GET request  
  - Outputs: Passes request to "Get many workflows"  
  - Failures: Missing webhook registration or path conflicts may cause 404.

- **Get many workflows**  
  - Type: n8n API node  
  - Role: Queries all workflows filtered by tag `aimodel`  
  - Config: Filter on tag equals `aimodel`  
  - Inputs: From GET models  
  - Outputs: Sends workflows data to "Edit Fields"  
  - Failures: API permission or connectivity issues.

- **Edit Fields**  
  - Type: Set node  
  - Role: Extracts workflow fields: name, id, and the `path` parameter from a node inside each workflow.  
  - Config: Assigns three fields: `name` (workflow name), `id` (workflow id), `path` (from node with `path` parameter)  
  - Key Expressions:
    - `{{$json.name}}`, `{{$json.id}}`
    - `{{$json.nodes.find(node => node.parameters?.path).parameters.path}}`  
  - Inputs: Workflows data  
  - Outputs: To Aggregate  
  - Failures: If no node has `path` parameter, expression fails.

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Aggregates all items into a single JSON for response formatting  
  - Config: Aggregate all item data (concatenate array)  
  - Inputs: From Edit Fields  
  - Outputs: To Models Response

- **Models Response**  
  - Type: Respond to Webhook  
  - Role: Returns the final JSON response formatted to OpenAI `/models` schema  
  - Config: Responds with JSON body that maps each workflow's `path` as `id`, sets `object: model`, current timestamp as `created`, and `owned_by: system`.  
  - Inputs: Aggregated workflows data  
  - Outputs: HTTP response to requester  
  - Failures: Expression errors if input malformed.

- **Sticky Note2**  
  - Content: Explains this block is the first endpoint for listing all models using GET webhook node and mapping responses to OpenAI compatible format.  
  - Link: [Webhook Trigger node documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)

---

#### 2.2 Processing Chat Completion Requests

**Overview:**  
This block listens for POST requests at `/youragents/chat/completions` with OpenAI-compatible chat completion requests. It remaps the request’s messages, calls the appropriate AI workflow webhook, and formats the response according to whether streaming is requested.

**Nodes Involved:**  
- POST ChatCompletions (Webhook Trigger)  
- Remap to Response API Schema (Code node)  
- Is Stream? (If node)  
- Call workflow webhook (HTTP Request)  
- Format Stream Response (Code node)  
- Text Response (Respond to Webhook)  
- Call workflow webhook1 (HTTP Request)  
- Format Completion Response (Code node)  
- JSON Response (Respond to Webhook)  
- Sticky Note3 (documentation)

**Node Details:**

- **POST ChatCompletions**  
  - Type: Webhook Trigger  
  - Role: Entry point for chat completion requests, path `/youragents/chat/completions`, method POST  
  - Config: Authentication via header, response mode is response node  
  - Inputs: Incoming POST request with chat messages  
  - Outputs: Passes JSON body to "Remap to Response API Schema"  
  - Failures: Missing or invalid auth headers; invalid JSON body.

- **Remap to Response API Schema**  
  - Type: Code (JavaScript)  
  - Role: Transforms incoming chat messages to a normalized internal format with content type mapping  
  - Key Logic:  
    - Iterates over messages, transforms string content to `{type: "input_text", text: content}`  
    - Supports `image_url` and `file_url` mapped to respective input types  
    - Outputs a normalized `input` array for downstream processing  
  - Inputs: POST ChatCompletions JSON body  
  - Outputs: To Is Stream? node  
  - Failures: Input missing `body.messages` causes failure.

- **Is Stream?**  
  - Type: If node  
  - Role: Checks if `stream` flag is true in incoming request body  
  - Condition: `{{$json.body.stream}} === true`  
  - Inputs: Remap to Response API Schema  
  - Outputs: True branch to "Call workflow webhook" (streaming), False branch to "Call workflow webhook1" (non-streaming)

- **Call workflow webhook**  
  - Type: HTTP Request  
  - Role: Calls the target AI workflow webhook (model name passed in URL) with streaming enabled  
  - Config:  
    - URL dynamically set to `https://n8n.lucidusfortis.com/webhook/{{model}}`  
    - POST method, JSON body includes model, stream flag, chatInput (JSON string), sessionId (cf-ray header), and fromLLM flag  
    - Authentication using predefined n8n API credentials  
  - Inputs: From Is Stream? true branch  
  - Outputs: To Format Stream Response  
  - Failures: Network errors, invalid URL, authentication failures.

- **Format Stream Response**  
  - Type: Code (JavaScript)  
  - Role: Formats the streaming response chunk to OpenAI streaming format  
  - Key Logic:  
    - Builds a chunk object with id, object type, timestamp, model, system fingerprint, and a delta with content  
    - Constructs a data string with two messages: chunk JSON and a `[DONE]` signal  
  - Outputs: To Text Response node  
  - Failures: Missing expected headers or output data.

- **Text Response**  
  - Type: Respond to Webhook  
  - Role: Returns the streaming response as plain text (`text/plain`)  
  - Inputs: From Format Stream Response  
  - Outputs: HTTP response stream to client

- **Call workflow webhook1**  
  - Type: HTTP Request  
  - Role: Calls the target AI workflow webhook for non-streaming requests (stream flag false)  
  - Config: Same as Call workflow webhook but with `stream: false`  
  - Inputs: From Is Stream? false branch  
  - Outputs: To Format Completion Response  
  - Failures: Same as streaming call.

- **Format Completion Response**  
  - Type: Code (JavaScript)  
  - Role: Formats the non-streaming completion response into OpenAI completion schema  
  - Key Logic:  
    - Constructs an object with id (from cf-ray header), object type, timestamp, model, choices array containing message content, and usage placeholders  
  - Inputs: From Call workflow webhook1  
  - Outputs: To JSON Response node  
  - Failures: Missing headers or response body.

- **JSON Response**  
  - Type: Respond to Webhook  
  - Role: Returns the final JSON chat completion response  
  - Inputs: From Format Completion Response  
  - Outputs: HTTP JSON response to client

- **Sticky Note3**  
  - Content: Explains this second endpoint handles chat completions, notes differences in streaming and non-streaming response handling.  
  - Link: [HTTP Request node documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest)

---

#### 2.3 Internal AI Agent Invocation and Supporting Nodes

**Overview:**  
This block includes nodes for invoking internal AI agents via webhooks, Langchain model and memory nodes, and setup instructions for credentials. It supports the actual AI processing behind the API endpoints.

**Nodes Involved:**  
- When chat message received (Langchain Chat Trigger)  
- n8n Webhooks (Langchain OpenAI model node)  
- Powered By n8n Workflow Models (Langchain Agent node)  
- Simple Memory (Langchain Memory Buffer Window node)  
- Sticky Note1 (documentation)

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Listens for chat messages to trigger AI workflows  
  - Config: Webhook ID specified, no additional options  
  - Inputs: External chat message event  
  - Outputs: To "Powered By n8n Workflow Models"

- **n8n Webhooks**  
  - Type: Langchain LM Chat OpenAI model node  
  - Role: Acts as a language model interface using OpenAI-compatible credentials  
  - Config: Model name set dynamically (`youragentname`), options default  
  - Inputs: Receives chat input from trigger  
  - Outputs: To Langchain Agent

- **Powered By n8n Workflow Models**  
  - Type: Langchain Agent node  
  - Role: Manages AI agent logic, decides responses, orchestrates AI workflows  
  - Config: Default options, connects memory and language model  
  - Inputs: From LM Chat node and Memory node  
  - Outputs: Final AI output for response

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains conversation memory buffer for context in chat  
  - Inputs: To Agent node  
  - Outputs: To Agent node  
  - Config: Defaults

- **Sticky Note1**  
  - Content: Advises on creating a custom OpenAI credential with changed Base URL to enable chat with n8n workflows, mimicking OpenAI-compatible API.  
  - Link: [OpenAI Credentials documentation](https://docs.n8n.io/integrations/builtin/credentials/openai/)

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                                  | Input Node(s)                           | Output Node(s)                    | Sticky Note                                                                                                          |
|-----------------------------|----------------------------------|-------------------------------------------------|---------------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| GET models                  | Webhook Trigger                  | Entry point for listing models at `/youragents/models` | External HTTP GET                    | Get many workflows                | ## 1. Listing All Available Models [Read more about the Webhook Trigger node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/) |
| Get many workflows          | n8n API node                    | Fetch workflows tagged `aimodel`                 | GET models                           | Edit Fields                      |                                                                                                                      |
| Edit Fields                | Set                             | Extracts and remaps workflow fields               | Get many workflows                   | Aggregate                       |                                                                                                                      |
| Aggregate                  | Aggregate                       | Aggregates all workflows into a single response   | Edit Fields                         | Models Response                 |                                                                                                                      |
| Models Response            | Respond to Webhook              | Sends formatted models list response               | Aggregate                         | (HTTP response to client)         |                                                                                                                      |
| POST ChatCompletions        | Webhook Trigger                 | Entry point for chat completions at `/youragents/chat/completions` | External HTTP POST                  | Remap to Response API Schema     | ## 2. Request a Chat Completion [Read more about the HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest)         |
| Remap to Response API Schema | Code                           | Normalizes incoming chat messages for internal use | POST ChatCompletions                | Is Stream?                      |                                                                                                                      |
| Is Stream?                 | If                              | Checks if streaming response is requested          | Remap to Response API Schema        | Call workflow webhook (T), Call workflow webhook1 (F) |                                                                                                                      |
| Call workflow webhook      | HTTP Request                    | Calls AI workflow webhook for streaming responses | Is Stream? (true branch)            | Format Stream Response           |                                                                                                                      |
| Format Stream Response     | Code                           | Formats streaming response chunks                   | Call workflow webhook               | Text Response                   |                                                                                                                      |
| Text Response              | Respond to Webhook              | Sends streaming response as plain text              | Format Stream Response              | (HTTP response to client)         |                                                                                                                      |
| Call workflow webhook1     | HTTP Request                    | Calls AI workflow webhook for standard responses   | Is Stream? (false branch)           | Format Completion Response       |                                                                                                                      |
| Format Completion Response | Code                           | Formats non-streaming completion response            | Call workflow webhook1              | JSON Response                   |                                                                                                                      |
| JSON Response              | Respond to Webhook              | Sends JSON completion response                        | Format Completion Response          | (HTTP response to client)         |                                                                                                                      |
| When chat message received | Langchain Chat Trigger          | Trigger node for chat messages on internal AI agents | External event                     | Powered By n8n Workflow Models   |                                                                                                                      |
| n8n Webhooks               | Langchain LM Chat OpenAI Model  | Language model interface using OpenAI-compatible credential | When chat message received          | Powered By n8n Workflow Models   |                                                                                                                      |
| Powered By n8n Workflow Models | Langchain Agent                 | Orchestrates AI agent workflow and response         | n8n Webhooks, Simple Memory         | (final AI output)                |                                                                                                                      |
| Simple Memory              | Langchain Memory Buffer Window  | Conversation memory for context                       | (none)                            | Powered By n8n Workflow Models   |                                                                                                                      |
| Sticky Note1               | Sticky Note                    | Guidance on creating custom OpenAI credentials       | (none)                            | (none)                         | ## 3. Create a New Custom OpenAI Credential [Learn more about OpenAI Credentials](https://docs.n8n.io/integrations/builtin/credentials/openai/)                  |
| Sticky Note2               | Sticky Note                    | Explains listing models endpoint                      | (none)                            | (none)                         | ## 1. Listing All Available Models [Read more about the Webhook Trigger node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)         |
| Sticky Note3               | Sticky Note                    | Explains chat completion endpoint and streaming logic | (none)                            | (none)                         | ## 2. Request a Chat Completion [Read more about the HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest)             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook for Listing Models**  
   - Add a **Webhook Trigger** node named `GET models`  
   - Set path to `youragents/models`  
   - Method: GET  
   - Response Mode: `responseNode`  
   - No authentication required

2. **Fetch Workflows Tagged 'aimodel'**  
   - Add an **n8n API** node named `Get many workflows`  
   - Configure to query workflows with filter: tag = `aimodel`  
   - Connect output of `GET models` to this node

3. **Extract Required Fields**  
   - Add a **Set** node named `Edit Fields`  
   - Map fields:  
     - `name` = `{{$json.name}}`  
     - `id` = `{{$json.id}}`  
     - `path` = expression: `{{$json.nodes.find(node => node.parameters?.path).parameters.path}}`  
   - Connect output of `Get many workflows` to this node

4. **Aggregate Workflow Data**  
   - Add an **Aggregate** node named `Aggregate`  
   - Aggregate all items data to a single JSON array  
   - Connect output of `Edit Fields` to this node

5. **Respond with Models List**  
   - Add a **Respond to Webhook** node named `Models Response`  
   - Set response type JSON with body:  
     ```javascript
     {
       object: "list",
       data: $json.data.map(item => ({
         id: item.path,
         object: "model",
         created: Math.floor(Date.now() / 1000),
         owned_by: "system"
       }))
     }
     ```  
   - Connect output of `Aggregate` to this node

6. **Create Webhook for Chat Completions**  
   - Add a **Webhook Trigger** node named `POST ChatCompletions`  
   - Set path: `youragents/chat/completions`  
   - Method: POST  
   - Authentication: Header Auth (set up header key as needed)  
   - Response Mode: `responseNode`  

7. **Remap Incoming Chat Messages**  
   - Add a **Code** node named `Remap to Response API Schema`  
   - Use JavaScript code to normalize message content types (text, image_url, file_url) into internal schema  
   - Connect output of `POST ChatCompletions` to this node

8. **Branch on Stream Flag**  
   - Add an **If** node named `Is Stream?`  
   - Condition: Check if `{{$json.body.stream}} === true`  
   - Connect output of `Remap to Response API Schema` to this node

9. **Call AI Workflow Webhook (Streaming)**  
   - Add **HTTP Request** node named `Call workflow webhook`  
   - Method: POST  
   - URL: `https://n8n.lucidusfortis.com/webhook/{{ $json.body.model }}` (use expression)  
   - JSON Body: include model, stream flag, chatInput (JSON stringified), sessionId, fromLLM boolean  
   - Authentication: Use n8n API credentials (predefined)  
   - Connect `Is Stream?` True output to this node

10. **Format Streaming Response**  
    - Add **Code** node named `Format Stream Response`  
    - Format streaming chunks following OpenAI streaming data format  
    - Connect output of `Call workflow webhook` to this node

11. **Respond with Streaming Text**  
    - Add **Respond to Webhook** node named `Text Response`  
    - Respond as `text/plain` with body from `Format Stream Response`  
    - Connect output of `Format Stream Response` to this node

12. **Call AI Workflow Webhook (Non-Streaming)**  
    - Add **HTTP Request** node named `Call workflow webhook1`  
    - Same config as streaming call but `stream: false` in body  
    - Connect `Is Stream?` False output to this node

13. **Format Completion Response**  
    - Add **Code** node named `Format Completion Response`  
    - Format response to OpenAI chat completion schema with message content  
    - Connect output of `Call workflow webhook1` to this node

14. **Respond with JSON Completion**  
    - Add **Respond to Webhook** node named `JSON Response`  
    - Respond with JSON body from `Format Completion Response`  
    - Connect output of `Format Completion Response` to this node

15. **Add Langchain AI Agent Nodes (Optional for Internal Agent Processing)**  
    - Add **Langchain Chat Trigger** node named `When chat message received`  
    - Set webhook ID and options accordingly  
    - Add **Langchain LM Chat OpenAI Model** node named `n8n Webhooks`  
    - Set model to your agent name, base URL configured through custom OpenAI credential  
    - Add **Langchain Agent** node named `Powered By n8n Workflow Models`  
    - Connect LM Chat and Memory nodes as inputs  
    - Add **Langchain Memory Buffer Window** node named `Simple Memory`  
    - Connect to Agent node

16. **Setup Credentials**  
    - Create a new OpenAI credential in n8n with custom Base URL pointing to your n8n instance or API gateway, enabling it to mimic OpenAI API calls to your agents.  
    - Create or configure HTTP Header Auth for the POST ChatCompletions webhook for security.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow mimics OpenAI’s API endpoints to enable clients to interact with custom AI workflows transparently.              | n8n integration use case                                                                                     |
| For detailed docs on Webhook Trigger node: [Webhook Trigger Documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/) | Documentation for webhook setup                                                                              |
| For detailed docs on HTTP Request node: [HTTP Request Documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest)            | Documentation for HTTP requests                                                                              |
| To create custom OpenAI credentials (e.g., change Base URL), see: [OpenAI Credentials Documentation](https://docs.n8n.io/integrations/builtin/credentials/openai/) | Required for integration with custom AI agents                                                               |
| The workflow uses Langchain nodes for AI agent orchestration and memory management, leveraging n8n's AI capabilities.            | Requires n8n Langchain integration                                                                           |
| The workflow’s streaming implementation uses the `cf-ray` header as a session identifier and system fingerprint for tracking.  | Important for request/response correlation and caching                                                      |

---

**Disclaimer:**  
The text provided is extracted exclusively from a workflow automated via n8n, an integration and automation tool. This processing complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.