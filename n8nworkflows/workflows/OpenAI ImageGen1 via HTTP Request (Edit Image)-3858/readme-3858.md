OpenAI ImageGen1 via HTTP Request (Edit Image)

https://n8nworkflows.xyz/workflows/openai-imagegen1-via-http-request--edit-image--3858


# OpenAI ImageGen1 via HTTP Request (Edit Image)

### 1. Workflow Overview

This workflow automates the process of editing an existing image using OpenAI's ImageGen1 API via an HTTP request. It is designed for content creators, marketers, and developers who want to streamline image editing by programmatically applying prompts to images and receiving edited versions without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the incoming chat message with an attached image and prompt.
- **1.2 API Key Injection:** Inserts the OpenAI API key into the workflow data for authentication.
- **1.3 OpenAI Image Edit Request:** Sends the image and prompt to OpenAI’s ImageGen1 edit endpoint using an HTTP Request node.
- **1.4 Image Conversion:** Converts the base64 JSON response from OpenAI into a binary file for further use.
- **1.5 Output & Automation:** The final edited image is ready for downstream actions such as storage or sharing (not included but easily extendable).

Sticky notes provide setup instructions, usage tips, and promotional information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives the chat message input, including the image and the text prompt, via a webhook-enabled chat trigger node.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **Type:** Chat Trigger (Langchain)  
  - **Role:** Entry point for the workflow, listens for chat messages with optional file uploads.  
  - **Configuration:**  
    - Webhook enabled with ID `449bbfbc-0523-406f-94a2-089bca9d7295`  
    - Allows file uploads of any MIME type (`*`)  
  - **Expressions/Variables:**  
    - The prompt text is later accessed via `$('When chat message received').item.json.chatInput`  
  - **Connections:**  
    - Output connected to `API KEY` node  
  - **Version:** 1.1  
  - **Edge Cases / Failure Modes:**  
    - Missing or invalid file upload  
    - Empty or malformed chat input  
    - Webhook connectivity issues  
  - **Sub-workflow:** None

#### 2.2 API Key Injection

- **Overview:**  
  Injects the OpenAI API key into the workflow data to authenticate subsequent API requests.

- **Nodes Involved:**  
  - API KEY

- **Node Details:**  
  - **Type:** Set node  
  - **Role:** Adds a new field `openAIKey` containing the OpenAI secret key to the JSON data.  
  - **Configuration:**  
    - Assigns `openAIKey` with a string value (e.g., `sk-proj-...`)  
    - Includes other fields from input unchanged  
  - **Expressions/Variables:** None dynamic; static key assignment  
  - **Connections:**  
    - Input from `When chat message received`  
    - Output to `HTTP Request`  
  - **Version:** 3.4  
  - **Edge Cases / Failure Modes:**  
    - Missing or invalid API key will cause authentication errors downstream  
  - **Sub-workflow:** None

#### 2.3 OpenAI Image Edit Request

- **Overview:**  
  Sends a multipart/form-data HTTP POST request to OpenAI’s ImageGen1 edit endpoint with the original image and prompt, requesting an edited image.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Calls OpenAI ImageGen1 API to edit the image based on the prompt.  
  - **Configuration:**  
    - URL: `https://api.openai.com/v1/images/edits`  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Body Parameters:  
      - `image`: binary data from input field `data0` (the uploaded image)  
      - `prompt`: dynamic expression `={{ $('When chat message received').item.json.chatInput }}`  
      - `model`: fixed string `gpt-image-1`  
      - `n`: 1 (number of images to generate)  
      - `size`: `1024x1024` (image resolution)  
      - `quality`: `high`  
    - Header Parameters:  
      - Authorization: Bearer token from `{{$json.openAIKey}}`  
  - **Expressions/Variables:**  
    - Prompt dynamically taken from chat input  
    - Authorization header dynamically injected from previous node  
  - **Connections:**  
    - Input from `API KEY`  
    - Output to `Convert to File`  
  - **Version:** 4.2  
  - **Edge Cases / Failure Modes:**  
    - Authentication failure if API key is invalid or missing  
    - Network timeouts or API rate limits  
    - Invalid or corrupt image data in binary field  
    - API errors if prompt is empty or malformed  
  - **Sub-workflow:** None

#### 2.4 Image Conversion

- **Overview:**  
  Converts the base64-encoded JSON response from the OpenAI API into a binary file format for downstream processing or storage.

- **Nodes Involved:**  
  - Convert to File

- **Node Details:**  
  - **Type:** Convert To File  
  - **Role:** Converts base64 JSON string (`data[0].b64_json`) from the API response into binary file data.  
  - **Configuration:**  
    - Operation: `toBinary`  
    - Source Property: `data[0].b64_json` (path to base64 image data in API response)  
  - **Expressions/Variables:** None dynamic; fixed source property  
  - **Connections:**  
    - Input from `HTTP Request`  
    - Output: terminal (ready for further extension)  
  - **Version:** 1.1  
  - **Edge Cases / Failure Modes:**  
    - Missing or malformed base64 data in API response  
    - Conversion errors if data is corrupted  
  - **Sub-workflow:** None

#### 2.5 Sticky Notes (Documentation & Promotion)

