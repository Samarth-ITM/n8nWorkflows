🤖💬 Conversational AI Chatbot with Google Gemini for Text & Image | Telegram

https://n8nworkflows.xyz/workflows/-----conversational-ai-chatbot-with-google-gemini-for-text---image---telegram-4365


# 🤖💬 Conversational AI Chatbot with Google Gemini for Text & Image | Telegram

---

# 🤖💬 Conversational AI Chatbot with Google Gemini for Text & Image | Telegram

---

## 1. Workflow Overview

This workflow implements a conversational AI chatbot integrated with Telegram, powered by Google Gemini (PaLM) models for generating both text and image responses. It is designed to handle user messages, classify their intent, generate appropriate conversational replies or visual content, and deliver responses through Telegram. The workflow maintains conversational context to ensure coherent, context-aware interactions.

The workflow is logically divided into the following blocks:

- **1.1 Input and Session Management:** Receives Telegram messages and manages session data for context continuity.
- **1.2 Memory and Conversational Context:** Stores and retrieves conversation history to maintain context for AI processing.
- **1.3 Intent Processing and Prompt Generation:** Analyzes user input to classify intent and prepares corresponding prompts.
- **1.4 Core Generation and Conversation Management:** Uses Google Gemini models to generate text or image content based on prompts and context.
- **1.5 Content Classification and User Delivery:** Determines response type, processes content accordingly, and sends it back to the user via Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Input and Session Management

**Overview:**  
Captures incoming Telegram messages and extracts essential session data such as message text, chat ID, language, and username to maintain user session state.

**Nodes Involved:**  
- `userInput`  
- `sessionData`

**Node Details:**

- **userInput**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point for user messages from Telegram.  
  - *Configuration:* Listens for "message" updates only. Authenticated via Telegram API credentials.  
  - *Input/Output:* No input; outputs raw Telegram message JSON.  
  - *Edge Cases:* Missing or malformed messages; rate limits from Telegram API.

- **sessionData**  
  - *Type:* Set  
  - *Role:* Extracts and stores relevant message and session information for downstream use.  
  - *Configuration:* Extracts `Mensaje` (message text), `sessionId` (chat ID), `Lenguaje` (language code), and `Username` from Telegram message JSON. Keeps only these fields.  
  - *Input:* Output from `userInput`.  
  - *Output:* Structured session data JSON.  
  - *Edge Cases:* Messages without expected fields; empty text or non-text messages.

---

### 2.2 Memory and Conversational Context

**Overview:**  
Manages conversation history by saving past messages and retrieving recent context to maintain conversational continuity.

**Nodes Involved:**  
- `conversationStore`  
- `memoryRetriever`  
- `latestContext`

**Node Details:**

- **conversationStore**  
  - *Type:* Langchain Memory Manager  
  - *Role:* Stores entire conversation history grouped by session.  
  - *Configuration:* Groups messages by session key.  
  - *Input:* Session data from `sessionData`.  
  - *Output:* Full conversation memory.  
  - *Edge Cases:* Memory overflow, loss of context if session key changes unexpectedly.

- **memoryRetriever**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Retrieves a limited window (last 2 messages) of conversation history for context.  
  - *Configuration:* Uses custom session key from `sessionData.sessionId`, limits context window length to 2 messages.  
  - *Input:* Connects to `conversationStore` memory output.  
  - *Output:* Partial conversation relevant for current processing.  
  - *Edge Cases:* Insufficient history; new sessions with no prior context.

- **latestContext**  
  - *Type:* Code  
  - *Role:* Formats retrieved conversation messages into a structured string for prompt input.  
  - *Configuration:* Extracts the last one or two messages, formats them as "Message 1: human: ... ai: ..." for clarity.  
  - *Input:* Output from `memoryRetriever`.  
  - *Output:* JSON with formatted conversation history string.  
  - *Edge Cases:* Empty or malformed messages.

---

### 2.3 Intent Processing and Prompt Generation

