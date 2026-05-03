Create OpenAI-Compatible API Using GitHub Models for Free AI Access

https://n8nworkflows.xyz/workflows/create-openai-compatible-api-using-github-models-for-free-ai-access-4217


# Create OpenAI-Compatible API Using GitHub Models for Free AI Access

### 1. Workflow Overview

This workflow enables free AI access by creating an OpenAI-compatible API interface that utilizes GitHub's AI models through their public API. It acts as a bridge, allowing users or n8n LLM nodes configured with a custom OpenAI credential to transparently access GitHub models without refactoring existing AI workflows.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Webhook nodes receive incoming HTTP requests simulating OpenAI API calls, including requests for available models and chat completions.
- **1.2 GitHub Model API Interaction:** HTTP Request nodes call GitHub's Models API endpoints to fetch model lists or to perform chat completions, with data transformation to fit OpenAI-compatible formats.
- **1.3 Response Handling and Logic Branching:** Conditional and response nodes manage output formatting and decision making, including handling streaming responses for chat completions and returning appropriately formatted JSON or text to clients.

Supporting these are several sticky notes providing documentation and usage instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block handles incoming HTTP requests that simulate OpenAI API calls. It exposes two webhook endpoints: one for fetching the list of models and another for chat completions.

**Nodes Involved:**  
- GET models (Webhook)  
- POST ChatCompletions (Webhook)  
- When chat message received (LangChain Chat Trigger)

**Node Details:**  

- **GET models**  
  - *Type:* Webhook  
  - *Role:* Receives GET requests at `/github-models/models` to list available AI models.  
  - *Configuration:* Path set to `github-models/models`, response mode is `responseNode` to delegate response to downstream nodes.  
  - *Input:* External HTTP GET requests.  
  - *Output:* Forwards data to "Github Models" node.  
  - *Edge cases:* Missing or invalid requests may result in webhook errors; ensure the webhook is active and publicly accessible.

- **POST ChatCompletions**  
  - *Type:* Webhook  
  - *Role:* Accepts POST requests at `/github-models/chat/completions` to perform chat completions.  
  - *Configuration:* Path set to `github-models/chat/completions`, HTTP method POST, response mode `responseNode`.  
  - *Input:* JSON payloads mimicking OpenAI chat completion requests.  
  - *Output:* Forwards payload to "Github Chat Completions".  
  - *Edge cases:* Invalid JSON body or missing keys (e.g., model, messages) may cause errors.

- **When chat message received**  
  - *Type:* LangChain Chat Trigger node  
  - *Role:* Triggers workflow on incoming chat messages in n8n AI workflows; bridges existing n8n LLM nodes to this custom API.  
  - *Configuration:* Uses a webhook ID for external connection.  
  - *Input:* Chat messages from connected AI agents or users.  
  - *Output:* Sends data to "Powered By Github Models" node.  
  - *Edge cases:* Requires active webhook URL and properly configured OpenAI credential with base URL adjusted.

---

#### 2.2 GitHub Model API Interaction

**Overview:**  
This block makes HTTP requests to GitHub's AI Models API endpoints to retrieve data and perform chat completions, transforming the data as needed to maintain OpenAI compatibility.

**Nodes Involved:**  
- Github Models (HTTP Request)  
- Github Chat Completions (HTTP Request)  
- Aggregate (Aggregate)  
- Powered By Github Models (LangChain Chain LLM)

**Node Details:**  

- **Github Models**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the list of available GitHub models from `https://models.github.ai/catalog/models`.  
  - *Configuration:*  
    - Uses predefined GitHub API credential.  
    - Sends headers: Accept `application/vnd.github+json`, `X-GitHub-Api-Version` `2022-11-28`.  
    - Handles redirect option enabled.  
  - *Input:* Triggered by "GET models" webhook node.  
  - *Output:* Sends raw GitHub API response to "Aggregate".  
  - *Edge cases:*  
    - GitHub API rate limits.  
    - Authentication failure if GitHub credential invalid or expired.  
    - Network timeouts.

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all items from the GitHub models response into a single data array.  
  - *Configuration:* Uses `aggregateAllItemData` option to combine data.  
  - *Input:* Receives data from "Github Models".  
  - *Output:* Passes aggregated data to "Models Response".  
  - *Edge cases:* Empty or malformed data may cause aggregation to fail.

