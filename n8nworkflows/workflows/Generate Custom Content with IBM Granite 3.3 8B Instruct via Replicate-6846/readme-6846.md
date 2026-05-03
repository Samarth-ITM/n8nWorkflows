Generate Custom Content with IBM Granite 3.3 8B Instruct via Replicate

https://n8nworkflows.xyz/workflows/generate-custom-content-with-ibm-granite-3-3-8b-instruct-via-replicate-6846


# Generate Custom Content with IBM Granite 3.3 8B Instruct via Replicate

### 1. Workflow Overview

This workflow automates generating custom text content using the IBM Granite 3.3 8B Instruct language model via the Replicate API. It is designed for use cases such as instruction-following text generation, content creation, and AI-assisted writing with fine-tuned control over generation parameters.

The workflow is logically divided into the following blocks:

- **1.1 Input Setup and Trigger**: Receives the manual trigger and sets authentication and generation parameters.
- **1.2 Text Generation Request**: Sends a text generation request to the Replicate API.
- **1.3 Status Polling Loop**: Waits and checks the prediction status repeatedly until completion or failure.
- **1.4 Result Handling and Response**: Processes success or failure responses and formats output.
- **1.5 Logging and Monitoring**: Logs request details for monitoring purposes.
- **1.6 Documentation and Notes**: Provides embedded documentation and helpful resources via sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup and Trigger

- **Overview:** Initiates the workflow manually, sets the Replicate API token, and configures all parameters required for the text generation model.
- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token  
  - Set Text Parameters

##### Node Details:

- **Manual Trigger**  
  - Type: Trigger node for manual workflow execution.  
  - Configuration: Default, no parameters.  
  - Inputs: None (entry point).  
  - Outputs: Connects to "Set API Token".  
  - Potential failures: None; purely manual initiation.  

- **Set API Token**  
  - Type: Set node to store API token.  
  - Configuration: Assigns a string variable `api_token` with placeholder "YOUR_REPLICATE_API_TOKEN". Users must replace this with their actual token.  
  - Inputs: From Manual Trigger.  
  - Outputs: Connects to "Set Text Parameters".  
  - Edge cases: Missing or invalid API token will cause authentication failures downstream.  
  - Notes: Critical for API authentication.  

- **Set Text Parameters**  
  - Type: Set node to define all text generation input parameters.  
  - Configuration: Copies `api_token` from previous node; sets model parameters such as `seed` (-1 for random), `top_k` (50), `top_p` (0.9), `prompt` (empty by default), `max_tokens` (512), `min_tokens` (0), `temperature` (0.6), `chat_template`, `system_prompt`, `stop_sequences`, `presence_penalty` (0), `frequency_penalty` (0).  
  - Inputs: From "Set API Token".  
  - Outputs: Connects to "Create Text Prediction".  
  - Edge cases: Empty or invalid parameters may affect model output quality or cause API errors.  

#### 2.2 Text Generation Request

- **Overview:** Sends an HTTP POST request to Replicate API with the configured parameters to start text generation, receiving a prediction ID in response.  
- **Nodes Involved:**  
  - Create Text Prediction

##### Node Details:

- **Create Text Prediction**  
  - Type: HTTP Request.  
  - Configuration: POST to `https://api.replicate.com/v1/predictions`.  
  - Headers: Authorization Bearer token from `api_token`; Prefer header set to "wait" to keep connection until result is ready if possible.  
  - Body: JSON containing model version ID and input parameters dynamically pulled from previous node.  
  - Inputs: From "Set Text Parameters".  
  - Outputs: Connects to "Log Request".  
  - Edge cases: Network errors, API rate limits, invalid token or parameters may cause request failure. Response always JSON with error or prediction ID.  

#### 2.3 Status Polling Loop

- **Overview:** Implements a polling mechanism to check the status of the prediction until it completes or fails, including wait intervals and conditional branching for retries or error handling.  
- **Nodes Involved:**  
  - Log Request  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

##### Node Details:

