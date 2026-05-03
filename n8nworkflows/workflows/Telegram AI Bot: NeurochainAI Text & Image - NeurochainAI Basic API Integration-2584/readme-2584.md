Telegram AI Bot: NeurochainAI Text & Image - NeurochainAI Basic API Integration

https://n8nworkflows.xyz/workflows/telegram-ai-bot--neurochainai-text---image---neurochainai-basic-api-integration-2584


# Telegram AI Bot: NeurochainAI Text & Image - NeurochainAI Basic API Integration

---

## 1. Workflow Overview

This workflow is designed to integrate a Telegram bot with NeurochainAI's advanced AI capabilities, enabling both text-based conversational AI responses and AI-driven image generation via the "Flux" model. It supports interactive Telegram chats where users can either send messages to receive AI text replies or enter image generation commands prefixed with `/flux`.

The logical structure is grouped into the following blocks:

- **1.1 Telegram Input Handling**  
  Captures incoming Telegram messages and routes them based on content (text command, flux image request, or direct message).

- **1.2 Command Parsing and Validation**  
  Cleans user inputs and validates prompt length or command format to ensure suitable requests.

- **1.3 Image Generation via NeurochainAI Flux API**  
  Handles `/flux` image generation commands: requests image creation, fetches the image URL, and sends the generated photo back to the user.

- **1.4 Text Processing via NeurochainAI Language Model API**  
  Sends user messages to NeurochainAI's text inference API, receives AI-generated text responses, and replies in Telegram.

- **1.5 Error Handling and User Feedback**  
  Provides graceful error messages and retry options to the user when issues occur, including prompt validation errors and API failures.

- **1.6 User Interaction Enhancements**  
  Sends chat action indicators (e.g., typing emoji) and manages Telegram message lifecycle (deleting temporary messages or error notifications).

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Input Handling

**Overview:**  
This block receives all incoming Telegram messages via webhook and directs flow based on message content type.

**Nodes Involved:**  
- Telegram Trigger  
- Switch (primary routing)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger Node  
  - Role: Webhook entry point for all Telegram updates (messages, commands, etc.)  
  - Config: Listens to all update types (`*`), uses Telegram Bot Token credential (named TEMPLATE)  
  - Input/Output: Input from Telegram webhook; outputs message JSON to Switch node  
  - Edge Cases: Telegram webhook misconfiguration, invalid bot token, message format variations  

- **Switch**  
  - Type: Switch Node  
  - Role: Routes messages into three paths based on content:  
    - Messages starting with `/flux` → Flux image generation path  
    - Messages containing mention of bot username `@NCNAI_BOT` → Text processing path  
    - Private chat messages → Text processing path (DM Text)  
  - Key Expressions: Uses string operations like `startsWith` and `contains` on `$json.message.text`  
  - Input: From Telegram Trigger  
  - Output: 3 outputs named Flux, text, DM Text  
  - Edge Cases: Case sensitivity handled, messages not matching any condition will be ignored  

---

### 2.2 Command Parsing and Validation

**Overview:**  
Cleans `/flux` command by removing the prefix and validates prompt length for both text and image commands.

**Nodes Involved:**  
- Code1 (clean message extraction)  
- Telegram2 (sending typing indicator emoji)  
- Telegram6 (prompt too short error message)  
- Telegram7 (deleting prompt too short error message)  
- Switch2 (error type handling for text AI API)  
- Switch1 (error handling for Flux API)

**Node Details:**

- **Code1**  
  - Type: Code Node (JavaScript)  
  - Role: Strips `/flux` prefix from user message to isolate image prompt  
  - Config: Uses regex to remove `/flux` and trailing spaces  
  - Output: JSON with `cleanMessage` (prompt without prefix)  
  - Input: From Switch (Flux output)  
  - Edge Cases: Messages without `/flux` prefix routed away, empty prompt results handled downstream  

- **Telegram2**  
  - Type: Telegram Node (Send Message)  
  - Role: Sends "⌛" emoji to user indicating prompt processing started  
  - Config: Dynamic chat ID from Telegram Trigger message; replies to original message  
  - Input: From Code1  
  - Output: Leads to NeurochainAI Flux HTTP request  
  - Edge Cases: Telegram API errors, message ID missing  

