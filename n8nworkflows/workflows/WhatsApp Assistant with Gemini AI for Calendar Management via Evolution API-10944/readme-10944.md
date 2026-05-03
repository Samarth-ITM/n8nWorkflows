WhatsApp Assistant with Gemini AI for Calendar Management via Evolution API

https://n8nworkflows.xyz/workflows/whatsapp-assistant-with-gemini-ai-for-calendar-management-via-evolution-api-10944


# WhatsApp Assistant with Gemini AI for Calendar Management via Evolution API

### 1. Workflow Overview

This workflow enables WhatsApp users to interact with a calendar management assistant powered by Google Gemini AI and integrated with Google Calendar via an MCP Server and the Evolution API. It processes incoming WhatsApp messages and media, converts them into structured inputs for an AI agent that interprets user intents, executes calendar-related actions through MCP tools, and replies to the user accordingly.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception:** Receives WhatsApp messages via webhook and classifies message types.
- **1.2 Media Retrieval and Conversion:** For media messages (image, audio, document), retrieves Base64 data and converts it to binary files.
- **1.3 AI Content Processing:** Uses Google Gemini AI nodes to analyze or transcribe media and generate textual content.
- **1.4 AI Agent Orchestration:** An AI agent interprets user requests, determines necessary tools, interacts with MCP Server calendar tools, and manages conversational memory.
- **1.5 MCP Server Calendar Tools:** Executes calendar operations (create, update, delete, list events) via MCP Server endpoints.
- **1.6 Output Messaging:** Sends processed responses back to the WhatsApp user through the Evolution API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block receives incoming WhatsApp messages through a webhook and determines the type of each message to route it accordingly.

**Nodes Involved:**  
- Webhook  
- Message Type  
- Message not supported

**Node Details:**

- **Webhook**  
  - Type: Webhook (HTTP POST listener)  
  - Configuration: Listens on path `/Calendar` for incoming POST requests from Evolution API.  
  - Inputs: HTTP POST from Evolution API with WhatsApp message payload.  
  - Outputs: Passes message JSON to "Message Type" node.  
  - Edge Cases: Invalid payloads or unauthorized requests may cause failure. Requires proper Evolution API webhook setup.

- **Message Type**  
  - Type: Switch node  
  - Configuration: Routes messages based on `body.data.messageType` JSON field, distinguishing 'conversation' (text), 'imageMessage', 'audioMessage', and 'documentMessage'.  
  - Outputs: Branches to respective processing nodes per media type or fallback to "Message not supported".  
  - Edge Cases: Unknown or unsupported message types trigger the fallback output.

- **Message not supported**  
  - Type: Evolution API node (send message)  
  - Configuration: Sends a fixed reply "This type of message is not supported by the system." to the sender's WhatsApp remoteJid.  
  - Inputs: From fallback of Message Type node.  
  - Edge Cases: Failures in sending message due to API errors or invalid remoteJid.

---

#### 2.2 Media Retrieval and Conversion

**Overview:**  
For media messages (image, audio, document), this block retrieves the Base64-encoded media via Evolution API, converts it into binary files suitable for AI analysis.

**Nodes Involved:**  
- Get base64 image  
- Get base64 audio  
- Get base64 document  
- Convert to image  
- Convert to audio  
- Convert to document

**Node Details:**

- **Get base64 image / audio / document**  
  - Type: Evolution API node (chat-api resource)  
  - Configuration: Uses messageId from incoming JSON to fetch Base64 media data from Evolution API instance named "Example".  
  - Inputs: From respective branches of Message Type node.  
  - Outputs: Base64 media content passed downstream.  
  - Edge Cases: Failures if media is unavailable, expired, or Evolution API instance misconfigured.

- **Convert to image / audio / document**  
  - Type: ConvertToFile node  
  - Configuration: Converts Base64 string (under `data.base64`) to binary file for image/audio/document analysis. For audio and document, binary property named `data_1`.  
  - Inputs: Base64 data from Get base64 nodes.  
  - Outputs: Binary data passed to AI analysis nodes.  
  - Edge Cases: Conversion errors if Base64 is corrupted or missing.

---

#### 2.3 AI Content Processing

**Overview:**  
Analyzes or transcribes media content into structured textual summaries or transcriptions using Google Gemini AI.

**Nodes Involved:**  
- Analyze an image  
- Transcribe a recording  
- Analyze document  
- Image (set node)  
- Audio (set node)  
- Document (set node)

**Node Details:**