- **Log Request**  
  - Type: Code node.  
  - Configuration: Logs timestamp, prediction ID, and model type to console for monitoring.  
  - Inputs: From "Create Text Prediction".  
  - Outputs: Connects to "Wait 5s".  
  - Edge cases: Logging failure does not stop workflow; mainly for debugging.  

- **Wait 5s**  
  - Type: Wait node.  
  - Configuration: Waits 5 seconds before next action.  
  - Inputs: From "Log Request".  
  - Outputs: Connects to "Check Status".  
  - Edge cases: None significant; delay to avoid API flooding.  

- **Check Status**  
  - Type: HTTP Request.  
  - Configuration: GET request to `https://api.replicate.com/v1/predictions/{prediction_id}` to fetch current status. Authorization header with API token.  
  - Inputs: From "Wait 5s" and "Wait 10s" (loop-back).  
  - Outputs: Connects to "Is Complete?".  
  - Edge cases: Network issues, token expiration, or API errors could cause failures or stale data.  

- **Is Complete?**  
  - Type: If node.  
  - Configuration: Checks if `status` field equals "succeeded".  
  - Inputs: From "Check Status".  
  - Outputs:  
    - True branch: "Success Response" node.  
    - False branch: "Has Failed?" node.  
  - Edge cases: Status might be other values like "starting", "processing", requiring re-polling.  

- **Has Failed?**  
  - Type: If node.  
  - Configuration: Checks if `status` equals "failed".  
  - Inputs: From "Is Complete?" (false branch).  
  - Outputs:  
    - True branch: "Error Response".  
    - False branch: "Wait 10s" (retry with longer wait).  
  - Edge cases: Other statuses (e.g. "canceled") not explicitly handled; could cause indefinite loops.  

- **Wait 10s**  
  - Type: Wait node.  
  - Configuration: Waits 10 seconds before retrying status check.  
  - Inputs: From "Has Failed?" (false branch).  
  - Outputs: Connects back to "Check Status" (loop).  
  - Edge cases: Extended retries could cause workflow delays or consume execution time limits.  

#### 2.4 Result Handling and Response

- **Overview:** Formats the final response for success or error and prepares the data for output or further processing.  
- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result

##### Node Details:

- **Success Response**  
  - Type: Set node.  
  - Configuration: Creates a JSON object containing success flag, output URL, prediction ID, status, and success message.  
  - Inputs: From "Is Complete?" (true branch).  
  - Outputs: Connects to "Display Result".  

- **Error Response**  
  - Type: Set node.  
  - Configuration: Creates a JSON object containing failure flag, error message (from API or generic), prediction ID, status, and failure message.  
  - Inputs: From "Has Failed?" (true branch).  
  - Outputs: Connects to "Display Result".  

- **Display Result**  
  - Type: Set node.  
  - Configuration: Sets field `final_result` to the `response` object from either success or error node.  
  - Inputs: From "Success Response" and "Error Response".  
  - Outputs: Workflow end / output.  

#### 2.5 Logging and Monitoring

- **Overview:** Contains a code node for logging request metadata for traceability and debugging.  
- **Nodes Involved:**  
  - Log Request (also described in status polling block)

##### Node Details:

- Covered previously in 2.3.

#### 2.6 Documentation and Notes

- **Overview:** Contains sticky note nodes that embed extensive documentation, usage instructions, troubleshooting, and external resource links for users and maintainers.  
- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

##### Node Details:

- **Sticky Note9**  
  - Content: Contact info for support and links to YouTube and LinkedIn channels for tutorials.  
  - Position: Top-left area for visibility.  

