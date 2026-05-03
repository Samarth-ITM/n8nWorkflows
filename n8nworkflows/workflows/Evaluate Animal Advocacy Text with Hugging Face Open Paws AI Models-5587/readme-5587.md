Evaluate Animal Advocacy Text with Hugging Face Open Paws AI Models

https://n8nworkflows.xyz/workflows/evaluate-animal-advocacy-text-with-hugging-face-open-paws-ai-models-5587


# Evaluate Animal Advocacy Text with Hugging Face Open Paws AI Models

---

### 1. Workflow Overview

This n8n sub-workflow is designed to evaluate and score animal advocacy texts using two Hugging Face Open Paws AI regression models deployed as inference endpoints. The workflow takes a text input and produces two separate scores: a **performance score** and a **preference score**, which can be used to quantify the quality and appeal of the advocacy text.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the input text from an external workflow.
- **1.2 AI Model Requests:** Sends the text to two Hugging Face inference endpoints to get separate scores.
- **1.3 Score Extraction and Setting:** Extracts the scores from the AI responses and prepares them for output.
- **1.4 Result Aggregation:** Merges the individual scores into a single data structure.
- **1.5 Output Preparation:** Formats and sets the final output containing both scores and the original text.
- **1.6 Documentation and Setup Guidance:** Provides instructions for deploying the required AI models and configuring the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives the input text payload from another workflow that triggers this sub-workflow.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **Name:** When Executed by Another Workflow  
  - **Type:** Execute Workflow Trigger  
  - **Role:** Entry trigger node that accepts workflow inputs from an external workflow execution.  
  - **Configuration:** Accepts a single input parameter named `text`.  
  - **Expressions/Variables:** Inputs accessed as `$json.text`.  
  - **Connections:** Outputs to both "Get Performance Score" and "Get Preference Score" nodes.  
  - **Edge Cases:** If triggering workflow does not supply `text`, subsequent HTTP requests may fail or produce invalid scores.  
  - **Version Requirements:** n8n Execute Workflow Trigger, version 1.1 or higher.  

---

#### 2.2 AI Model Requests

- **Overview:**  
  Sends HTTP POST requests with the input text to two separate Hugging Face inference endpoints to retrieve the performance and preference scores.

- **Nodes Involved:**  
  - Get Performance Score  
  - Get Preference Score

- **Node Details:**  

  1. **Get Performance Score**  
     - **Type:** HTTP Request  
     - **Role:** Calls the Hugging Face inference endpoint for text performance prediction.  
     - **Configuration:**  
       - Method: POST  
       - URL: Placeholder (`INSERT_YOUR_ENDPOINT_URL_HERE`) to be replaced with the deployed Hugging Face endpoint URL for performance scoring.  
       - Body: JSON with `inputs` field containing stringified input text from `$json.text`.  
       - Headers: `Accept: application/json`  
       - Authentication: Uses predefined Hugging Face API credential with an API key.  
     - **Expressions:** Uses an expression to inject the input text: `{{ JSON.stringify($json.text) }}`  
     - **Connections:** Outputs to "Set Performance Score" node.  
     - **Edge Cases:**  
       - Endpoint URL missing or incorrect → HTTP request failure.  
       - Invalid API key or permissions → 401 Unauthorized error.  
       - Timeout or network issues → request failure.  
       - Unexpected response structure → downstream extraction errors.  
     - **Version:** HTTP Request node, version 4.2 or higher.  

  2. **Get Preference Score**  
     - **Type:** HTTP Request  
     - **Role:** Calls the Hugging Face inference endpoint for animal advocate preference prediction.  
     - **Configuration:** Same as above, with a different endpoint URL placeholder.  
     - **Expressions:** Same input text expression used.  
     - **Connections:** Outputs to "Get Preference Score2" node.  
     - **Edge Cases:** Same as above.  
     - **Version:** HTTP Request node, version 4.2 or higher.  

---

#### 2.3 Score Extraction and Setting

- **Overview:**  
  Parses the JSON responses from the AI model endpoints to extract the numerical scores and store them under distinct keys for later aggregation.

- **Nodes Involved:**  
  - Set Performance Score  
  - Get Preference Score2