- **Analyze an image**  
  - Type: Langchain Google Gemini node (image analysis)  
  - Configuration: Uses Gemini 2.5 Flash model with a detailed prompt instructing factual image description. Input is binary image file.  
  - Outputs: JSON with image description text.  
  - Edge Cases: Model API failure or unclear images may yield poor descriptions.

- **Transcribe a recording**  
  - Type: Langchain Google Gemini node (audio transcription)  
  - Configuration: Gemini 2.5 Flash model transcribes audio binary input.  
  - Outputs: JSON with transcription text.  
  - Edge Cases: Audio quality issues or API timeouts.

- **Analyze document**  
  - Type: Langchain Google Gemini node (document summarization)  
  - Configuration: Gemini 2.5 Flash model summarizes document binary input briefly.  
  - Outputs: JSON with summary text.  
  - Edge Cases: Complex documents or unsupported formats may affect summary quality.

- **Image / Audio / Document (Set nodes)**  
  - Type: Set  
  - Configuration: Extracts textual content from AI node output under `content.parts[0].text` and assigns to `text` property. Image node also captures image caption from original message JSON.  
  - Outputs: Text assigned for AI Agent input.  
  - Edge Cases: Missing or malformed AI output JSON.

---

#### 2.4 AI Agent Orchestration

**Overview:**  
Receives user input text (from text messages or media analysis), maintains conversational memory, and uses an AI agent to decide and execute calendar-related actions via MCP tools.

**Nodes Involved:**  
- Edit Fields  
- Archestrator Agent  
- Simple Memory  
- Google Gemini Chat Model  
- MCP Calendar

**Node Details:**

- **Edit Fields**  
  - Type: Set  
  - Configuration: Extracts conversation text from incoming message JSON to `text` field for agent input.  
  - Inputs: From "Message Type" text branch.  
  - Outputs: Passes `text` to AI Agent.  
  - Edge Cases: Missing conversation text.

- **Simple Memory**  
  - Type: Langchain memoryBufferWindow  
  - Configuration: Maintains last 8 messages in session keyed by sender's remoteJid to provide conversational context to AI agent.  
  - Inputs: AI Agent's `ai_memory` input.  
  - Outputs: Provides memory context to AI Agent.  
  - Edge Cases: Limited context window may truncate long conversations.

- **Google Gemini Chat Model**  
  - Type: Langchain lmChatGoogleGemini  
  - Configuration: Language model used by AI Agent for interpreting inputs and generating outputs.  
  - Inputs: AI Agent's `ai_languageModel` input.  
  - Outputs: Textual AI responses and tool calls.  
  - Edge Cases: Model API failures or rate limits.

- **Archestrator Agent**  
  - Type: Langchain agent  
  - Configuration: Core AI agent that processes text input, uses system prompt defining rules for tool invocation, temporal references, and safe operation. Connects to MCP Calendar tool and uses memory and language model nodes.  
  - Inputs: Text from media or text message (via Edit Fields, Image, Audio, Document set nodes), memory context, language model.  
  - Outputs: Generates text response and calls MCP Calendar tools as needed.  
  - Edge Cases: Ambiguous user requests, missing data, or misinterpretation may require additional user prompts.

- **MCP Calendar**  
  - Type: MCP Client Tool (HTTP client to MCP Server)  
  - Configuration: Connects to MCP Server endpoint for calendar operations at `https://hexagom-n8n.cloudfy.cloud/mcp/calendar`.  
  - Inputs: Called by AI Agent to perform calendar actions (get, create, update, delete events).  
  - Outputs: Returns results of calendar operations to AI Agent.  
  - Edge Cases: Server downtime, authentication issues, or malformed requests.

---

#### 2.5 MCP Server Calendar Tools

**Overview:**  
These nodes implement specific Google Calendar operations, callable by the MCP Server via the AI agent.

**Nodes Involved:**  
- MCP Server Google Calendar  
- Create an event  
- Get many events  
- Update an event  
- Delete an event

**Node Details:**

- **MCP Server Google Calendar**  
  - Type: MCP Trigger node (webhook)  
  - Configuration: Listens on `/calendar` path for MCP Server tool calls; triggers respective Google Calendar nodes.  
  - Inputs: Incoming MCP Server requests (called by AI Agent)  
  - Outputs: Triggers calendar tool nodes accordingly.  
  - Edge Cases: Webhook downtime or misrouting.

- **Create an event**  
  - Type: Google Calendar Tool node (create)  
  - Configuration: Creates event on calendar `example@gmail.com` with parameters (start, end, summary, description) provided by AI agent overrides.  
  - Inputs: Called by MCP Server Google Calendar node.  
  - Outputs: Confirmation of event creation.  
  - Edge Cases: Invalid date formats, permission errors.

