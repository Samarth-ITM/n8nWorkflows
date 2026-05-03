Command-based Telegram Bot for Article Summarization & Image Prompts with OpenAI

https://n8nworkflows.xyz/workflows/command-based-telegram-bot-for-article-summarization---image-prompts-with-openai-4392


# Command-based Telegram Bot for Article Summarization & Image Prompts with OpenAI

### 1. Workflow Overview

This workflow implements a **command-based Telegram bot** designed primarily for two functionalities: summarizing articles from URLs and processing image generation prompts, both powered by OpenAI. It listens to incoming Telegram messages, routes commands to appropriate handlers, and sends back responses via Telegram.

**Target Use Cases:**
- Users want to quickly get concise summaries of online articles by sending a `/summary <URL>` command.
- Users want to request image generation prompts via `/img <prompt>` (currently returns a placeholder text response).
- A help command `/help` provides users with usage instructions.

**Logical Blocks:**

- **1.1 Input Reception and Command Routing**  
  Listens for Telegram messages, inspects command prefixes (`/help`, `/summary`, `/img`), and routes messages to the corresponding processing branches.

- **1.2 Help Command Processing**  
  Sends a formatted help menu in response to `/help`.

- **1.3 Article Summarization Workflow**  
  For `/summary` commands: downloads article HTML, extracts clean text, generates a structured summary via OpenAI, and sends the summary back to the user.

- **1.4 Image Generation Workflow**  
  For `/img` commands: processes the image prompt with OpenAI (currently text-only response), then notifies the user that the image generation request was processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Command Routing

- **Overview:**  
  This block captures all incoming Telegram messages and determines which command (if any) the user has sent, routing the flow accordingly.

- **Nodes Involved:**  
  - Trigger: Telegram Webhook  
  - Route: Check for Help Command  
  - Route: Check for Summary Command  
  - Route: Check for Image Command

- **Node Details:**

  - **Trigger: Telegram Webhook**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point that listens for all incoming messages from Telegram bot users.  
    - *Config:* Listens for `"message"` updates only.  
    - *Input:* External Telegram messages (via webhook).  
    - *Output:* Message JSON containing chat and text details.  
    - *Failures:* Webhook misconfiguration, Telegram API downtime.  
    - *Version:* 1.2

  - **Route: Check for Help Command**  
    - *Type:* If  
    - *Role:* Checks if the message text starts with `/help`.  
    - *Config:* Condition: string starts with `/help`.  
    - *Input:* Telegram message JSON.  
    - *Output:* Two outputs; true (send help), false (check summary).  
    - *Failures:* Expression evaluation errors if message text missing.  
    - *Version:* 1

  - **Route: Check for Summary Command**  
    - *Type:* If  
    - *Role:* Checks if the message text starts with `/summary`.  
    - *Config:* Condition: string starts with `/summary`.  
    - *Input:* Message JSON (false output of help check).  
    - *Output:* True (start article summary), false (check image).  
    - *Failures:* Expression errors if text missing.  
    - *Version:* 1

  - **Route: Check for Image Command**  
    - *Type:* If  
    - *Role:* Checks if the message text starts with `/img`.  
    - *Config:* Condition: string starts with `/img`.  
    - *Input:* Message JSON (false output of summary check).  
    - *Output:* True (process image request), false (no further handling).  
    - *Failures:* Expression errors if text missing.  
    - *Version:* 1

---

#### 2.2 Help Command Processing

- **Overview:**  
  Sends a predefined Markdown-formatted help message that lists available commands and usage examples.

- **Nodes Involved:**  
  - Response: Send Help Menu

- **Node Details:**

  - **Response: Send Help Menu**  
    - *Type:* Telegram  
    - *Role:* Sends help text back to the Telegram chat.  
    - *Config:*  
      - Text includes commands `/summary <link>` and `/img <prompt>`, with usage examples.  
      - Markdown parsing enabled for formatting.  
      - Chat ID extracted from incoming message JSON.  
    - *Input:* True output from help command check.  
    - *Output:* None (message sent to Telegram).  
    - *Failures:* Telegram API errors, invalid chat ID.  
    - *Version:* 1

---

#### 2.3 Article Summarization Workflow

- **Overview:**  
  Retrieves article HTML content from the URL provided in the `/summary` command, extracts meaningful text, generates a concise summary using OpenAI, and responds with the summary.

- **Nodes Involved:**  
  - Fetch: Download Article Content  
  - Parse: Extract Text from HTML  
  - AI: Generate Article Summary  
  - Response: Send Article Summary

