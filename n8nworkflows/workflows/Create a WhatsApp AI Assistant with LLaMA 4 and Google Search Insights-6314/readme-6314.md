Create a WhatsApp AI Assistant with LLaMA 4 and Google Search Insights

https://n8nworkflows.xyz/workflows/create-a-whatsapp-ai-assistant-with-llama-4-and-google-search-insights-6314


# Create a WhatsApp AI Assistant with LLaMA 4 and Google Search Insights

### 1. Workflow Overview

This workflow implements a **real-time AI assistant on WhatsApp** using Groq’s **LLaMA 4 AI model** combined with **Google Search (via SerpAPI)** to provide fresh, context-aware responses. It is designed primarily for handling **text messages** from WhatsApp users, with an extendable architecture to support images or other media in the future.

#### Logical Blocks

- **1.1 Input Reception & Routing**  
  Captures incoming WhatsApp messages and routes them based on message type (text or image).

- **1.2 Text Message Preparation**  
  Extracts raw text content from WhatsApp message JSON and prepares it for AI processing.

- **1.3 AI Processing & Memory Context**  
  Uses the Groq LLaMA 4 model and maintains conversation memory to generate context-aware replies. Integrates Google Search via SerpAPI as an external tool for up-to-date information.

- **1.4 Output Delivery**  
  Sends the generated AI response back to the user via WhatsApp.

- **1.5 Documentation & Guidance (Sticky Notes)**  
  Provides embedded documentation for workflow understanding and future extensibility.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing

- **Overview:**  
  This block listens for incoming WhatsApp messages and determines their type (text or image), routing messages to appropriate processing paths.

- **Nodes Involved:**  
  - WhatsApp Trigger  
  - Route Based on Input (Switch Node)  
  - Sticky Note (Node Guide: Input Switch)

- **Node Details:**

  - **WhatsApp Trigger**  
    - Type: WhatsApp Trigger  
    - Role: Entry point for WhatsApp messages; triggers workflow on new messages.  
    - Configuration: Listens for message updates via WhatsApp Business API or Twilio sandbox webhook.  
    - Inputs: Incoming WhatsApp event data.  
    - Outputs: JSON containing message details.  
    - Credentials: OAuth2-based WhatsApp API credentials.  
    - Failure modes: Authentication errors, webhook misconfiguration, message format changes.

  - **Route Based on Input (Switch Node)**  
    - Type: Switch  
    - Role: Checks if the incoming message contains an image or text field.  
    - Configuration:  
      - Two outputs: "Image" if `messages[0].image` exists, "Text" if `messages[0].text` exists.  
      - Strict type validation and case-sensitive checks.  
    - Inputs: WhatsApp Trigger output.  
    - Outputs:  
      - "Text" output leads to text processing flow.  
      - "Image" output currently unconnected (reserved for future extension).  
    - Failure modes: Missing or malformed message JSON structure may cause routing failure.

  - **Sticky Note (Node Guide: Input Switch)**  
    - Provides explanation of this routing mechanism and notes future image message support.

---

#### 2.2 Text Message Preparation

- **Overview:**  
  Extracts the actual text message content from WhatsApp JSON and formats it for use by the AI Agent.

- **Nodes Involved:**  
  - Prepare Text Prompt  
  - Sticky Note2 (Execution Logic)

- **Node Details:**

  - **Prepare Text Prompt (Set Node)**  
    - Type: Set  
    - Role: Assigns the WhatsApp message text body to a new `text` field in the JSON payload.  
    - Configuration: Sets `text` = `messages[0].text.body` from incoming JSON.  
    - Inputs: Routed from "Text" output of Switch node.  
    - Outputs: JSON with `text` property ready for AI processing.  
    - Failure modes: If `messages[0].text.body` is missing or empty, `text` will be empty, potentially causing AI input errors.

  - **Sticky Note2 (Execution Logic)**  
    - Describes this block’s function: processing text inputs only, preparing for AI Agent consumption.

---