**Overview:**  
Analyzes the user input and conversation context to classify user intention (text chat, image generation, or other), then prepares the appropriate prompt for response generation.

**Nodes Involved:**  
- `GeminiModel1`  
- `structuredOutput`  
- `inputProcessor`  
- `intentHandler`  
- `generatePrompt`  
- `chatPrompt`  
- `otherPrompt`  
- `buildPrompt`

**Node Details:**

- **GeminiModel1**  
  - *Type:* Google Gemini (Langchain LM Chat Google Gemini)  
  - *Role:* Processes input with Google Gemini model for intent classification.  
  - *Configuration:* Model: "models/gemini-2.0-flash", uses Google Palm API credentials.  
  - *Input:* Receives formatted conversation and current user input from `latestContext` and `sessionData`.  
  - *Output:* Raw AI classification output.  
  - *Edge Cases:* API timeouts, invalid responses, auth failures.

- **structuredOutput**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Parses GeminiModel1 output into JSON with fields `intention` and `typeData`.  
  - *Input:* AI raw output from `GeminiModel1`.  
  - *Output:* Structured JSON for intent handling.  
  - *Edge Cases:* Parsing errors if AI output deviates from expected format.

- **inputProcessor**  
  - *Type:* Langchain Chain LLM  
  - *Role:* Implements detailed prompt logic to classify user intention and data type, considering conversation history and specific rules.  
  - *Configuration:* Complex prompt text embedded with instructions for intention classification and data type assignment (text or image).  
  - *Input:* Structured output from `structuredOutput`, conversation history from `latestContext`, and session user message.  
  - *Output:* JSON with `intention` ("generate", "chat", or "other") and `typeData` ("text" or "image").  
  - *Edge Cases:* Ambiguous inputs, vague user requests.

- **intentHandler**  
  - *Type:* Switch  
  - *Role:* Routes workflow based on user intention.  
  - *Configuration:* Routes to outputs "generate", "chat", or "other" based on `inputProcessor`'s `intention` field.  
  - *Input:* Output from `inputProcessor`.  
  - *Output:* Routes to prompt selection nodes accordingly.  
  - *Edge Cases:* Unhandled intentions, missing keys.

- **generatePrompt, chatPrompt, otherPrompt**  
  - *Type:* Set  
  - *Role:* Define prompt templates for each user intention category.  
  - *Configuration:*  
    - `generatePrompt`: Detailed instructions for generating a visual description for image generation (composition, color palette, clarity, aesthetics).  
    - `chatPrompt`: Formal, concise, respectful rules for chat responses, emphasizing clarity and positivity.  
    - `otherPrompt`: Similar to chatPrompt but adds instructions for refusing inappropriate or irrelevant requests politely.  
  - *Input:* None (static content).  
  - *Output:* Assigned prompt string for building final prompt.  
  - *Edge Cases:* None (static text).

- **buildPrompt**  
  - *Type:* Set  
  - *Role:* Passes selected prompt text downstream.  
  - *Configuration:* Assigns prompt field from the selected prompt node.  
  - *Input:* One of `generatePrompt`, `chatPrompt`, or `otherPrompt`.  
  - *Output:* Prompt string for core AI generation.  
  - *Edge Cases:* None.

---

### 2.4 Core Generation and Conversation Management

**Overview:**  
Generates the AI response (text or image prompt) using Google Gemini, integrating conversation context for coherent replies and storing conversation memory.

**Nodes Involved:**  
- `ChatCore`  
- `GeminiModel`  
- `conversationMemory`

**Node Details:**

- **ChatCore**  
  - *Type:* Langchain Agent  
  - *Role:* Central orchestrator integrating user input, prompt, and conversation history to generate AI responses.  
  - *Configuration:*  
    - System message defines the chatbot as professional, responding in Spanish for text, English for images, and strictly following company rules.  
    - Input text combines user message plus selected prompt instructions.  
  - *Input:* Prompt from `buildPrompt` and user message from `sessionData`.  
  - *Output:* Generated AI text or image prompt.  
  - *Edge Cases:* API failures, rate limits, context loss.

