Send Daily AI-Generated Quotes & Images to Telegram with GPT-4o and Flux-pro

https://n8nworkflows.xyz/workflows/send-daily-ai-generated-quotes---images-to-telegram-with-gpt-4o-and-flux-pro-7425


# Send Daily AI-Generated Quotes & Images to Telegram with GPT-4o and Flux-pro

# Send Daily AI-Generated Quotes & Images to Telegram with GPT-4o and Flux-pro

---

## 1. Workflow Overview

This workflow automates the daily delivery of an AI-generated inspirational quote along with a matching AI-generated image to a Telegram chat. It supports two trigger modes:  
- **Scheduled trigger**: automatically runs daily at a specified time  
- **Manual trigger via Telegram**: captures a user’s chat ID by receiving any message sent to the bot, enabling personalized delivery.

The workflow logically divides into four main blocks:

1.1 **Trigger Reception**  
- Receives input either from a daily schedule or a Telegram message  
- Captures the Telegram chat ID for sending messages  

1.2 **AI Quote Generation**  
- Uses the OpenAI GPT-4o model via AI/ML API to generate a short, original, uplifting quote without attribution or formatting  

1.3 **AI Image Creation**  
- Uses the flux-pro model via AI/ML API HTTP request to generate a cinematic, text-free image inspired by the quote  

1.4 **Telegram Delivery**  
- Sends the generated image and quote caption to the captured Telegram chat ID  

---

## 2. Block-by-Block Analysis

### 1.1 Trigger Reception

**Overview:**  
This block initiates the workflow either daily at 7:30 AM or upon receiving any Telegram message. The Telegram trigger also captures the chat ID needed for sending messages back to the user.

**Nodes Involved:**  
- 📩 Receive Telegram Message1  
- ⏰ Schedule Trigger (Daily)1  

**Node Details:**  

- **📩 Receive Telegram Message1**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages to capture chat ID  
  - Configuration:  
    - Trigger on "message" updates only  
    - Uses Telegram API credentials linked to a bot created via @BotFather  
  - Inputs: None (trigger node)  
  - Outputs: Emits the message JSON containing the chat ID (`item.json.message.chat.id`)  
  - Failure Types: Network issues, invalid or expired Telegram credentials, webhook misconfiguration  
  - Notes: Essential for manual triggering and capturing chat IDs dynamically  

- **⏰ Schedule Trigger (Daily)1**  
  - Type: Schedule Trigger  
  - Role: Automatically starts the workflow daily at 7:30 AM  
  - Configuration:  
    - Fixed daily interval trigger set to 07:30  
  - Inputs: None (trigger node)  
  - Outputs: Emits empty data triggering the chain  
  - Failure Types: System time misconfiguration  
  - Notes: Enables fully automated daily execution without manual interaction  

---

### 1.2 AI Quote Generation

**Overview:**  
Generates a concise, original, uplifting quote using the OpenAI GPT-4o model via AI/ML API. The output is plain text, max 180 characters, no author attribution or markdown.

**Nodes Involved:**  
- ✍️ Generate Quote (AI/ML API)1  

**Node Details:**  

- **✍️ Generate Quote (AI/ML API)1**  
  - Type: AI/ML API (OpenAI GPT-4o)  
  - Role: Produces the inspirational quote text  
  - Configuration:  
    - Model: `openai/gpt-4o`  
    - Prompt instructs to generate a short, original uplifting quote, max 180 chars, no quotes or author, in English  
    - No additional options or request modifications  
  - Inputs: Output from either schedule or Telegram trigger  
  - Outputs: JSON with the quote text in `item.json.content`  
  - Failure Types: API key expiry, rate limits, prompt misformatting, network errors  
  - Credentials: AI/ML API key from aimlapi.com configured in n8n credentials  
  - Notes: This node’s output feeds directly into the image generation node  

---

### 1.3 AI Image Creation

**Overview:**  
Creates a cinematic, aesthetically pleasing image inspired by the generated quote using the flux-pro model of the AI/ML API via HTTP request.

**Nodes Involved:**  
- 🎨 Generate Image (AI/ML API | flux-pro)2  

**Node Details:**  

