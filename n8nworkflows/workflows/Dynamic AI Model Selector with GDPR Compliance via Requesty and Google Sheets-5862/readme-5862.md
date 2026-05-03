Dynamic AI Model Selector with GDPR Compliance via Requesty and Google Sheets

https://n8nworkflows.xyz/workflows/dynamic-ai-model-selector-with-gdpr-compliance-via-requesty-and-google-sheets-5862


# Dynamic AI Model Selector with GDPR Compliance via Requesty and Google Sheets

### 1. Workflow Overview

This workflow provides a dynamic, GDPR-compliant AI language model selection and interaction system using Requesty API and Google Sheets for configuration and state management. It enables a user to select an AI model from up-to-date options fetched dynamically from Requesty, ensures the selection is stored and managed with GDPR compliance, and facilitates chat interactions using the selected model.

The workflow logic is grouped into the following blocks:

- **1.1 Initialization & Model Fetching:** Retrieve available AI models dynamically from Requesty and update the workflow’s model selection dropdown accordingly.
- **1.2 Model Selection Management:** Handle user selection of AI models through a form trigger, update Google Sheets to persist selected model state.
- **1.3 AI Chat Interaction:** Process chat inputs using the currently selected AI model via Requesty’s chat completions endpoint.
- **1.4 Auxiliary & Optional AI Agent:** Includes optional more advanced AI agent and OpenAI integration nodes for extended capabilities.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Model Fetching

**Overview:**  
This block initializes the workflow, fetches the current list of available AI models from Requesty’s API, processes the returned data to extract model IDs, and updates the workflow’s form trigger dropdown options dynamically to reflect these models.

**Nodes Involved:**  
- Initialize Workflow  
- Get Current Workflow  
- Fetch Available Models  
- Split Model Data  
- Extract Model IDs  
- Merge Data  
- Update Dropdown Options (Code Node)  
- Update Workflow

**Node Details:**

- **Initialize Workflow**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow for initialization or updates.  
  - *Config:* No parameters; manual trigger only.  
  - *Connections:* Outputs to Get Current Workflow and Fetch Available Models.  
  - *Failures:* None typical; manual trigger may be forgotten.  

- **Get Current Workflow**  
  - *Type:* n8n node (n8n core)  
  - *Role:* Retrieve the current workflow JSON to modify it dynamically.  
  - *Config:* Uses current workflow ID to fetch JSON.  
  - *Connections:* Outputs to Merge Data.  
  - *Failures:* Workflow fetch failure possible if workflow ID invalid or permissions insufficient.  

- **Fetch Available Models**  
  - *Type:* HTTP Request  
  - *Role:* Requesty API call to fetch AI models available under provider “coding”.  
  - *Config:* GET request to `https://router.requesty.ai/v1/models?provider=coding`.  
  - *Connections:* Outputs to Split Model Data.  
  - *Failures:* HTTP errors, network errors, API key or quota issues.  

- **Split Model Data**  
  - *Type:* Split Out  
  - *Role:* Split the array of models in the response’s `data` field into individual items.  
  - *Config:* Splits on `data` field.  
  - *Connections:* Outputs to Extract Model IDs.  
  - *Failures:* If API response format changes or `data` is missing, node errors.  

- **Extract Model IDs**  
  - *Type:* Set  
  - *Role:* Extract and assign the `id` field of each model to a simplified JSON structure.  
  - *Config:* Sets `id` property from `$json.data.id`.  
  - *Connections:* Outputs to Merge Data (second input).  
  - *Failures:* Missing or malformed `id` fields cause empty or invalid data.  

- **Merge Data**  
  - *Type:* Merge  
  - *Role:* Merges the current workflow JSON (first input) with the array of model IDs (second input).  
  - *Config:* Default merge settings.  
  - *Connections:* Outputs to Update Dropdown Options.  
  - *Failures:* Incorrect input order or missing inputs cause merge failure.  