#### 2.3 AI Processing & Memory Context

- **Overview:**  
  Core AI processing block that generates replies using Groq’s LLaMA 4 model, augmented by Google Search when necessary, and maintains conversation history for contextual awareness.

- **Nodes Involved:**  
  - AI Agent  
  - Groq Chat Model  
  - Google Search  
  - Conversation Memory  
  - Sticky Note3 (Meet Your AI Agent)

- **Node Details:**

  - **AI Agent (Langchain Agent Node)**  
    - Type: Langchain Agent node (version 2.1)  
    - Role: Orchestrates AI response generation by combining language model, tools, and memory.  
    - Configuration:  
      - Input text: `={{ $json.text }}` from previous node.  
      - System Message: Defines AI personality as “Seventeen,” a casual, light-sarcastic AI speaking in American vernacular English, alternating greetings, referencing current user name and timestamp dynamically.  
      - Prompt type: 'define' (custom prompt).  
      - Inputs from:  
        - Groq Chat Model (as language model)  
        - Google Search (as AI tool)  
        - Conversation Memory (as memory buffer)  
      - Outputs: AI-generated text reply.  
    - Failure modes: API quota limits, malformed prompt variables, network timeouts, authentication errors on Groq or SerpAPI.  
    - Version-specific: Requires n8n version supporting Langchain Agent v2.1.  
    - Sub-workflow: None (main workflow node).

  - **Groq Chat Model (Langchain LLM Node)**  
    - Type: Langchain Language Model node  
    - Role: Provides the core LLaMA 4 language model for text generation.  
    - Configuration:  
      - Model: `meta-llama/llama-4-scout-17b-16e-instruct`  
      - No additional options set.  
      - Credentials: Groq API key.  
    - Inputs: From AI Agent (as language model).  
    - Outputs: Text generation results to AI Agent.  
    - Failure modes: API key invalid, timeout, or quota exceeded.

  - **Google Search (SerpAPI Tool Node)**  
    - Type: Langchain Tool node for SerpAPI  
    - Role: Provides real-time Google Search results to AI Agent for fresh information retrieval.  
    - Configuration: Uses default SerpAPI options.  
    - Credentials: SerpAPI API key.  
    - Inputs: ai_tool input from AI Agent.  
    - Outputs: Search results back to AI Agent.  
    - Failure modes: API quota limits, incorrect API key, network errors.

  - **Conversation Memory (Memory Buffer Window Node)**  
    - Type: Langchain Memory Node  
    - Role: Maintains a sliding window of the last 20 messages for conversational context.  
    - Configuration:  
      - Session key: User phone number (`={{ $('WhatsApp Trigger').item.json.messages[0].from }}`) used to maintain distinct conversations per user.  
      - Context window length: 20 messages.  
    - Inputs: ai_memory input from AI Agent.  
    - Outputs: Conversational context to AI Agent.  
    - Failure modes: Missing session key, memory overflow if window size is too large for system resources.

  - **Sticky Note3 (Meet Your AI Agent)**  
    - Describes the AI agent’s capabilities and integration with Groq LLaMA 4 and SerpAPI for real-time responses and context retention.

---

#### 2.4 Output Delivery

- **Overview:**  
  Sends the AI-generated response back to the user via WhatsApp.

- **Nodes Involved:**  
  - Send WhatsApp Reply

- **Node Details:**

  - **Send WhatsApp Reply**  
    - Type: WhatsApp Node (Send Message)  
    - Role: Sends text message replies to the user’s WhatsApp number.  
    - Configuration:  
      - Text body: `={{ $json.output }}` (output from AI Agent).  
      - Operation: Send.  
      - Phone Number ID: Configured WhatsApp Business phone number ID.  
      - Recipient Phone Number: Extracted from incoming message sender (`={{ $('WhatsApp Trigger').item.json.messages[0].from }}`).  
    - Credentials: WhatsApp API credentials (OAuth2).  
    - Inputs: From AI Agent.  
    - Outputs: None (terminal node).  
    - Failure modes: WhatsApp API quota limits, invalid phone number format, authentication failures.