- **Node Details:**

  - **Fetch: Download Article Content**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the article's raw HTML from the URL embedded in the Telegram message.  
    - *Config:*  
      - URL extracted from `message.link_preview_options.url` (Telegram message preview metadata).  
      - Custom User-Agent header to mimic a browser (`Mozilla/5.0`).  
      - No additional options for redirects or timeouts specified.  
    - *Input:* True output from summary command check.  
    - *Output:* Raw HTML content of the article.  
    - *Failures:* Invalid URL, network errors, HTTP errors (404, 500), missing URL in message.  
    - *Version:* 4.2

  - **Parse: Extract Text from HTML**  
    - *Type:* HTML  
    - *Role:* Cleans and extracts the main textual content from the downloaded HTML.  
    - *Config:*  
      - Extracts content from `<body>` element.  
      - Skips SVG and anchor (`<a>`) tags to avoid menus and navigation links.  
    - *Input:* HTML content from HTTP Request node.  
    - *Output:* Plain text extracted from the article body.  
    - *Failures:* Malformed HTML, unexpected document structure, empty extraction.  
    - *Version:* 1.2

  - **AI: Generate Article Summary**  
    - *Type:* OpenAI (LangChain)  
    - *Role:* Generates a structured summary with 10-12 bullet points from the extracted text.  
    - *Config:*  
      - Uses OpenAI model (credential "OpenAi account 2").  
      - Prompt instructs to summarize uniquely and professionally for decision-makers.  
      - Input is the extracted text from previous node.  
    - *Input:* Extracted article text.  
    - *Output:* AI-generated summary text (structured bullet points).  
    - *Failures:* OpenAI API errors, rate limits, empty input.  
    - *Version:* 1.8

  - **Response: Send Article Summary**  
    - *Type:* Telegram  
    - *Role:* Sends the AI-generated summary back to the user's Telegram chat.  
    - *Config:*  
      - Text content extracted from AI response JSON path `candidates[0].content.parts[0].text`.  
      - Chat ID from original Telegram message context.  
    - *Input:* AI summary output.  
    - *Output:* None (message sent).  
    - *Failures:* Telegram API errors, missing chat ID, malformed AI response.  
    - *Version:* 1

---

#### 2.4 Image Generation Workflow

- **Overview:**  
  Processes `/img` commands by sending the prompt to OpenAI's image resource endpoint and then sends a notification message back to the user. Currently, the node returns only a text response placeholder.

- **Nodes Involved:**  
  - AI: Process Image Generation Request  
  - Response: Send Image Generation Notice

- **Node Details:**

  - **AI: Process Image Generation Request**  
    - *Type:* OpenAI (LangChain)  
    - *Role:* Sends the image prompt to OpenAI's image resource (text-based placeholder).  
    - *Config:*  
      - Resource set to `"image"` (but no actual image generation endpoint integration).  
      - No model ID specified.  
      - Uses incoming Telegram message JSON for prompt input.  
    - *Input:* True output from image command check.  
    - *Output:* Text response placeholder for image generation.  
    - *Failures:* OpenAI API errors, unsupported image resource, empty prompt.  
    - *Version:* 1.8

  - **Response: Send Image Generation Notice**  
    - *Type:* Telegram  
    - *Role:* Notifies the user that the image generation request was received, explains current limitations, and suggests alternative APIs.  
    - *Config:*  
      - Fixed text message about Gemini image model limitations and recommends Stability AI for real images.  
      - Chat ID from Telegram message JSON.  
    - *Input:* AI image processing output.  
    - *Output:* None (message sent).  
    - *Failures:* Telegram API errors, missing chat ID.  
    - *Version:* 1

---

### 3. Summary Table

| Node Name                        | Node Type                  | Functional Role                               | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                                                         |
|---------------------------------|----------------------------|-----------------------------------------------|-------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Trigger: Telegram Webhook        | Telegram Trigger           | Listens for incoming Telegram messages        | (External)                    | Route: Check for Help Command          |                                                                                                                                     |
| Route: Check for Help Command    | If                         | Checks if message is `/help` command           | Trigger: Telegram Webhook      | Response: Send Help Menu, Route: Check for Summary Command |                                                                                                                                     |
| Response: Send Help Menu         | Telegram                   | Sends help menu message                         | Route: Check for Help Command  | (End)                                 |                                                                                                                                     |
| Route: Check for Summary Command| If                         | Checks if message is `/summary` command        | Route: Check for Help Command  | Fetch: Download Article Content, Route: Check for Image Command |                                                                                                                                     |
| Fetch: Download Article Content  | HTTP Request               | Downloads article HTML content                  | Route: Check for Summary Command | Parse: Extract Text from HTML         |                                                                                                                                     |
| Parse: Extract Text from HTML    | HTML                       | Extracts clean text from HTML                    | Fetch: Download Article Content | AI: Generate Article Summary          |                                                                                                                                     |
| AI: Generate Article Summary     | OpenAI (LangChain)         | Generates article summary from extracted text  | Parse: Extract Text from HTML  | Response: Send Article Summary         |                                                                                                                                     |
| Response: Send Article Summary   | Telegram                   | Sends AI-generated summary to user              | AI: Generate Article Summary   | (End)                                 |                                                                                                                                     |
| Route: Check for Image Command   | If                         | Checks if message is `/img` command             | Route: Check for Summary Command | AI: Process Image Generation Request |                                                                                                                                     |
| AI: Process Image Generation Request | OpenAI (LangChain)   | Processes image prompt (text placeholder)       | Route: Check for Image Command | Response: Send Image Generation Notice |                                                                                                                                     |
| Response: Send Image Generation Notice | Telegram             | Notifies user about image generation request    | AI: Process Image Generation Request | (End)                              |                                                                                                                                     |
| StickyNote                      | Sticky Note                | Documentation and overview                       | None                         | None                                  | # 🤖 Telegram Multi-Function Bot Workflow\n\n**Purpose:** This workflow creates a Telegram bot that handles multiple commands for article summarization and image generation.\n\n**Supported Commands:**\n- `/help` - Shows available commands and usage examples\n- `/summary <URL>` - Fetches and summarizes articles from web links\n- `/img <prompt>` - Processes image generation requests (currently placeholder)\n\n**Flow Logic:**\n1. Telegram webhook receives all messages\n2. Command routing checks message content and directs to appropriate handler\n3. Article summarization: URL → HTTP fetch → HTML parsing → AI summary → Response\n4. Image generation: Prompt processing → AI handling → Response notification\n\n**Note:** Image generation currently returns text confirmation instead of actual images. Consider integrating with Stability AI or similar services for real image generation. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure webhook to listen for `"message"` updates only.  
   - Save webhook URL and set it in your Telegram Bot settings.