- **GeminiModel**  
  - *Type:* Google Gemini (Langchain LM Chat Google Gemini)  
  - *Role:* Provides final language model response to `ChatCore`.  
  - *Configuration:* Uses "models/gemini-2.0-flash-001" with Google Palm API credentials.  
  - *Input:* Triggered by `ChatCore`'s AI languageModel call.  
  - *Output:* AI generated text or image instructions.  
  - *Edge Cases:* Similar to `ChatCore`.

- **conversationMemory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Stores last 10 messages for conversation continuity with `ChatCore`.  
  - *Configuration:* Uses session ID from `sessionData`.  
  - *Input:* AI memory input from `ChatCore`.  
  - *Output:* Maintains context for future calls.  
  - *Edge Cases:* Memory size limits.

---

### 2.5 Content Classification and User Delivery

**Overview:**  
Determines if the AI response is text or image, processes content accordingly, and sends the final output to the Telegram user.

**Nodes Involved:**  
- `contentType`  
- `errorPreprocessor`  
- `textCleaner`  
- `imageGeneration`  
- `imageBuilder`  
- `sendImage`  
- `sendTextMessage`

**Node Details:**

- **contentType**  
  - *Type:* Switch  
  - *Role:* Routes workflow based on response typeData (`image` or `text`) from `ChatCore` output.  
  - *Configuration:* Case-sensitive string contains check for "image" or "text".  
  - *Input:* Output from `ChatCore`.  
  - *Output:* Routes to image processing or text sending.  
  - *Edge Cases:* Unexpected typeData values.

- **errorPreprocessor**  
  - *Type:* Set  
  - *Role:* Appends mandatory image description prompt if image generation is requested without description.  
  - *Configuration:* Adds instruction to include an image description.  
  - *Input:* AI output from `ChatCore`.  
  - *Output:* Modified prompt text for image generation.  
  - *Edge Cases:* Missing or incomplete image descriptions.

- **textCleaner**  
  - *Type:* Code  
  - *Role:* Cleans prompt text for image generation by removing newlines, quotes, backslashes, and extra spaces.  
  - *Configuration:* JavaScript code replacing unwanted characters and trimming.  
  - *Input:* Output from `errorPreprocessor`.  
  - *Output:* Clean text string for image generation API.  
  - *Edge Cases:* Unexpected characters in text.

- **imageGeneration**  
  - *Type:* HTTP Request  
  - *Role:* Calls Google Gemini image generation API with cleaned prompt text.  
  - *Configuration:*  
    - POST request to Google Gemini image generation endpoint.  
    - Sends prompt text within JSON body requesting Text and Image modalities.  
    - Uses Google Palm API credentials.  
  - *Input:* Cleaned prompt text from `textCleaner`.  
  - *Output:* Image generation response including image data.  
  - *Edge Cases:* API errors, quota limits, malformed responses.

- **imageBuilder**  
  - *Type:* Convert To File  
  - *Role:* Converts base64 image data from API response into a binary file for Telegram.  
  - *Configuration:* Extracts inline base64 image data from API response and names file "generated_image.png".  
  - *Input:* API response from `imageGeneration`.  
  - *Output:* Binary file data of generated image.  
  - *Edge Cases:* Missing or corrupted image data.

- **sendImage**  
  - *Type:* Telegram  
  - *Role:* Sends the generated image file to the user in Telegram chat.  
  - *Configuration:* Uses Telegram chat ID from session data, sends photo operation with binary data, sets filename.  
  - *Input:* Binary image from `imageBuilder`.  
  - *Output:* Message confirmation from Telegram.  
  - *Edge Cases:* Telegram API errors, invalid chat ID.