---

#### 2.5 Documentation & Guidance (Sticky Notes)

- **Nodes Involved:**  
  - Main Workflow Overview  
  - Sticky Note (Node Guide: Input Switch)  
  - Sticky Note2 (Execution Logic)  
  - Sticky Note3 (Meet Your AI Agent)

- **Description:**  
  These nodes provide embedded, rich documentation directly inside the workflow editor to guide users on the workflow’s purpose, usage, and extension points.

---

### 3. Summary Table

| Node Name            | Node Type                      | Functional Role                     | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                                                                    |
|----------------------|--------------------------------|-----------------------------------|-----------------------|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger     | WhatsApp Trigger               | Receive incoming WhatsApp messages | None                  | Route Based on Input  |                                                                                                                                                |
| Route Based on Input | Switch                        | Route based on message type (text/image) | WhatsApp Trigger      | Prepare Text Prompt (Text output), none (Image output) | Node Guide: Input Switch explains routing logic and future image support.                                                                        |
| Prepare Text Prompt  | Set                          | Extract and prepare text message  | Route Based on Input   | AI Agent              | Execution Logic note clarifies text processing focus.                                                                                          |
| AI Agent             | Langchain Agent               | AI response generation with LLaMA 4, SerpAPI, and memory | Prepare Text Prompt, Groq Chat Model, Google Search, Conversation Memory | Send WhatsApp Reply      | Meet Your AI Agent note details AI personality, context memory, and search integration.                                                         |
| Groq Chat Model      | Langchain Language Model      | LLaMA 4 language model for text generation | AI Agent (ai_languageModel) | AI Agent              |                                                                                                                                                |
| Google Search        | Langchain Tool (SerpAPI)      | Real-time Google Search tool       | AI Agent (ai_tool)     | AI Agent              |                                                                                                                                                |
| Conversation Memory  | Langchain Memory Buffer Window| Stores last 20 messages for context | AI Agent (ai_memory)   | AI Agent              |                                                                                                                                                |
| Send WhatsApp Reply  | WhatsApp Node (Send Message)  | Sends AI-generated reply to user  | AI Agent              | None                  |                                                                                                                                                |
| Sticky Note          | Sticky Note                   | Documentation on input routing    | None                  | None                  | Node Guide: Input Switch                                                                                                                       |
| Sticky Note2         | Sticky Note                   | Documentation on execution logic  | None                  | None                  | Execution Logic                                                                                                                                |
| Sticky Note3         | Sticky Note                   | Documentation on AI Agent features | None                  | None                  | Meet Your AI Agent                                                                                                                             |
| Main Workflow Overview| Sticky Note                   | Overall workflow description and setup instructions | None                  | None                  | Main Workflow Overview                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node**  
   - Type: WhatsApp Trigger  
   - Configure webhook to listen for message updates (`updates`: messages)  
   - Set OAuth2 credentials for WhatsApp Business API or Twilio sandbox  
   - Position: Entry point

2. **Add Switch node "Route Based on Input"**  
   - Type: Switch (version 3.2)  
   - Add two outputs: "Image" and "Text"  
   - Condition for "Image": Check if `messages[0].image` exists (object exists)  
   - Condition for "Text": Check if `messages[0].text` exists (object exists)  
   - Connect WhatsApp Trigger main output to this node

3. **Create Set node "Prepare Text Prompt"**  
   - Type: Set (version 3.4)  
   - Add field `text` (string) with value expression `={{ $json.messages[0].text.body }}`  
   - Connect "Text" output of Switch node to this node