- **Telegram6**  
  - Type: Telegram Node (Send Message)  
  - Role: Sends "Prompt too short" error message if prompt length invalid or empty  
  - Config: Markdown parse mode, replies to original message  
  - Input: From Switch1 error path  
  - Output: Leads to Telegram7 for cleanup  
  - Edge Cases: Telegram API errors  

- **Telegram7**  
  - Type: Telegram Node (Delete Message)  
  - Role: Deletes the "Prompt too short" message after a delay or on retry  
  - Config: Uses chat ID and message ID from Telegram2's message response  
  - Input: From Telegram6  
  - Edge Cases: Message may already be deleted or unavailable  

- **Switch2**  
  - Type: Switch Node  
  - Role: Handles error messages from NeurochainAI text API responses, distinguishing between "No response from worker" and "Prompt string is invalid"  
  - Input: From NeurochainAI REST API error output  
  - Output: Routes to appropriate Telegram error messages  
  - Edge Cases: Unexpected errors  

- **Switch1**  
  - Type: Switch Node  
  - Role: Handles Flux API errors, mainly differentiating invalid prompt errors (400) from other errors  
  - Input: From NeurochainAI Flux API error output  
  - Output: Routes to Telegram6 (prompt too short) or Telegram3 (general error)  
  - Edge Cases: API throttling, malformed JSON, authorization failure  

---

### 2.3 Image Generation via NeurochainAI Flux API

**Overview:**  
Processes image generation requests by calling NeurochainAI Flux API and returns generated images to Telegram chat.

**Nodes Involved:**  
- NeurochainAI - Flux (HTTP Request)  
- Code (extract image URL)  
- HTTP Request3 (download image binary)  
- Telegram1 (send photo to user)  
- Telegram4 (delete typing indicator emoji)  
- Telegram5 (delete error message if any)  
- Switch1 (error routing)  
- Telegram2 (send processing emoji)

**Node Details:**

- **NeurochainAI - Flux**  
  - Type: HTTP Request Node  
  - Role: Sends POST request to NeurochainAI Flux endpoint for Text-to-Image generation  
  - Config:  
    - URL: `https://ncmb.neurochain.io/tasks/tti`  
    - Model: `flux1-schnell-gguf` (configurable)  
    - Prompt: dynamic, from cleaned user message  
    - Image size: 1024x1024  
    - Quality: standard  
    - Seed: random integer 1-999 for variability  
    - Headers: Authorization Bearer token (replace `YOUR-API-KEY-HERE` with actual key)  
  - Input: From Telegram2 (after processing emoji sent)  
  - Output: JSON with image URL or error message  
  - Edge Cases: API key invalid, quota exceeded, prompt too short, network timeout  

- **Code**  
  - Type: Code Node  
  - Role: Parses Flux API response to extract image URL from JSON string  
  - Config: Uses JSON.parse on API returned string to get first image URL  
  - Input: From NeurochainAI - Flux API response  
  - Output: JSON containing only `imageUrl` property  
  - Edge Cases: Malformed JSON response, empty array response  

- **HTTP Request3**  
  - Type: HTTP Request Node  
  - Role: Downloads the actual image binary data from extracted image URL  
  - Config: URL is dynamic from Code node output  
  - Input: From Code node  
  - Output: Binary image data forwarded to Telegram1  
  - Edge Cases: URL invalid, 404 Not Found, timeout  

- **Telegram1**  
  - Type: Telegram Node (Send Photo)  
  - Role: Sends the downloaded image as a photo message in Telegram chat  
  - Config:  
    - Chat ID from original Telegram message  
    - Caption includes original prompt  
    - Markdown parse mode  
    - Replies to original user message ID  
  - Input: From HTTP Request3 binary image data  
  - Output: Leads to Telegram4 (delete indicator)  
  - Edge Cases: Telegram API rate limits, file size limits  

- **Telegram4**  
  - Type: Telegram Node (Delete Message)  
  - Role: Deletes the temporary "⌛" emoji message after sending photo  
  - Input: From Telegram1 result message  
  - Output: None (end of image flow)  
  - Edge Cases: Message may already be deleted or unavailable  

- **Telegram5**  
  - Type: Telegram Node (Delete Message)  
  - Role: Deletes error message after user retry or error resolved  
  - Input: From Telegram3 (error message)  
  - Edge Cases: Message may be missing  