- **Update Dropdown Options**  
  - *Type:* Code  
  - *Role:* Updates the dropdown options in the form trigger node dynamically based on fetched model IDs.  
  - *Config:* JavaScript code that:  
    - Retrieves workflow JSON and model IDs from inputs.  
    - Updates dropdown values in the form trigger’s `formFields.values` structure.  
    - Returns modified workflow JSON.  
  - *Key expressions:* `$input.all()`, iterates inputs to collect IDs.  
  - *Connections:* Outputs updated workflow JSON to Update Workflow node.  
  - *Failures:* Code errors, malformed input, or missing nodes in workflow JSON cause incorrect updates. Includes error handling returning debug info.  

- **Update Workflow**  
  - *Type:* n8n (workflow update)  
  - *Role:* Updates the current workflow with the new JSON containing refreshed dropdown options.  
  - *Config:* Uses workflow ID and updated JSON from previous node.  
  - *Connections:* Terminal node of this block.  
  - *Failures:* Permission issues, concurrency conflicts, or invalid JSON cause update failure.  

---

#### 1.2 Model Selection Management

**Overview:**  
Manages user interaction for selecting an AI model via a form, saves the selected model to Google Sheets for persistence, and clears older selections to maintain GDPR compliance.

**Nodes Involved:**  
- Model Selection Form  
- Set Selected Model  
- Clear Previous Selection  
- Save Model Selection

**Node Details:**

- **Model Selection Form**  
  - *Type:* Form Trigger  
  - *Role:* Provides a web form interface for users to select an AI model from the updated dropdown.  
  - *Config:* Dropdown field labeled "AI Language Model" with dynamic options set by the initialization block.  
  - *Connections:* Outputs to Set Selected Model.  
  - *Failures:* Webhook misconfiguration, form option mismatch, or frontend caching errors.  

- **Set Selected Model**  
  - *Type:* Set  
  - *Role:* Extracts the selected model name from form submission and assigns it to variable `model`.  
  - *Config:* Sets `model` to the form field name received (`fieldName`).  
  - *Connections:* Outputs to Clear Previous Selection.  
  - *Failures:* Missing form data or unexpected field names could cause empty assignment.  

- **Clear Previous Selection**  
  - *Type:* Google Sheets  
  - *Role:* Clears the "Current_Model" sheet to remove old model selection data for GDPR compliance.  
  - *Config:*  
    - Operation: Clear  
    - Sheet: "Current_Model"  
    - Document ID: Configured Google Sheet ID (user replaces placeholder).  
  - *Connections:* Outputs to Save Model Selection.  
  - *Failures:* Google Sheets authentication errors, incorrect sheet name, or permissions issues.  

- **Save Model Selection**  
  - *Type:* Google Sheets  
  - *Role:* Appends or updates the "Current_Model" sheet with the newly selected model information.  
  - *Config:*  
    - Operation: Append or update  
    - Sheet: "Current_Model"  
    - Document ID: Same as above  
    - Mapping: Auto-mapped input data (the selected model field)  
  - *Connections:* Terminal node of this block.  
  - *Failures:* Same as Clear Previous Selection; also potential data mapping issues.  

---

#### 1.3 AI Chat Interaction

**Overview:**  
Processes user chat inputs on a webhook, retrieves the currently selected AI model from Google Sheets, sends a chat completion request to Requesty’s API using that model, extracts the response, and formats it for output.

**Nodes Involved:**  
- Chat Interface  
- Read Current Model  
- Get Current Model  
- AI API Request  
- Extract Response  
- Format Response

**Node Details:**

- **Chat Interface**  
  - *Type:* Langchain Chat Trigger (Webhook)  
  - *Role:* Entry point for receiving user chat messages.  
  - *Config:* Default parameters; webhook ID configured.  
  - *Connections:* Outputs to Read Current Model.  
  - *Failures:* Webhook not reachable, authentication issues if required.  

