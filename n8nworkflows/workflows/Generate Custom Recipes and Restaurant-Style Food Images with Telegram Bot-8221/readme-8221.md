Generate Custom Recipes and Restaurant-Style Food Images with Telegram Bot

https://n8nworkflows.xyz/workflows/generate-custom-recipes-and-restaurant-style-food-images-with-telegram-bot-8221


# Generate Custom Recipes and Restaurant-Style Food Images with Telegram Bot

---
### 1. Workflow Overview

This workflow implements a Telegram chatbot named "AI Chef" that generates custom cooking recipes and photorealistic restaurant-style food images based on user input. Users send dish names or cooking-related queries via Telegram, and the bot replies with a detailed recipe followed by a high-quality image of the plated dish.

The workflow is logically divided into these blocks:

- **1.1 Input Reception (Telegram Trigger):** Receives user messages from Telegram.
- **1.2 Recipe Generation (AI Recipe Agent):** Uses an AI language model to generate cooking recipes and culinary guidance.
- **1.3 Recipe Text Delivery (Send Text Message):** Sends the generated recipe back to the user in Telegram chat.
- **1.4 Image Prompt Generation (Restaurant-Style Plating Prompt Agent):** Converts the recipe text into a concise, detailed prompt for photorealistic image generation.
- **1.5 Food Image Generation (Nano 🍌 HTTP Request):** Calls an AI image generation API to create a restaurant-quality food image.
- **1.6 Image Processing & Delivery:** Extracts the image URL, converts it into a file, and sends it as a photo message on Telegram.

Additionally, the workflow maintains conversational context using a **Window Buffer Memory** node integrated with the AI Recipe generation, ensuring coherent multi-turn interactions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming Telegram messages from users to start the recipe/image generation process.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - *Type:* Telegram trigger node; event listener for Telegram Bot API.  
    - *Configuration:* Listens to update type "message" to capture user texts.  
    - *Credentials:* Uses Telegram Bot API credentials named "AI Chef Assistant".  
    - *Input/Output:* Initiates the workflow on new messages; outputs message data JSON including chat ID and message text.  
    - *Potential Failures:* Telegram API authentication errors, webhook misconfiguration, or connectivity issues.

---

#### 2.2 Recipe Generation

- **Overview:**  
  Processes the input text using an AI agent specialized for cooking. It generates a friendly, professional recipe or food-related guidance based on the user’s request while maintaining context via memory.

- **Nodes Involved:**  
  - AI Recipe (LangChain Agent)  
  - Window Buffer Memory  
  - OpenRouter Chat Model

- **Node Details:**  
  - **Window Buffer Memory**  
    - *Type:* LangChain buffer memory node for conversational context.  
    - *Configuration:* Uses the Telegram chat ID as the session key to store recent conversation. Context window length set to 200 tokens.  
    - *Input:* Receives messages and outputs context-enriched data for AI agent.  
    - *Potential Failures:* Expression evaluation errors if chat ID missing; memory overflow on long sessions.  

  - **OpenRouter Chat Model**  
    - *Type:* Language model node (OpenRouter API using GPT-4o-mini).  
    - *Configuration:* GPT-4o-mini model selected for text generation.  
    - *Credentials:* OpenRouter API credentials "Ai chef".  
    - *Input/Output:* Receives prompt with context, outputs AI-generated recipe text.  
    - *Failures:* API key errors, rate limits, network timeouts.  

  - **AI Recipe**  
    - *Type:* LangChain Agent node configured for culinary assistance.  
    - *Configuration:*  
      - System message defines the assistant’s role as a friendly, professional virtual chef.  
      - Instructions cover tone, content scope (recipes, substitutions, nutrition tips), and safety warnings.  
      - Prompt type set to "define" to control AI behavior strictly.  
    - *Input:* User message text with memory and language model integration.  
    - *Output:* Recipe or cooking guidance text.  
    - *Failures:* Model errors, improper prompt formatting, or context loss.