- **Github Chat Completions**  
  - *Type:* HTTP Request  
  - *Role:* Sends chat completion requests to GitHub's chat completion API endpoint `https://models.github.ai/inference/chat/completions`.  
  - *Configuration:*  
    - POST method with JSON body constructed from incoming webhook JSON: uses `model`, `messages`, and `stream` keys.  
    - Sends headers same as "Github Models" node.  
    - Uses predefined GitHub API credential.  
  - *Input:* Triggered by "POST ChatCompletions" webhook node.  
  - *Output:* Passes response to "Is Agent?" node for branching.  
  - *Edge cases:*  
    - Streamed responses require special handling.  
    - API errors, auth failures, or invalid payloads.

- **Powered By Github Models**  
  - *Type:* LangChain Chain LLM node  
  - *Role:* Acts as a placeholder or pass-through node linked to the GitHub Models API, used in conjunction with the "When chat message received" trigger.  
  - *Configuration:* No custom parameters; relies on attached subnode with custom OpenAI credential.  
  - *Input:* Receives chat messages from the trigger node.  
  - *Output:* Sends data downstream (not explicitly connected in this JSON, but used internally).  
  - *Edge cases:* Requires correct OpenAI credential with Base URL pointed to this workflow's webhook endpoints.

---

#### 2.3 Response Handling and Logic Branching

**Overview:**  
This block manages the formatting of API responses, decides how to respond based on whether the chat completion is streamed or not, and returns data to the client accordingly.

**Nodes Involved:**  
- Models Response (Respond to Webhook)  
- Is Agent? (If)  
- Agent Response (Respond to Webhook)  
- Chat Response (Respond to Webhook)

**Node Details:**  

- **Models Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Formats and returns the list of models in an OpenAI-compatible structure.  
  - *Configuration:*  
    - Responds with JSON.  
    - Transforms GitHub model data into OpenAI model list format with fields: `id`, `object` ("model"), `created` (hardcoded timestamp), and `owned_by` ("system").  
  - *Input:* Receives aggregated model data from "Aggregate" node.  
  - *Output:* Ends webhook response cycle for model listing.  
  - *Edge cases:* Mapping function may fail if data structure changes.

- **Is Agent?**  
  - *Type:* If (Conditional)  
  - *Role:* Checks if the GitHub chat completion response contains a `stream` flag set to true.  
  - *Configuration:*  
    - Condition: `{{$json.body.stream}}` is `true` (strict boolean).  
  - *Input:* Receives response from "Github Chat Completions".  
  - *Output:*  
    - If true: routes to "Agent Response".  
    - If false: routes to "Chat Response".  
  - *Edge cases:* Missing `stream` property could cause condition to fail or misroute.

- **Agent Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Returns streamed chat completion data as plain text.  
  - *Configuration:*  
    - Responds with MIME type `text/plain`.  
    - Response body is raw data from GitHub API's `data` field.  
  - *Input:* Triggered when `stream` is true.  
  - *Output:* Final webhook response for streaming chat completions.  
  - *Edge cases:* Streaming could be interrupted or malformed.

- **Chat Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Returns non-streamed chat completion data in JSON format.  
  - *Configuration:*  
    - Responds with JSON.  
    - Response body is the entire JSON from GitHub chat completion response.  
  - *Input:* Triggered when `stream` is false.  
  - *Output:* Final webhook response for standard chat completions.  
  - *Edge cases:* Malformed responses or missing data keys.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                     | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                                                        |
