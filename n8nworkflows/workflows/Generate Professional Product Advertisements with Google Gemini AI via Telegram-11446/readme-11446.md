Generate Professional Product Advertisements with Google Gemini AI via Telegram

https://n8nworkflows.xyz/workflows/generate-professional-product-advertisements-with-google-gemini-ai-via-telegram-11446


# Generate Professional Product Advertisements with Google Gemini AI via Telegram

### 1. Workflow Overview

This workflow enables the generation of professional, luxury-brand-quality product advertisements by leveraging Telegram for input and Google Gemini AI for image and text analysis and enhancement. Users send a product image with an accompanying caption to a Telegram bot. The workflow processes this input to generate a commercial-grade advertisement image with detailed design instructions and sends the enhanced ad back to the user via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives images and captions from Telegram users, downloads the image file.
- **1.2 AI Processing:** Converts images to base64 and uses Google Gemini AI to analyze the caption text and generate detailed design instructions.
- **1.3 Data Preparation:** Merges the image data and AI analysis, prepares the structured payload for the image generation API.
- **1.4 Image Generation:** Sends the combined payload to Google’s Nano Banana Pro model, receives generated content including enhanced images in base64, converts these into binary image files.
- **1.5 Output Delivery:** Sends the final enhanced advertisement image back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for messages sent to the Telegram bot, downloads the product image file from Telegram servers, and prepares the image data in a usable format for AI processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Download Image File  
  - Image to Base65  

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Trigger node for Telegram messages  
    - *Configuration:* Listens for new messages (`updates: ["message"]`) from users to the bot. Uses a webhook ID as placeholder for deployment.  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Emits raw Telegram message data including images and captions.  
    - *Edge Cases:*  
      - Missing or invalid Telegram Bot API credentials.  
      - Messages without images or captions.  
      - Telegram API rate limits or downtime.

  - **Download Image File**  
    - *Type:* Telegram node to download media files  
    - *Configuration:* Uses the file ID from the first photo in the Telegram message to download the actual product image file.  
    - *Inputs:* Output from Telegram Trigger  
    - *Outputs:* Binary image file data.  
    - *Edge Cases:*  
      - File ID missing or invalid.  
      - Network errors during download.

  - **Image to Base65 (Extract From File)**  
    - *Type:* Extract binary data as base64 string  
    - *Configuration:* Converts the downloaded binary image file into a base64 property in JSON for further processing.  
    - *Inputs:* Binary image from Download Image File  
    - *Outputs:* JSON with base64-encoded image data.  
    - *Edge Cases:*  
      - File conversion errors.  
      - Large image sizes causing memory issues.

---

#### 2.2 AI Processing

- **Overview:**  
  This block uses Google Gemini AI (Nano Banana Pro model) to analyze the product image and caption text. It extracts structured text fields from the caption and generates detailed design instructions for a professional advertisement.

- **Nodes Involved:**  
  - AI Design Analysis  

- **Node Details:**

  - **AI Design Analysis**  
    - *Type:* Google Gemini AI node (`@n8n/n8n-nodes-langchain.googleGemini`)  
    - *Configuration:*  
      - System prompt defines the AI as a seasoned creative director specializing in luxury brand ads.  
      - Instructions enforce strict rules: product must remain unaltered, use only user caption text, no AI-generated taglines if caption is missing, focus on premium visual design.  
      - Caption text is parsed into headline, subheadline, description, and call to action fields strictly from user's caption.  
      - Design suggestions are extensive, covering composition, lighting, background, color grading, typography, and more, with 400+ words of detailed guidance.  
      - Output is structured JSON with all extracted and generated fields.  
    - *Inputs:* Binary image (base64) from Image to Base65, original caption from Telegram Trigger node JSON.  
    - *Outputs:* JSON with parsed caption text fields and detailed design suggestions.  
    - *Edge Cases:*  
      - API authentication failures (Google Gemini API key).  
      - Caption missing or empty — fields remain empty, no hallucination.  
      - Response latency or errors from Google Gemini.  
      - Expression errors parsing Telegram caption text.  
    - *Version Requirements:* Requires n8n version supporting `@n8n/n8n-nodes-langchain.googleGemini` node.