- **sendTextMessage**  
  - *Type:* Telegram  
  - *Role:* Sends text message responses to Telegram user.  
  - *Configuration:* Sends text from `ChatCore` output, uses chat ID from session data, sets HTML parse mode, disables attribution append.  
  - *Input:* Text output from `ChatCore` via `contentType`.  
  - *Output:* Confirmation of sent message.  
  - *Edge Cases:* Telegram API errors, message length limits.

---

## 3. Summary Table

| Node Name         | Node Type                              | Functional Role                              | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                   |
|-------------------|--------------------------------------|----------------------------------------------|------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| userInput         | Telegram Trigger                     | Receives Telegram messages                     |                              | sessionData                 | ### 1. Input and Session Management<br>Receives messages from Telegram and manages the session to maintain context. |
| sessionData       | Set                                 | Extracts and stores session info               | userInput                    | conversationStore           | ### 1. Input and Session Management<br>Receives messages from Telegram and manages the session to maintain context. |
| conversationStore | Langchain Memory Manager             | Stores full conversation history               | sessionData                  | latestContext, memoryRetriever | ### 2. Memory and Conversational Context<br>Retrieves the necessary context to properly infer intentions.     |
| memoryRetriever   | Langchain Memory Buffer Window       | Retrieves recent conversation snippets         | conversationStore            | latestContext               | ### 2. Memory and Conversational Context<br>Retrieves the necessary context to properly infer intentions.     |
| latestContext     | Code                                | Formats conversation history for AI input      | memoryRetriever              | inputProcessor              | ### 2. Memory and Conversational Context<br>Retrieves the necessary context to properly infer intentions.     |
| GeminiModel1      | Google Gemini LM Chat                | Classifies user intention and content type     | latestContext                | inputProcessor              | ### 3. Intent Processing and Prompt Generation<br>Analyzes the intention and selects prompts accordingly.      |
| structuredOutput  | Langchain Structured Output Parser   | Parses AI classification output as JSON        | GeminiModel1                 | inputProcessor              | ### 3. Intent Processing and Prompt Generation<br>Analyzes the intention and selects prompts accordingly.      |
| inputProcessor    | Langchain Chain LLM                  | Processes user input and context to classify   | structuredOutput, latestContext, sessionData | intentHandler           | ### 3. Intent Processing and Prompt Generation<br>Analyzes the intention and selects prompts accordingly.      |
| intentHandler     | Switch                             | Routes flow based on user intention             | inputProcessor               | generatePrompt, chatPrompt, otherPrompt | ### 3. Intent Processing and Prompt Generation<br>Analyzes the intention and selects prompts accordingly.      |
| generatePrompt    | Set                                 | Prompt template for image generation requests  | intentHandler                | buildPrompt                 | ### 3. Intent Processing and Prompt Generation<br>Analyzes the intention and selects prompts accordingly.      |
| chatPrompt        | Set                                 | Prompt template for chat responses              | intentHandler                | buildPrompt                 | ### 3. Intent Processing and Prompt Generation<br>Analyzes the intention and selects prompts accordingly.      |
| otherPrompt       | Set                                 | Prompt template for other (non-generate/chat)  | intentHandler                | buildPrompt                 | ### 3. Intent Processing and Prompt Generation<br>Analyzes the intention and selects prompts accordingly.      |
| buildPrompt       | Set                                 | Passes selected prompt downstream                | generatePrompt, chatPrompt, otherPrompt | ChatCore                   | ### 3. Intent Processing and Prompt Generation<br>Analyzes the intention and selects prompts accordingly.      |
| ChatCore          | Langchain Agent                     | Generates AI response integrating context        | buildPrompt                  | contentType                 | ### 4. Core of Generation and Conversation Management<br>Generates responses with Google Gemini integrating context. |
| GeminiModel       | Google Gemini LM Chat                | Provides final AI response for ChatCore          | ChatCore (ai_languageModel)  | ChatCore                   | ### 4. Core of Generation and Conversation Management<br>Generates responses with Google Gemini integrating context. |
| conversationMemory| Langchain Memory Buffer Window       | Stores conversation context for ChatCore          | ChatCore (ai_memory)         |                             | ### 4. Core of Generation and Conversation Management<br>Generates responses with Google Gemini integrating context. |
| contentType       | Switch                             | Routes response by content type (image/text)      | ChatCore                     | errorPreprocessor, sendTextMessage | ### 5. Content Classification and User Delivery<br>Determines content type and manages delivery via Telegram.  |
| errorPreprocessor | Set                                 | Adds mandatory image description if missing       | contentType (image)          | textCleaner                 | ### 5. Content Classification and User Delivery<br>Determines content type and manages delivery via Telegram.  |
| textCleaner       | Code                                | Cleans prompt text for image generation            | errorPreprocessor            | imageGeneration             | ### 5. Content Classification and User Delivery<br>Determines content type and manages delivery via Telegram.  |
| imageGeneration   | HTTP Request                       | Calls Google Gemini API for image generation       | textCleaner                  | imageBuilder                | ### 5. Content Classification and User Delivery<br>Determines content type and manages delivery via Telegram.  |
| imageBuilder      | Convert To File                    | Converts base64 image data to binary file          | imageGeneration              | sendImage                   | ### 5. Content Classification and User Delivery<br>Determines content type and manages delivery via Telegram.  |
| sendImage         | Telegram                           | Sends image file to user via Telegram               | imageBuilder                 |                             | ### 5. Content Classification and User Delivery<br>Determines content type and manages delivery via Telegram.  |
| sendTextMessage   | Telegram                           | Sends text message to user via Telegram             | contentType (text)           |                             | ### 5. Content Classification and User Delivery<br>Determines content type and manages delivery via Telegram.  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node (`userInput`):**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates only.  
   - Authenticate using Telegram Bot API credentials.