|-------------------------|----------------------------------|----------------------------------------------------|------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger           | Receives chat messages from users/agents           |                        | Powered By Github Models     |                                                                                                                                                    |
| Powered By Github Models  | LangChain Chain LLM              | Acts as LLM node connected to GitHub API            | When chat message received |                             | "LLM Models via N8N Webhooks" note nearby describing setup of OpenAI credential with webhook base URL                                               |
| Sticky Note1             | Sticky Note                      | Documentation for creating custom OpenAI credential |                        |                             | "Create a New Custom OpenAI Credential" with setup link                                                                                           |
| Sticky Note              | Sticky Note                      | Instructions on LLM models via n8n webhooks         |                        |                             | Steps to create OpenAI credential with base URL pointing to webhook                                                                                |
| Sticky Note2             | Sticky Note                      | Documentation for listing available models          |                        |                             | Explains GET models webhook and mapping                                                                                                          |
| Sticky Note3             | Sticky Note                      | Documentation about chat completions                 |                        |                             | Explains POST chat completions webhook and streaming response handling                                                                            |
| Sticky Note4             | Sticky Note                      | General project and usage overview                   |                        |                             | Extensive usage instructions, links to GitHub Models docs, Discord, forum, and project notes                                                      |
| GET models               | Webhook                         | Receives GET request for available models            |                        | Github Models                |                                                                                                                                                    |
| Github Models            | HTTP Request                    | Calls GitHub Models API to fetch models list          | GET models              | Aggregate                   |                                                                                                                                                    |
| Aggregate                | Aggregate                      | Aggregates all model data into a single array         | Github Models           | Models Response             |                                                                                                                                                    |
| Models Response          | Respond to Webhook              | Formats and returns model list in OpenAI format       | Aggregate               |                             |                                                                                                                                                    |
| POST ChatCompletions     | Webhook                         | Receives POST request for chat completions             |                        | Github Chat Completions     |                                                                                                                                                    |
| Github Chat Completions  | HTTP Request                    | Sends chat completion request to GitHub API            | POST ChatCompletions    | Is Agent?                   |                                                                                                                                                    |
| Is Agent?                | If                             | Checks if response is streaming or not                 | Github Chat Completions | Agent Response / Chat Response |                                                                                                                                                    |
| Agent Response           | Respond to Webhook              | Returns streaming chat completion as plain text        | Is Agent?               |                             |                                                                                                                                                    |
| Chat Response            | Respond to Webhook              | Returns standard chat completion JSON response         | Is Agent?               |                             |                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create New OpenAI Credential**  
   - Go to n8n Credentials, create a new OpenAI credential named e.g. "n8n-webhook".  
   - Enter any API key (e.g., "12345") — this is dummy since requests are proxied.  
   - Set the Base URL to your n8n instance's webhook URL for models, e.g., `https://<your_n8n_url>/webhook/github-models`.  
   - Save credential.

2. **Create Webhook for Model Listing (`GET models` node)**  
   - Add a Webhook node, method GET, path `github-models/models`.  
   - Set Response Mode to "Response Node".  
   - Connect the output to the next node "Github Models".

3. **Create HTTP Request Node to Fetch GitHub Models (`Github Models`)**  
   - Add HTTP Request node.  
   - Set URL to `https://models.github.ai/catalog/models`.  
   - Method: GET.  
   - Enable Redirect handling.  
   - Add headers:  
     - `Accept: application/vnd.github+json`  
     - `X-GitHub-Api-Version: 2022-11-28`  
   - Use predefined GitHub API credential (must be pre-configured with appropriate auth).  
   - Connect to "Aggregate".

4. **Add Aggregate Node**  
   - Add Aggregate node.  
   - Set aggregation to `Aggregate All Item Data`.  
   - Connect output to "Models Response".

