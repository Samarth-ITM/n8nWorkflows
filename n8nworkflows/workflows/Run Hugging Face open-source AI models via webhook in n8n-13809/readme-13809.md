Run Hugging Face open-source AI models via webhook in n8n

https://n8nworkflows.xyz/workflows/run-hugging-face-open-source-ai-models-via-webhook-in-n8n-13809


# Run Hugging Face open-source AI models via webhook in n8n

# n8n Hugging Face — Open-Source AI Model Runner

This workflow provides a standardized interface to interact with the **Hugging Face Inference API**. It allows users to execute various AI tasks—such as text generation, summarization, sentiment analysis, translation, and image generation—using open-source models without managing local infrastructure or GPUs.

---

### 1. Workflow Overview

The workflow acts as a bridge between an external request (Webhook) and the Hugging Face ecosystem. It automates the selection of models based on task types, handles the technical API payload construction, and cleans up complex JSON responses into a structured format.

**Logical Blocks:**
*   **1.1 Input & Configuration:** Captures the incoming request and initializes environmental variables (API keys and task parameters).
*   **1.2 Logic & Payload Building:** A specialized script determines which model to use and builds the specific JSON structure required by Hugging Face for that specific task.
*   **1.3 API Execution:** Performs the secure HTTPS handshake with Hugging Face servers.
*   **1.4 Response Processing:** Normalizes the varying data structures returned by different AI models into a consistent JSON output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Configuration
*   **Overview:** Receives the task details via a POST request and sets up the internal variables.
*   **Nodes Involved:** `Receive Task Request`, `Set API Config`.
*   **Node Details:**
    *   **Receive Task Request (Webhook):** Listens on the `/hf-runner` path. It is set to `Respond to Webhook` mode, meaning it stays open until the final node sends the result back.
    *   **Set API Config (Set):** Normalizes inputs. It extracts `task`, `input`, `model`, and `parameters` from the request body. 
        *   *Key Variable:* `HF_API_KEY` (Placeholder for the user's Hugging Face token).
        *   *Edge Case:* If no task is provided, it defaults to `summarization`.

#### 2.2 Logic & Payload Building
*   **Overview:** Translates a simple user request into a model-specific API call.
*   **Nodes Involved:** `Build Model Payload`.
*   **Node Details:**
    *   **Build Model Payload (Code):** A JavaScript block that maps tasks to default models (e.g., `Mistral-7B` for text, `BART` for summarization, `Stable Diffusion` for images).
    *   *Logic:* It applies default parameters (like `max_length` or `temperature`) but allows user-provided `extraParams` to override them.
    *   *Failure Type:* Throws an error if the `input` field is missing.

#### 2.3 API Execution
*   **Overview:** Communicates with the external Hugging Face Inference server.
*   **Nodes Involved:** `Call Hugging Face API`.
*   **Node Details:**
    *   **Call Hugging Face API (HTTP Request):** Sends a POST request to `https://api-inference.huggingface.co/models/{{$json.model}}`.
    *   *Headers:* Uses `Bearer` token authentication. Includes `x-wait-for-model: true`, which forces the API to wait for the model to load into memory if it is currently idle.
    *   *Timeout:* Set to 60 seconds to accommodate large model cold starts.

#### 2.4 Response Processing
*   **Overview:** Cleans the raw data for the end-user.
*   **Nodes Involved:** `Parse and Format Response`, `Return Result`.
*   **Node Details:**
    *   **Parse and Format Response (Code):** Different models return data in different shapes (nested arrays vs. objects). This node identifies the task type and extracts the core result (e.g., `generated_text`, `summary_text`, or sentiment `label`).
    *   **Return Result (Respond to Webhook):** Returns a clean JSON object containing the `jobId`, `success` status, the model used, and the final `result`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Receive Task Request** | Webhook | Entry Point | None | Set API Config | Webhook receives the task request with input text and task type... |
| **Set API Config** | Set | Config Management | Receive Task Request | Build Model Payload | ...stores your Hugging Face API key and normalizes all inputs... |
| **Build Model Payload** | Code | Payload Logic | Set API Config | Call Hugging Face API | ...selects the right model and builds the correct API payload for the task |
| **Call Hugging Face API** | HTTP Request | External Integration | Build Model Payload | Parse and Format Response | HTTP Request calls the Hugging Face Inference API with the built payload |
| **Parse and Format Response** | Code | Data Transformation | Call Hugging Face API | Return Result | ...parses and formats the raw API response into clean structured output |
| **Return Result** | Respond to Webhook | Final Output | Parse and Format Response | None | ...returns the final result as JSON to the caller |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Webhook** node. Set HTTP Method to `POST` and Path to `hf-runner`. Set "HTTP Response" to `Using 'Respond to Webhook' Node`.
2.  **Configuration:** Add a **Set** node. Define the following string expressions:
    *   `task`: `={{ $json.body.task || 'summarization' }}`
    *   `inputText`: `={{ $json.body.input || '' }}`
    *   `HF_API_KEY`: Paste your token from [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens).
3.  **Logic Engine:** Add a **Code** node. Use JavaScript to define a `defaultModels` object mapping tasks to model strings. Ensure the code outputs a single JSON object containing the `payload` and the `model` string.
4.  **API Integration:** Add an **HTTP Request** node.
    *   URL: `https://api-inference.huggingface.co/models/{{ $json.model }}`
    *   Method: `POST`
    *   Authentication: Add a Header named `Authorization` with value `Bearer YOUR_API_KEY`.
    *   Body: Send the `payload` generated in the previous step.
5.  **Data Cleanup:** Add another **Code** node. Write logic to check if the response is an array or object. Extract the specific text field based on the task (e.g., if task is `summarization`, look for `summary_text`).
6.  **Response:** Add a **Respond to Webhook** node. Configure the JSON body to return the finalized result variables.
7.  **Connection:** Connect the nodes in the order described above.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Hugging Face Tokens** | Get your free API key at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens) |
| **Model Selection** | You can use any of the 400,000+ models by passing the `model` parameter in your POST request. |
| **Image Generation Note** | For `image_generation`, the API returns binary data. This workflow currently returns a placeholder string; for actual image use, set the HTTP node to "Buffer" response. |