---

#### 2.3 Data Preparation

- **Overview:**  
  Combines the base64 image data and AI analysis output into a unified payload and prepares the structured input for the Nano Banana Pro image generation API.

- **Nodes Involved:**  
  - Combine Image & Analysis  
  - Prepare API Payload  

- **Node Details:**

  - **Combine Image & Analysis**  
    - *Type:* Merge node  
    - *Configuration:* Combines two inputs (base64 image and AI JSON analysis) into one data stream for downstream processing.  
    - *Inputs:*  
      - From Image to Base65 (image base64)  
      - From AI Design Analysis (AI JSON)  
    - *Outputs:* Single combined JSON object with image and analysis.  
    - *Edge Cases:*  
      - Mismatched or missing inputs causing merge errors.

  - **Prepare API Payload**  
    - *Type:* Code node (JavaScript)  
    - *Configuration:*  
      - Collects all items from the merge node inputs.  
      - Returns a single JSON object with input1 and input2 keys holding the respective combined data arrays.  
      - Note: The current code duplicates input arrays, which might need adjustment for specific API payload structure.  
    - *Inputs:* Combined data from Merge node  
    - *Outputs:* JSON object structured for API consumption.  
    - *Edge Cases:*  
      - Errors in JavaScript code execution.  
      - Unexpected data structure input.

---

#### 2.4 Image Generation

- **Overview:**  
  Sends the combined image and AI-generated instructions payload to the Nano Banana Pro Google Generative Language API to generate enhanced advertisement images. Converts the returned base64 images into binary files.

- **Nodes Involved:**  
  - Generate Enhanced Image  
  - Convert Base64 to Image  
  - Convert Base64 to Image (secondary)  

- **Node Details:**

  - **Generate Enhanced Image**  
    - *Type:* HTTP Request node  
    - *Configuration:*  
      - Sends POST request to Google Generative Language API endpoint for `nano-banana-pro-preview` model.  
      - Payload includes:  
        - Text part: JSON stringified AI design instructions extracted from previous step.  
        - Image part: Base64 image data.  
      - Requests response modalities of TEXT and IMAGE.  
      - Uses predefined Google Palm API credentials for authentication.  
    - *Inputs:* JSON payload from Prepare API Payload  
    - *Outputs:* JSON response containing generated text and base64-encoded image(s).  
    - *Edge Cases:*  
      - API authentication errors.  
      - API rate limits or downtime.  
      - Invalid or malformed request payloads.  
      - Large payload size causing timeouts.

  - **Convert Base64 to Image**  
    - *Type:* Convert To File node  
    - *Configuration:* Converts base64 string from first image content in API response into a binary file in n8n.  
    - *On Error:* Configured to continue workflow execution on errors.  
    - *Inputs:* API response JSON  
    - *Outputs:* Binary image file data.  
    - *Edge Cases:*  
      - Conversion failure due to invalid base64 data.

  - **Convert Base64 to Image (secondary)**  
    - *Type:* Convert To File node  
    - *Configuration:* Converts base64 string from second image content in API response (if present) into binary file.  
    - *Inputs:* Output of the first Convert Base64 to Image node  
    - *Outputs:* Binary image file data  
    - *Edge Cases:*  
      - Missing second image content.  
      - Conversion errors.

---

#### 2.5 Output Delivery

- **Overview:**  
  Sends the generated enhanced advertisement image back to the Telegram user who initiated the input.

- **Nodes Involved:**  
  - Send Image  