---

#### 2.3 Recipe Text Delivery

- **Overview:**  
  Sends the generated recipe text back to the user on Telegram as a chat message.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**  
  - **Send a text message**  
    - *Type:* Telegram node to send text messages.  
    - *Configuration:* Sends the recipe text stored in the `output` property. Uses the chat ID from Telegram Trigger for targeting.  
    - *Credentials:* Same Telegram API credentials.  
    - *Input:* Receives recipe text from AI Recipe node.  
    - *Output:* Confirmation of message sent.  
    - *Failures:* Telegram API errors, invalid chat IDs.

---

#### 2.4 Image Prompt Generation

- **Overview:**  
  Converts the generated recipe text into a clear, detailed prompt tailored for photorealistic image generation of a professionally plated dish.

- **Nodes Involved:**  
  - Restaurant-Style Plating prompt  
  - OpenRouter Chat Model1

- **Node Details:**  
  - **Restaurant-Style Plating prompt**  
    - *Type:* LangChain Agent node for prompt engineering.  
    - *Configuration:*  
      - System message guides the AI to generate a concise, visual-only image prompt based on the recipe text.  
      - Emphasizes plating style, colors, garnishes, lighting, and restaurant-appropriate dishware.  
      - Output is a single paragraph in English, ready for image generation.  
      - Prompt type "define" ensures strict output format.  
    - *Input:* Receives recipe text as input.  
    - *Output:* Photorealistic image prompt text.  
    - *Failures:* Misinterpretation of recipe text, incomplete prompt output.

  - **OpenRouter Chat Model1**  
    - *Type:* GPT-4o-mini language model node (same as earlier but separate instance).  
    - *Configuration:* Used implicitly by the LangChain agent above for text generation.  
    - *Failures:* Same as OpenRouter Chat Model.

---

#### 2.5 Food Image Generation

- **Overview:**  
  Sends the image prompt to an AI image generation API (OpenRouter with Gemini 2.5 Flash model) to produce a photorealistic image of the plated dish.

- **Nodes Involved:**  
  - Nano 🍌 (HTTP Request)

- **Node Details:**  
  - **Nano 🍌**  
    - *Type:* HTTP Request node.  
    - *Configuration:*  
      - POST request to `https://openrouter.ai/api/v1/chat/completions`.  
      - Body contains model "google/gemini-2.5-flash-image-preview:free" with the image prompt as user message content.  
      - Authorization header uses Bearer token from environment variable `$OPENROUTER_API_KEY`.  
    - *Input:* Receives image prompt text from the previous block.  
    - *Output:* JSON response with image URL(s).  
    - *Failures:* API authentication errors, malformed request, network timeouts, rate limits.

---

#### 2.6 Image Processing & Delivery

- **Overview:**  
  Extracts the image URL, decodes and converts it into a binary file format, then sends it as a photo message to the user on Telegram.

- **Nodes Involved:**  
  - Edit Fields  
  - Convert to File  
  - Send a photo message

