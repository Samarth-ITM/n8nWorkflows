Generate AI System Prompts for LLMs with Unli.dev

https://n8nworkflows.xyz/workflows/generate-ai-system-prompts-for-llms-with-unli-dev-7696


# Generate AI System Prompts for LLMs with Unli.dev

### 1. Workflow Overview

This workflow, titled **"System Prompt Generator Using Unli.dev"**, is designed to generate highly structured and effective AI system prompts for Large Language Models (LLMs) based on user requests. It targets users who need tailored system prompts that precisely instruct AI assistants to perform specific tasks flawlessly and in a single interaction.

The workflow logically divides into the following blocks:

- **1.1 Input Reception**: Receives incoming HTTP POST requests containing user prompts.
- **1.2 Prompt Construction**: Builds a comprehensive system prompt by embedding a detailed instruction set for prompt engineering.
- **1.3 API Request Preparation**: Formats the message payload for the Unli.dev AI chat completions API.
- **1.4 AI Invocation**: Sends the constructed prompt and user message to Unli.dev’s API to generate the system prompt.
- **1.5 Response Extraction & Delivery**: Extracts the AI-generated system prompt from the API response and returns it to the requester.
- **1.6 Ancillary Notes**: Includes a sticky note with a Postman test screenshot for reference.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming POST HTTP requests at the endpoint `/systempromptgenerator`. It serves as the entry point to receive user requests describing the desired AI assistant’s function.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - *Type & Role:* HTTP Webhook node; accepts incoming HTTP POST requests.  
    - *Configuration:*  
      - Path: `systempromptgenerator`  
      - HTTP Method: POST  
      - Response Mode: `responseNode` (delays response until downstream node sends it)  
    - *Expressions/Variables:* None  
    - *Input:* External HTTP request  
    - *Output:* JSON containing user payload, expected to include `body.prompt` with the user's description.  
    - *Edge Cases:*  
      - Invalid or missing prompt in request body may cause downstream failures.  
      - Unsupported HTTP methods will be rejected automatically.  
      - Network or permission issues on the webhook URL could block requests.

#### 2.2 Prompt Construction

- **Overview:**  
  Constructs a detailed system prompt that instructs an AI "Metaprompt-Architect" to analyze the user's request and generate a comprehensive system prompt for another AI assistant. It embeds a sophisticated multi-part instruction set into the workflow data.

- **Nodes Involved:**  
  - Set Prompt/Model

- **Node Details:**  
  - **Set Prompt/Model**  
    - *Type & Role:* Set node; assigns or transforms data fields for downstream use.  
    - *Configuration:*  
      - Sets two fields:  
        - `prompt` from `$json.body.prompt` (user’s input)  
        - `system` containing a very detailed multi-paragraph system prompt template that:  
          - Defines the persona as "Metaprompt-Architect"  
          - Specifies a rigorous methodology for prompt design including persona, mission, protocol, rules, output format, tone, and meta analysis  
          - Includes example interaction  
        - Sets `model` to `"auto"` (automatic model selection)  
    - *Expressions:*  
      - `={{ $json.body.prompt }}` for user prompt extraction  
      - Large static string with embedded instructions for system prompt  
    - *Input:* Data from Webhook  
    - *Output:* JSON with `prompt`, `system`, and `model` fields prepared for API call  
    - *Edge Cases:*  
      - If the incoming JSON lacks `body.prompt`, the system prompt will be constructed with a missing user prompt, potentially resulting in less relevant output.  
      - Expression errors if input data is malformed.

#### 2.3 API Request Preparation

- **Overview:**  
  Prepares the body of the HTTP request to the Unli.dev chat completions API, formatting the conversation messages correctly based on presence or absence of the system prompt.

- **Nodes Involved:**  
  - Prepare API Body

- **Node Details:**  
  - **Prepare API Body**  
    - *Type & Role:* Set node; formats the JSON request body for the API call.  
    - *Configuration:*  
      - Sets fields:  
        - `model` from `$json.model`  
        - `messages` as a JSON string array:  
          - If `system` exists, messages array includes a system role message with its content and a user role message with the user prompt  
          - Otherwise, only a user role message  
        - `stream` set to `"false"` (non-streaming response)  
    - *Expressions:*  
      - Conditional JSON.stringify expressions for `messages` field  
    - *Input:* Data from Set Prompt/Model node  
    - *Output:* JSON ready for POST to Unli.dev API  
    - *Edge Cases:*  
      - Possible JSON parsing or stringification errors if input fields are malformed.  
      - If `system` is empty or null, the system message is skipped, possibly reducing prompt effectiveness.

