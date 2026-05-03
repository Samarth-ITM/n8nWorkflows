Generate hyper-realistic images from Telegram via Gemini Nano Banana 2

https://n8nworkflows.xyz/workflows/generate-hyper-realistic-images-from-telegram-via-gemini-nano-banana-2-13778


# Generate hyper-realistic images from Telegram via Gemini Nano Banana 2

# Reference Document: Generate Hyper-realistic Images from Telegram via Gemini Nano Banana 2

This document provides a technical breakdown and reconstruction guide for an n8n workflow designed to generate high-fidelity, photorealistic images using a dual-model AI pipeline via OpenRouter.

---

### 1. Workflow Overview

The workflow automates the transformation of simple natural language descriptions from Telegram into hyper-realistic images. It leverages a "Dense Narrative" prompting strategy to bypass common AI aesthetic biases, resulting in unretouched, documentary-style photography.

**Logic Blocks:**
*   **1.1 Input & Trigger:** Receives the message from Telegram and provides immediate user feedback.
*   **1.2 Prompt Engineering (LLM):** Uses Gemini 3.1 Pro to expand a simple user request into a complex, technical JSON prompt (including camera settings and material physics).
*   **1.3 Image Generation:** Uses the expanded prompt to invoke Gemini 3.1 Flash Image Preview.
*   **1.4 Post-Processing & Delivery:** Extracts the image data, converts it to a binary file, sends it back to Telegram, and archives it to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Acknowledgment
This block initializes the process and manages user expectations regarding latency.

*   **Nodes Involved:** `Telegram Trigger`, `Send Acknowledgment`.
*   **Node Details:**
    *   **Telegram Trigger:** Monitors for new messages (`updates: [message]`).
    *   **Send Acknowledgment:** Sends a hardcoded text message: *"🎨 Got it! Generating your image..."* using the `chatId` from the trigger.
    *   **Edge Cases:** If the Telegram API is unreachable, the workflow fails at the start. Retry is enabled on the Send node.

#### 2.2 Prompt Engineering
This block converts "human-speak" into "AI-optimal" technical specifications.

*   **Nodes Involved:** `Config`, `Expand to JSON Prompt`.
*   **Node Details:**
    *   **Config (Set):** Stores the `chatId`, `userMessage`, and a massive `systemPrompt`. The system prompt contains strict "Prompt Rules" for Camera Math (focal length, ISO < 800), explicit skin imperfections, and lighting physics.
    *   **Expand to JSON Prompt (HTTP Request):** Calls OpenRouter (`google/gemini-3.1-pro-preview`).
        *   **Method:** POST to `https://openrouter.ai/api/v1/chat/completions`.
        *   **Input:** System prompt and user message formatted as a JSON body.
        *   **Output:** A structured JSON object containing prompt, negative_prompt, and camera settings.

#### 2.3 Parsing & Generation
Prepares the data for the image model and executes the generation.