- **Node Details:**

  - **Send Image**  
    - *Type:* Telegram node  
    - *Configuration:*  
      - Sends a photo message to the original user’s chat ID (extracted from the first Telegram Trigger node).  
      - Uses binary image data from the final Convert Base64 to Image node.  
      - Operation set to `sendPhoto`.  
    - *Inputs:* Binary image file from Convert Base64 to Image nodes  
    - *Outputs:* Confirmation of message sent.  
    - *Edge Cases:*  
      - Telegram API authentication or permission errors.  
      - User chat ID missing or invalid.  
      - Network failures sending the image.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                         | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                                                                                                                                   |
|-------------------------|----------------------------------|---------------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger         | telegramTrigger                  | Receives user messages from Telegram  | None                         | Download Image File          |                                                                                                                                                                                                                                               |
| Download Image File      | telegram                        | Downloads product image from Telegram | Telegram Trigger             | Image to Base65, AI Design Analysis |                                                                                                                                                                                                                                               |
| Image to Base65          | extractFromFile                 | Converts binary image to base64       | Download Image File          | Combine Image & Analysis     | ## 📥 Telegram Input<br>Receives images & captions, downloads file for processing.                                                                                                                                                             |
| AI Design Analysis       | googleGemini (LangChain node)  | Analyzes caption and generates design instructions | Download Image File          | Combine Image & Analysis     | ## 🤖 AI Analysis<br>Converts image to base64. Gemini AI extracts caption text & generates detailed design instructions.                                                                                                                     |
| Combine Image & Analysis | merge                          | Merges image data and AI analysis     | Image to Base65, AI Design Analysis | Prepare API Payload          | ## 🔧 Payload Setup<br>Combines image & analysis. Prepares structured API request for Nano Banana Pro.                                                                                                                                          |
| Prepare API Payload      | code                           | Prepares API request payload           | Combine Image & Analysis     | Generate Enhanced Image      |                                                                                                                                                                                                                                               |
| Generate Enhanced Image  | httpRequest                    | Calls Nano Banana Pro API to generate enhanced ad image | Prepare API Payload          | Convert Base64 to Image      | ## 🎯 Generate & Convert<br>Sends request to Nano Banana Pro API. Converts base64 response to binary image files.                                                                                                                              |
| Convert Base64 to Image  | convertToFile                  | Converts first base64 image to binary | Generate Enhanced Image      | Send Image, Convert Base64 to Image (secondary) |                                                                                                                                                                                                                                               |
| Convert Base64 to Image  | convertToFile                  | Converts second base64 image to binary| Convert Base64 to Image      | Send Image                  |                                                                                                                                                                                                                                               |
| Send Image               | telegram                       | Sends enhanced ad image back to user  | Convert Base64 to Image, Convert Base64 to Image (secondary) | None                        | ## 📤 Telegram Output<br>Sends enhanced ad image back to user.                                                                                                                                                                                  |
| Main Overview            | stickyNote                     | Workflow summary and instructions     | None                        | None                       | ## 🎨 AI Product Ad Generator via Telegram<br>Send a product image with caption text to your Telegram bot. The workflow uses Google's Nano Banana Pro AI to analyze your image, extract caption text, generate professional design instructions, and return the ad. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure to listen for `message` updates.  
   - Use your Telegram bot webhook ID.  
   - Ensure Telegram Bot API credentials are set up in n8n.

2. **Add Download Image File Node:**  
   - Type: Telegram  
   - Operation: Download file using `fileId` from incoming photo (`{{$json.message.photo[0].file_id}}`).  
   - Connect input from Telegram Trigger node.  
   - Use same webhook ID for Telegram.

3. **Add Extract From File Node (Image to Base65):**  
   - Type: Extract From File  
   - Operation: Convert binary to property (base64).  
   - Connect input from Download Image File node.