- **Read Current Model**  
  - *Type:* Google Sheets  
  - *Role:* Reads the "Current_Model" sheet to fetch the currently selected AI model.  
  - *Config:*  
    - Operation: Read  
    - Sheet: "Current_Model"  
    - Document ID: Configured Google Sheet ID  
  - *Connections:* Outputs to Get Current Model.  
  - *Failures:* Authentication, permissions, missing or empty sheet data.  

- **Get Current Model**  
  - *Type:* Set  
  - *Role:* Extracts the model string from the data read from Google Sheets to assign it to `model`.  
  - *Config:* Sets `model` to the value read from the sheet.  
  - *Connections:* Outputs to AI API Request.  
  - *Failures:* Missing or malformed data from spreadsheet.  

- **AI API Request**  
  - *Type:* HTTP Request  
  - *Role:* Sends chat completion request to Requesty API using the selected model and user’s chat input.  
  - *Config:*  
    - POST to `https://router.requesty.ai/v1/chat/completions`  
    - JSON body includes model name, system prompt, user prompt, max tokens, temperature.  
    - Authentication: HTTP header with Bearer token from Requesty API credentials.  
  - *Connections:* Outputs to Extract Response.  
  - *Failures:* API errors, auth failures, timeout, invalid model name.  

- **Extract Response**  
  - *Type:* Set  
  - *Role:* Extracts the AI’s chat response content from API response JSON.  
  - *Config:* Sets `content` to `choices[0].message.content`.  
  - *Connections:* Outputs to Format Response.  
  - *Failures:* Unexpected API response structure or empty responses.  

- **Format Response**  
  - *Type:* Code  
  - *Role:* Formats the extracted response content into a JSON object with a `text` field for downstream consumption.  
  - *Config:* Returns `{ text: content }`.  
  - *Connections:* Terminal node for chat interaction output.  
  - *Failures:* None typical.  

---

#### 1.4 Auxiliary & Optional AI Agent

**Overview:**  
Optional nodes for integrating an AI Agent and OpenAI chat model for advanced AI workflows. This is not connected to the main chat workflow but available for extension.

**Nodes Involved:**  
- OpenAI Chat Model  
- AI Agent (Optional)

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Provides an OpenAI GPT-4o-mini model interface.  
  - *Config:* Model set to `openai/gpt-4o-mini`.  
  - *Connections:* Outputs to AI Agent (Optional).  
  - *Failures:* Requires valid OpenAI API credentials.  