2. **Create Set Node (`sessionData`):**  
   - Extract fields from `userInput`:  
     - `Mensaje` = `message.text`  
     - `sessionId` = `message.chat.id`  
     - `Lenguaje` = `message.from.language_code`  
     - `Username` = `message.chat.username`  
   - Keep only these fields for downstream use.

3. **Add Langchain Memory Manager Node (`conversationStore`):**  
   - Configure to store conversation history grouped by `sessionId` from `sessionData`.

4. **Add Langchain Memory Buffer Window Node (`memoryRetriever`):**  
   - Configure with `sessionKey` = `sessionData.sessionId`  
   - Set `contextWindowLength` = 2 messages.

5. **Add Code Node (`latestContext`):**  
   - JS code to:  
     - Extract last one or two messages from `memoryRetriever` output.  
     - Format messages as:  
       ```
       Message 1:  
       human: [first line of human message]  
       ai: [AI response without trailing newlines]
       ```  
   - Output this formatted string as `messages`.

6. **Add Google Gemini Chat Node (`GeminiModel1`):**  
   - Model: `models/gemini-2.0-flash`  
   - Credentials: Google Palm API  
   - Input: Pass formatted conversation from `latestContext` and user message from `sessionData`.  
   - Purpose: Classify user intention and content type.

7. **Add Langchain Structured Output Parser (`structuredOutput`):**  
   - Configure with example JSON:  
     ```json
     {
       "intention": "generate",
       "typeData": "text"
     }
     ```  
   - Parses output from `GeminiModel1`.

8. **Add Langchain Chain LLM Node (`inputProcessor`):**  
   - Use a detailed prompt that:  
     - Classifies intention as "generate", "chat", or "other".  
     - Determines data type as "text" or "image".  
     - Considers conversation history and user input.  
   - Input from `structuredOutput`, `latestContext`, and `sessionData`.

9. **Add Switch Node (`intentHandler`):**  
   - Routes based on `inputProcessor`'s `intention` field:  
     - "generate" → `generatePrompt`  
     - "chat" → `chatPrompt`  
     - "other" → `otherPrompt`

