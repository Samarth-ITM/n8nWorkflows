Free AI Image Generator - n8n Automation Workflow with Gemini/ChatGPT

https://n8nworkflows.xyz/workflows/free-ai-image-generator---n8n-automation-workflow-with-gemini-chatgpt-5626


# Free AI Image Generator - n8n Automation Workflow with Gemini/ChatGPT

### 1. Workflow Overview

This workflow automates AI-driven image creation triggered by chat messages, using Google Gemini’s image generation capabilities. It is designed for users who want to generate high-quality, realistic images from textual prompts efficiently and at scale. The workflow can be triggered via an n8n chat interface or Telegram and supports previewing and saving the generated images either locally or as Telegram replies.

The logic is divided into four main blocks:

- **1.1 Input Reception and Initialization:** Captures incoming chat messages and sets default parameters such as image size and model.
- **1.2 AI Prompt Generation:** Uses an AI agent powered by Google Gemini to convert the chat message into a structured, detailed image generation prompt.
- **1.3 Image Generation and Processing:** Cleans AI output, formats prompts, and sends them to an image generation API to create images.
- **1.4 Image Delivery and Storage:** Prepares images for preview and saving, allowing output either as Telegram messages or local files.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
This block listens for incoming chat messages (either via n8n chat or Telegram) and sets default values such as image size and model type for image generation.

**Nodes Involved:**  
- When chat message received  
- Telegram Trigger (disabled)  
- Fields - Set Values

**Node Details:**

- **When chat message received**  
  - *Type:* Chat trigger (Langchain chatTrigger)  
  - *Role:* Entry point for workflow via n8n chat interface  
  - *Config:* Uses a webhook ID; no additional options set  
  - *Inputs:* External chat message  
  - *Outputs:* Passes chat input text downstream  
  - *Edge cases:* Webhook misconfiguration or network failure may cause trigger failure

- **Telegram Trigger** (disabled)  
  - *Type:* Telegram message trigger  
  - *Role:* Optional alternative entry point via Telegram messages  
  - *Config:* Watches for "message" updates; currently disabled  
  - *Edge cases:* Disabled, so no run impact

- **Fields - Set Values**  
  - *Type:* Set node  
  - *Role:* Initializes default parameters for image generation  
  - *Config:* Sets `model` to `"flux"`, `width` to `"1080"`, and `height` to `"1920"`  
  - *Inputs:* From chat trigger nodes  
  - *Outputs:* Supplies defaults to AI agent and HTTP request nodes  
  - *Edge cases:* Hardcoded defaults; user must modify manually to change

---

#### 1.2 AI Prompt Generation

**Overview:**  
Transforms the raw chat input into a structured image generation prompt using a specialized AI agent configured with Google Gemini’s chat model.

**Nodes Involved:**  
- AI Agent - Create Image From Prompt  
- Google Gemini Chat Model

**Node Details:**

- **Google Gemini Chat Model**  
  - *Type:* Google Gemini language model node  
  - *Role:* Provides AI model backend for the agent node  
  - *Config:* Uses "models/gemini-2.0-flash" with specific safety settings disabled to avoid blocking content  
  - *Credentials:* Requires Google Gemini (PaLM) API credentials  
  - *Edge cases:* API quota limits, invalid credentials, or safety filter changes could cause failure

- **AI Agent - Create Image From Prompt**  
  - *Type:* Langchain agent node  
  - *Role:* Generates a detailed image prompt JSON from the chat input  
  - *Config:*  
    - Input text is the chat input from the trigger node  
    - System prompt instructs the AI to format output as a structured JSON with detailed instructions on prompt construction, including scene, elements, lighting, and technical parameters  
    - Output parser enabled to extract structured response  
  - *Inputs:* Receives chat input and Google Gemini model reference  
  - *Outputs:* Produces JSON containing image prompt(s)  
  - *Edge cases:* AI output may be malformed or incomplete; requires robust parsing downstream

---

#### 1.3 Image Generation and Processing

**Overview:**  
Cleans and extracts the relevant image prompts from the AI agent's output, formats them with parameters, and calls an external image generation HTTP API to create images.