- **Get many events**  
  - Type: Google Calendar Tool node (getAll)  
  - Configuration: Retrieves events in time range specified by AI agent.  
  - Inputs: Called by MCP Server Google Calendar node.  
  - Outputs: List of events.  
  - Edge Cases: Large result sets or API limits.

- **Update an event**  
  - Type: Google Calendar Tool node (update)  
  - Configuration: Updates event by ID with supplied fields (start, end, summary, description).  
  - Inputs: Called by MCP Server Google Calendar node.  
  - Outputs: Confirmation of update.  
  - Edge Cases: Event not found, invalid fields.

- **Delete an event**  
  - Type: Google Calendar Tool node (delete)  
  - Configuration: Deletes event by ID.  
  - Inputs: Called by MCP Server Google Calendar node.  
  - Outputs: Confirmation of deletion.  
  - Edge Cases: Event not found or permission denied.

---

#### 2.6 Output Messaging

**Overview:**  
Sends the AI agent's response back to the WhatsApp user through Evolution API.

**Nodes Involved:**  
- Send message

**Node Details:**

- **Send message**  
  - Type: Evolution API node (messages-api resource)  
  - Configuration: Sends text message using `remoteJid` from original message and text from AI Agent output.  
  - Inputs: Output from Archestrator Agent main output.  
  - Outputs: Delivery confirmation.  
  - Edge Cases: API failures, invalid remoteJid, or message length limits.

---

### 3. Summary Table

| Node Name                 | Node Type                                 | Functional Role                                  | Input Node(s)              | Output Node(s)                    | Sticky Note                                                                                  |
|---------------------------|-------------------------------------------|-------------------------------------------------|----------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Webhook                   | Webhook                                   | Receive WhatsApp messages                         | -                          | Message Type                     | ## AI Agent that uses MCP Server to execute actions requested via Evolution API...           |
| Message Type              | Switch                                    | Classify message type and route                   | Webhook                    | Edit Fields, Get base64 image, Get base64 audio, Get base64 document, Message not supported | ## Here it will identify the type and forward the message.                                  |
| Message not supported     | Evolution API                             | Reply unsupported message type                     | Message Type (fallback)     | -                                |                                                                                              |
| Get base64 image          | Evolution API                             | Retrieve Base64 image media                        | Message Type (Image)        | Convert to image                  |                                                                                              |
| Get base64 audio          | Evolution API                             | Retrieve Base64 audio media                        | Message Type (Audio)        | Convert to audio                  |                                                                                              |
| Get base64 document       | Evolution API                             | Retrieve Base64 document media                     | Message Type (Document)     | Convert to document               |                                                                                              |
| Convert to image          | ConvertToFile                            | Convert Base64 to binary image file                | Get base64 image            | Analyze an image                  |                                                                                              |
| Convert to audio          | ConvertToFile                            | Convert Base64 to binary audio file                | Get base64 audio            | Transcribe a recording            |                                                                                              |
| Convert to document       | ConvertToFile                            | Convert Base64 to binary document file             | Get base64 document         | Analyze document                 |                                                                                              |
| Analyze an image          | Langchain Google Gemini (image)          | Analyze image content                              | Convert to image            | Image                           |                                                                                              |
| Transcribe a recording    | Langchain Google Gemini (audio)          | Transcribe audio recordings                        | Convert to audio            | Audio                           |                                                                                              |
| Analyze document          | Langchain Google Gemini (document)       | Summarize document content                         | Convert to document         | Document                        |                                                                                              |
| Image                     | Set                                       | Extract image description text and caption        | Analyze an image            | Archestrator Agent               |                                                                                              |
| Audio                     | Set                                       | Extract transcription text                         | Transcribe a recording      | Archestrator Agent               |                                                                                              |
| Document                  | Set                                       | Extract document summary text                      | Analyze document            | Archestrator Agent               |                                                                                              |
| Edit Fields               | Set                                       | Extract conversation text from plain messages      | Message Type (Text)         | Archestrator Agent               |                                                                                              |
| Archestrator Agent        | Langchain agent                          | Interpret user input, call MCP tools, generate responses | Image, Audio, Document, Edit Fields, Simple Memory, Google Gemini Chat Model, MCP Calendar | Send message                    | ## The orchestrating agent will analyze the input, request the necessary tools, and generate the output. |
| Simple Memory             | Langchain memoryBufferWindow              | Maintain conversation context                      | Archestrator Agent (ai_memory) | Archestrator Agent (ai_memory) |                                                                                              |
| Google Gemini Chat Model  | Langchain lmChatGoogleGemini              | Language model for AI agent                        | Archestrator Agent (ai_languageModel) | Archestrator Agent (ai_languageModel) |                                                                                              |
| MCP Calendar              | MCP Client Tool                          | Execute calendar operations via MCP Server         | Archestrator Agent (ai_tool) | Archestrator Agent (ai_tool)    |                                                                                              |
| MCP Server Google Calendar| MCP Trigger                              | Receive MCP Server calendar requests               | MCP Tool nodes (Create, Get, Update, Delete) | MCP Tool nodes               | ## MCP Server with the tools connected.                                                     |
| Create an event           | Google Calendar Tool                      | Create Google Calendar event                       | MCP Server Google Calendar | MCP Server Google Calendar       |                                                                                              |
| Get many events           | Google Calendar Tool                      | Retrieve list of events                            | MCP Server Google Calendar | MCP Server Google Calendar       |                                                                                              |
| Update an event           | Google Calendar Tool                      | Update Google Calendar event                       | MCP Server Google Calendar | MCP Server Google Calendar       |                                                                                              |
| Delete an event           | Google Calendar Tool                      | Delete Google Calendar event                       | MCP Server Google Calendar | MCP Server Google Calendar       |                                                                                              |
| Send message              | Evolution API                             | Send response message to WhatsApp user            | Archestrator Agent          | -                                |                                                                                              |
| Sticky Note               | Sticky Note                              | Documentation and explanation                      | -                          | -                                | See 1.1 for full content: AI Agent that uses MCP Server to execute actions requested via Evolution API. |
| Sticky Note1              | Sticky Note                              | Section header: Search, convert, and analyze media| -                          | -                                |                                                                                              |
| Sticky Note2              | Sticky Note                              | Section header: MCP Server with the tools connected | -                          | -                                |                                                                                              |
| Sticky Note3              | Sticky Note                              | Section header: Identify message type and forward | -                          | -                                |                                                                                              |
| Sticky Note4              | Sticky Note                              | Section header: Orchestrating agent details       | -                          | -                                |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node:**  
   - Type: Webhook  
   - Set HTTP Method to POST  
   - Path: `Calendar`  
   - Save Webhook URL for Evolution API webhook configuration.

