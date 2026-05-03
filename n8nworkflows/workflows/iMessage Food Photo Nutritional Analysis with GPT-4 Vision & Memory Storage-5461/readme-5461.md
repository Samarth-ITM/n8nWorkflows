iMessage Food Photo Nutritional Analysis with GPT-4 Vision & Memory Storage

https://n8nworkflows.xyz/workflows/imessage-food-photo-nutritional-analysis-with-gpt-4-vision---memory-storage-5461


# iMessage Food Photo Nutritional Analysis with GPT-4 Vision & Memory Storage

### 1. Workflow Overview

This workflow automates nutritional analysis of food photos received via iMessage using GPT-4 Vision capabilities and maintains conversational memory for personalized reporting. It integrates with Blooio.com to receive iMessage events, processes attached food images or text messages with an AI nutrition analyst agent, and replies with detailed, friendly nutritional summaries including daily, weekly, and monthly reports. The workflow also stores chat history in a Postgres database to maintain context across sessions.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception and Filtering:** Receives incoming iMessage events from Blooio.com via webhook, filters out self-messages, and checks for attached images.
- **1.2 Image Handling:** Extracts and downloads image URLs from message attachments for AI processing.
- **1.3 AI Nutrition Analysis:** Sends food photos or text content to a customized GPT-4 Vision agent that estimates nutritional content, confidence, and health scores.
- **1.4 Memory Storage:** Saves conversation context and session data in a Postgres chat memory node for ongoing personalized analysis.
- **1.5 Aggregation and Summary Generation:** Aggregates AI responses and generates a summarized report message.
- **1.6 Response Sending:** Uses Blooio.com API to send the nutrition summary back to the user via SMS/iMessage.

Supporting sticky notes provide setup instructions, webhook event examples, and detailed guidance for configuring nodes and authentication.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

- **Overview:** This block listens for incoming iMessage events from Blooio.com, prevents the workflow from responding to its own messages, and determines if the message contains image attachments.
- **Nodes Involved:**  
  - Receive Message (From Blooio)  
  - Don't respond to yourself  
  - If has images, download them

- **Node Details:**

  - **Receive Message (From Blooio)**  
    - Type: Webhook  
    - Role: Entry point that receives POST requests from Blooio.com containing message events (new or updated) including message metadata and attachments.  
    - Configuration: Webhook path `/receive-event`, HTTP method POST.  
    - Inputs: External HTTP request containing Blooio event JSON.  
    - Outputs: JSON payload with full message details.  
    - Edge Cases: Missing or malformed webhook payloads, connection timeouts, invalid HTTP method.  

  - **Don't respond to yourself**  
    - Type: If  
    - Role: Filters out messages sent by self to prevent feedback loops.  
    - Configuration: Checks `selfMessage` field in incoming JSON; proceeds only if false.  
    - Inputs: Output of webhook node.  
    - Outputs: True branch if not self-message, false branch otherwise.  
    - Edge Cases: Missing `selfMessage` field or unexpected data types could cause expression failures.

  - **If has images, download them**  
    - Type: If  
    - Role: Checks if message attachments array length > 0 to decide if image download is required.  
    - Configuration: Condition checks `length` of `attachments` array in message JSON strictly greater than 0.  
    - Inputs: Filtered messages from previous node.  
    - Outputs: True branch for messages with images, false branch for text-only messages.  
    - Edge Cases: No attachments field or empty arrays, type validation errors.

---

#### 2.2 Image Handling

- **Overview:** Extracts URLs from message attachments and triggers downloading images for AI analysis.
- **Nodes Involved:**  
  - Code  
  - HTTP Request  
  - Loop Over Items  

- **Node Details:**

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Transforms the attachments array from the message into individual items containing only the URL field for subsequent processing.  
    - Configuration: Iterates over `body.message.attachments` and outputs objects with `json.url`.  
    - Inputs: Messages with attachments from If node (true branch).  
    - Outputs: Array of URLs as separate items.  
    - Edge Cases: Empty attachments or missing fields cause empty outputs.

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Downloads each image by requesting the URL extracted from the previous node.  
    - Configuration: URL taken dynamically from item JSON `url`. No authentication or special headers configured.  
    - Inputs: URLs from Code node.  
    - Outputs: Binary image data for AI processing.  
    - Edge Cases: Broken URLs, expired URLs, network timeouts, unsupported MIME types.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Splits the array of URLs into individual items to process each image separately.  
    - Configuration: Default batch size (1 item per batch).  
    - Inputs: Output of Code and HTTP Request nodes.  
    - Outputs: One item per batch for AI processing.  
    - Edge Cases: Large numbers of attachments may slow workflow execution.