- **AI Agent (Optional)**  
  - *Type:* Langchain Agent  
  - *Role:* Uses the OpenAI Chat Model for agent-based AI tasks.  
  - *Config:* Default.  
  - *Connections:* Terminal optional node.  
  - *Failures:* Dependent on OpenAI Chat Model node and credentials.  

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                          | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                              |
|-------------------------|-------------------------------|----------------------------------------|-----------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| Initialize Workflow      | Manual Trigger                | Start workflow initialization          |                             | Get Current Workflow, Fetch Available Models |                                                                                                        |
| Get Current Workflow     | n8n workflow node             | Fetch current workflow JSON             | Initialize Workflow          | Merge Data                |                                                                                                        |
| Fetch Available Models   | HTTP Request                 | Fetch AI models list from Requesty API | Initialize Workflow          | Split Model Data          |                                                                                                        |
| Split Model Data         | Split Out                    | Split models array to individual items | Fetch Available Models       | Extract Model IDs          |                                                                                                        |
| Extract Model IDs        | Set                          | Extract model IDs from data             | Split Model Data             | Merge Data                |                                                                                                        |
| Merge Data              | Merge                        | Merge workflow JSON with model IDs     | Get Current Workflow, Extract Model IDs | Update Dropdown Options    |                                                                                                        |
| Update Dropdown Options  | Code                         | Update form dropdown options dynamically| Merge Data                  | Update Workflow           | Updates dropdown options in form trigger nodes dynamically based on fetched models                      |
| Update Workflow          | n8n workflow node             | Save updated workflow JSON              | Update Dropdown Options      |                           |                                                                                                        |
| Model Selection Form     | Form Trigger                 | User selects AI model                   |                             | Set Selected Model        |                                                                                                        |
| Set Selected Model       | Set                          | Assign selected model variable          | Model Selection Form         | Clear Previous Selection  |                                                                                                        |
| Clear Previous Selection | Google Sheets                | Clear old model selection for GDPR      | Set Selected Model           | Save Model Selection      |                                                                                                        |
| Save Model Selection     | Google Sheets                | Save new model selection                | Clear Previous Selection     |                           |                                                                                                        |
| Chat Interface           | Langchain Chat Trigger       | Receive chat user input                 |                             | Read Current Model        |                                                                                                        |
| Read Current Model       | Google Sheets                | Read currently selected model          | Chat Interface              | Get Current Model         |                                                                                                        |
| Get Current Model        | Set                          | Assign current model variable           | Read Current Model           | AI API Request            |                                                                                                        |
| AI API Request           | HTTP Request                 | Requesty chat completions API call      | Get Current Model            | Extract Response          |                                                                                                        |
| Extract Response         | Set                          | Extract AI response content             | AI API Request              | Format Response           |                                                                                                        |
| Format Response          | Code                         | Format response for output              | Extract Response            |                           |                                                                                                        |
| OpenAI Chat Model        | Langchain OpenAI Chat Model  | OpenAI GPT-4o-mini integration (optional) |                           | AI Agent (Optional)       |                                                                                                        |
| AI Agent (Optional)      | Langchain Agent              | AI agent using OpenAI Chat Model        | OpenAI Chat Model            |                           |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Initialize Workflow`  
   - Purpose: Manual start point for workflow initialization.

2. **Add n8n Workflow Node to Get Current Workflow**  
   - Name: `Get Current Workflow`  
   - Operation: Get  
   - Workflow ID: Use expression `{{$workflow.id}}` to get current workflow’s ID.  
   - Connect output of `Initialize Workflow` to this node.

3. **Add HTTP Request Node to Fetch Available Models**  
   - Name: `Fetch Available Models`  
   - Method: GET  
   - URL: `https://router.requesty.ai/v1/models?provider=coding`  
   - Connect output of `Initialize Workflow` to this node.

4. **Add Split Out Node to Split Model Data**  
   - Name: `Split Model Data`  
   - Field to split out: `data`  
   - Connect output of `Fetch Available Models` to this node.

5. **Add Set Node to Extract Model IDs**  
   - Name: `Extract Model IDs`  
   - Set field `id` to expression `{{$json.data.id}}`  
   - Connect output of `Split Model Data` to this node.

6. **Add Merge Node to Combine Workflow JSON with Model IDs**  
   - Name: `Merge Data`  
   - Connect output of `Get Current Workflow` to first input.  
   - Connect output of `Extract Model IDs` to second input.

7. **Add Code Node to Update Dropdown Options**  
   - Name: `Update Dropdown Options`  
   - Paste JavaScript code (see node details above) to dynamically update form dropdown options based on model IDs.  
   - Connect output of `Merge Data` to this node.

8. **Add n8n Workflow Node to Update Workflow**  
   - Name: `Update Workflow`  
   - Operation: Update  
   - Workflow ID: Use expression `{{$json.id}}` from previous node.  
   - Workflow Object: Use expression `{{JSON.stringify($json)}}` from previous node.  
   - Connect output of `Update Dropdown Options` to this node.

9. **Add Form Trigger Node for Model Selection**  
   - Name: `Model Selection Form`  
   - Form Title: “Select AI Model”  
   - Add Dropdown field labeled “AI Language Model” with initial empty values (will be updated dynamically).  
   - Configure webhook ID as needed.  