**Nodes Involved:**  
- Code - Clean Json  
- Code - Get Prompt  
- Code - Set Filename  
- HTTP Request - Create Image

**Node Details:**

- **Code - Clean Json**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Parses AI agent's raw output string to extract the `"prompt"` fields into an array  
  - *Config:* Splits output by lines, extracts text after `"prompt":` occurrences, returns array under `image_prompt` key  
  - *Inputs:* AI Agent output JSON string  
  - *Outputs:* JSON object with `image_prompt` array  
  - *Edge cases:* If AI output format changes or no prompts found, returns empty array gracefully

- **Code - Get Prompt**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Converts the array of prompts into HTTP request body objects with image parameters  
  - *Config:* For each prompt, creates JSON with prompt text, width and height (from Fields - Set Values), and additional image generation parameters (12 inference steps, guidance scale 3.5, 1 image, safety checker enabled)  
  - *Inputs:* Output of Clean Json, also references Fields - Set Values for parameters  
  - *Outputs:* Array of JSON objects ready for HTTP request body  
  - *Edge cases:* Empty prompt array results in no image requests

- **Code - Set Filename**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Assigns filenames to each generated image sequentially (`images_001.png`, etc.)  
  - *Inputs:* Image generation response items  
  - *Outputs:* Items updated with `fileName` property  
  - *Edge cases:* Handles multiple images; if no images, no filenames set

- **HTTP Request - Create Image**  
  - *Type:* HTTP Request node  
  - *Role:* Sends prompt data to external image generation service (`https://image.pollinations.ai/prompt/`) with query parameters for size and model  
  - *Config:*  
    - URL dynamically constructed with prompt text  
    - Sends JSON query with width, height, model (from Fields - Set Values), fixed seed (42), and disables logo  
    - Headers specify JSON content type and accept JSON  
    - Retries on failure with 5-second interval  
  - *Inputs:* Receives formatted prompt JSON from previous node  
  - *Outputs:* Image creation response, including URLs or binary data for images  
  - *Edge cases:* Network timeouts, API service downtime, invalid parameters, rate limiting

---

#### 1.4 Image Delivery and Storage

**Overview:**  
Once images are generated, this block handles previewing and saving images either by sending them as Telegram messages or writing them to local storage.

**Nodes Involved:**  
- Telegram Response (disabled)  
- Save Image To Disk (disabled)

**Node Details:**

- **Telegram Response**  
  - *Type:* Telegram node  
  - *Role:* Sends generated image as a photo reply in Telegram chat  
  - *Config:* Operation set to `sendPhoto` with binary data enabled  
  - *Credentials:* Requires Telegram API credentials  
  - *Inputs:* Image data from HTTP Request node  
  - *Disabled:* This node is disabled by default; user must enable to use  
  - *Edge cases:* Telegram API limits, invalid credentials, user chat restrictions

