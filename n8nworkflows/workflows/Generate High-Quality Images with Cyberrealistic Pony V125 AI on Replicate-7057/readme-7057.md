Generate High-Quality Images with Cyberrealistic Pony V125 AI on Replicate

https://n8nworkflows.xyz/workflows/generate-high-quality-images-with-cyberrealistic-pony-v125-ai-on-replicate-7057


# Generate High-Quality Images with Cyberrealistic Pony V125 AI on Replicate

---

### 1. Workflow Overview

This workflow automates the generation of high-quality cyberrealistic pony images using the **0xdino/cyberrealistic-pony-v125** AI model hosted on Replicate. It is designed for users who want to programmatically request and retrieve AI-generated images with predefined parameters.

The workflow’s logic is grouped into the following blocks:

- **1.1 Manual Trigger & API Key Setup**: User initiates execution and sets the Replicate API key.
- **1.2 Prediction Creation**: Sends a request to Replicate’s API to start the image generation prediction.
- **1.3 Prediction Monitoring Loop**: Polls the prediction status periodically until completion.
- **1.4 Result Processing**: Processes and structures the final prediction output for downstream use or display.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger & API Key Setup

**Overview:**  
This initial block waits for the user to manually start the workflow and configures the necessary authentication by setting the Replicate API key.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow manually.  
  - *Configuration:* No parameters needed; simply triggers the workflow when the user clicks "Execute".  
  - *Inputs:* None  
  - *Outputs:* One output connected to "Set API Key".  
  - *Failure Modes:* Minimal; only user interaction required.

- **Set API Key**  
  - *Type:* Set Node  
  - *Role:* Assigns the Replicate API key as a workflow variable.  
  - *Configuration:* Sets a string variable `replicate_api_key` with the placeholder value `"YOUR_REPLICATE_API_KEY"`.  
  - *Expressions Used:* None.  
  - *Inputs:* From manual trigger.  
  - *Outputs:* Passes data to "Create Prediction".  
  - *Failure Modes:* Misconfiguration if API key is left as the placeholder or incorrect key used.  
  - *Note:* User must replace `"YOUR_REPLICATE_API_KEY"` with a valid API token.

---

#### 1.2 Prediction Creation

**Overview:**  
This block initiates the AI image generation by sending an authenticated POST request to Replicate’s API with model version and input parameters.

**Nodes Involved:**  
- Create Prediction

**Node Details:**

- **Create Prediction**  
  - *Type:* HTTP Request  
  - *Role:* Calls Replicate API endpoint `/v1/predictions` to create a new prediction job.  
  - *Configuration:*  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Request body (JSON):  
      ```json
      {
        "version": "bf02541d927899ad63dae3abbfbd5185f5a00f09401ccbe5bfd0f66d6283ea65",
        "input": {
          "cfg": 4,
          "steps": 40,
          "width": 768,
          "height": 1152,
          "denoise": 0.98,
          "facerestore": true
        }
      }
      ```  
    - Headers include:  
      - `Authorization: Bearer <replicate_api_key>` (dynamic from Set API Key node)  
      - `Content-Type: application/json`  
    - Timeout: 60 seconds  
  - *Expressions:* Authorization header uses expression:  
    ```javascript
    'Bearer ' + $('Set API Key').item.json.replicate_api_key
    ```  
  - *Inputs:* From "Set API Key"  
  - *Outputs:* Response containing prediction metadata, including prediction ID and status. Connected to "Extract Prediction ID".  
  - *Failure Modes:*  
    - Auth errors if API key invalid or missing.  
    - Network timeouts or API errors if Replicate is down.  
    - Incorrect parameter formats may cause API rejection.  
  - *Version-specific:* Uses Replicate API version v1 prediction endpoint and a specific model version hash.

---

#### 1.3 Prediction Monitoring Loop

**Overview:**  
This block extracts the prediction ID from the initial response, then enters a polling loop that checks the status of the prediction at 2-second intervals until completion.