---

#### 2.3 AI Nutrition Analysis

- **Overview:** Uses a LangChain agent node configured with GPT-4 Vision to analyze food photos and generate nutritional summaries in a conversational style.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Postgres Chat Memory  

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent (AI Agent)  
    - Role: Receives user message content or image binary data and prompts GPT-4 Vision with a detailed system prompt to identify foods, estimate portions, calculate nutrition, and produce a friendly iMessage-style nutritional summary including confidence and health scores.  
    - Configuration:  
      - Text input dynamically set from incoming message content or image data.  
      - System message defines identity, mission, analysis protocol, response format, and error handling instructions.  
      - Passthrough of binary images enabled for vision analysis.  
    - Inputs: Image binaries or message texts from Loop Over Items node.  
    - Outputs: AI-generated text nutrition summaries.  
    - Version Requirements: LangChain agent with GPT-4 Vision support.  
    - Edge Cases: Unclear images, unsupported food items, API rate limits, AI hallucination or incomplete responses.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4.1-mini model access for the AI agent node to generate chat completions.  
    - Configuration: Model set to "gpt-4.1-mini", credentials linked to OpenAI API key.  
    - Inputs: Prompt and system message from AI Agent node.  
    - Outputs: Generated chat text.  
    - Edge Cases: API authentication failure, rate limits, network errors.

  - **Postgres Chat Memory**  
    - Type: LangChain Postgres Chat Memory  
    - Role: Stores conversation history keyed to the message conversation ID to maintain context and enable cumulative nutrition reporting.  
    - Configuration:  
      - Table: `n8n_calorie_tracker`  
      - Session Key: Conversation ID from message JSON  
      - Context window length set to 200 messages.  
      - Credentials linked to Neon Postgres instance.  
    - Inputs: AI Agent processed messages.  
    - Outputs: Updated memory context for AI agent.  
    - Edge Cases: Database connectivity issues, table schema mismatch, session key missing.

---

#### 2.4 Aggregation and Summary Generation

- **Overview:** Aggregates AI-generated nutritional insights and produces a consolidated report for the user.
- **Nodes Involved:**  
  - Aggregate  
  - AI Agent1  
  - OpenAI Chat Model1  

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all AI agent outputs into a single JSON object or array for summary generation.  
    - Configuration: Aggregates all item data from upstream AI Agent nodes.  
    - Inputs: AI Agent outputs (nutrition summaries).  
    - Outputs: Aggregated data for summarization.  
    - Edge Cases: Empty inputs, large data causing performance issues.

  - **AI Agent1**  
    - Type: LangChain Agent  
    - Role: Summarizes aggregated nutritional data into a concise iMessage-style report without meta-text or framing.  
    - Configuration:  
      - System message instructs to produce a direct summary only.  
      - Input text is the aggregated JSON string of combined AI agent outputs.  
    - Inputs: Aggregated nutritional data.  
    - Outputs: Final nutrition summary message text.  
    - Edge Cases: Incomplete or contradictory aggregated data.

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Supports AI Agent1 with GPT-4.1-mini model for generating the summary text.  
    - Configuration: Same as OpenAI Chat Model node with shared credentials.  
    - Inputs: Summarization prompt from AI Agent1.  
    - Outputs: Final summary text.  
    - Edge Cases: Same as OpenAI Chat Model.

---

#### 2.5 Response Sending

- **Overview:** Sends the generated nutrition summary back to the user via Blooio.com messaging API.
- **Nodes Involved:**  
  - Send Message