10. **Add Set Node to Store Selected Model**  
    - Name: `Set Selected Model`  
    - Set field `model` to expression `{{$json.fieldName}}` (captures selected form value).  
    - Connect output of `Model Selection Form` to this node.

11. **Add Google Sheets Node to Clear Previous Selection**  
    - Name: `Clear Previous Selection`  
    - Operation: Clear  
    - Sheet Name: “Current_Model”  
    - Document ID: Your Google Sheet ID (replace placeholder).  
    - Connect output of `Set Selected Model` to this node.

12. **Add Google Sheets Node to Save New Model Selection**  
    - Name: `Save Model Selection`  
    - Operation: Append or Update  
    - Sheet Name: “Current_Model”  
    - Document ID: Use same Google Sheet ID.  
    - Connect output of `Clear Previous Selection` to this node.

13. **Add Langchain Chat Trigger Node for Chat Interface**  
    - Name: `Chat Interface`  
    - Configure webhook ID.  

14. **Add Google Sheets Node to Read Current Model**  
    - Name: `Read Current Model`  
    - Operation: Read  
    - Sheet Name: “Current_Model”  
    - Document ID: Same Google Sheet ID.  
    - Connect output of `Chat Interface` to this node.

15. **Add Set Node to Get Current Model**  
    - Name: `Get Current Model`  
    - Set field `model` to expression `{{$json.model}}` extracted from Google Sheets data.  
    - Connect output of `Read Current Model` to this node.

16. **Add HTTP Request Node for AI API Request**  
    - Name: `AI API Request`  
    - Method: POST  
    - URL: `https://router.requesty.ai/v1/chat/completions`  
    - Body Type: JSON  
    - JSON Body:  
      ```json
      {
        "model": "{{ $json.model }}",
        "messages": [
          { "role": "system", "content": "You are a helpful assistant." },
          { "role": "user", "content": "{{ $('Chat Interface').item.json.chatInput }}" }
        ],
        "max_tokens": 200,
        "temperature": 0.7
      }
      ```  
    - Authentication: HTTP Header Bearer token using credential `requestyApi` API key.  
    - Headers: Content-Type: application/json, Authorization: Bearer {{ $credentials.requestyApi.apiKey }}  
    - Connect output of `Get Current Model` to this node.

17. **Add Set Node to Extract Response Content**  
    - Name: `Extract Response`  
    - Set field `content` to expression `{{$json.choices[0].message.content}}`  
    - Connect output of `AI API Request` to this node.

18. **Add Code Node to Format Response**  
    - Name: `Format Response`  
    - JavaScript code:  
      ```js
      return [{
        json: {
          text: $json.content
        }
      }];
      ```  
    - Connect output of `Extract Response` to this node.

19. **(Optional) Setup OpenAI Chat Model Node**  
    - Name: `OpenAI Chat Model`  
    - Model: `openai/gpt-4o-mini`  
    - Configure OpenAI credentials.  

20. **(Optional) Setup AI Agent Node**  
    - Name: `AI Agent (Optional)`  
    - Connect input from `OpenAI Chat Model`.  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow dynamically updates the dropdown options in a form trigger node by modifying its JSON via the n8n API.   | See Update Dropdown Options node’s code for implementation details.                                  |
| Google Sheets is used as a GDPR-compliant storage mechanism to keep track of the current model selection, clearing old data before writing new. | Requires correct Google Sheets credentials and sheet setup with a "Current_Model" sheet.             |
| Requesty API is used for model listing and chat completions; API key must be configured in credentials named `requestyApi`. | https://router.requesty.ai/                                                                          |
| The workflow integrates Langchain nodes for advanced AI use but these are optional and not connected to main chat flow.| https://docs.n8n.io/integrations/ai/nodes/langchain/                                                 |
| Replace all placeholders such as `YOUR_GOOGLE_SHEET_ID` with actual IDs before execution.                              |                                                                                                     |

---

_Disclaimer: This description is based on a fully automated n8n workflow export and respects all content and data usage policies._