**Nodes Involved:**  
- Extract Prediction ID  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Extract Prediction ID**  
  - *Type:* Code  
  - *Role:* Parses the prediction creation response to extract the prediction ID and initial status, constructs URL for subsequent status checks.  
  - *Configuration:* Runs once per item; JavaScript code:  
    ```javascript
    const prediction = $input.item.json;
    const predictionId = prediction.id;
    const initialStatus = prediction.status;

    return {
      predictionId: predictionId,
      status: initialStatus,
      predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
    };
    ```  
  - *Inputs:* From "Create Prediction"  
  - *Outputs:* JSON object with predictionId, status, and predictionUrl. Connected to "Wait".  
  - *Failure Modes:* If API response lacks expected fields, code may error or return undefined URLs.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses workflow execution for 2 seconds between status checks to avoid rate limiting.  
  - *Configuration:* Wait time set to 2 seconds.  
  - *Inputs:* From "Extract Prediction ID" or "Check If Complete" (if not done)  
  - *Outputs:* Triggers "Check Prediction Status".  
  - *Failure Modes:* Minimal; but long runs may cause workflow timeout in n8n instance settings.

- **Check Prediction Status**  
  - *Type:* HTTP Request  
  - *Role:* Sends a GET request to the prediction status URL to retrieve updated status and output.  
  - *Configuration:*  
    - URL: Dynamic from `predictionUrl` field in JSON  
    - Method: GET (default)  
    - Headers: Authorization with Bearer token from "Set API Key" node  
  - *Inputs:* From "Wait"  
  - *Outputs:* Passes updated prediction JSON to "Check If Complete".  
  - *Failure Modes:*  
    - Auth errors if token expired.  
    - Network or API errors.  
    - Unexpected response format.

- **Check If Complete**  
  - *Type:* If  
  - *Role:* Evaluates if prediction status equals `"succeeded"` to determine if polling should continue.  
  - *Configuration:* Boolean condition:  
    ```javascript
    $json.status === 'succeeded'
    ```  
  - *Inputs:* From "Check Prediction Status"  
  - *Outputs:*  
    - True branch: to "Process Result"  
    - False branch: loops back to "Wait" for another polling iteration.  
  - *Failure Modes:* Status may be `"failed"`, `"canceled"`, or other non-success states not explicitly handled here, possibly causing indefinite polling.

---

#### 1.4 Result Processing

**Overview:**  
Once the prediction is successfully completed, this block processes the output data, extracting key metadata and output URLs for further use.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - *Type:* Code  
  - *Role:* Extracts relevant fields from the final prediction JSON and formats them into a simplified object.  
  - *Configuration:* JavaScript code:  
    ```javascript
    const result = $input.item.json;

    return {
      status: result.status,
      output: result.output,
      metrics: result.metrics,
      created_at: result.created_at,
      completed_at: result.completed_at,
      model: '0xdino/cyberrealistic-pony-v125',
      other_url: result.output
    };
    ```  
  - *Inputs:* From "Check If Complete" (true branch)  
  - *Outputs:* Structured result object, suitable for logging, storage, or UI display.  
  - *Failure Modes:* If `output` field is missing or empty, downstream nodes may fail or produce no image.  
  - *Notes:* The field `other_url` duplicates `output` for convenience.

---

### 3. Summary Table

| Node Name              | Node Type         | Functional Role                      | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                         |
|------------------------|-------------------|-------------------------------------|------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger    | Entry point to start workflow        | -                      | Set API Key                |                                                                                                                     |
| Set API Key            | Set               | Sets Replicate API key               | On clicking 'execute'   | Create Prediction          |                                                                                                                     |
| Create Prediction      | HTTP Request      | Sends prediction creation request   | Set API Key             | Extract Prediction ID      |                                                                                                                     |
| Extract Prediction ID  | Code              | Extracts prediction ID & status      | Create Prediction       | Wait                      |                                                                                                                     |
| Wait                   | Wait              | Pauses between polling cycles       | Extract Prediction ID / Check If Complete (false) | Check Prediction Status |                                                                                                                     |
| Check Prediction Status| HTTP Request      | Polls prediction status              | Wait                   | Check If Complete          |                                                                                                                     |
| Check If Complete      | If                | Checks if prediction succeeded      | Check Prediction Status | Process Result / Wait      |                                                                                                                     |
| Process Result         | Code              | Processes final prediction output   | Check If Complete (true)| -                         |                                                                                                                     |
| Sticky Note            | Sticky Note       | Provides workflow overview & setup  | -                      | -                         | ## 0xdino Cyberrealistic Pony V125 AI Generator<br><br>This workflow uses the **0xdino/cyberrealistic-pony-v125** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: 0xdino<br>- **Required Fields**: None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a node of type **Manual Trigger** named `"On clicking 'execute'"`.  
   - No parameters needed.