10. **Create Set Nodes for Prompts:**  
    - `generatePrompt`: Detailed instructions for image description generation.  
    - `chatPrompt`: Rules for formal, clear, respectful chat replies.  
    - `otherPrompt`: Similar rules with polite refusal instructions.  

11. **Add Set Node (`buildPrompt`):**  
    - Passes selected prompt (`prompt` field) from above nodes downstream.

12. **Add Langchain Agent Node (`ChatCore`):**  
    - System message defines chatbot role and language policies.  
    - Input combines user message and selected prompt.  
    - Outputs AI generated text or image prompt.

13. **Add Google Gemini Chat Node (`GeminiModel`):**  
    - Model: `models/gemini-2.0-flash-001`  
    - Credentials: Google Palm API  
    - Connected as AI language model to `ChatCore`.

14. **Add Langchain Memory Buffer Window Node (`conversationMemory`):**  
    - Stores last 10 messages using `sessionId` from `sessionData`.  
    - Connect as AI memory to `ChatCore`.

15. **Add Switch Node (`contentType`):**  
    - Routes based on `ChatCore` output `typeData`:  
      - If contains "image" → `errorPreprocessor`  
      - If contains "text" → `sendTextMessage`

16. **Add Set Node (`errorPreprocessor`):**  
    - Adds mandatory instruction for image description if missing.

17. **Add Code Node (`textCleaner`):**  
    - Cleans prompt text by removing newlines, quotes, backslashes, multiple spaces.

18. **Add HTTP Request Node (`imageGeneration`):**  
    - POST to Google Gemini image generation endpoint.  
    - Body includes prompt text from `textCleaner`.  
    - Use Google Palm API credentials.

19. **Add Convert To File Node (`imageBuilder`):**  
    - Converts base64 image data from `imageGeneration` response to binary PNG file named "generated_image.png".

20. **Add Telegram Node (`sendImage`):**  
    - Sends photo to Telegram chat ID from `sessionData.sessionId`.  
    - Uses binary data from `imageBuilder`.

21. **Add Telegram Node (`sendTextMessage`):**  
    - Sends text message to Telegram chat from `ChatCore` output.  
    - Uses HTML parse mode, disables attribution.  
    - Chat ID from `sessionData.sessionId`.

22. **Connect all nodes according to logical flow:**  
    - `userInput` → `sessionData` → `conversationStore` → `memoryRetriever` → `latestContext` → `GeminiModel1` → `structuredOutput` → `inputProcessor` → `intentHandler` → prompt nodes → `buildPrompt` → `ChatCore` → `contentType` → conditional routes to image or text delivery nodes.

23. **Credential Setup:**  
    - Telegram API credentials for `userInput`, `sendImage`, and `sendTextMessage`.  
    - Google Palm API credentials for `GeminiModel1`, `GeminiModel`, and `imageGeneration`.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow enforces strict communication rules to maintain professionalism, clarity, and respect during conversations.                                         | Embedded in prompt texts within `chatPrompt` and `otherPrompt` nodes.                            |
| Image generation prompts instruct a minimalistic, balanced, and visually appealing design with specific color and resolution constraints.                        | Defined in `generatePrompt` node.                                                               |
| Conversation memory management uses Langchain memory nodes to maintain context windows of recent messages for coherence.                                         | See `conversationStore`, `memoryRetriever`, `conversationMemory` nodes.                          |
| Google Gemini (PaLM) models are used extensively for both intent classification and response generation, leveraging different model versions for tasks.          | Credentials and model names must be updated according to Google API availability.                |
| Telegram integration requires registering a bot and obtaining API credentials.                                                                                     | Telegram Bot API: https://core.telegram.org/bots/api                                                |
| The workflow includes several sticky notes explaining block roles and node groups, useful for onboarding and maintenance.                                         | Sticky notes positioned near corresponding node groups in the original workflow editor.          |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and does not contain illegal, offensive, or protected material. All processed data are legal and public.

---