2. **Create Message Type node:**  
   - Type: Switch  
   - Rules based on JSON path `$json.body.data.messageType`:  
     - Text: equals `"conversation"`  
     - Image: equals `"imageMessage"`  
     - Audio: equals `"audioMessage"`  
     - Document: equals `"documentMessage"`  
   - Fallback output labeled "Message not supported".

3. **Create Message not supported node:**  
   - Type: Evolution API (messages-api resource)  
   - Parameters:  
     - remoteJid: `={{ $json.body.data.key.remoteJid }}`  
     - messageText: "This type of message is not supported by the system."  
     - instanceName: Your Evolution API instance name  
   - Connect fallback output of Message Type here.

4. **Create Get base64 media nodes (3):**  
   - For Image, Audio, Document:  
     - Type: Evolution API (chat-api resource)  
     - Operation: `get-media-base64`  
     - messageId: `={{ $json.body.data.key.id }}`  
     - instanceName: Your Evolution API instance name  
   - Connect respective outputs of Message Type.

5. **Create Convert to file nodes (3):**  
   - For Image, Audio, Document:  
     - Type: ConvertToFile  
     - Operation: `toBinary`  
     - Source Property: `data.base64`  
     - For Audio and Document, set Binary Property Name to `data_1`  
   - Connect outputs of corresponding Get base64 nodes.

6. **Create AI analysis nodes:**  
   - **Analyze an image:**  
     - Type: Langchain Google Gemini (image resource)  
     - Model ID: `models/gemini-2.5-flash`  
     - Input Type: binary  
     - Operation: analyze  
     - Text prompt as per workflow instructions for factual image description.  
     - Connect Convert to image output.  
   - **Transcribe a recording:**  
     - Type: Langchain Google Gemini (audio resource)  
     - Model ID: `models/gemini-2.5-flash`  
     - Input Type: binary  
     - Connect Convert to audio output.  
   - **Analyze document:**  
     - Type: Langchain Google Gemini (document resource)  
     - Model ID: `models/gemini-2.5-flash`  
     - Input Type: binary  
     - Text prompt: "What is this document about? Please provide a summary..."  
     - Connect Convert to document output.

