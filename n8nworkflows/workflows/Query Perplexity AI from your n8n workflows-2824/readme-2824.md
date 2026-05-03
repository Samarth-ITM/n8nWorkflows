Query Perplexity AI from your n8n workflows

https://n8nworkflows.xyz/workflows/query-perplexity-ai-from-your-n8n-workflows-2824


# Query Perplexity AI from your n8n workflows

### 1. Workflow Overview

This workflow demonstrates how to integrate Perplexity AI, a free AI-powered answer engine, into n8n workflows to obtain accurate, trusted, and real-time answers to user questions. It is designed for users who want to query Perplexity AI programmatically and process the response within n8n.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger to start the workflow and setting up the input parameters including system and user prompts, and domain filters.
- **1.2 Perplexity AI Query:** Sending a POST request to the Perplexity AI API with the prepared parameters and authentication.
- **1.3 Response Processing:** Extracting and cleaning the relevant parts of the API response for further use or display.
- **1.4 Documentation & Guidance:** Sticky notes providing credential setup instructions and model selection guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block initializes the workflow execution manually and sets the parameters required for the Perplexity AI query, including system and user prompts and domain filters.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Params (Set Node)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow execution.  
    - Configuration: No parameters; triggers workflow on manual activation.  
    - Inputs: None  
    - Outputs: Connected to "Set Params" node.  
    - Edge Cases: None specific; manual trigger requires user interaction.

  - **Set Params**  
    - Type: Set  
    - Role: Defines and assigns the input parameters for the AI query.  
    - Configuration:  
      - `system_prompt`: "You are an n8n fanboy." (context for AI)  
      - `user_prompt`: "What are the differences between n8n and Make?" (user question)  
      - `domains`: "n8n.io, make.com" (comma-separated domain filters)  
    - Expressions: Static string assignments.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Perplexity Request" node  
    - Edge Cases: If parameters are empty or malformed, the API request may fail or return irrelevant results.

#### 2.2 Perplexity AI Query

- **Overview:**  
  This block sends a POST request to the Perplexity AI API endpoint with the parameters set previously, using HTTP header authentication with an API key.

- **Nodes Involved:**  
  - Perplexity Request (HTTP Request)

- **Node Details:**

  - **Perplexity Request**  
    - Type: HTTP Request  
    - Role: Sends the AI query to Perplexity’s chat completions API.  
    - Configuration:  
      - URL: `https://api.perplexity.ai/chat/completions`  
      - Method: POST  
      - Body (JSON):  
        - `model`: "sonar" (Sonar Pro model)  
        - `messages`: Array with system and user roles populated from input parameters (`system_prompt` and `user_prompt`)  
        - `temperature`: 0.2 (controls randomness)  
        - `top_p`: 0.9 (nucleus sampling parameter)  
        - `search_domain_filter`: JSON array of domains parsed from the `domains` string input  
        - `return_images`: false  
        - `return_related_questions`: false  
        - `search_recency_filter`: "month" (limits search to last month)  
        - `top_k`: 0  
        - `stream`: false  
        - `presence_penalty`: 0  
        - `frequency_penalty`: 1  
        - `response_format`: null  
      - Authentication: Generic HTTP Header Auth  
        - Header Name: "Authorization"  
        - Header Value: "Bearer <Your Perplexity API Key>"  
    - Expressions: Uses n8n expressions to inject input parameters dynamically into the JSON body.  
    - Inputs: From "Set Params"  
    - Outputs: To "Clean Output" node  
    - Version Requirements: HTTP Request node version 4.2 or higher recommended for latest features.  
    - Edge Cases:  
      - Authentication failure if API key is invalid or missing.  
      - Network timeouts or API rate limits.  
      - Malformed JSON body causing API errors.  
      - Unexpected API response structure changes.

#### 2.3 Response Processing

- **Overview:**  
  This block extracts and cleans the relevant parts of the Perplexity AI response, specifically the main answer content and citations, for easier downstream use.

- **Nodes Involved:**  
  - Clean Output (Set Node)

- **Node Details:**

  - **Clean Output**  
    - Type: Set  
    - Role: Extracts the AI-generated answer and citations from the API response JSON.  
    - Configuration:  
      - `output`: Extracted from `choices[0].message.content` in the API response JSON.  
      - `citations`: Extracted from `citations` array in the API response JSON.  
    - Expressions: Uses n8n expressions to parse nested JSON properties.  
    - Inputs: From "Perplexity Request"  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - If the API response does not contain expected fields, these expressions may fail or return undefined.  
      - Empty or null responses.

#### 2.4 Documentation & Guidance

- **Overview:**  
  Provides users with instructions on setting up credentials and selecting AI models via sticky notes within the workflow canvas.

- **Nodes Involved:**  
  - Sticky Note (Credentials Setup)  
  - Sticky Note1 (Model Information)