- **Node Details:**  
  - **Edit Fields**  
    - *Type:* Set node to manipulate and extract data fields.  
    - *Configuration:*  
      - Extracts the image URL from the nested JSON response path `choices[0].message.images[0].image_url.url`.  
      - Splits this URL string to separate the base64 image data and MIME type.  
      - Sets file extension as ".png".  
    - *Input:* Receives HTTP response from Nano 🍌 node.  
    - *Output:* JSON with fields `data` (full URL), `base` (base64 image data), `mime`, and `fileName`.  
    - *Failures:* If the URL structure changes, splitting could fail; missing fields cause expression errors.

  - **Convert to File**  
    - *Type:* ConvertToFile node.  
    - *Configuration:* Converts the base64 string in `base` property to a binary file.  
    - *Input:* Receives base64 image data from Edit Fields.  
    - *Output:* Binary image file ready for sending.  
    - *Failures:* Invalid base64 data could cause conversion failure.

  - **Send a photo message**  
    - *Type:* Telegram node to send photos.  
    - *Configuration:* Sends the binary image file to the same chat ID as before.  
    - *Credentials:* Telegram API credentials.  
    - *Input:* Binary image file from Convert to File node.  
    - *Output:* Confirmation of photo message sent.  
    - *Failures:* Telegram API errors, large file size, invalid chat ID.

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                         | Input Node(s)            | Output Node(s)              | Sticky Note                                                                                  |
|----------------------------|----------------------------------------|---------------------------------------|--------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Telegram Trigger            | telegramTrigger                        | Receive user messages from Telegram   |                          | AI Recipe                  |                                                                                              |
| AI Recipe                  | langchain.agent                        | Generate custom recipes and guidance  | Telegram Trigger, Window Buffer Memory, OpenRouter Chat Model | Send a text message           |                                                                                              |
| Send a text message         | telegram                              | Send recipe text to Telegram chat     | AI Recipe                 | Restaurant-Style Plating prompt |                                                                                              |
| Restaurant-Style Plating prompt | langchain.agent                    | Generate image prompt from recipe     | Send a text message       | Nano 🍌                     |                                                                                              |
| Nano 🍌                    | httpRequest                           | Generate photorealistic food image    | Restaurant-Style Plating prompt | Edit Fields                 |                                                                                              |
| Edit Fields                | set                                  | Extract and decode image data         | Nano 🍌                   | Convert to File             |                                                                                              |
| Convert to File            | convertToFile                        | Convert base64 to binary file          | Edit Fields               | Send a photo message        |                                                                                              |
| Send a photo message        | telegram                              | Send food image to Telegram chat       | Convert to File           |                            |                                                                                              |
| Window Buffer Memory        | langchain.memoryBufferWindow          | Maintain conversation context          |                          | AI Recipe                  |                                                                                              |
| OpenRouter Chat Model       | langchain.lmChatOpenRouter            | Language model for recipe generation   |                          | AI Recipe                  |                                                                                              |
| OpenRouter Chat Model1      | langchain.lmChatOpenRouter            | Language model for image prompt gen.   |                          | Restaurant-Style Plating prompt |                                                                                              |
| Sticky Note                | stickyNote                           | Documentation and instructions         |                          |                            | ## AI Chef Bot – Recipe + Food Image Generator \nImport this workflow into your n8n instance.\n\nConfigure your Telegram Bot Token (from BotFather).\n\nConfigure your OpenRouter API Key for AI text + image generation.\n\nSave and activate the workflow.\n\nGo to Telegram and send any dish name (e.g., Polpette di pesce).\n\nThe bot replies with:\n\n📖 A full recipe.\n\n📸 A restaurant-plated realistic food image. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `telegramTrigger`  
   - Configure to listen for "message" updates.  
   - Attach Telegram API credentials ("AI Chef Assistant").  
   - Position: Start of workflow.

2. **Add Window Buffer Memory Node**  
   - Type: `langchain.memoryBufferWindow`  
   - Set `sessionKey` to `={{ $json.message.chat.id }}` for chat-specific context.  
   - Set `sessionIdType` to `customKey`.  
   - Set `contextWindowLength` to `200`.  
   - Connect input from Telegram Trigger (for context tracking).

3. **Add OpenRouter Chat Model Node**  
   - Type: `langchain.lmChatOpenRouter`  
   - Select model: `openai/gpt-4o-mini`.  
   - Attach OpenRouter API credentials ("Ai chef").  
   - Connect output to AI Recipe Agent node.

4. **Add AI Recipe Agent Node**  
   - Type: `langchain.agent`  
   - Set prompt text as `={{ $json.message.text }}` (user input).  
   - System message: Use the detailed "Virtual Chef Assistant" instructions from the workflow (see block 2.2).  
   - Set prompt type to "define".  
   - Connect inputs from Telegram Trigger, Window Buffer Memory (ai_memory), and OpenRouter Chat Model (ai_languageModel).  
   - Connect output to Send a text message node.