*   **Nodes Involved:** `Parse and Prepare`, `Generate Image`.
*   **Node Details:**
    *   **Parse and Prepare (Code):** Uses JavaScript to clean Markdown code fences (e.g., ` ```json `) from the LLM response and parses the string into a valid object. It outputs a flat structure: `chatId` and `imagePrompt`.
    *   **Generate Image (HTTP Request):** Calls OpenRouter (`google/gemini-3.1-flash-image-preview`).
        *   **Specifics:** Uses `modalities: ['image', 'text']`.
        *   **Input:** Prefixes the user content with "Generate an image: " followed by the JSON prompt string.
    *   **Failure Handling:** If either the prompt expansion or image generation fails, the flow redirects to "Send Failure Notification" nodes.

#### 2.4 Delivery & Archiving
Handles the binary data and sends it to the final destinations.

*   **Nodes Involved:** `Get Image URL`, `Convert to File`, `Send Photo to Telegram`, `Upload to Google Drive`.
*   **Node Details:**
    *   **Get Image URL (Set):** Extracts the Base64 string from the API response. It uses a `.split(",")[1]` expression to strip the data URI prefix.
    *   **Convert to File:** Converts the Base64 string into a binary file named `image_file`.
    *   **Send Photo to Telegram:** Sends the binary file back to the original `chatId` with a success caption.
    *   **Upload to Google Drive:** Saves the image as `image.png` into a specific folder (ID: `1CYgRUELLCpM1tHx5cM6hBpf5RdfxItR0`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Telegram Trigger** | telegramTrigger | Workflow Entry | (None) | Send Acknowledgment | Trigger & Acknowledgment |
| **Send Acknowledgment** | telegram | User Feedback | Telegram Trigger | Config | Trigger & Acknowledgment |
| **Config** | set | Variable Definition | Send Acknowledgment | Expand to JSON Prompt | Prompt Engineering |
| **Expand to JSON Prompt**| httpRequest | Prompt Expansion | Config | Parse and Prepare | Prompt Engineering |
| **Parse and Prepare** | code | Data Sanitization | Expand to JSON Prompt | Generate Image | (None) |
| **Generate Image** | httpRequest | Image Creation | Parse and Prepare | Get Image URL | Image Generation & Delivery |
| **Get Image URL** | set | String Extraction | Generate Image | Convert to File | Image Generation & Delivery |
| **Convert to File** | convertToFile | Binary Processing | Get Image URL | Send Photo, Google Drive | Image Generation & Delivery |
| **Send Photo to Telegram**| telegram | Result Delivery | Convert to File | (None) | Image Generation & Delivery |
| **Upload to Google Drive**| googleDrive | Archiving | Convert to File | (None) | Image Generation & Delivery |
| **Send Failure Notif...** | telegram | Error Handling | Gen. Image / Expand | (None) | (None) |

---

### 4. Reproducing the Workflow from Scratch

1.  **Telegram Setup:** Create a bot via `@BotFather`. Create **Telegram API** credentials in n8n.
2.  **OpenRouter Setup:** Create an account at `openrouter.ai`, generate an API key, and add it to n8n as **OpenRouter API** (Header Auth: `Authorization: Bearer <key>`).
3.  **Step 1: Trigger:** Add a `Telegram Trigger`. Set updates to `message`.
4.  **Step 2: Acknowledgment:** Add a `Telegram` node (Operation: `sendMessage`). Set `Chat ID` to `={{ $json.message.chat.id }}`.
5.  **Step 3: Configuration:** Add a `Set` node. Create:
    *   `chatId`: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
    *   `userMessage`: `={{ $('Telegram Trigger').item.json.message.text }}`
    *   `systemPrompt`: Copy the detailed technical prompt provided in the `Config` node of the JSON (this is critical for the "Banana 2" realism style).
6.  **Step 4: LLM Expansion:** Add an `HTTP Request` node.
    *   **URL:** `https://openrouter.ai/api/v1/chat/completions`
    *   **Method:** POST.
    *   **Body:** JSON including `model: google/gemini-3.1-pro-preview` and the messages array using variables from Step 3.
7.  **Step 5: Sanitization:** Add a `Code` node. Use the JS snippet to trim backticks (```) and `JSON.parse` the content to ensure it is valid.
8.  **Step 6: Generation:** Add an `HTTP Request` node.
    *   **Model:** `google/gemini-3.1-flash-image-preview`.
    *   **Modalities:** `['image', 'text']`.
9.  **Step 7: Extraction:** Add a `Set` node to isolate the image string: `{{ $json.choices[0].message.images[0].image_url.url.split(",")[1] }}`.
10. **Step 8: Binary Conversion:** Add a `Convert to File` node. Operation: `Base64 to Binary`. Source: `image_file`.
11. **Step 9: Output:** 
    *   Add a `Telegram` node (Operation: `sendPhoto`). Enable `Binary Data` and set `Binary Property` to `data`.
    *   Add a `Google Drive` node (Operation: `upload`). Set destination folder and filename.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prompt Engineering Logic** | Based on "Nano Banana 2" principles for Gemini 3.1 models. |
| **Required Models** | Ensure your OpenRouter account has credits for `gemini-3.1-pro` and `gemini-3.1-flash-image-preview`. |
| **OpenRouter Documentation** | [https://openrouter.ai/docs](https://openrouter.ai/docs) |
| **Google Drive Requirement** | Ensure the folder ID in the node exists and the service account/OAuth has write permissions. |