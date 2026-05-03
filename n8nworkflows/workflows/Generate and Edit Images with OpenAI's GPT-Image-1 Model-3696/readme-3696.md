Generate and Edit Images with OpenAI's GPT-Image-1 Model

https://n8nworkflows.xyz/workflows/generate-and-edit-images-with-openai-s-gpt-image-1-model-3696


# Generate and Edit Images with OpenAI's GPT-Image-1 Model

### 1. Workflow Overview

This workflow enables the generation and editing of images using OpenAI's GPT-Image-1 model via n8n. It is designed for creative professionals, marketers, and e-commerce teams who want to automate image creation and modification based on textual prompts. The workflow includes logical blocks for manual triggering, image generation, base64-to-binary conversion, image editing, and final conversion for output.

**Logical Blocks:**

- **1.1 Manual Trigger:** Initiates the workflow manually for testing or controlled execution.
- **1.2 Image Generation:** Sends a prompt to OpenAI’s image generation endpoint to create an initial image in base64 format.
- **1.3 Base64 to Binary Conversion:** Converts the generated base64 image into a binary PNG file suitable for editing.
- **1.4 Image Editing:** Sends the binary image along with an edit prompt to OpenAI’s image editing endpoint.
- **1.5 Final Conversion:** Converts the edited image from base64 back to a binary PNG file for download or further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  Starts the workflow manually via a button click in the n8n UI. This block is primarily for testing and debugging the entire image generation and editing process.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **Node:** When clicking ‘Test workflow’  
    - Type: Manual Trigger  
    - Configuration: No parameters; triggers workflow on user action  
    - Inputs: None (start node)  
    - Outputs: Connects to "Create image call" node  
    - Edge Cases: None significant; user must manually trigger  
    - Version: 1  
    - Notes: Ideal for controlled testing without external triggers

---

#### 1.2 Image Generation

- **Overview:**  
  Sends a POST request to OpenAI’s `/v1/images/generations` endpoint using the GPT-Image-1 model to generate an image from a textual prompt. The response includes a base64-encoded image.

- **Nodes Involved:**  
  - Create image call

- **Node Details:**  
  - **Node:** Create image call  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://api.openai.com/v1/images/generations`  
      - Method: POST  
      - Authentication: OpenAI API credentials (named "OpenAi AIFB account")  
      - Body Parameters:  
        - model: `gpt-image-1`  
        - prompt: `"A cute red panda like dark super hero"` (example prompt)  
        - n: 1 (number of images)  
        - size: `1024x1024`  
        - moderation: `low`  
        - background: `auto`  
      - Sends body as JSON  
    - Inputs: From manual trigger  
    - Outputs: JSON response containing `data[0].b64_json` (base64 image)  
    - Edge Cases:  
      - API authentication errors  
      - Rate limiting or quota exceeded  
      - Invalid prompt or model access restrictions  
      - Network timeouts  
    - Version: 4.2

---

#### 1.3 Base64 to Binary Conversion

- **Overview:**  
  Converts the base64-encoded image string from the generation step into a binary PNG file. This binary format is required for the subsequent image editing step.

- **Nodes Involved:**  
  - Convert json binary to File

- **Node Details:**  
  - **Node:** Convert json binary to File  
    - Type: Convert To File  
    - Configuration:  
      - Operation: `toBinary`  
      - Source Property: `data[0].b64_json` (base64 string from previous node)  
      - File Name: `name_example` (static filename)  
      - MIME Type: `image/png`  
    - Inputs: From "Create image call" node  
    - Outputs: Binary data under the `data` field  
    - Edge Cases:  
      - Missing or malformed base64 data  
      - Conversion failures if input is invalid  
    - Version: 1.1

---

#### 1.4 Image Editing

- **Overview:**  
  Sends the binary PNG file along with an editing prompt to OpenAI’s `/v1/images/edits` endpoint. The node uses multipart/form-data to upload the image file and apply semantic edits.

- **Nodes Involved:**  
  - Edit Image (OpenAI)

- **Node Details:**  
  - **Node:** Edit Image (OpenAI)  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://api.openai.com/v1/images/edits`  
      - Method: POST  
      - Authentication: OpenAI API credentials (same as above)  
      - Content Type: `multipart/form-data`  
      - Body Parameters:  
        - image: binary file from previous node (`data` field)  
        - prompt: `"add a mask with horns"` (example edit prompt)  
        - model: `gpt-image-1`  
        - n: 1  
        - size: `1024x1024`  
        - quality: `high`  
    - Inputs: Binary image file from "Convert json binary to File"  
    - Outputs: JSON response with edited image base64 in `data[0].b64_json`  
    - Edge Cases:  
      - Requires binary file input; base64 input will fail  
      - API authentication or permission errors  
      - Network or timeout issues  
      - Invalid or unsupported prompt for editing  
    - Version: 4.2

---

#### 1.5 Final Conversion

- **Overview:**  
  Converts the edited image’s base64 string back into a binary PNG file, making it ready for download, preview, or further processing.

- **Nodes Involved:**  
  - Convert json binary to File final