- **🎨 Generate Image (AI/ML API | flux-pro)2**  
  - Type: HTTP Request (API call to AI/ML API image generation endpoint)  
  - Role: Generates one 1024x1024 px image based on the quote text  
  - Configuration:  
    - POST to `https://api.aimlapi.com/v1/images/generations`  
    - Body parameters:  
      - model: `flux-pro`  
      - prompt: dynamic expression combining static instructions with the quote text from previous node (`$('✍️ Generate Quote (AI/ML API)1').item.json.content`)  
      - n: 1 (one image)  
      - size: 1024x1024  
      - response_formats: 1 (default image URL response)  
    - Authentication: Predefined AI/ML API credentials  
  - Inputs: Quote text from the Generate Quote node  
  - Outputs: JSON containing `images[0].url` with the generated image URL  
  - Failure Types: HTTP errors, API key issues, malformed prompt, quota exceeded  
  - Notes: The prompt explicitly excludes text in the image and stresses mood and composition  

---

### 1.4 Telegram Delivery

**Overview:**  
Sends the generated image and a caption containing the quote to the Telegram chat ID captured earlier.

**Nodes Involved:**  
- 📤 Send to Telegram1  

**Node Details:**  

- **📤 Send to Telegram1**  
  - Type: Telegram node (sendPhoto operation)  
  - Role: Delivers the final image and caption to the user’s Telegram chat  
  - Configuration:  
    - Operation: sendPhoto  
    - Chat ID: dynamic expression from Telegram trigger node's captured chat ID (`$('📩 Receive Telegram Message1').item.json.message.chat.id`)  
    - File URL: dynamic from generated image URL (`$('🎨 Generate Image (AI/ML API | flux-pro)2').item.json.images[0].url`)  
    - Caption: the quote prefixed with 🌅 emoji, dynamically from quote node  
  - Inputs: Output from image generation node  
  - Outputs: None (end of workflow)  
  - Failure Types: Invalid chat ID, Telegram API errors, network issues, expired credentials  
  - Credentials: Telegram Bot API key configured in n8n  
  - Notes: Works for both scheduled and manual runs after chat ID capture  

---

## 3. Summary Table