- **Node Details:**

  - **Send Message**  
    - Type: HTTP Request  
    - Role: Posts the nutrition summary message back to the user's phone number using Blooio.com's send-message API endpoint.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.blooio.com/send-message`  
      - Authentication: HTTP Bearer Token via Blooio Bearer Auth credentials.  
      - Headers: Accept `application/json`, Content-Type `application/json`, Authorization header with Bearer token.  
      - Body Parameters:  
        - `identifier`: Sender's phone number from original message JSON path `body.message.sender`  
        - `message`: Text from AI Agent1 output `$json.output`  
    - Inputs: Final summarized text from AI Agent1.  
    - Outputs: API response confirming message sent.  
    - Edge Cases: Invalid or expired API token, network failures, invalid phone numbers, rate limits.

---

#### 2.6 Supporting Documentation (Sticky Notes)

- Multiple sticky notes provide detailed instructions on setup, expected webhook payloads, image downloader explanation, and Blooio.com API integration.  
- Notable content includes:  
  - How to configure the Send Message HTTP Request node with Blooio.com API.  
  - Sample JSON payloads for message received and message read events.  
  - Step-by-step instructions for obtaining API tokens and testing the webhook.  
  - Visual guide image linked from Imgur for interface orientation.

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                      |
|----------------------------|-------------------------------|----------------------------------------------|--------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------|
| Receive Message (From Blooio) | Webhook                      | Entry point receiving iMessage events        | —                              | Don't respond to yourself       |                                                                                                                  |
| Don't respond to yourself   | If                            | Filters out self-sent messages                 | Receive Message (From Blooio)  | If has images, download them    |                                                                                                                  |
| If has images, download them| If                            | Checks if message has attachments             | Don't respond to yourself       | Code, AI Agent                 |                                                                                                                  |
| Code                       | Code (JavaScript)             | Extracts image URLs from attachments          | If has images, download them    | Loop Over Items                 |                                                                                                                  |
| HTTP Request               | HTTP Request                  | Downloads images from URLs                      | Loop Over Items (parallel branch)| Loop Over Items                |                                                                                                                  |
| Loop Over Items            | SplitInBatches                | Splits attachments list into individual items | Code, HTTP Request              | AI Agent, HTTP Request          |                                                                                                                  |
| AI Agent                   | LangChain Agent               | Analyzes food photos/text, generates nutrition summary | Loop Over Items               | Aggregate                      |                                                                                                                  |
| OpenAI Chat Model          | LangChain OpenAI Model        | Provides GPT-4.1-mini model for AI Agent      | AI Agent                       | AI Agent                       |                                                                                                                  |
| Postgres Chat Memory       | LangChain Memory Postgres     | Stores conversation context for personalized reports | AI Agent                   | AI Agent                       |                                                                                                                  |
| Aggregate                  | Aggregate                    | Combines multiple AI nutrition outputs        | AI Agent                       | AI Agent1                      |                                                                                                                  |
| AI Agent1                  | LangChain Agent               | Summarizes aggregated nutrition data          | Aggregate                      | Send Message                   |                                                                                                                  |
| OpenAI Chat Model1         | LangChain OpenAI Model        | Supports AI Agent1 for summary generation     | AI Agent1                     | AI Agent1                     |                                                                                                                  |
| Send Message               | HTTP Request                  | Sends nutrition summary message via Blooio API | AI Agent1                     | —                              | "## Using the n8n HTTP Request Node to Send a Blooio.com Message" (Instructions on API setup and headers)       |
| Sticky Note                | Sticky Note                   | Provides setup instructions and webhook examples | —                            | —                              | Multiple sticky notes with setup guides, webhook samples, and visual aids distributed through the workflow nodes |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive Message (From Blooio)"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `receive-event`  
   - Purpose: Receive iMessage events from Blooio.com webhook.

2. **Create If Node: "Don't respond to yourself"**  
   - Condition: `body.message.selfMessage` is false  
   - Connect input from "Receive Message (From Blooio)"  
   - Purpose: Filter out messages sent by the workflow itself.

3. **Create If Node: "If has images, download them"**  
   - Condition: Length of `body.message.attachments` > 0  
   - Connect input from "Don't respond to yourself" (true branch)  
   - Purpose: Detect if incoming message has image attachments.

4. **Create Code Node: "Code"**  
   - JavaScript code:

     ```js
     let output = [];
     for (const item of $input.first().json.body.message.attachments) {
       output.push({ json: { url: item.url } });
     }
     return output;
     ```

   - Connect input from "If has images, download them" (true branch)  
   - Purpose: Extract URLs from attachments for downloading.

5. **Create HTTP Request Node: "HTTP Request"**  
   - URL: `={{ $json.url }}` (dynamic)  
   - Method: GET (default)  
   - Purpose: Download each image from extracted URLs.  
   - Connect input from "Code".

6. **Create SplitInBatches Node: "Loop Over Items"**  
   - Default batch size (1)  
   - Connect inputs from both "Code" and "HTTP Request" (parallel outputs)  
   - Purpose: Process each image individually in following nodes.

7. **Create LangChain Agent Node: "AI Agent"**  
   - Text input: `=User message: {{ $('Receive Message (From Blooio)').item.json.body.message.content }}`  
   - Enable binary image passthrough  
   - System message: Use the detailed nutrition analysis prompt including identity, mission, analysis protocol, response format, and error handling (as described in 2.3).  
   - Connect input from "Loop Over Items".  
   - Purpose: Analyze food photos or text to generate nutrition info.

8. **Create LangChain OpenAI Chat Model Node: "OpenAI Chat Model"**  
   - Model: `gpt-4.1-mini`  
   - Credentials: OpenAI API key configured  
   - Connect to AI Agent's language model input.  
   - Purpose: Provide GPT-4 model for AI Agent.

9. **Create LangChain Postgres Chat Memory Node: "Postgres Chat Memory"**  
   - Table Name: `n8n_calorie_tracker`  
   - Session Key: `={{ $('Receive Message (From Blooio)').item.json.body.message.conversation.id }}`  
   - Context Window Length: 200  
   - Credentials: Postgres (Neon) connection  
   - Connect to AI Agent's memory input.  
   - Purpose: Store conversation context for personalized reports.

10. **Create Aggregate Node: "Aggregate"**  
    - Aggregate all item data  
    - Connect input from "AI Agent".  
    - Purpose: Combine multiple AI outputs into one.

11. **Create LangChain Agent Node: "AI Agent1"**  
    - Text input: `={{ $json.data.toJsonString() }}` (aggregated output)  
    - System message: Instruct to summarize messages as a direct iMessage-style report, no framing text.  
    - Connect input from "Aggregate".  
    - Purpose: Generate final nutrition summary message.

12. **Create LangChain OpenAI Chat Model Node: "OpenAI Chat Model1"**  
    - Model: `gpt-4.1-mini`  
    - Credentials: Same OpenAI API key  
    - Connect to AI Agent1.  
    - Purpose: Assist in summary generation.

13. **Create HTTP Request Node: "Send Message"**  
    - Method: POST  
    - URL: `https://api.blooio.com/send-message`  
    - Authentication: HTTP Bearer Token (Blooio API token)  
    - Headers:  
      - `Accept: application/json`  
      - `Content-Type: application/json`  
      - `Authorization: Bearer YOUR_API_TOKEN` (replace with actual token)  
    - Body Parameters (JSON):  
      - `identifier`: `={{ $('Receive Message (From Blooio)').item.json.body.message.sender }}`  
      - `message`: `={{ $json.output }}` (final summary text)  
    - Connect input from "AI Agent1".  
    - Purpose: Send nutritional summary back to user via Blooio.