---

### 2.4 Text Processing via NeurochainAI Language Model API

**Overview:**  
Processes user messages directed at the bot (either via mention or private chat) with NeurochainAI language model, sending the AI response back into the Telegram conversation.

**Nodes Involved:**  
- TYPING - ACTION (Telegram sendChatAction)  
- NeurochainAI - REST API (HTTP Request)  
- AI Response (Telegram send message)  
- No response (Telegram error message)  
- Prompt too short (Telegram error message)  
- Switch2 (error handling)

**Node Details:**

- **TYPING - ACTION**  
  - Type: Telegram Node (sendChatAction)  
  - Role: Indicates bot is typing to the user while processing AI request  
  - Config: Chat ID dynamic from Telegram Trigger  
  - Input: From Switch (text or DM Text output)  
  - Output: NeurochainAI REST API call  
  - Edge Cases: Telegram API failure  

- **NeurochainAI - REST API**  
  - Type: HTTP Request Node  
  - Role: Sends user message to NeurochainAI text inference endpoint  
  - Config:  
    - URL: `https://ncmb.neurochain.io/tasks/message`  
    - Model: `Meta-Llama-3.1-8B-Instruct-Q6_K.gguf` (configurable)  
    - Prompt: Injects exact user message text  
    - Parameters: max_tokens=1024, temperature=0.6, top_p=0.95, frequency_penalty=0, presence_penalty=1.1  
    - Headers: Authorization Bearer token (replace `YOUR-API-KEY-HERE`)  
  - Input: From TYPING - ACTION  
  - Output: AI text completion or error  
  - Edge Cases: API quota, invalid prompt, network errors  

- **AI Response**  
  - Type: Telegram Node (Send Message)  
  - Role: Sends AI generated text back to user chat  
  - Config:  
    - Text from NeurochainAI `choices[0].text` property  
    - Markdown parse mode  
    - Replies to original user message  
  - Input: From NeurochainAI REST API success output  
  - Edge Cases: Telegram API limits, empty AI response  

- **No response**  
  - Type: Telegram Node (Send Message)  
  - Role: Sends "No response from worker" error to user on specific error from API  
  - Config: Markdown parse mode, replies to original message  
  - Input: From Switch2 error output  
  - Edge Cases: Telegram API failure  

- **Prompt too short**  
  - Type: Telegram Node (Send Message)  
  - Role: Informs user their prompt is too short for AI processing  
  - Config: Markdown parse mode, inline keyboard with retry button  
  - Input: From Switch2 error output  
  - Edge Cases: Telegram API failure  

- **Switch2**  
  - Type: Switch Node  
  - Role: Differentiates error messages from NeurochainAI text API to route appropriate Telegram error messages  
  - Input: From NeurochainAI REST API error output  
  - Edge Cases: Unrecognized errors  

---

### 2.5 Error Handling and User Feedback

**Overview:**  
Handles various error scenarios, presenting informative messages and offering retry options to the user.

**Nodes Involved:**  
- Telegram3 (send error messages with retry button)  
- Telegram5 (delete error message)  
- Switch1 and Switch2 (error routing)  
- Telegram6 (prompt too short)  
- Telegram7 (delete prompt too short message)  
- No response and Prompt too short nodes (text API errors)

**Node Details:**

- **Telegram3**  
  - Type: Telegram Node (Send Message)  
  - Role: Sends generic error messages with a retry inline button for Flux API errors  
  - Config: Inline keyboard with callback data to retry prompt  
  - Input: From Switch1 error path  
  - Output: Leads to Telegram5 for message deletion on retry  
  - Edge Cases: Telegram API failures  

- Other nodes in error handling are explained in prior sections.

---

### 2.6 User Interaction Enhancements

**Overview:**  
Improves user experience by indicating processing status and cleaning up temporary messages.

**Nodes Involved:**  
- Telegram2 (processing emoji)  
- Telegram4 (delete processing emoji)  
- Telegram5 (delete error message)  
- TYPING - ACTION (chat action indicator)

**Node Details:**  
Covered in blocks above.

---

## 3. Summary Table