#### 2.4 AI Invocation

- **Overview:**  
  Calls the Unli.dev API’s chat completions endpoint with the prepared body to generate the AI system prompt as per the instructions.

- **Nodes Involved:**  
  - Unli.Dev (Chat Completions)

- **Node Details:**  
  - **Unli.Dev (Chat Completions)**  
    - *Type & Role:* HTTP Request node; sends POST request to Unli.dev API for AI completion.  
    - *Configuration:*  
      - URL: `https://api.unli.dev/v1/chat/completions`  
      - Method: POST  
      - Timeout: 60 seconds (to handle potentially long processing)  
      - JSON Body: Uses parsed `model`, `messages`, and `stream` fields from previous node data  
      - Authentication: HTTP Header Auth credential named "Unli.dev - JACCBot" (must be preconfigured with valid API key)  
    - *Input:* Prepared JSON from Prepare API Body  
    - *Output:* JSON response from Unli.dev including AI generated content and usage stats  
    - *Edge Cases:*  
      - Authentication failure (invalid or expired API key) causes 401 errors  
      - Timeout if API response is delayed over 60 seconds  
      - API rate limits or quota exhaustion may cause errors  
      - Network issues interrupting the HTTP request  
      - Unexpected response format causing downstream parsing errors

#### 2.5 Response Extraction & Delivery

- **Overview:**  
  Extracts the AI-generated system prompt text from the API response and sends it back as the HTTP response of the webhook request.

- **Nodes Involved:**  
  - Extract Answer  
  - Respond to Webhook

- **Node Details:**  
  - **Extract Answer**  
    - *Type & Role:* Set node; extracts relevant data from the Unli.dev API response for final output.  
    - *Configuration:*  
      - Extracts:  
        - `answer` from `$json.choices[0].message.content` (main AI response text)  
        - `model_used` from `$json.model`  
        - `usage` from `$json.usage` (token usage info)  
    - *Input:* API response JSON from Unli.Dev node  
    - *Output:* JSON containing extracted fields for response  
    - *Edge Cases:*  
      - If `choices` array is empty or missing, `$json.choices[0]` access fails  
      - Unexpected API response structure may cause extraction failure

  - **Respond to Webhook**  
    - *Type & Role:* Respond to webhook node; sends the final AI-generated prompt text back to the HTTP client.  
    - *Configuration:*  
      - Response Body: uses extracted `answer` field  
      - Responds as plain text  
    - *Input:* Extracted answer JSON  
    - *Output:* HTTP response content to original POST request  
    - *Edge Cases:*  
      - If input is empty or null, response will be empty  
      - Network issues or client disconnects can affect delivery

#### 2.6 Ancillary Notes

- **Overview:**  
  Provides a sticky note in the editor containing a screenshot illustrating a Postman test of the webhook endpoint.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - *Type & Role:* Visual annotation within n8n editor; no functional impact on workflow execution.  
    - *Configuration:*  
      - Content includes markdown heading "Postman Test" and an embedded image URL:  
        `https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/NewPlatform/unli.dev/Screenshot%202025-08-21%20at%2016.03.40.png`  
      - Size and color configured for visibility  
    - *Input/Output:* None  
    - *Notes:* Useful documentation for developers testing the webhook endpoint.

---

### 3. Summary Table