5. **Add Send a text message Node**  
   - Type: `telegram`  
   - Set `text` to `={{ $json.output }}` (recipe text).  
   - Set `chatId` to `={{ $('Telegram Trigger').item.json.message.chat.id }}`.  
   - Attach Telegram API credentials.  
   - Connect input from AI Recipe node.  
   - Connect output to Restaurant-Style Plating prompt node.

6. **Add Restaurant-Style Plating prompt Agent Node**  
   - Type: `langchain.agent`  
   - Set prompt text to `={{ $('AI Recipe').item.json.output }}` (recipe text).  
   - System message: Use the expert AI prompt generator instructions for photorealistic plating (see block 2.4).  
   - Set prompt type to "define".  
   - Connect input from Send a text message node.

7. **Add OpenRouter Chat Model1 Node**  
   - Type: `langchain.lmChatOpenRouter` (duplicate of previous language model node).  
   - Model: `openai/gpt-4o-mini`.  
   - Credentials: same as previous OpenRouter Chat Model.  
   - Connect input/output to Restaurant-Style Plating prompt node (ai_languageModel).

8. **Add Nano 🍌 HTTP Request Node**  
   - Type: `httpRequest`  
   - Method: POST  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Authentication: Predefined credential using OpenRouter API key (set environment variable `$OPENROUTER_API_KEY`).  
   - Headers: Add `Authorization: Bearer $OPENROUTER_API_KEY`.  
   - Body (JSON):  
     ```json
     {
       "model": "google/gemini-2.5-flash-image-preview:free",
       "messages": [
         {
           "role": "user",
           "content": [
             {
               "type": "text",
               "text": "Generate a photorealistic image of {{ $json.output }}"
             }
           ]
         }
       ]
     }
     ```  
   - Connect input from Restaurant-Style Plating prompt node.

9. **Add Edit Fields (Set) Node**  
   - Type: `set`  
   - Assign variables:  
     - `data`: full image URL from `choices[0].message.images[0].image_url.url`  
     - `base`: extract base64 part after comma from the URL  
     - `mime`: extract MIME type from URL prefix  
     - `fileName`: `.png`  
   - Connect input from Nano 🍌 node.

10. **Add Convert to File Node**  
    - Type: `convertToFile`  
    - Operation: `toBinary`  
    - Source Property: `base` (base64 image string)  
    - Connect input from Edit Fields node.

11. **Add Send a photo message Node**  
    - Type: `telegram`  
    - Operation: `sendPhoto`  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Attach Telegram API credentials.  
    - Connect input from Convert to File node.

12. **Link all nodes according to the logical flow:**  
    - Telegram Trigger → Window Buffer Memory → OpenRouter Chat Model → AI Recipe → Send a text message → Restaurant-Style Plating prompt → OpenRouter Chat Model1 → Nano 🍌 → Edit Fields → Convert to File → Send a photo message.

13. **Test the workflow:**  
    - Activate and send dish names or cooking questions via Telegram.  
    - Verify recipe text and corresponding food image responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| AI Chef Bot – Recipe + Food Image Generator: Import workflow, configure Telegram Bot Token and OpenRouter API Key, activate & test.        | Workflow sticky note content                      |
| Use OpenRouter API keys for both text and image generation nodes (GPT-4o-mini for text, Gemini 2.5 Flash for images).                      | API credential setup                             |
| Telegram Bot token obtained via BotFather on Telegram platform.                                                                             | Telegram Bot setup                               |
| The system messages in LangChain agents are carefully crafted to ensure clear, safe, and domain-specific AI behavior.                      | AI prompt engineering guidelines                 |
| For large images or slow API response times, consider adding error handling or retries in HTTP Request node.                               | Potential failure mitigation                      |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.