| Node Name              | Node Type           | Functional Role                                | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                                             |
|------------------------|---------------------|-----------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger     | Entry point: receives all Telegram updates    | —                           | Switch                      | ## Instructions for Using the Template: Setup steps for Telegram Bot, API keys, and config (Sticky Note8)                              |
| Switch                 | Switch              | Routes messages based on content (/flux, mention, DM) | Telegram Trigger            | Code1, TYPING - ACTION       |                                                                                                                                         |
| Code1                  | Code                | Removes `/flux` prefix from commands           | Switch (Flux output)         | Telegram2                   | ## This node removes the /flux prefix. (Sticky Note3)                                                                                   |
| Telegram2              | Telegram             | Sends "⌛" emoji indicating processing started | Code1                       | NeurochainAI - Flux         | ## This node sends an emoji to indicate that the prompt is being processed. (Sticky Note6)                                              |
| NeurochainAI - Flux    | HTTP Request        | Calls NeurochainAI Flux API for image generation | Telegram2                   | Code, Switch1               | ## HTTP REQUEST: Replace MODEL and API key. Lists models compatible with Flux. (Sticky Note5)                                           |
| Code                   | Code                | Extracts image URL from Flux API response       | NeurochainAI - Flux          | HTTP Request3               | Extract the URL from the previous node (Node Note)                                                                                      |
| HTTP Request3          | HTTP Request        | Downloads image binary from extracted URL       | Code                        | Telegram1                   |                                                                                                                                         |
| Telegram1              | Telegram             | Sends generated image photo to Telegram chat   | HTTP Request3               | Telegram4                   | ## SUCCESS: Sends AI response directly to Telegram chat. (Sticky Note7)                                                                 |
| Telegram4              | Telegram             | Deletes temporary processing emoji message      | Telegram1                   | —                           |                                                                                                                                         |
| Switch1                | Switch              | Handles Flux API errors and routes accordingly  | NeurochainAI - Flux          | Telegram6, Telegram3        |                                                                                                                                         |
| Telegram6              | Telegram             | Sends "Prompt too short" error message          | Switch1                     | Telegram7                   |                                                                                                                                         |
| Telegram7              | Telegram             | Deletes "Prompt too short" message               | Telegram6                   | —                           |                                                                                                                                         |
| Telegram3              | Telegram             | Sends generic error with retry button            | Switch1                     | Telegram5                   |                                                                                                                                         |
| Telegram5              | Telegram             | Deletes error message after retry                 | Telegram3                   | —                           |                                                                                                                                         |
| TYPING - ACTION        | Telegram             | Sends "typing" status to Telegram chat           | Switch (text or DM Text)    | NeurochainAI - REST API     |                                                                                                                                         |
| NeurochainAI - REST API| HTTP Request        | Sends user message to NeurochainAI text model   | TYPING - ACTION             | AI Response, Switch2        | ## HTTP REQUEST: Replace MODEL and API key. Lists supported text models. (Sticky Note2)                                                  |
| AI Response            | Telegram             | Sends AI-generated text back to Telegram chat   | NeurochainAI - REST API     | —                           | ## SUCCESS: Sends AI response directly to Telegram chat. (Sticky Note7)                                                                 |
| Switch2                | Switch              | Routes text API errors to appropriate messages  | NeurochainAI - REST API     | No response, Prompt too short|                                                                                                                                         |
| No response            | Telegram             | Sends "No response from worker" error message   | Switch2                     | —                           | ## ERROR (Sticky Note1)                                                                                                                  |
| Prompt too short       | Telegram             | Sends "Prompt too short" message for text API   | Switch2                     | —                           | ## ERROR (Sticky Note1)                                                                                                                  |
| Sticky Note            | Sticky Note          | Provides error section heading                    | —                           | —                           | ## ERROR (Sticky Note)                                                                                                                   |
| Sticky Note1           | Sticky Note          | SUCCESS section note about sending AI response  | —                           | —                           | ## SUCCESS (Sticky Note1)                                                                                                                |
| Sticky Note2           | Sticky Note          | HTTP REQUEST instructions and model lists       | —                           | —                           | ## HTTP REQUEST instructions (Sticky Note2)                                                                                            |
| Sticky Note3           | Sticky Note          | Explains Code1 node removing /flux prefix        | —                           | —                           | ## Removes /flux prefix (Sticky Note3)                                                                                                 |
| Sticky Note4           | Sticky Note          | ERROR section note near Telegram1 and HTTP Request3 | —                           | —                           | ## ERROR (Sticky Note4)                                                                                                                  |
| Sticky Note5           | Sticky Note          | HTTP REQUEST instructions for Flux API           | —                           | —                           | ## HTTP REQUEST instructions for Flux (Sticky Note5)                                                                                   |
| Sticky Note6           | Sticky Note          | Explains Telegram2 node sending processing emoji | —                           | —                           | ## Sends emoji to indicate processing (Sticky Note6)                                                                                   |
| Sticky Note7           | Sticky Note          | SUCCESS note near Telegram1 and AI Response nodes | —                           | —                           | ## SUCCESS note describing sending AI response (Sticky Note7)                                                                          |
| Sticky Note8           | Sticky Note          | Detailed setup instructions for the entire template | —                           | —                           | ## Instructions for Using the Template (Sticky Note8) [https://www.neurochain.ai/](https://www.neurochain.ai/) [Guides](https://docs.neurochain.ai/nc/neurochainai-guides) |