- **Overview:**  
  Provides user guidance, setup instructions, and promotional content within the workflow canvas.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  
  - **Type:** Sticky Note  
  - **Role:**  
    - First sticky note explains setup steps for API key, organization verification, and usage tips.  
    - Second sticky note promotes a paid AI image automation template with links and marketing copy.  
  - **Configuration:**  
    - Text content with markdown formatting and embedded links/images  
  - **Connections:** None (informational only)  
  - **Version:** 1  
  - **Edge Cases / Failure Modes:** None (non-executable)  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                     | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                        |
|---------------------------|-------------------------------|-----------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger (Langchain)       | Entry point; receives chat input and image | None                         | API KEY                     | See Sticky Note for setup instructions and usage tips                                                                            |
| API KEY                   | Set                           | Injects OpenAI API key into data  | When chat message received    | HTTP Request                | See Sticky Note for setup instructions and usage tips                                                                            |
| HTTP Request              | HTTP Request                  | Calls OpenAI ImageGen1 edit API   | API KEY                      | Convert to File             | See Sticky Note for setup instructions and usage tips                                                                            |
| Convert to File           | Convert To File               | Converts base64 JSON to binary file | HTTP Request                 | None                       | See Sticky Note for setup instructions and usage tips                                                                            |
| Sticky Note               | Sticky Note                   | Setup instructions and tips       | None                         | None                       | Contains detailed setup guide and OpenAI organization verification link                                                          |
| Sticky Note1              | Sticky Note                   | Promotional content for AI Image Cash Machine template | None                         | None                       | Contains marketing banner and links to premium AI image automation template                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: `When chat message received`  
   - Enable webhook with a unique ID (e.g., `449bbfbc-0523-406f-94a2-089bca9d7295`)  
   - Allow file uploads with MIME type set to `*` (all types)  
   - Position: Left side for input

2. **Create the Set Node for API Key Injection**  
   - Type: `Set`  
   - Name: `API KEY`  
   - Add an assignment:  
     - Field Name: `openAIKey`  
     - Type: String  
     - Value: Your OpenAI secret key (e.g., `sk-...`)  
   - Enable "Include Other Fields" to pass through existing data  
   - Connect output of `When chat message received` to input of `API KEY`

3. **Create the HTTP Request Node to Call OpenAI Image Edit API**  
   - Type: `HTTP Request`  
   - Name: `HTTP Request`  
   - Set URL to `https://api.openai.com/v1/images/edits`  
   - Method: POST  
   - Content Type: `multipart/form-data`  
   - Enable "Send Body" and "Send Headers"  
   - Add Body Parameters:  
     - `image`: type `formBinaryData`, input field name `data0` (the binary image from chat trigger)  
     - `prompt`: expression `={{ $('When chat message received').item.json.chatInput }}`  
     - `model`: `gpt-image-1`  
     - `n`: `1`  
     - `size`: `1024x1024`  
     - `quality`: `high`  
   - Add Header Parameter:  
     - `Authorization`: expression `=Bearer {{ $json.openAIKey }}`  
   - Connect output of `API KEY` to input of `HTTP Request`

4. **Create the Convert To File Node**  
   - Type: `Convert To File`  
   - Name: `Convert to File`  
   - Operation: `toBinary`  
   - Source Property: `data[0].b64_json` (path to base64 image data in HTTP response)  
   - Connect output of `HTTP Request` to input of `Convert to File`

5. **(Optional) Add Nodes for Further Processing**  
   - For example, upload to S3, post to Slack, or store in Google Drive  
   - Connect output of `Convert to File` accordingly

6. **Add Sticky Notes for Documentation**  
   - Create Sticky Note nodes with setup instructions and promotional content as desired  
   - Position them clearly on the canvas for user guidance

7. **Credential Setup**  
   - In n8n Credentials, create or update an OpenAI API credential with your secret key  
   - Ensure the key is valid and has access to the ImageGen1 API  
   - No additional credentials are required for the HTTP Request node since the key is passed dynamically

8. **Activate and Test the Workflow**  
   - Activate the workflow  
   - Trigger the webhook by sending a chat message with an attached image and prompt  
   - Verify the edited image is received and converted correctly

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| ### 🖼️ Edit Images with the **OpenAI ImageGen v1** API  1. Verify your OpenAI organization is confirmed: [OpenAI Settings → Organization](https://platform.openai.com/settings/organization/general)  2. Add your OpenAI API key in n8n credentials  3. Run the chat trigger node supplying text prompt and source image  4. Preview the new image in the `Convert to File` node, then automate sending or storage  *Tip:* Chain nodes to watermark, resize, or schedule social posts automatically.                                                                                                                     | Sticky Note content in workflow for setup and usage instructions                                            |
| [![AI-Image Cash Machine – banner](https://public-files.gumroad.com/r2rfepypwzxebylbzutm7lk577m7)](https://drauscher.gumroad.com/l/PremiumAISaaSTemplateBeginnerFriendlyCustomizable)  Launch the **AI-Image Cash Machine** template with Next.js frontend, Stripe payments, Supabase storage, and n8n backend automation. Includes step-by-step PDF setup guide and sample environment file.  Special summer deal with discount code `SUMMER25`. Try live demo at Pixarify Online.                                                                                                                                                                                                                 | Promotional sticky note linking to premium AI image automation template on Gumroad                         |

---

This documentation provides a complete, structured reference to understand, reproduce, and extend the OpenAI ImageGen1 image editing workflow in n8n. It anticipates key failure points such as API authentication, image upload issues, and response parsing, enabling robust customization and integration.