- **Node Details:**

  - **Sticky Note (Credentials Setup)**  
    - Type: Sticky Note  
    - Role: Informs users how to obtain API keys and configure authentication in the HTTP Request node.  
    - Content Highlights:  
      - Link to Perplexity API settings: https://www.perplexity.ai/settings/api  
      - Instructions to use Generic Credentials with HTTP Header Auth in n8n.  
      - Header name: "Authorization"  
      - Header value format: "Bearer pplx-e4...59ea" (example API key)  
    - Position: Top-left of the canvas for visibility.

  - **Sticky Note1 (Model Information)**  
    - Type: Sticky Note  
    - Role: Advises on the AI model used ("Sonar Pro") and where to find alternatives.  
    - Content Highlights:  
      - Sonar Pro is the current top model.  
      - Link to model cards documentation: https://docs.perplexity.ai/guides/model-cards  
    - Positioned near the Perplexity Request node for contextual relevance.

---

### 3. Summary Table

| Node Name                 | Node Type       | Functional Role                  | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                  |
|---------------------------|-----------------|--------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger  | Workflow entry point            | None                        | Set Params                |                                                                                              |
| Set Params                | Set             | Define input parameters         | When clicking ‘Test workflow’ | Perplexity Request        |                                                                                              |
| Perplexity Request        | HTTP Request    | Send query to Perplexity AI API | Set Params                  | Clean Output              | See sticky note for credentials setup: https://www.perplexity.ai/settings/api                |
| Clean Output              | Set             | Extract and clean API response  | Perplexity Request          | None                      |                                                                                              |
| Sticky Note              | Sticky Note     | Credential setup instructions   | None                        | None                      | 1/ Go to https://www.perplexity.ai/settings/api to get API key. 2/ Use Generic Credentials with HTTP Header Auth in Perplexity Request node. |
| Sticky Note1             | Sticky Note     | Model usage guidance             | None                        | None                      | Sonar Pro is the current top model. See https://docs.perplexity.ai/guides/model-cards        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - No configuration needed. This node will start the workflow manually.

2. **Create Set Node for Parameters**  
   - Add a **Set** node named "Set Params".  
   - Configure three fields with static values:  
     - `system_prompt` (String): "You are an n8n fanboy."  
     - `user_prompt` (String): "What are the differences between n8n and Make?"  
     - `domains` (String): "n8n.io, make.com"  
   - Connect the output of "When clicking ‘Test workflow’" to this node.

3. **Create HTTP Request Node for Perplexity API**  
   - Add an **HTTP Request** node named "Perplexity Request".  
   - Set the following parameters:  
     - HTTP Method: POST  
     - URL: `https://api.perplexity.ai/chat/completions`  
     - Authentication: Generic Credential with HTTP Header Auth  
       - Header Name: "Authorization"  
       - Header Value: "Bearer YOUR_API_KEY" (replace with your actual Perplexity API key)  
     - Body Content Type: JSON  
     - Request Body (JSON) using expressions:  
       ```json
       {
         "model": "sonar",
         "messages": [
           {
             "role": "system",
             "content": "{{ $json.system_prompt }}"
           },
           {
             "role": "user",
             "content": "{{ $json.user_prompt }}"
           }
         ],
         "temperature": 0.2,
         "top_p": 0.9,
         "search_domain_filter": {{ (JSON.stringify($json.domains.split(','))) }},
         "return_images": false,
         "return_related_questions": false,
         "search_recency_filter": "month",
         "top_k": 0,
         "stream": false,
         "presence_penalty": 0,
         "frequency_penalty": 1,
         "response_format": null
       }
       ```  
   - Connect the output of "Set Params" to this node.

4. **Create Set Node to Clean Output**  
   - Add a **Set** node named "Clean Output".  
   - Configure two fields with expressions to extract API response data:  
     - `output` (String): `={{ $json.choices[0].message.content }}`  
     - `citations` (Array): `={{ $json.citations }}`  
   - Connect the output of "Perplexity Request" to this node.

5. **Add Sticky Notes for Documentation**  
   - Add a **Sticky Note** near the top-left of the canvas with the following content:  
     ```
     ## Credentials Setup

     1/ Go to the perplexity dashboard, purchase some credits and create an API Key

     https://www.perplexity.ai/settings/api

     2/ In the Perplexity Request node, use Generic Credentials, Header Auth.

     For the name, use the value "Authorization"
     And for the value "Bearer pplx-e4...59ea" (Your Perplexity Api Key)
     ```
   - Add another **Sticky Note** near the "Perplexity Request" node with the content:  
     ```
     **Sonar Pro** is the current top model used by perplexity.
     If you want to use a different one, check this page:

     https://docs.perplexity.ai/guides/model-cards
     ```

6. **Verify Credentials Setup**  
   - In n8n, create a new Generic Credential with HTTP Header Auth type.  
   - Set header name to "Authorization" and value to "Bearer YOUR_API_KEY".  
   - Assign this credential to the "Perplexity Request" node.

7. **Test the Workflow**  
   - Click "Execute Workflow" or "Test Workflow" to trigger manually.  
   - Confirm that the output node "Clean Output" returns the AI answer and citations.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                      |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Perplexity AI requires purchasing credits and generating an API key at their dashboard.            | https://www.perplexity.ai/settings/api              |
| The current recommended AI model is "Sonar Pro". Alternative models and their details are listed.  | https://docs.perplexity.ai/guides/model-cards       |
| Use Generic Credential with HTTP Header Auth in n8n for API authentication with Perplexity AI.     | Credential setup instructions in Sticky Note node   |

---

This documentation provides a complete, stepwise understanding of the workflow, enabling users and automation agents to reproduce, modify, and troubleshoot the integration of Perplexity AI within n8n.