4. **Add Langchain Agent node "AI Agent"**  
   - Type: Langchain Agent (version 2.1)  
   - Set input text: `={{ $json.text }}`  
   - Configure system message prompt:  
     ```
     You are Seventeen, a friendly and helpful AI assistant created by Manav.  
     You speak in casual AAVE (American tone) with a light sarcastic vibe.

     Alternate your greetings (e.g., “yo, what’s poppin’?”, “hey, what’s goin’ on?”) so replies don’t feel repetitive.  
     You are currently talking to {{ $('WhatsApp Trigger').item.json.contacts[0].profile.name }}.  
     The current date and time is {{ $now.toISO() }}.

     Avoid writing “G’” at the start of sentences. Keep responses casual but clear.
     ```
   - Prompt type: define

5. **Add Langchain Language Model node "Groq Chat Model"**  
   - Type: Langchain LLM (version 1)  
   - Model: `meta-llama/llama-4-scout-17b-16e-instruct`  
   - Attach Groq API credentials  
   - Connect output to AI Agent’s `ai_languageModel` input

6. **Add Langchain Tool node "Google Search"**  
   - Type: Langchain Tool SerpApi (version 1)  
   - Attach SerpAPI credentials  
   - Connect output to AI Agent’s `ai_tool` input

7. **Add Langchain Memory node "Conversation Memory"**  
   - Type: Memory Buffer Window (version 1.3)  
   - Session key: `={{ $('WhatsApp Trigger').item.json.messages[0].from }}` (unique user identifier)  
   - Context window length: 20 messages  
   - Connect output to AI Agent’s `ai_memory` input

8. **Connect "Prepare Text Prompt" output to AI Agent main input**

9. **Create WhatsApp Node "Send WhatsApp Reply"**  
   - Type: WhatsApp (Send Message)  
   - Operation: Send  
   - Text Body: `={{ $json.output }}` (AI Agent reply)  
   - Phone Number ID: Your WhatsApp Business number ID (e.g., "768049963047541")  
   - Recipient Phone Number: `={{ $('WhatsApp Trigger').item.json.messages[0].from }}`  
   - Attach WhatsApp API credentials  
   - Connect AI Agent main output to this node

10. **Add Sticky Notes for documentation** (optional but recommended)  
    - Add informational sticky notes as per the original workflow to guide users on routing, execution, and AI capabilities.

11. **Activate and test the workflow**  
    - Link WhatsApp webhook URL to your WhatsApp Business API or sandbox  
    - Ensure Groq and SerpAPI credentials are valid and have sufficient quota  
    - Send test WhatsApp text messages and verify AI replies

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                                                                                                |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow integrates Groq’s LLaMA 4 model for fast, high-quality AI responses combined with SerpAPI to provide real-time Google Search insights, ensuring up-to-date answers on WhatsApp.                                      | Core workflow purpose                                                                                                                                                                         |
| Requires n8n with support for Langchain Agent node (version 2.1) and related Langchain nodes.                                                                                                                                   | n8n version compatibility                                                                                                                                                                    |
| Setup requires API keys for Groq and SerpAPI, and either WhatsApp Business API or Twilio sandbox for WhatsApp messaging.                                                                                                         | Credential requirements                                                                                                                                                                      |
| Future extensions may include image and audio message processing by expanding the "Route Based on Input" switch and adding corresponding handlers.                                                                               | Extensibility note                                                                                                                                                                           |
| The AI persona “Seventeen” uses casual American English with a light sarcastic tone, which can be customized in the AI Agent system message prompt.                                                                              | Customization tip                                                                                                                                                                            |
| Embedded sticky notes provide helpful documentation directly in the workflow for ease of use and onboarding new users.                                                                                                           | Workflow documentation best practice                                                                                                                                                        |
| Groq API details and model info: [Groq LLaMA 4 Models](https://www.groq.com/llama-4) (example link, verify with your provider)                                                                                                  | External reference (verify link validity)                                                                                                                                                   |
| SerpAPI documentation and usage: [https://serpapi.com/](https://serpapi.com/)                                                                                                                                                      | External reference                                                                                                                                                                           |
| WhatsApp Business API documentation: [https://developers.facebook.com/docs/whatsapp](https://developers.facebook.com/docs/whatsapp)                                                                                              | External reference                                                                                                                                                                           |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.