| Node Name                             | Node Type                   | Functional Role                   | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                              |
|-------------------------------------|-----------------------------|---------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Telegram Setup         | Sticky Note                 | Info on capturing chat ID via Telegram message |                               |                               | ## Incoming Message<br>### Use this to capture the chat ID for Telegram delivery. Send any message to this bot to get started.                                                                                                           |
| 📩 Receive Telegram Message1          | Telegram Trigger            | Capture chat ID on Telegram message |                               | ✍️ Generate Quote (AI/ML API)1 |                                                                                                                                                                                                                                          |
| ⏰ Schedule Trigger (Daily)1           | Schedule Trigger            | Daily scheduled workflow start  |                               | ✍️ Generate Quote (AI/ML API)1 |                                                                                                                                                                                                                                          |
| ✍️ Generate Quote (AI/ML API)1         | AI/ML API                   | Generate original inspirational quote | 📩 Receive Telegram Message1, ⏰ Schedule Trigger (Daily)1 | 🎨 Generate Image (AI/ML API | flux-pro)2 | ## 2️⃣ Generate Quote<br>### AI/ML API creates a short, original, uplifting quote with no author attribution, ready for visual inspiration.                                                                                             |
| 🎨 Generate Image (AI/ML API | flux-pro)2 | HTTP Request                | Generate image from quote       | 📤 Send to Telegram1           | ## 3️⃣ Create Image<br>### The generated quote is turned into an AI image via AI/ML API (flux-pro) with cinematic and aesthetic composition.                                                                                            |
| 📤 Send to Telegram1                  | Telegram                    | Send image and caption to Telegram chat | 🎨 Generate Image (AI/ML API | flux-pro)2 |                               | ## 4️⃣ Send to Telegram<br>### The final image and quote caption are sent to the captured chat ID from the Telegram trigger.                                                                                                           |
| Sticky Note - Generate Quote          | Sticky Note                 | Explains quote generation step  |                               |                               | ## 2️⃣ Generate Quote<br>### AI/ML API creates a short, original, uplifting quote with no author attribution, ready for visual inspiration.                                                                                             |
| Sticky Note - Create Image            | Sticky Note                 | Explains image generation step  |                               |                               | ## 3️⃣ Create Image<br>### The generated quote is turned into an AI image via AI/ML API (flux-pro) with cinematic and aesthetic composition.                                                                                            |
| Sticky Note - Send to Telegram        | Sticky Note                 | Explains Telegram delivery step |                               |                               | ## 4️⃣ Send to Telegram<br>### The final image and quote caption are sent to the captured chat ID from the Telegram trigger.                                                                                                           |
| Sticky Note - Telegram Setup1          | Sticky Note                 | Explains trigger start options  |                               |                               | ## 1️⃣ Trigger<br>### The workflow starts either on schedule (daily) or when a message is received from Telegram to capture the chat ID.                                                                                                |
| Sticky Note - Intro                   | Sticky Note                 | Workflow overview and purpose   |                               |                               | # 🌅 Daily AI Inspiration — Telegram + AI/ML API<br><br>This workflow sends a daily AI-generated inspirational quote with a matching image directly to your Telegram chat. It can also be triggered manually by sending a message to the bot. |
| Sticky Note - Step 1 Trigger          | Sticky Note                 | Setup instructions for triggers |                               |                               | ## 1️⃣ Trigger Setup (Telegram / Schedule)<br>- Start on a **daily schedule** (`Schedule Trigger` node) or via **Telegram message** (`Telegram Trigger` node).<br>- **Telegram Bot Setup:**<br>  1. Create a bot via [@BotFather](https://t.me/BotFather)<br>  2. Save the **Telegram Bot API key**<br>  3. In n8n: `Credentials → Telegram API` → paste the key<br>- Sending any message to the bot captures the **chat ID** for deliveries. |
| Sticky Note - Step 2 Quote            | Sticky Note                 | Setup instructions for quote generation |                               |                               | ## 2️⃣ Generate Quote (AI/ML API)<br>- Model: `openai/gpt-4o`<br>- Generates a short, original, uplifting quote:<br>  * No author name<br>  * No quotes around text<br>  * Max 180 characters<br>  * Avoid clichés<br>- **AI/ML API Setup:**<br>  1. Get your API key from [aimlapi.com](https://aimlapi.com)<br>  2. In n8n: `Credentials → AI/ML API` → paste the key<br>- Output is passed to the image generation step. |
| Sticky Note - Step 3 Image            | Sticky Note                 | Setup instructions for image generation |                               |                               | ## 3️⃣ Create Image (AI/ML API — flux-pro)<br>- Sends the generated quote as a prompt to `flux-pro` model.<br>- Produces a cinematic, aesthetically pleasing illustration:<br>  * No text in image<br>  * Focus on mood, lighting, composition<br>- Default parameters:<br>  * Size: 1024×1024<br>  * n: 1 (one image)<br>- Configurable in the HTTP Request node for different styles, sizes, or counts. |
| Sticky Note - Step 4 Send             | Sticky Note                 | Setup instructions for Telegram delivery |                               |                               | ## 4️⃣ Send to Telegram<br>- Sends the generated image to the captured **chat ID**.<br>- Adds the quote as a caption with a 🌅 emoji prefix.<br>- Works for both scheduled and manual runs.<br>- Pro Tip: Test the workflow manually once to confirm delivery before enabling the schedule. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram API Credential**  
   - In n8n credentials, add a new Telegram API credential  
   - Obtain Bot API key from [@BotFather](https://t.me/BotFather) on Telegram  
   - Paste the API key into the credential setup  

2. **Create AI/ML API Credential**  
   - In n8n credentials, add a new AI/ML API credential  
   - Obtain API key from [aimlapi.com](https://aimlapi.com)  
   - Paste the key into the credential setup  

3. **Add Telegram Trigger Node** (`📩 Receive Telegram Message1`)  
   - Type: Telegram Trigger  
   - Parameters: listen for "message" updates  
   - Credential: select Telegram API credential  
   - Position: e.g., X=448, Y=1760  

4. **Add Schedule Trigger Node** (`⏰ Schedule Trigger (Daily)1`)  
   - Type: Schedule Trigger  
   - Parameters: set daily trigger at 07:30 (7:30 AM)  
   - Position: e.g., X=448, Y=2112  

5. **Add AI/ML API Node for Quote Generation** (`✍️ Generate Quote (AI/ML API)1`)  
   - Type: AI/ML API  
   - Parameters:  
     - Model: `openai/gpt-4o`  
     - Prompt:  
       ```
       You are a generator of short, original, uplifting quotes.

       Requirements:
       - Output ONLY the quote text, no author, no quotes, no markdown.
       - Max 180 characters.
       - Avoid clichés.
       - Language: the same as this instruction.

       Generate 1 quote.
       ```  
   - Credential: select AI/ML API credential  
   - Position: e.g., X=752, Y=1936  

6. **Connect Triggers to Quote Node**  
   - Connect `📩 Receive Telegram Message1` main output to `✍️ Generate Quote (AI/ML API)1` main input  
   - Connect `⏰ Schedule Trigger (Daily)1` main output to `✍️ Generate Quote (AI/ML API)1` main input  

7. **Add HTTP Request Node for Image Generation** (`🎨 Generate Image (AI/ML API | flux-pro)2`)  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.aimlapi.com/v1/images/generations`  
     - Method: POST  
     - Authentication: Predefined Credential (select AI/ML API credential)  
     - Body Parameters (JSON):  
       - model: `flux-pro`  
       - prompt:  
         ```javascript
         "Create a cinematic, aesthetically pleasing illustration that captures the spirit of this quote. No text in the image. Emphasize mood, lighting, composition. Quote: " + $('✍️ Generate Quote (AI/ML API)1').item.json.content
         ```  
       - n: 1  
       - size: 1024x1024  
       - response_formats: 1  
   - Position: e.g., X=1072, Y=1936  

8. **Connect Quote Node Output to Image Node Input**  

9. **Add Telegram Node for Sending Image** (`📤 Send to Telegram1`)  
   - Type: Telegram  
   - Parameters:  
     - Operation: sendPhoto  
     - Chat ID: dynamic expression: `$('📩 Receive Telegram Message1').item.json.message.chat.id`  
     - File: dynamic expression: `$('🎨 Generate Image (AI/ML API | flux-pro)2').item.json.images[0].url`  
     - Additional Fields → Caption: dynamic expression: `"🌅 " + $('✍️ Generate Quote (AI/ML API)1').item.json.content`  
   - Credential: select Telegram API credential  
   - Position: e.g., X=1392, Y=1936  

10. **Connect Image Node Output to Telegram Send Node Input**  

11. **Test Workflow**  
    - Send a message to your Telegram bot to trigger the manual flow and capture your chat ID  
    - Confirm the quote and image are sent successfully  
    - Enable schedule trigger for daily automated delivery  

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                 | Context or Link                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Telegram Bot Setup: Create your bot via [@BotFather](https://t.me/BotFather) and save the API key for n8n integration.                                                                                                                                          | Telegram Bot Creation                                |
| AI/ML API Key: Obtain from [aimlapi.com](https://aimlapi.com) and configure in n8n credentials.                                                                                                                                                                 | AI/ML API Credential Setup                           |
| Prompt guidelines for quote generation emphasize brevity, originality, no attribution, and no markdown formatting to ensure clean text for captioning.                                                                                                        | AI Prompt Design                                     |
| Image generation prompt excludes text and focuses on cinematic mood, lighting, and composition for aesthetic appeal.                                                                                                                                           | Image Generation Prompt                              |
| Test manual trigger by sending a message to the bot before enabling the scheduled trigger to ensure chat ID capture and message delivery work correctly.                                                                                                       | Testing Recommendation                              |
| This workflow demonstrates integration of AI-generated content with messaging apps, combining GPT-4o for text and flux-pro for images, orchestrated by n8n automation platform.                                                                                  | Workflow Context                                    |

---

This documentation enables users and AI agents to understand, reproduce, and maintain the workflow effectively, ensuring smooth operation and easy customization.