- **Node Details:**  

  1. **Set Performance Score**  
     - **Type:** Set  
     - **Role:** Extracts the `score` field from the JSON response of the performance model and assigns it to a new field `performance_score`.  
     - **Configuration:** Assigns `performance_score` as a number with value `={{ $json.score }}`.  
     - **Connections:** Outputs to "Merge Branches" node on the first input.  
     - **Edge Cases:** If `$json.score` is missing or not a number, the output may be invalid.  

  2. **Get Preference Score2**  
     - **Type:** Set  
     - **Role:** Extracts the `score` field from the preference model response and assigns it to `preference_score`.  
     - **Configuration:** Assigns `preference_score` as a number with value `={{ $json.score }}`.  
     - **Connections:** Outputs to "Merge Branches" node on the second input.  
     - **Edge Cases:** Same as above.  

---

#### 2.4 Result Aggregation

- **Overview:**  
  Merges the two streams containing the performance and preference scores into a single combined data stream.

- **Nodes Involved:**  
  - Merge Branches  
  - Create Single Item

- **Node Details:**  

  1. **Merge Branches**  
     - **Type:** Merge  
     - **Role:** Combines two input branches (performance score and preference score) into one.  
     - **Configuration:** Default merge settings (join inputs from two sources).  
     - **Connections:** Outputs merged data to "Create Single Item".  
     - **Edge Cases:** If one input is delayed or missing, merge may stall or produce incomplete data.

  2. **Create Single Item**  
     - **Type:** Aggregate  
     - **Role:** Aggregates all incoming items into one single JSON object for easier output handling.  
     - **Configuration:** Uses "aggregateAllItemData" to combine all items into a single object under `data` array.  
     - **Connections:** Outputs to "Set Output" for final formatting.  
     - **Edge Cases:** If no input items, results in empty aggregation.  

---

#### 2.5 Output Preparation

- **Overview:**  
  Formats the final output JSON object containing the original input text and both scores.

- **Nodes Involved:**  
  - Set Output

- **Node Details:**  
  - **Type:** Set  
  - **Role:** Creates a structured output JSON with fields:  
    - `performance_score` (number) from aggregated data  
    - `preference_score` (number) from aggregated data  
    - `text` (string) from original input text  
  - **Configuration:** Assigns values using expressions:  
    - `performance_score = {{$json.data[0].performance_score}}`  
    - `preference_score = {{$json.data[1].preference_score}}`  
    - `text = {{$('When Executed by Another Workflow').item.json.text}}`  
  - **Edge Cases:** If aggregated data structure changes or is missing, expressions may fail.  

---

#### 2.6 Documentation and Setup Guidance

- **Overview:**  
  Provides detailed instructions on how to deploy the necessary Hugging Face models as inference endpoints and configure the sub-workflow accordingly.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Type:** Sticky Note  
  - **Role:** Contains textual documentation for users configuring this sub-workflow.  
  - **Content Summary:**  
    - Lists two required models with links:  
      1. Animal Advocate Preference Prediction (Longform)  
      2. Text Performance Prediction (Longform)  
    - Steps to deploy models as inference endpoints on Hugging Face.  
    - Instructions to copy and paste endpoint URLs into the HTTP Request nodes.  
    - Reminder to configure Hugging Face API credentials with appropriate access tokens.  
    - Warning about permissions and API token scopes.  
  - **Edge Cases:** N/A (informational only)  

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                                  | Input Node(s)                 | Output Node(s)                | Sticky Note                                           |
|-------------------------------|--------------------------|-------------------------------------------------|------------------------------|------------------------------|-------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger | Receives input text from external workflow       |                              | Get Performance Score, Get Preference Score |                                                       |
| Get Performance Score          | HTTP Request             | Sends text to Hugging Face Performance endpoint | When Executed by Another Workflow | Set Performance Score         |                                                       |
| Get Preference Score           | HTTP Request             | Sends text to Hugging Face Preference endpoint  | When Executed by Another Workflow | Get Preference Score2         |                                                       |
| Set Performance Score          | Set                      | Extracts and sets performance score              | Get Performance Score         | Merge Branches               |                                                       |
| Get Preference Score2          | Set                      | Extracts and sets preference score               | Get Preference Score          | Merge Branches               |                                                       |
| Merge Branches                | Merge                    | Combines performance and preference scores       | Set Performance Score, Get Preference Score2 | Create Single Item           |                                                       |
| Create Single Item             | Aggregate                | Aggregates merged data into single item           | Merge Branches                | Set Output                   |                                                       |
| Set Output                    | Set                      | Formats final output with scores and original text | Create Single Item            |                              |                                                       |
| Sticky Note                   | Sticky Note              | Provides deployment and configuration instructions |                              |                              | Contains detailed instructions and links to Hugging Face models and deployment steps. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new sub-workflow** in n8n named "Scoring Text (Sub-Workflow)".