---

## 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow entirely in n8n:

### Step 1: Create Telegram Trigger Node

- Node Type: Telegram Trigger  
- Configuration:  
  - Updates: All (`*`)  
  - Credentials: Add Telegram Bot Token (OAuth2 or Token-based; name it as preferred)  
- Position: Entry point of the workflow

### Step 2: Add Switch Node for Routing Incoming Messages

- Node Type: Switch  
- Configuration:  
  - Add 3 outputs with names: Flux, text, DM Text  
  - Conditions:  
    - Output "Flux": message text starts with `/flux` (case-insensitive)  
    - Output "text": message text contains `@NCNAI_BOT` (case-insensitive)  
    - Output "DM Text": message chat type equals `private`  
- Connect Telegram Trigger main output to this Switch input

### Step 3: Command Parsing for `/flux` Commands

- Add a Code node (Code1) connected from Switch Flux output  
- Code snippet (JavaScript):  
  ```javascript
  const userMessage = $json.message.text;
  const cleanMessage = userMessage.replace(/^\/flux\s*/, '');
  return { json: { cleanMessage } };
  ```

### Step 4: Send Processing Emoji to User

- Add a Telegram node (Telegram2) connected from Code1 output  
- Operation: Send Message  
- Text: `⌛`  
- Chat ID: `{{$json.message.chat.id}}` (from Telegram Trigger)  
- Reply to message ID: `{{$json.message.message_id}}`  
- Credentials: Use same Telegram Bot Token credential

### Step 5: Call NeurochainAI Flux API for Image Generation

- Add HTTP Request node (NeurochainAI - Flux) connected from Telegram2  
- Method: POST  
- URL: `https://ncmb.neurochain.io/tasks/tti`  
- Headers:  
  - Authorization: `Bearer YOUR-API-KEY-HERE` (replace with your actual key)  
  - Content-Type: `application/json`  
- JSON Body:  
  ```json
  {
    "model": "flux1-schnell-gguf",
    "prompt": "Generate an image that matches exactly this: {{$json.cleanMessage}}",
    "size": "1024x1024",
    "quality": "standard",
    "n": 1,
    "seed": {{ Math.floor(Math.random() * 999) + 1 }}
  }
  ```
- Enable "Send Body" and "Send Headers" options

### Step 6: Extract Image URL from Flux API Response

- Add Code node (Code) connected from NeurochainAI - Flux output  
- Code snippet:  
  ```javascript
  const rawUrl = $json.choices[0].text;
  const imageUrl = JSON.parse(rawUrl)[0];
  return { json: { imageUrl } };
  ```

### Step 7: Download Image Binary

- Add HTTP Request node (HTTP Request3) connected from Code node  
- Method: GET  
- URL: `={{ $json.imageUrl }}`  
- Enable "Response Format" as "File" or "Binary" to download image data

### Step 8: Send Generated Image to Telegram Chat

- Add Telegram node (Telegram1) connected from HTTP Request3  
- Operation: Send Photo  
- Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
- Photo binary property: Use binary data from HTTP Request3  
- Caption: `=*Prompt:* \`{{ $('Code1').item.json.cleanMessage }}\``  
- Parse mode: Markdown  
- Reply to message ID: `={{ $('Telegram Trigger').item.json.message.message_id }}`