14. **Connect Workflow Nodes as per the connection map:**  
    - Webhook → Don't respond to yourself → If has images → Code and AI Agent (parallel)  
    - Code → Loop Over Items → HTTP Request → Loop Over Items → AI Agent  
    - AI Agent → Aggregate → AI Agent1 → Send Message  
    - OpenAI Chat Model and Postgres Memory connected as AI Agent’s model and memory respectively  
    - OpenAI Chat Model1 connected to AI Agent1

15. **Credential Setup:**  
    - Configure Blooio Bearer Auth credentials with your API token.  
    - Configure OpenAI API credentials with a valid GPT-4 capable key.  
    - Configure Postgres credentials for Neon database access.

16. **Testing:**  
    - Send an iMessage with a meal photo or description to your Blooio number.  
    - Confirm webhook triggers, AI processes image/text, and summary is sent back.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                  |
|------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Detailed instructions for configuring the Blooio.com HTTP Request node to send SMS or email messages.      | Present in workflow Sticky Note near "Send Message" node.       |
| Sample JSON payloads for message received and message read webhook events illustrating expected data format.| Sticky Notes near webhook input node provide examples.          |
| Setup guide for obtaining Blooio API token and connecting it to the workflow for message sending.          | Sticky Note titled "🚀 Start Here: Blooio Nutrition Tracker Setup" with stepwise instructions. |
| Visual guide image linked from Imgur demonstrating user interface or workflow concept.                      | Sticky Note near workflow start with image `https://i.imgur.com/bOPrv4H.png`                       |
| Reminder: The AI agent is designed not to ask for user input and to respond conversationally without JSON or markdown formatting. | Embedded in AI Agent's system prompt configuration.             |

---

**Disclaimer:** This document is generated solely from an n8n workflow JSON export. It complies with content policies and contains no illegal, offensive, or protected content. All processed data are legal and publicly available.