- **Node Details:**  
  - **Node:** Convert json binary to File final  
    - Type: Convert To File  
    - Configuration:  
      - Operation: `toBinary`  
      - Source Property: `data[0].b64_json` (from "Edit Image (OpenAI)")  
      - File Name: (empty, defaults to dynamic or generic)  
      - MIME Type: `image/png`  
    - Inputs: From "Edit Image (OpenAI)" node  
    - Outputs: Final binary image file under `data` field  
    - Edge Cases:  
      - Missing or malformed base64 data from edit response  
      - Conversion failures if input is invalid  
    - Version: 1.1

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                     | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                           |
|-----------------------------|--------------------|-----------------------------------|----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger     | Starts workflow manually           | None                       | Create image call            | 🧪 Manual Trigger: Starts the workflow manually. Ideal for testing and debugging purposes.          |
| Create image call            | HTTP Request       | Generates image from prompt        | When clicking ‘Test workflow’ | Convert json binary to File  | 🎨 Image Generation (OpenAI): Sends POST to `/v1/images/generations` with `gpt-image-1` model.      |
| Convert json binary to File  | Convert To File    | Converts base64 image to binary    | Create image call           | Edit Image (OpenAI)          | 🧾 Convert base64 to File: Converts `b64_json` to binary PNG file for next step.                     |
| Edit Image (OpenAI)          | HTTP Request       | Edits image using OpenAI endpoint  | Convert json binary to File | Convert json binary to File final | ✏️ Image Editing (OpenAI): Sends binary image to `/v1/images/edits` with prompt and model.          |
| Convert json binary to File final | Convert To File | Converts edited base64 to binary   | Edit Image (OpenAI)         | None                        | 🧾 Final Conversion (base64 → File): Converts edited image to downloadable PNG file.                 |
| Sticky Note                 | Sticky Note        | Notes and explanations             | None                       | None                        | 🧠 Image AI Workflow Overview: Detailed workflow description, requirements, cost warning, use cases. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.  
   - No parameters needed. This node starts the workflow on manual click.

2. **Create Image Generation Node**  
   - Add an **HTTP Request** node named `Create image call`.  
   - Set method to `POST`.  
   - URL: `https://api.openai.com/v1/images/generations`.  
   - Authentication: Select or create OpenAI API credentials (OAuth2 or API key) named e.g. `OpenAi AIFB account`.  
   - Body Parameters (as JSON):  
     - `model`: `gpt-image-1`  
     - `prompt`: `"A cute red panda like dark super hero"` (replaceable)  
     - `n`: 1  
     - `size`: `1024x1024`  
     - `moderation`: `low`  
     - `background`: `auto`  
   - Connect output of manual trigger node to this node.

3. **Create Base64 to Binary Conversion Node**  
   - Add a **Convert To File** node named `Convert json binary to File`.  
   - Operation: `toBinary`.  
   - Source Property: `data[0].b64_json`.  
   - File Name: `name_example`.  
   - MIME Type: `image/png`.  
   - Connect output of `Create image call` to this node.

4. **Create Image Editing Node**  
   - Add an **HTTP Request** node named `Edit Image (OpenAI)`.  
   - Method: `POST`.  
   - URL: `https://api.openai.com/v1/images/edits`.  
   - Authentication: Use the same OpenAI API credentials as before.  
   - Content Type: `multipart/form-data`.  
   - Body Parameters:  
     - `image`: Set as form binary data, input field name `data` (connect binary data from previous node).  
     - `prompt`: `"add a mask with horns"` (replaceable).  
     - `model`: `gpt-image-1`.  
     - `n`: 1.  
     - `size`: `1024x1024`.  
     - `quality`: `high`.  
   - Connect output of `Convert json binary to File` node to this node.

5. **Create Final Conversion Node**  
   - Add a **Convert To File** node named `Convert json binary to File final`.  
   - Operation: `toBinary`.  
   - Source Property: `data[0].b64_json`.  
   - File Name: leave empty or set dynamically.  
   - MIME Type: `image/png`.  
   - Connect output of `Edit Image (OpenAI)` node to this node.

6. **Credentials Setup**  
   - Create or verify OpenAI API credentials in n8n under Credentials.  
   - Ensure the API key has access to the `gpt-image-1` model and image generation/editing endpoints.  
   - Assign these credentials to both HTTP Request nodes.

7. **Testing**  
   - Save the workflow.  
   - Click the manual trigger node’s “Execute Node” button to run the workflow step-by-step or run the full workflow.  
   - Inspect outputs at each step, especially the binary image files.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow requires an OpenAI account with access to the `gpt-image-1` model, which may require organizational verification.  | https://platform.openai.com/account/api-keys                                                    |
| Cost per image generation/edit varies between $0.020 and $0.190 depending on resolution and usage. Monitor usage carefully.       | https://platform.openai.com/account/usage                                                       |
| Suggested expansions: add Webhook triggers, Telegram/Slack integration, dynamic prompt injection, conditional logic, and merging. | Workflow Sticky Note5 content                                                                    |
| Example prompts can be customized to suit marketing, design, e-commerce, or creative workflows.                                   | Workflow description and sticky notes                                                           |
| Use multipart/form-data for image editing requests; binary file input is mandatory (base64 will not work).                       | Sticky Note3 content                                                                             |
| Consider adding rate limiting or batch processing for large-scale image generation to avoid API throttling or cost overruns.      | Workflow Sticky Note5 content                                                                    |

---

This document provides a complete, structured reference for understanding, reproducing, and extending the "Generate and Edit Images with OpenAI's GPT-Image-1 Model" workflow in n8n.