| Node Name                | Node Type                | Functional Role                     | Input Node(s)              | Output Node(s)               | Sticky Note                                           |
|--------------------------|--------------------------|-----------------------------------|----------------------------|-----------------------------|------------------------------------------------------|
| Webhook                  | Webhook                  | Receives HTTP POST requests        | -                          | Set Prompt/Model             | Postman test screenshot linked in Sticky Note        |
| Set Prompt/Model         | Set                      | Constructs detailed system prompt  | Webhook                    | Prepare API Body             |                                                      |
| Prepare API Body         | Set                      | Formats API request body           | Set Prompt/Model            | Unli.Dev (Chat Completions)  |                                                      |
| Unli.Dev (Chat Completions)| HTTP Request           | Calls Unli.dev API for AI prompt   | Prepare API Body            | Extract Answer               |                                                      |
| Extract Answer           | Set                      | Extracts AI response content       | Unli.Dev (Chat Completions) | Respond to Webhook           |                                                      |
| Respond to Webhook       | Respond to Webhook        | Returns AI-generated prompt to user| Extract Answer             | -                           |                                                      |
| Sticky Note              | Sticky Note               | Documentation / annotation         | -                          | -                           | Postman Test ![](https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/NewPlatform/unli.dev/Screenshot%202025-08-21%20at%2016.03.40.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook`  
   - HTTP Method: POST  
   - Path: `systempromptgenerator`  
   - Response Mode: `responseNode` (to delay response)  

2. **Create Set Node for Prompt and Model**  
   - Type: Set  
   - Name: `Set Prompt/Model`  
   - Add Fields:  
     - `prompt` (String) = `={{ $json.body.prompt }}` to extract user prompt from webhook JSON body  
     - `system` (String) = Paste the detailed multi-part system prompt text exactly as provided in the original workflow. This includes the persona definition, methodology, rules, output format, tone, example, and constraints.  
     - `model` (String) = `auto` (to auto-select the AI model)  
   - Connect Webhook main output to this node’s input

3. **Create Set Node to Prepare API Body**  
   - Type: Set  
   - Name: `Prepare API Body`  
   - Fields:  
     - `model` = `={{ $json.model }}`  
     - `messages` = `={{ $json.system ? JSON.stringify([{ "role": "system", "content": $json.system }, { "role": "user", "content": $json.prompt }]) : JSON.stringify([{ "role": "user", "content": $json.prompt }]) }}`  
     - `stream` = `false` (string)  
   - Connect `Set Prompt/Model` main output to this node’s input

4. **Create HTTP Request Node to Call Unli.dev API**  
   - Type: HTTP Request  
   - Name: `Unli.Dev (Chat Completions)`  
   - HTTP Method: POST  
   - URL: `https://api.unli.dev/v1/chat/completions`  
   - Timeout: 60000 ms (60 seconds)  
   - Authentication: HTTP Header Authentication (create credential with API key for Unli.dev)  
   - Request Body Type: JSON  
   - Body Content:  
     ```json
     {
       "model": {{$json.model}},
       "messages": {{$json.messages | parseJson}}, 
       "stream": {{$json.stream === 'true'}}
     }
     ```  
   - Connect `Prepare API Body` output to this node input

5. **Create Set Node to Extract Answer**  
   - Type: Set  
   - Name: `Extract Answer`  
   - Fields:  
     - `answer` = `={{ $json.choices[0].message.content }}`  
     - `model_used` = `={{ $json.model }}`  
     - `usage` = `={{ $json.usage }}`  
   - Connect `Unli.Dev (Chat Completions)` output to this node input

6. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Name: `Respond to Webhook`  
   - Respond With: Text  
   - Response Body: `={{ $json.answer }}`  
   - Connect `Extract Answer` output to this node input

7. **Connect Nodes in Sequence**  
   - Webhook → Set Prompt/Model → Prepare API Body → Unli.Dev (Chat Completions) → Extract Answer → Respond to Webhook

8. **Optional: Create Sticky Note for Documentation**  
   - Type: Sticky Note  
   - Content:  
     ```
     ### Postman Test

     ![](https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/NewPlatform/unli.dev/Screenshot%202025-08-21%20at%2016.03.40.png)
     ```  
   - Place near the Webhook node for easy reference

9. **Credential Setup**  
   - Create HTTP Header Auth credential in n8n with Unli.dev API key  
   - Assign this credential to the HTTP Request node  

10. **Activate Workflow and Test**  
    - Deploy and activate the workflow  
    - Test by sending POST request with JSON body containing `prompt` field to the webhook URL  
    - Validate that response contains a structured AI system prompt as designed

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow leverages Unli.dev API for chat completions, an alternative to OpenAI’s API specialized for chat-based LLMs.   | https://unli.dev                                                                                         |
| The detailed system prompt template embedded in "Set Prompt/Model" node is crafted to enforce single-turn, comprehensive AI prompt generation, illustrating advanced prompt engineering methodology. | N/A                                                                                                     |
| Postman screenshot illustrates testing the webhook endpoint with sample data, useful for debugging or API client setup.     | ![](https://raw.githubusercontent.com/khmuhtadin/n8n-template/main/NewPlatform/unli.dev/Screenshot%202025-08-21%20at%2016.03.40.png) |
| Ensure that the Unli.dev API key has sufficient quota and permissions; monitor usage to avoid rate limiting or failures.     | N/A                                                                                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow constructed with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.