2. **Add If node for Help Command check**  
   - Type: If  
   - Condition: Message text starts with `/help`  
   - Connect Telegram Trigger output to this node.

3. **Add Telegram node to send Help Menu**  
   - Type: Telegram  
   - Text:  
     ```
     🤖 *Help Menu*

     Use `/summary <link>` to summarize an article.
     Use `/img <prompt>` to generate an image.

     _Example:_
     /summary https://example.com
     /img a futuristic cityscape
     ```  
   - Enable Markdown parse mode.  
   - Chat ID: `{{$json["message"]["chat"]["id"]}}`  
   - Connect true output of Help Command check here.

4. **Add If node for Summary Command check**  
   - Type: If  
   - Condition: Message text starts with `/summary`  
   - Connect false output of Help Command check here.

5. **Add HTTP Request node to fetch article content**  
   - Type: HTTP Request  
   - URL: `={{ $json.message.link_preview_options.url }}`  
   - Set header: User-Agent = `"Mozilla/5.0"`  
   - Connect true output of Summary Command check here.

6. **Add HTML node to extract text from HTML**  
   - Type: HTML  
   - Operation: Extract HTML content  
   - CSS Selector: `body`  
   - Skip Selectors: `svg, a`  
   - Connect HTTP Request output here.

7. **Add OpenAI node for article summary generation**  
   - Type: OpenAI (LangChain)  
   - Credential: Your OpenAI account (ensure API key configured)  
   - Prompt message:  
     ```
     Summarize the entire content provided below into 10–12 concise bullet points. Ensure each point captures a unique and important aspect of the information, covering the core ideas, key facts, major findings, and essential takeaways. Avoid repetition and use clear, professional language suitable for quick understanding by a decision-maker.

     Content:
     {{ $json.text }}
     ```  
   - Connect HTML extraction output here.

8. **Add Telegram node to send article summary**  
   - Type: Telegram  
   - Text: `={{$json["candidates"][0]["content"]["parts"][0]["text"]}}`  
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Connect OpenAI summary output here.

9. **Add If node for Image Command check**  
   - Type: If  
   - Condition: Message text starts with `/img`  
   - Connect false output of Summary Command check here.

10. **Add OpenAI node for image generation request**  
    - Type: OpenAI (LangChain)  
    - Resource: `image` (note: will return text placeholder)  
    - Connect true output of Image Command check here.

11. **Add Telegram node to send image generation notice**  
    - Type: Telegram  
    - Text:  
      ```
      🖼️ Generated image prompt submitted! Gemini image model doesn't return images directly. Use image generation APIs like Stability for actual image URLs.
      ```  
    - Chat ID: `={{$json["message"]["chat"]["id"]}}`  
    - Connect OpenAI image node output here.

12. **Set up credentials**  
    - Configure Telegram API credentials for all Telegram nodes.  
    - Configure OpenAI API credentials for AI nodes.

13. **Test the workflow**  
    - Send `/help` to verify help message.  
    - Send `/summary <valid URL>` to test article summarization.  
    - Send `/img <text prompt>` to test image generation placeholder.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The image generation workflow currently returns only a textual confirmation, not an actual image URL. Integration with APIs like Stability AI is recommended. | Image generation placeholder note in Response: Send Image Generation Notice node.                   |
| This workflow uses LangChain-powered OpenAI nodes for advanced prompt engineering and structured responses.                                                | AI nodes configuration and prompt content.                                                         |
| Project credits and comprehensive workflow details are embedded in the StickyNote node within the workflow JSON for user reference.                        | StickyNote content attached to the workflow for documentation purposes.                             |

---

**Disclaimer:** This content is generated solely from an n8n workflow automation. It strictly respects content policies and contains no illegal or offensive material. All processed data is legal and publicly accessible.