2. **Create Set Node for API Key**  
   - Add a **Set** node named `"Set API Key"`.  
   - Add a string field `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"`.  
   - Connect output of `"On clicking 'execute'"` to `"Set API Key"`.  
   - Replace `"YOUR_REPLICATE_API_KEY"` with your actual Replicate API token before use.

3. **Create HTTP Request Node to Create Prediction**  
   - Add an **HTTP Request** node named `"Create Prediction"`.  
   - Configure as follows:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: Generic Credential with HTTP Header Auth  
     - Add Header:  
       - `Authorization`: Set expression to `'Bearer ' + $('Set API Key').item.json.replicate_api_key`  
       - `Content-Type`: `application/json`  
     - Request Body Type: JSON  
     - JSON Body:  
       ```json
       {
         "version": "bf02541d927899ad63dae3abbfbd5185f5a00f09401ccbe5bfd0f66d6283ea65",
         "input": {
           "cfg": 4,
           "steps": 40,
           "width": 768,
           "height": 1152,
           "denoise": 0.98,
           "facerestore": true
         }
       }
       ```  
     - Timeout: 60000 ms (60 seconds)  
   - Connect output of `"Set API Key"` to this node.

4. **Create Code Node to Extract Prediction ID**  
   - Add a **Code** node named `"Extract Prediction ID"`.  
   - Set mode: Run Once For Each Item.  
   - JavaScript code:  
     ```javascript
     const prediction = $input.item.json;
     const predictionId = prediction.id;
     const initialStatus = prediction.status;

     return {
       predictionId,
       status: initialStatus,
       predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
     };
     ```  
   - Connect output of `"Create Prediction"` to this node.

5. **Create Wait Node**  
   - Add a **Wait** node named `"Wait"`.  
   - Set wait time to 2 seconds.  
   - Connect output of `"Extract Prediction ID"` to `"Wait"`.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Add an **HTTP Request** node named `"Check Prediction Status"`.  
   - Configure as:  
     - URL: Expression `{{$json.predictionUrl}}`  
     - Method: GET (default)  
     - Authentication: Generic Credential, HTTP Header Auth  
     - Header:  
       - `Authorization`: `'Bearer ' + $('Set API Key').item.json.replicate_api_key`  
   - Connect output of `"Wait"` to this node.

7. **Create If Node to Check Completion**  
   - Add an **If** node named `"Check If Complete"`.  
   - Condition: Boolean  
     - Value 1: `{{$json.status}}`  
     - Operator: equals  
     - Value 2: `succeeded`  
   - Connect output of `"Check Prediction Status"` to this node.

8. **Create Code Node to Process Result**  
   - Add a **Code** node named `"Process Result"`.  
   - Mode: Run Once For Each Item  
   - JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: '0xdino/cyberrealistic-pony-v125',
       other_url: result.output
     };
     ```  
   - Connect the **true** output of `"Check If Complete"` to this node.

9. **Connect False Branch Back to Wait**  
   - Connect the **false** output of `"Check If Complete"` back to the `"Wait"` node to continue polling.

10. **Optional: Add Sticky Note**  
    - Add a **Sticky Note** node with the following content to document the workflow:  
      ```
      ## 0xdino Cyberrealistic Pony V125 AI Generator

      This workflow uses the **0xdino/cyberrealistic-pony-v125** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: 0xdino
      - **Required Fields**: None
      ```  
    - Position it clearly near the starting nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The model version string `"bf02541d927899ad63dae3abbfbd5185f5a00f09401ccbe5bfd0f66d6283ea65"` is specific to 0xdino's Cyberrealistic Pony V125 model on Replicate. | Model version identifier - keep updated from Replicate if model updates are released.           |
| Polling interval is set to 2 seconds to balance responsiveness and API rate limits. Adjust if needed based on your usage tier.                              | Rate limiting considerations for Replicate API.                                                |
| The workflow does not explicitly handle prediction failure or cancellation statuses, which could cause indefinite polling loops. Consider adding logic to handle these states. | Recommended improvement for robustness.                                                        |
| Replace `"YOUR_REPLICATE_API_KEY"` with a valid API token from your Replicate account to authenticate API calls.                                             | Replicate API key setup.                                                                        |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---