2. **Add an "Execute Workflow Trigger" node**:  
   - Name: `When Executed by Another Workflow`  
   - Configure to accept workflow inputs with one parameter named `text` (type string).  
   - Position it as the entry point.

3. **Add two "HTTP Request" nodes** for calling Hugging Face endpoints:  

   a. **Get Performance Score**  
      - Method: POST  
      - URL: Paste the Hugging Face inference endpoint URL for the **Text Performance Prediction (Longform)** model.  
      - Authentication: Use the Hugging Face API credential (configured with your API access token).  
      - Headers: Add `Accept: application/json`.  
      - Body: Raw JSON with:  
        ```json
        {
          "inputs": {{ JSON.stringify($json.text) }},
          "parameters": {}
        }
        ```  
      - Connect input from `When Executed by Another Workflow`.

   b. **Get Preference Score**  
      - Same setup as above, but URL is for **Animal Advocate Preference Prediction (Longform)** model endpoint.  
      - Connect input from `When Executed by Another Workflow`.

4. **Add two "Set" nodes** to extract scores:  

   a. **Set Performance Score**  
      - Assign new field `performance_score` of type number with value `={{ $json.score }}`.  
      - Connect input from `Get Performance Score`.

   b. **Get Preference Score2**  
      - Assign new field `preference_score` of type number with value `={{ $json.score }}`.  
      - Connect input from `Get Preference Score`.

5. **Add a "Merge" node** named `Merge Branches`:  
   - Use default merge mode to combine two inputs.  
   - Connect inputs from `Set Performance Score` and `Get Preference Score2`.

6. **Add an "Aggregate" node** named `Create Single Item`:  
   - Set aggregation mode to `aggregateAllItemData`.  
   - Connect input from `Merge Branches`.

7. **Add a final "Set" node** named `Set Output`:  
   - Assign the following fields:  
     - `performance_score`: `={{ $json.data[0].performance_score }}` (number)  
     - `preference_score`: `={{ $json.data[1].preference_score }}` (number)  
     - `text`: `={{ $('When Executed by Another Workflow').item.json.text }}` (string)  
   - Connect input from `Create Single Item`.

8. **Add a "Sticky Note" node** with deployment instructions:  
   - Include detailed steps for deploying Hugging Face models as inference endpoints.  
   - Provide links to the models:  
     - https://huggingface.co/open-paws/animal_advocate_preference_prediction_longform  
     - https://huggingface.co/open-paws/text_performance_prediction_longform  
   - Remind to configure API credentials with access tokens.

9. **Configure Credentials:**  
   - Create or use existing Hugging Face API credentials in n8n with an API token that has access to inference endpoints.  
   - Assign these credentials to both HTTP Request nodes.

10. **Test the sub-workflow:**  
    - Trigger the sub-workflow by executing it from another workflow with a `text` input.  
    - Verify that the output contains both `performance_score` and `preference_score` fields with valid numeric values.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| To use the scoring sub-workflow, deploy the two Hugging Face regression models as inference endpoints and configure their URLs in the HTTP Request nodes. Ensure API tokens have the required permissions for Hugging Face API access. | See sticky note inside the workflow for setup instructions and model links: https://huggingface.co/open-paws/animal_advocate_preference_prediction_longform and https://huggingface.co/open-paws/text_performance_prediction_longform |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow built with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or proprietary elements. All data processed is legal and publicly accessible.

---