- **Save Image To Disk**  
  - *Type:* Read/Write File node  
  - *Role:* Saves image files locally, using filenames assigned earlier  
  - *Config:* Writes files to `/files/{{filename}}` path  
  - *Inputs:* Image binary data and filenames  
  - *Disabled:* Disabled by default; must be enabled to use  
  - *Edge cases:* File system permission errors, disk space limitations

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                               | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                  |
|-----------------------------|----------------------------------|----------------------------------------------|---------------------------------|---------------------------------|--------------------------------------------------------------|
| When chat message received   | @n8n/n8n-nodes-langchain.chatTrigger | Entry point for chat input                     | -                               | Fields - Set Values             | ## **1. Get The Inputs** We’ll take your image idea from the text you send in the chat, along with any settings like image size or style. This information is parsed and prepared for the image generation step. |
| Telegram Trigger            | n8n-nodes-base.telegramTrigger   | Optional Telegram message trigger (disabled)  | -                               | Fields - Set Values             | ## **1. Get The Inputs** (same as above)                      |
| Fields - Set Values          | n8n-nodes-base.set               | Initialize default parameters (model, size)   | When chat message received, Telegram Trigger | AI Agent - Create Image From Prompt | ## **1. Get The Inputs** (same as above)                     |
| AI Agent - Create Image From Prompt | @n8n/n8n-nodes-langchain.agent    | Generate structured image prompt JSON          | Fields - Set Values, Google Gemini Chat Model | Code - Clean Json                | ## **2. Use Google Gemini's Image Model To Generate The Image** Your prompt and settings are sent to Google Gemini’s Image Generation Model. Prefer a different model? You can easily swap in alternatives like OpenAI ChatGPT or Microsoft Copilot. |
| Google Gemini Chat Model     | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides AI model backend                       | AI Agent - Create Image From Prompt (ai_languageModel) | AI Agent - Create Image From Prompt | ## **2. Use Google Gemini's Image Model** (same as above)    |
| Code - Clean Json            | n8n-nodes-base.code             | Parse and extract prompts from AI output       | AI Agent - Create Image From Prompt | Code - Get Prompt               | ## **3. Prepare The Final Image Feasible For Preview And Saving** This stage helps to clean up the raw output from the AI Agent for your image request and format it into a complete, ready-to-view and downloadable format. |
| Code - Get Prompt            | n8n-nodes-base.code             | Format prompts and parameters for HTTP request | Code - Clean Json                | Code - Set Filename             | ## **3. Prepare The Final Image** (same as above)             |
| Code - Set Filename          | n8n-nodes-base.code             | Assign filenames to images                       | Code - Get Prompt                | HTTP Request - Create Image     | ## **3. Prepare The Final Image** (same as above)             |
| HTTP Request - Create Image  | n8n-nodes-base.httpRequest      | Call external image generation API              | Code - Set Filename              | Save Image To Disk, Telegram Response | ## **4. Preview And Save The Image** Once your image is ready in a downloadable format, you’ll be able to preview it. If you're happy with the result, you can save it in one of two ways by activating either of them before running the workflow: - As a reply in Telegram chat - To your local storage (disk) |
| Save Image To Disk           | n8n-nodes-base.readWriteFile    | Save images locally                              | HTTP Request - Create Image      | -                             | ## **4. Preview And Save The Image** (same as above)           |
| Telegram Response            | n8n-nodes-base.telegram         | Send images as Telegram photo                    | HTTP Request - Create Image      | -                             | ## **4. Preview And Save The Image** (same as above)           |
| Sticky Note1                 | n8n-nodes-base.stickyNote       | Documentation and instructions                   | -                               | -                             | ## [Agent Circle's N8N Workflow] Automated AI Image Creator - Try It Out! (Extensive usage notes and links) |
| Sticky Note                 | n8n-nodes-base.stickyNote       | Documentation for block 1                         | -                               | -                             | ## **1. Get The Inputs** (same as above)                        |
| Sticky Note2                 | n8n-nodes-base.stickyNote       | Documentation for block 2                         | -                               | -                             | ## **2. Use Google Gemini's Image Model To Generate The Image** (same as above) |
| Sticky Note4                 | n8n-nodes-base.stickyNote       | Documentation for block 3                         | -                               | -                             | ## **3. Prepare The Final Image Feasible For Preview And Saving** (same as above) |
| Sticky Note3                 | n8n-nodes-base.stickyNote       | Documentation for block 4                         | -                               | -                             | ## **4. Preview And Save The Image** (same as above)            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node (Chat Message):**  
   - Add **When chat message received** node (Langchain chatTrigger).  
   - Configure webhook with a unique ID or URL.  
   - No additional options needed.

2. **(Optional) Add Telegram Trigger:**  
   - Add **Telegram Trigger** node.  
   - Set update type to "message".  
   - Leave disabled unless Telegram integration is desired.

3. **Set Default Parameters:**  
   - Add **Set** node named "Fields - Set Values".  
   - Assign fields:  
     - `model` = "flux"  
     - `width` = "1080"  
     - `height` = "1920"  
   - Connect input from chat or Telegram trigger.

4. **Configure AI Language Model Node:**  
   - Add **Google Gemini Chat Model** node.  
   - Set Model Name: `models/gemini-2.0-flash`.  
   - Adjust options:  
     - topK=40, topP=1, temperature=0.5  
     - Disable safety blocking for harassment, hate speech, explicit content, dangerous content.  
     - Max output tokens: 65536  
   - Provide Google Palm API credentials.