5. **Add Respond to Webhook Node for Models (`Models Response`)**  
   - Add Respond to Webhook node.  
   - Set to respond with JSON.  
   - Set response body to transform GitHub model list to OpenAI model list format using JavaScript expression:  
     ```js
     ({
       "object": "list",
       "data":  $json.data.map(item => ({
         "id": item.id,
         "object": "model",
         "created": 1733945430,
         "owned_by": "system"
       }))
     })
     ```  
   - Connect as final response for `GET models`.

6. **Create Webhook for Chat Completions (`POST ChatCompletions`)**  
   - Add Webhook node, method POST, path `github-models/chat/completions`.  
   - Response Mode: "Response Node".  
   - Connect output to "Github Chat Completions".

7. **Create HTTP Request Node to Perform Chat Completion (`Github Chat Completions`)**  
   - Add HTTP Request node.  
   - URL: `https://models.github.ai/inference/chat/completions`.  
   - Method: POST.  
   - Enable Redirect handling.  
   - Headers same as "Github Models" node.  
   - Authentication: Use predefined GitHub API credential.  
   - Body: Set JSON body with expression:  
     ```js
     {
       model: $json.body.model,
       messages: $json.body.messages,
       stream: $json.body.stream
     }
     ```  
   - Connect output to "Is Agent?".

8. **Add Conditional Node to Check Streaming (`Is Agent?`)**  
   - Add If node.  
   - Condition: Check if `{{$json.body.stream}}` is boolean true.  
   - True output connects to "Agent Response".  
   - False output connects to "Chat Response".

9. **Add Respond to Webhook Node for Streaming Response (`Agent Response`)**  
   - Respond with text/plain.  
   - Response body: `{{$json.data}}`.  
   - Connect as final response for streaming chat completions.

10. **Add Respond to Webhook Node for Normal Response (`Chat Response`)**  
    - Respond with JSON.  
    - Response body: `{{$json}}`.  
    - Connect as final response for standard chat completions.

11. **Add LangChain Chat Trigger Node (`When chat message received`)**  
    - Add LangChain Chat Trigger node.  
    - Configure with webhook ID (auto-generated).  
    - Connect output to "Powered By Github Models".

12. **Add LangChain Chain LLM Node (`Powered By Github Models`)**  
    - Add Chain LLM node.  
    - No specific parameters; attach subnode with the custom OpenAI credential created in step 1.  
    - This node acts as the AI model interface in n8n workflows connecting to GitHub models via the custom OpenAI-compatible API.

13. **Verify Credentials**  
    - GitHub API credential configured with valid token and permission to access GitHub Models API.  
    - OpenAI credential with Base URL pointing to this workflow's webhook URL.

14. **Activate the Workflow**  
    - Activate all nodes.  
    - Ensure your n8n instance is publicly accessible if you want external calls.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow shows how to connect GitHub's free AI models to existing n8n AI workflows by creating a custom OpenAI-compatible API proxy inside n8n. It avoids refactoring existing AI nodes and allows experimentation with state-of-the-art models for free. Note GitHub's API is not intended for production usage. | https://docs.github.com/en/github-models/prototyping-with-ai-models                                        |
| For higher rate limits or production use, consider paid AI services instead of GitHub Models API.                                                                                                                                                                                                           | GitHub Models API documentation                                                                              |
| Setup requires creating a custom OpenAI credential with the Base URL pointed to this workflow's webhook endpoints.                                                                                                                                                                                         | https://docs.n8n.io/integrations/builtin/credentials/openai/                                                |
| Helpful links: Discord community for n8n (https://discord.com/invite/XPKeKXeB7d), n8n Forum (https://community.n8n.io/)                                                                                                                                                                                     | Community support                                                                                            |
| The workflow includes detailed sticky notes explaining each step and usage instructions; review them inside n8n canvas for guidance.                                                                                                                                                                       | Sticky notes nodes inside the workflow                                                                      |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.