4. **Add AI Design Analysis Node (Google Gemini):**  
   - Type: Google Gemini AI (LangChain node)  
   - Configure system prompt to describe the elite creative director role.  
   - Set instructions to parse Telegram caption text:  
     - Use expression to inject `{{$('Telegram Trigger ').item.json.message.caption}}`.  
     - Define output JSON structure: headline, subheadline, description, call to action, design suggestions.  
   - Connect input from Download Image File node (binary image).  
   - Set credentials for Google Gemini API (Google Cloud API key with Generative Language enabled).

5. **Add Merge Node (Combine Image & Analysis):**  
   - Type: Merge  
   - Mode: Default (merge inputs by index).  
   - Connect inputs from Image to Base65 and AI Design Analysis nodes.

6. **Add Code Node (Prepare API Payload):**  
   - Type: Code  
   - JavaScript code to combine inputs into single JSON object with keys `input1` and `input2`.  
   - Connect input from Merge node.

7. **Add HTTP Request Node (Generate Enhanced Image):**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/nano-banana-pro-preview:generateContent`  
   - Body (JSON):  
     - Construct JSON with `contents` array containing text (stringified from AI Design Analysis output) and inline image base64 data.  
     - Request `responseModalities` of ["TEXT", "IMAGE"].  
   - Use Google Palm API credentials.  
   - Connect input from Prepare API Payload node.

8. **Add Convert To File Node (Convert Base64 to Image):**  
   - Type: Convert To File  
   - Operation: `toBinary`  
   - Source property: `candidates[0].content.parts[0].inlineData.data` (first image base64)  
   - Connect input from HTTP Request node.  
   - Set "On Error" to continue workflow on error.

9. **Add Second Convert To File Node (Convert Base64 to Image secondary):**  
   - Type: Convert To File  
   - Operation: `toBinary`  
   - Source property: `candidates[0].content.parts[1].inlineData.data` (second image base64 if present)  
   - Connect input from first Convert To File node.

10. **Add Telegram Node (Send Image):**  
    - Type: Telegram  
    - Operation: `sendPhoto`  
    - Chat ID: Use expression to get original sender ID from Telegram Trigger (`{{$('Telegram Trigger ').first().json.message.from.id}}`).  
    - Use binary data from both Convert To File nodes (supports multiple outputs).  
    - Connect inputs from Convert To File nodes.

11. **Credential Setup:**  
    - Telegram: Add your Telegram bot API token.  
    - Google Gemini API: Add Google Cloud API key with Generative Language API enabled.  
    - Google Palm API: Add API credentials capable of calling the Nano Banana Pro model.

12. **Test the workflow:**  
    - Send a product image with caption text to your Telegram bot.  
    - Observe the enhanced advertisement image returned by the bot.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses Google’s experimental Nano Banana Pro model via the Generative Language API for advanced image and text generation tailored for luxury product advertisements.                                                                                                                                                                          | https://cloud.google.com/generative-language                                                                                                             |
| Telegram bot setup requires creating a bot via BotFather and adding the bot token credentials in n8n.                                                                                                                                                                                                                                                       | https://core.telegram.org/bots                                                                                                                           |
| The workflow strictly enforces no alteration of the original product image and uses only the user's caption text for advertisement content, preventing hallucinations or AI-generated taglines if caption text is missing.                                                                                                                                  | Design instructions embedded in AI Design Analysis node.                                                                                                |
| The detailed design suggestions cover professional photography techniques including composition rules, lighting setups, background choices, color grading, typography, and overall advertisement style aiming for Instagram and Pinterest-ready content.                                                                                                       | Embedded in AI Design Analysis node’s system prompt.                                                                                                     |
| The workflow includes sticky notes for clarity on functional blocks and instructions, enhancing maintainability and onboarding.                                                                                                                                                                                                                            | n8n sticky notes visible in the workflow diagram.                                                                                                        |

---

**Disclaimer:** The provided workflow is an automated process built with n8n, respecting content policies and processing only legal, public data. It does not produce or manipulate illegal, offensive, or protected content.