5. **Add AI Agent Node:**  
   - Add **AI Agent - Create Image From Prompt** (Langchain agent).  
   - Set `text` input to `{{$node["When chat message received"].json["chatInput"]}}`.  
   - Assign system message with detailed instructions for AI prompt creation (as per provided description).  
   - Enable Output Parser.  
   - Connect AI Model input to the Google Gemini Chat Model node.

6. **Clean AI Output with Code Node:**  
   - Add **Code - Clean Json** node.  
   - Use JavaScript code to parse AI output string, extracting all `"prompt"` values into an array `image_prompt`.  
   - Connect from AI Agent node.

7. **Format Prompts for HTTP Request:**  
   - Add **Code - Get Prompt** node.  
   - Map each prompt to JSON format including:  
     - prompt text  
     - image size (width, height) from Fields - Set Values node  
     - num_inference_steps=12, guidance_scale=3.5, num_images=1, enable_safety_checker=true  
   - Connect from Clean Json node.

8. **Assign Filenames:**  
   - Add **Code - Set Filename** node.  
   - JavaScript to assign sequential file names `images_001.png`, `images_002.png` etc.  
   - Connect from Get Prompt node.

9. **HTTP Request to Generate Image:**  
   - Add **HTTP Request - Create Image** node.  
   - URL: `https://image.pollinations.ai/prompt/{{ $json.body.prompt }}` (dynamic with prompt).  
   - Query JSON includes width, height, model, seed=42, nologo=true (values sourced from Fields - Set Values node).  
   - Headers: Content-Type and Accept set to application/json.  
   - Enable retry on fail with 5-second delay.  
   - Connect from Set Filename node.

10. **Output Options (Choose One or Both):**  
    - **Telegram Response:**  
      - Add **Telegram Response** node.  
      - Operation: sendPhoto, binaryData enabled.  
      - Provide Telegram API credentials.  
      - Connect from HTTP Request node.  
      - Enable this node if you want to send images back via Telegram.

    - **Save Image To Disk:**  
      - Add **Read/Write File** node named "Save Image To Disk".  
      - Operation: write.  
      - File name: `/files/{{ $json.fileName }}`.  
      - Connect from HTTP Request node.  
      - Enable this node if you want to save images locally.

11. **Add Sticky Notes:**  
    - Add sticky notes to document each block as per your needs for clarity.

12. **Activate Workflow:**  
    - Set trigger node webhook URL accessible.  
    - Enable either Telegram Response or Save Image To Disk node.  
    - Turn on workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow is a template by Agent Circle for fully automated AI image creation using Google Gemini. It is highly customizable to use other AI providers such as OpenAI or Microsoft AI Copilot. Default image size is 1080 x 1920 pixels, and the default model is "flux". To customize image size or model, update the "Fields - Set Values" node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | https://www.agentcircle.ai/                                                                                      |
| Community support and further resources are available on multiple platforms including Discord, Facebook, Gumroad, X (Twitter), YouTube, and LinkedIn.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Discord: https://discord.com/invite/jySQ2PNm Facebook Group: https://www.facebook.com/groups/aiagentcircle/ Gumroad: http://agentcircle.gumroad.com/ X: https://x.com/agent_circle YouTube: https://www.youtube.com/@agentcircle LinkedIn: https://www.linkedin.com/company/agentcircle |
| The image generation API used here is https://image.pollinations.ai/prompt/. This service supports prompt-driven image creation with parameters for size, model, and seed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | API Documentation (implied by usage)                                                                             |
| Important considerations include ensuring your Google Gemini API credentials have image generation permissions, managing rate limits, and enabling either Telegram or local file saving depending on your target output.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Credential setup required                                                                                         |
| The workflow includes extensive prompt engineering instructions embedded in the AI Agent node’s system prompt to maximize image quality, realism, and style consistency, which can be adapted or extended for different use cases.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Embedded in AI Agent node system message                                                                         |

---

*Disclaimer: The text above is generated exclusively from an automated n8n workflow export and contains no illegal or offensive content. All data processed are publicly accessible and legal.*