7. **Create Set nodes for media text extraction:**  
   - Image node:  
     - Assign `text` = `={{ $json.content.parts[0].text }}`  
     - Assign `image_caption` = `={{ $('Message Type').item.json.body.data.message.imageMessage.caption }}`  
     - Connect Analyze an image node output.  
   - Audio node:  
     - Assign `text` = `={{ $json.content.parts[0].text }}`  
     - Connect Transcribe a recording node output.  
   - Document node:  
     - Assign `text` = `={{ $json.content.parts[0].text }}`  
     - Connect Analyze document node output.

8. **Create Edit Fields node for text messages:**  
   - Type: Set  
   - Assign `text` = `={{ $json.body.data.message.conversation }}`  
   - Connect Message Type text output.

9. **Create Archestrator Agent node:**  
   - Type: Langchain agent  
   - Text input: `={{ $json.text }}` from Set or media nodes  
   - System prompt: Define AI assistant role, rules for tool invocation, temporal references, and safety as provided in workflow.  
   - Connect AI memory input to Simple Memory node.  
   - Connect AI language model input to Google Gemini Chat Model node.  
   - Connect AI tool input to MCP Calendar node.

10. **Create Simple Memory node:**  
    - Type: Langchain memoryBufferWindow  
    - Session key: `={{ $('Webhook').item.json.body.data.key.remoteJid }}`  
    - Context window length: 8 messages  
    - Connect to Archestrator Agent `ai_memory` input.

11. **Create Google Gemini Chat Model node:**  
    - Type: Langchain lmChatGoogleGemini  
    - Options default  
    - Connect to Archestrator Agent `ai_languageModel` input.

12. **Create MCP Calendar node:**  
    - Type: MCP Client Tool  
    - Endpoint URL: your MCP Server calendar URL (e.g. `https://hexagom-n8n.cloudfy.cloud/mcp/calendar`)  
    - Connect to Archestrator Agent `ai_tool` input.

13. **Create MCP Server Google Calendar webhook:**  
    - Type: MCP Trigger  
    - Path: `calendar`  
    - Connect outputs to following Google Calendar Tool nodes.

14. **Create Google Calendar Tool nodes:**  
    - Create an event (operation: create)  
    - Get many events (operation: getAll)  
    - Update an event (operation: update)  
    - Delete an event (operation: delete)  
    - Set calendar to your calendar ID (e.g. `example@gmail.com`)  
    - Use AI agent override parameters for event details.

15. **Connect MCP Server Google Calendar outputs:**  
    - Connect to each calendar operation node.

16. **Create Send message node:**  
    - Type: Evolution API (messages-api resource)  
    - remoteJid: `={{ $('Webhook').item.json.body.data.key.remoteJid }}`  
    - messageText: AI Agent output text (`={{ $json.output }}`)  
    - instanceName: Your Evolution API instance name  
    - Connect Archestrator Agent main output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires valid Evolution API account and configured instance for WhatsApp messaging integration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Evolution API docs and account setup.                                                                                                            |
| Google Gemini AI API access is mandatory for media analysis and natural language understanding.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Google Gemini AI documentation.                                                                                                                  |
| Google Calendar credentials must be configured in n8n as OAuth2 credentials and assigned to Google Calendar Tool nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | n8n Google Calendar OAuth2 credential setup.                                                                                                    |
| MCP Server must be accessible via URL and configured to trigger calendar tools. Can be internal or external implementation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | MCP Server documentation.                                                                                                                        |
| The AI Agent prompt enforces strict rules to minimize unnecessary tool calls, handle ambiguous requests, and parse temporal expressions consistently. Adjust prompt to extend or customize behavior.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Contains detailed instructions in the Archestrator Agent node configuration.                                                                   |
| Conversation memory uses a sliding window of 8 messages for context. Increase window or change memory type if longer memory is needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Simple Memory node configuration note.                                                                                                          |
| For webhook setup, ensure to update Evolution API instance webhook URL with the URL generated by the Webhook node to receive WhatsApp messages.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | See workflow first node configuration.                                                                                                         |
| The workflow supports only text, image, audio, and document message types. Other media types prompt a "not supported" message.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Message Type node fallback output.                                                                                                              |
| To extend functionality to other MCP tools (e.g., Notion, ClickUp), add corresponding MCP Client Tool nodes and update AI Agent prompt accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Adaptation suggestion in sticky note.                                                                                                          |
| If using alternative chatbot platforms (Chatwoot, TypeBot), update webhook URL and verify message payload structures for compatibility with Message Type switch node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Adaptation suggestion in sticky note.                                                                                                          |

---

**Disclaimer:** The provided text results exclusively from an automated workflow created with n8n, respecting all current content policies and containing no illegal, offensive, or protected elements. All data handled is legal and public.