### Step 9: Delete Processing Emoji Message

- Add Telegram node (Telegram4) connected from Telegram1  
- Operation: Delete Message  
- Chat ID and Message ID from Telegram2 result (the emoji message)  

### Step 10: Error Handling for Flux API

- Add a Switch node (Switch1) connected from NeurochainAI - Flux error output  
- Conditions:  
  - If error message equals `400 - "{\"error\":true,\"msg\":\"Prompt string is invalid\"}"` → output 1  
  - Else → output 2  
- Output 1 connects to Telegram6 (send "Prompt too short")  
- Output 2 connects to Telegram3 (send generic error with retry button)  
- Telegram6 sends "Prompt too short" text message with Markdown formatting  
- Telegram3 sends error message with inline keyboard button for retry  
- Both Telegram6 and Telegram3 connect to Telegram7 and Telegram5 respectively to delete those error messages later

### Step 11: Text Processing Path Setup

- Add Telegram node (TYPING - ACTION) connected from Switch outputs for text and DM Text  
- Operation: Send Chat Action  
- Chat ID dynamic from Telegram Trigger  

- Add HTTP Request node (NeurochainAI - REST API) connected from TYPING - ACTION  
- Method: POST  
- URL: `https://ncmb.neurochain.io/tasks/message`  
- Headers: Authorization Bearer token and Content-Type application/json  
- JSON Body:  
  ```json
  {
    "model": "Meta-Llama-3.1-8B-Instruct-Q6_K.gguf",
    "prompt": "You must respond directly to the user's message, and the message the user sent you is the following message: {{$json.message.text}}",
    "max_tokens": 1024,
    "temperature": 0.6,
    "top_p": 0.95,
    "frequency_penalty": 0,
    "presence_penalty": 1.1
  }
  ```

### Step 12: Send AI Text Response Back

- Add Telegram node (AI Response) connected from NeurochainAI - REST API success output  
- Send Message operation  
- Text: `={{ $json.choices[0].text }}`  
- Chat ID and Reply to message ID dynamic from Telegram Trigger  
- Markdown parse mode

### Step 13: Handle Text API Errors

- Add Switch node (Switch2) connected from NeurochainAI REST API error output  
- Routes error `"500 - \"{\"error\":true,\"msg\":\"No response from worker\"}\""` to No response Telegram node  
- Routes other errors to Prompt too short Telegram node  
- No response node sends "No response from worker" message  
- Prompt too short node sends "Prompt too short" message with retry button  

### Step 14: Final Touches and Cleanup

- Add Telegram nodes to delete error messages after retry (Telegram5, Telegram7)  
- Ensure proper linking for all nodes to maintain the flow  

### Step 15: Add Sticky Notes (optional)

- Add sticky notes with instructions, error/success notes, HTTP request guidelines, and setup steps for clarity

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                              | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| NeurochainAI Website and Guides: https://www.neurochain.ai/ and https://docs.neurochain.ai/nc/neurochainai-guides provide official documentation, API references, and helpful setup instructions.                                                                                         | Referenced in Sticky Note8                                                                          |
| Supported NeurochainAI Models for Text and Flux APIs include Meta-Llama and Mistral variants for text, and flux1-schnell-gguf for images; these can be customized in HTTP Request nodes.                                                                                                   | See Sticky Note2 and Sticky Note5                                                                  |
| The workflow assumes Telegram Bot Token and NeurochainAI API Key are properly configured as credentials in n8n. Ensure these credentials have the necessary permissions and sufficient API credits.                                                                                         | Setup prerequisite                                                                                   |
| Error handling is designed to gracefully inform users and offer retry options with inline keyboard buttons.                                                                                                                                                                              | Implemented via Telegram3 and Telegram6 nodes                                                      |
| The use of "typing" actions and temporary emoji messages improves the user experience by providing feedback during potentially long AI processing times.                                                                                                                                  | Nodes Telegram2, Telegram4, TYPING - ACTION                                                        |

---

This documentation provides a detailed and structured overview to fully understand, reproduce, and maintain the NeurochainAI Telegram bot integration workflow using n8n.