- **Sticky Note4**  
  - Content: Detailed model description, parameter references, workflow explanation, quick start instructions, troubleshooting, and links to official documentation.  
  - Position: Large block occupying left-bottom area.  

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                      | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                 |
|---------------------|--------------------|------------------------------------|--------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger     | Start workflow                     | None                     | Set API Token                | See Sticky Note4 for general workflow info                                                  |
| Set API Token       | Set                | Store API token                    | Manual Trigger            | Set Text Parameters          | See Sticky Note4 for general workflow info                                                  |
| Set Text Parameters | Set                | Configure text generation params   | Set API Token             | Create Text Prediction       | See Sticky Note4 for detailed parameters and usage instructions                             |
| Create Text Prediction | HTTP Request      | Send generation request to Replicate | Set Text Parameters       | Log Request                  | API request requires valid token and correct parameters                                    |
| Log Request         | Code               | Log prediction details             | Create Text Prediction    | Wait 5s                     | Logs timestamp and prediction ID for monitoring                                            |
| Wait 5s             | Wait               | Delay before status check          | Log Request               | Check Status                 |                                                                                             |
| Check Status        | HTTP Request       | Poll prediction status             | Wait 5s, Wait 10s         | Is Complete?                 | Requires valid prediction ID and token                                                    |
| Is Complete?        | If                 | Check if prediction succeeded     | Check Status              | Success Response, Has Failed? |                                                                                             |
| Has Failed?         | If                 | Check if prediction failed        | Is Complete?              | Error Response, Wait 10s     |                                                                                             |
| Wait 10s            | Wait               | Delay before retry on pending     | Has Failed?               | Check Status                 |                                                                                             |
| Success Response    | Set                | Format successful output           | Is Complete?              | Display Result               |                                                                                             |
| Error Response      | Set                | Format error output                | Has Failed?               | Display Result               |                                                                                             |
| Display Result      | Set                | Final response packaging           | Success Response, Error Response | None                    |                                                                                             |
| Sticky Note9        | Sticky Note        | Support contact and community links | None                     | None                        | Contact: Yaron@nofluff.online; YouTube & LinkedIn links                                    |
| Sticky Note4        | Sticky Note        | Detailed documentation and instructions | None                     | None                        | Extensive documentation on model, parameters, workflow, and troubleshooting                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Manual Trigger" node**  
   - Node Type: Manual Trigger  
   - Purpose: Start the workflow manually.  
   - No parameters needed.  

2. **Create "Set API Token" node**  
   - Node Type: Set  
   - Parameters: Add string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token).  
   - Connect output of "Manual Trigger" to this node.  

3. **Create "Set Text Parameters" node**  
   - Node Type: Set  
   - Parameters:  
     - Copy `api_token` from previous node using expression: `={{ $('Set API Token').item.json.api_token }}`  
     - Add fields:  
       - `seed` (number): -1  
       - `top_k` (number): 50  
       - `top_p` (number): 0.9  
       - `prompt` (string): "" (empty)  
       - `max_tokens` (number): 512  
       - `min_tokens` (number): 0  
       - `temperature` (number): 0.6  
       - `chat_template` (string): ""  
       - `system_prompt` (string): ""  
       - `stop_sequences` (string): ""  
       - `presence_penalty` (number): 0  
       - `frequency_penalty` (number): 0  
   - Connect output of "Set API Token" to this node.  

4. **Create "Create Text Prediction" node**  
   - Node Type: HTTP Request  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{ $json.api_token }}`  
     - Prefer: `wait`  
   - Body Type: JSON  
   - Body Content (use expressions to map fields):  
     ```json
     {
       "version": "ibm-granite/granite-3.3-8b-instruct:3ff9e6e20ff1f31263bf4f36c242bd9be1acb2025122daeefe2b06e883df0996",
       "input": {
         "seed": {{ $json.seed }},
         "top_k": {{ $json.top_k }},
         "top_p": {{ $json.top_p }},
         "prompt": "{{ $json.prompt }}",
         "max_tokens": {{ $json.max_tokens }},
         "min_tokens": {{ $json.min_tokens }},
         "temperature": {{ $json.temperature }},
         "chat_template": "{{ $json.chat_template }}",
         "system_prompt": "{{ $json.system_prompt }}",
         "stop_sequences": "{{ $json.stop_sequences }}",
         "presence_penalty": {{ $json.presence_penalty }},
         "frequency_penalty": {{ $json.frequency_penalty }}
       }
     }
     ```  
   - Connect output of "Set Text Parameters" to this node.  

5. **Create "Log Request" node**  
   - Node Type: Code  
   - JavaScript:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('ibm-granite/granite-3.3-8b-instruct Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'text'
     });
     return $input.all();
     ```  
   - Connect output of "Create Text Prediction" to this node.  

6. **Create "Wait 5s" node**  
   - Node Type: Wait  
   - Parameters: Unit = seconds, Amount = 5  
   - Connect output of "Log Request" to this node.  

7. **Create "Check Status" node**  
   - Node Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Text Prediction').item.json.id }}`  
   - Headers: Authorization: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect output of "Wait 5s" to this node.  
   - Also connect output of "Wait 10s" (created later) to this node for polling loop.  

8. **Create "Is Complete?" node**  
   - Node Type: If  
   - Condition: Check if `{{$json.status}}` equals `"succeeded"` (string comparison, case sensitive).  
   - Connect output of "Check Status" to this node.  

9. **Create "Has Failed?" node**  
   - Node Type: If  
   - Condition: Check if `{{$json.status}}` equals `"failed"` (string).  
   - Connect false output of "Is Complete?" to this node.  

10. **Create "Success Response" node**  
    - Node Type: Set  
    - Assign field `response` with object:  
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Text generated successfully"
      }
      ```  
    - Connect true output of "Is Complete?" to this node.  

11. **Create "Error Response" node**  
    - Node Type: Set  
    - Assign field `response` with object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Text generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate text"
      }
      ```  
    - Connect true output of "Has Failed?" to this node.  

12. **Create "Wait 10s" node**  
    - Node Type: Wait  
    - Parameters: Unit = seconds, Amount = 10  
    - Connect false output of "Has Failed?" to this node.  
    - Connect output of this node back to "Check Status" node to form polling loop.  

13. **Create "Display Result" node**  
    - Node Type: Set  
    - Assign field `final_result` with value `={{ $json.response }}`  
    - Connect outputs of both "Success Response" and "Error Response" to this node.  

14. **(Optional) Create Sticky Notes**  
    - Add two sticky note nodes with content copied from Sticky Note9 and Sticky Note4 in the original workflow for documentation and quick reference.  
    - Position them for best visibility in your canvas.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| =======================================\n        GRANITE-3.3-8B-INSTRUCT GENERATOR\n=======================================\nFor any questions or support, please contact:\n    Yaron@nofluff.online\n\nExplore more tips and tutorials here:\n   - YouTube: https://www.youtube.com/@YaronBeen/videos\n   - LinkedIn: https://www.linkedin.com/in/yaronbeen/\n=======================================                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Contact and community links for support and further learning                                      |
| ## 🤖 **IBM-GRANITE/GRANITE-3.3-8B-INSTRUCT - TEXT GENERATION WORKFLOW**\n\n**🔥 Powered by Replicate API and n8n Automation**\n\n---\n\n### 📝 **Model Overview**\n\n- **Owner**: ibm-granite\n- **Model**: granite-3.3-8b-instruct\n- **Type**: Text Generation\n- **API Endpoint**: https://api.replicate.com/v1/predictions\n\n**🎯 What This Model Does:**\nGranite-3.3-8B-Instruct is a 8-billion parameter 128K context length language model fine-tuned for improved reasoning and instruction-following capabilities.\n\n---\n\n### 📋 **Parameter Reference**\n\n**🔴 Required Parameters:** None\n**🔵 Optional Parameters:** seed, top_k, top_p, prompt, max_tokens, min_tokens, temperature, chat_template (and 4 more)\n\n**📖 Detailed Parameter Guide:**\n- **seed** (integer): Random seed. Leave blank to randomize the seed.\n- **top_k** (integer): The number of highest probability tokens to consider for generating the output. If > 0, only keep... (Default: 50)\n- **top_p** (number): A probability threshold for generating the output. If < 1.0, only keep the top tokens with cumula... (Default: 0.9)\n- **prompt** (string): User prompt to send to the model. (Default: )\n- **max_tokens** (integer): The maximum number of tokens the model should generate as output. (Default: 512)\n- **min_tokens** (integer): The minimum number of tokens the model should generate as output. (Default: 0)\n- **temperature** (number): The value used to modulate the next token probabilities. (Default: 0.6)\n- **chat_template** (string): A template to format the prompt with. If not provided, the default prompt template will be used.\n- *...and 4 more parameters*\n\n---\n\n### 🔧 **Workflow Components Explained**\n\n**🎯 Manual Trigger**\n- Starts the workflow execution\n- Click to begin text generation process\n\n**🔐 Set API Token** \n- Configures your Replicate API authentication\n- Replace 'YOUR_REPLICATE_API_TOKEN' with your actual token\n- Essential for accessing the ibm-granite/granite-3.3-8b-instruct model\n\n**⚙️ Set Text Parameters**\n- Configures all input parameters for the model\n- Includes both required and optional parameters\n- Pre-filled with sensible defaults for testing\n\n**🚀 Create Text Prediction**\n- Sends the generation request to Replicate API\n- Uses the text parameters you configured\n- Returns a prediction ID for status tracking\n\n**⏳ Wait & Status Checking Loop**\n- Waits 5 seconds then checks prediction status\n- Continues checking until completion or failure\n- Implements intelligent retry logic with 10-second delays\n\n**✅ Success/Error Handling**\n- Routes successful completions to success response\n- Handles failures gracefully with error details\n- Returns structured JSON response with URLs/errors\n\n**📊 Logging & Monitoring**\n- Logs all requests for debugging and monitoring\n- Tracks timestamps and prediction IDs\n- Helps identify issues during development\n\n---\n\n### 🌟 **Key Benefits**\n\n- **🎨 Instant Text Generation**: Transform ideas into texts using state-of-the-art AI\n- **🔄 Automated Workflow**: Handles the complete generation pipeline automatically\n- **🛡️ Error Resilience**: Built-in retry logic and comprehensive error handling\n- **📈 Production Ready**: Includes logging, monitoring, and structured responses\n- **🔧 Customizable**: Easy to modify parameters and extend functionality\n- **⚡ Efficient Processing**: Optimized API calls with intelligent status checking\n\n---\n\n### 🚀 **Quick Start Instructions**\n\n1. **🔑 Get Your API Key**\n   - Sign up at https://replicate.com\n   - Navigate to your account settings\n   - Copy your API token\n\n2. **🔧 Configure the Workflow**\n   - Replace 'YOUR_REPLICATE_API_TOKEN' with your actual token\n   - Adjust parameters in the 'Set Text Parameters' node\n   - Customize the prompt or other inputs as needed\n\n3. **▶️ Execute the Workflow**\n   - Click the 'Manual Trigger' to start\n   - Monitor the execution in the n8n interface\n   - Check logs for detailed execution information\n\n4. **📥 Get Your Results**\n   - Successful generations return a URL to your text\n   - Download or use the generated content as needed\n   - Results are available immediately upon completion\n\n---\n\n### 🔍 **Troubleshooting Guide**\n\n**Common Issues:**\n- **Invalid API Token**: Ensure your Replicate token is valid and has sufficient credits\n- **Parameter Validation**: Check that required parameters match expected types\n- **Generation Timeout**: Some texts take longer - monitor the logs\n- **Output Format**: Verify the model returns the expected output format\n\n**Best Practices:**\n- Test with default parameters first\n- Monitor your Replicate usage and billing\n- Keep API tokens secure and never commit them to code\n- Use appropriate parameter values for your use case\n\n---\n\n**🔗 Additional Resources:**\n- Model Documentation: https://replicate.com/ibm-granite/granite-3.3-8b-instruct\n- Replicate API Docs: https://replicate.com/docs\n- n8n Documentation: https://docs.n8n.io\n\n--- | Comprehensive embedded documentation including model details, parameters, troubleshooting, and links |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.