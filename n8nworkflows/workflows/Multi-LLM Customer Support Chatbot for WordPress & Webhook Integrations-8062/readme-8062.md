Multi-LLM Customer Support Chatbot for WordPress & Webhook Integrations

https://n8nworkflows.xyz/workflows/multi-llm-customer-support-chatbot-for-wordpress---webhook-integrations-8062


# Multi-LLM Customer Support Chatbot for WordPress & Webhook Integrations

### 1. Workflow Overview

This workflow implements a **Multi-LLM Customer Support Chatbot** designed for integration with WordPress and other live chat platforms via webhooks. It automates customer service and lead generation by handling user queries through AI-driven conversational agents. The chatbot maintains context, uses multiple Large Language Models (LLMs) for response generation, and supports conversational flow control, including graceful conversation termination.

**Target Use Cases:**

- Embedding an intelligent chatbot on WordPress or any live chat interface.
- Automating first-level customer support and lead qualification.
- Managing multi-turn conversations with memory and contextual awareness.
- Switching between multiple LLM providers (OpenAI GPT-4, Google Gemini, Anthropic Claude, OpenRouter, xAI Grok).
- Detecting conversation end signals and responding accordingly.

**Logical Blocks:**

- **1.1 Input Reception:** Accepts chat messages via webhook and extracts user input.
- **1.2 AI Processing and Memory:** Routes input through a customizable AI Agent that leverages conversational memory and integrates various LLMs.
- **1.3 Conversation Control:** Checks if the conversation should end based on AI output tags and sets appropriate flags.
- **1.4 Output Delivery:** Sends the AI-generated responses back to the live chat client.
- **1.5 Auxiliary Components:** Includes sticky notes for documentation and various LLM nodes for flexible AI model selection.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Receives incoming chat messages from the live chat frontend (e.g., WordPress plugin) through an HTTP webhook and prepares the chat input for AI processing.

- **Nodes Involved:**  
  - Website Chat Messages  
  - Convert Chat Text

- **Node Details:**

  - **Website Chat Messages**  
    - Type: Webhook (HTTP POST)  
    - Role: Entry point accepting POST requests at path `/demo-workflow`. Receives raw chat messages from the live chat interface.  
    - Configuration: Responds via response node mode, enabling synchronous responses.  
    - Inputs: HTTP POST request with JSON body containing `chatinput`.  
    - Outputs: Passes raw webhook data downstream.  
    - Edge Cases: Missing or malformed POST body, slow or failed webhook calls.

  - **Convert Chat Text**  
    - Type: Set  
    - Role: Extracts the user’s chat input from the webhook payload and stores it as `chatInput` for further use.  
    - Configuration: Sets variable `chatInput` to `{{$json["body"]["chatinput"]}}`.  
    - Inputs: Raw webhook data from "Website Chat Messages".  
    - Outputs: JSON with `chatInput` string.  
    - Edge Cases: If `chatinput` field is missing or empty, downstream nodes may fail or produce empty responses.

---

#### 1.2 AI Processing and Memory

- **Overview:**  
  Processes the user input through a custom AI Agent that integrates conversational memory and supports multiple LLMs. This block enriches the chat with context and applies brand and conversational guidelines.

- **Nodes Involved:**  
  - Forerunner™ AI Agent  
  - Simple Memory  
  - Think  
  - OpenRouter Chat Model  
  - OpenAI Chat Model  
  - Google Gemini Chat Model  
  - Anthropic Chat Model  
  - xAI Grok Chat Model

- **Node Details:**

  - **Forerunner™ AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Central conversational brain that processes input, applies system prompts, uses tools, and generates AI responses.  
    - Configuration:  
      - System message contains detailed brand voice, conversation rules, allowed topics, lead capture requirements, escalation instructions, compliance, and how to end conversations.  
      - Uses UK English spelling, concise and friendly tone, and practical advice.  
      - Embeds business logic like lead capture and escalation pathways.  
    - Inputs: Receives `chatInput` from "Convert Chat Text" and AI memory from "Simple Memory".  
    - Outputs: Generates an AI response including possible control tags like `[END_OF_CONVERSATION]`.  
    - Edge Cases: API quota limits, prompt injection risks, failure to follow system prompt instructions, memory synchronization issues.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversation history keyed by client IP (`x-real-ip`) to provide context across message exchanges.  
    - Configuration: Keeps a sliding window of the last 20 messages for context.  
    - Inputs: Chat messages and AI responses from the agent.  
    - Outputs: Supplies memory context to the AI Agent node.  
    - Edge Cases: Memory overflow, IP changes causing session breaks, stale or inconsistent context.

  - **Think**  
    - Type: LangChain Think Tool  
    - Role: Provides meta-processing or auxiliary reasoning capabilities to the AI Agent.  
    - Configuration: Includes a dynamic description with current UTC datetime, advising not to ask the user for time or date.  
    - Inputs/Outputs: Connected as an AI tool to the Forerunner™ AI Agent.  
    - Edge Cases: Latency or failure in tool invocation.

  - **OpenRouter Chat Model**  
    - Type: LangChain LLM Chat Model (OpenRouter)  
    - Role: Alternative LLM backend offering access to Google Gemini 2.5 flash model.  
    - Configuration: Uses credentials for OpenRouter API.  
    - Inputs: Accepts prompts from the Forerunner™ AI Agent.  
    - Outputs: Returns AI-generated completions.  
    - Edge Cases: API downtime, invalid credentials, rate limits.

  - **OpenAI Chat Model**  
    - Type: LangChain LLM Chat Model (OpenAI)  
    - Role: Provides access to GPT-4 model for response generation.  
    - Configuration: Uses OpenAI API credentials.  
    - Inputs: Can be configured for direct LLM calls or via AI Agent.  
    - Outputs: AI completions.  
    - Edge Cases: API key limits, network issues, model deprecation.

  - **Google Gemini Chat Model**  
    - Type: LangChain LLM Chat Model (Google PaLM API)  
    - Role: Supports Google Gemini 1.5 pro latest model for AI generation.  
    - Configuration: Uses Google Palm API credentials.  
    - Edge Cases: API quota, authentication failures.

  - **Anthropic Chat Model**  
    - Type: LangChain LLM Chat Model (Anthropic)  
    - Role: Uses Anthropic’s Claude 3.7 Sonnet model.  
    - Configuration: Uses Anthropic API credentials.  
    - Edge Cases: Rate limits, credential expiration.

  - **xAI Grok Chat Model**  
    - Type: LangChain LLM Chat Model (xAI)  
    - Role: Accesses xAI Grok chat model for AI responses.  
    - Configuration: Uses xAi API credentials.  
    - Edge Cases: API unavailability.

---

#### 1.3 Conversation Control

- **Overview:**  
  Determines whether the conversation should end by checking the AI response for an end-of-conversation tag. Splits the flow to either conclude or continue the chat.

- **Nodes Involved:**  
  - End Conversation?  
  - Yes - End  
  - No - Continue

- **Node Details:**

  - **End Conversation?**  
    - Type: If  
    - Role: Checks if the AI output string contains `[END_OF_CONVERSATION]`.  
    - Configuration: String containment condition on the AI response output field. Case sensitive and strict.  
    - Inputs: AI Agent output JSON containing the chat reply.  
    - Outputs:  
      - True branch: If tag found → "Yes - End" node.  
      - False branch: If tag absent → "No - Continue" node.  
    - Edge Cases: False positives if tag appears in normal text, tag missing causing endless chat.

  - **Yes - End**  
    - Type: Set  
    - Role: Cleans the AI response by removing the `[END_OF_CONVERSATION]` tag and marks the conversation as ended (`endOfConversation: true`).  
    - Configuration:  
      - Uses expression to replace tag with empty string and trim extra spaces.  
      - Sets a boolean flag `endOfConversation` to true.  
    - Outputs: Final cleaned response to be sent to client.  
    - Edge Cases: Tag not properly removed, flag missing.

  - **No - Continue**  
    - Type: Set  
    - Role: Passes the AI response unchanged for continuation of the chat.  
    - Configuration: Sets `reply` field equal to AI output string.  
    - Edge Cases: Missing or empty AI response.

---

#### 1.4 Output Delivery

- **Overview:**  
  Sends the AI-generated response (cleaned and flagged if conversation ended) back to the client’s live chat interface as an HTTP response to the original webhook call.

- **Nodes Involved:**  
  - Send Chat to Client

- **Node Details:**

  - **Send Chat to Client**  
    - Type: Respond to Webhook  
    - Role: Final node that returns the chatbot’s reply to the live chat frontend.  
    - Configuration: Default response node settings, sending JSON with the `reply` and optionally `endOfConversation` flags.  
    - Inputs: From both "Yes - End" and "No - Continue" nodes.  
    - Edge Cases: Response timeout, client disconnection.

---

#### 1.5 Auxiliary Components

- **Overview:**  
  Contains documentation and notes embedded within the workflow to assist users in understanding and customizing the chatbot.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note (Documentation)  
    - Role: Provides an extensive explanation of the workflow’s purpose, setup instructions, AI agent behavior, and conversation flow.  
    - Content Highlights:  
      - Describes the chatbot’s integration with WordPress and webhook chats.  
      - Explains how the AI Agent processes user input and maintains conversational context.  
      - Details the use of the `[END_OF_CONVERSATION]` tag to close chats.  
      - Encourages friendly, concise, non-technical responses.  
    - Position: Visually placed for easy reference.  
    - Edge Cases: None (documentation only).

---

### 3. Summary Table

| Node Name              | Node Type                        | Functional Role                                    | Input Node(s)          | Output Node(s)              | Sticky Note                                                                                                  |
|------------------------|---------------------------------|--------------------------------------------------|-----------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Website Chat Messages   | Webhook                         | Receives incoming chat messages from client      | —                     | Convert Chat Text            | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Convert Chat Text       | Set                             | Extracts user message from webhook payload       | Website Chat Messages  | Forerunner™ AI Agent        | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Forerunner™ AI Agent    | LangChain AI Agent              | Processes chat input with AI, applies system rules| Convert Chat Text, Simple Memory, Think, OpenRouter Chat Model | End Conversation?            | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Simple Memory          | LangChain Memory Buffer Window  | Maintains conversation history context            | —                     | Forerunner™ AI Agent        | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Think                  | LangChain Tool (Think)           | Provides auxiliary reasoning for AI Agent         | —                     | Forerunner™ AI Agent        | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| OpenRouter Chat Model   | LangChain LLM Chat Model         | Provides AI completions via OpenRouter API        | Forerunner™ AI Agent   | Forerunner™ AI Agent        | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| OpenAI Chat Model       | LangChain LLM Chat Model         | Provides AI completions via OpenAI GPT-4           | —                     | —                           | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Google Gemini Chat Model| LangChain LLM Chat Model         | Provides AI completions via Google Gemini          | —                     | —                           | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Anthropic Chat Model    | LangChain LLM Chat Model         | Provides AI completions via Anthropic Claude       | —                     | —                           | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| xAI Grok Chat Model    | LangChain LLM Chat Model         | Provides AI completions via xAI Grok                | —                     | —                           | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| End Conversation?      | If                              | Checks if AI response contains end-of-chat tag     | Forerunner™ AI Agent  | Yes - End, No - Continue     | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Yes - End              | Set                             | Cleans end tag and flags conversation as ended    | End Conversation? (True) | Send Chat to Client          | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| No - Continue          | Set                             | Passes AI response without modification            | End Conversation? (False) | Send Chat to Client          | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Send Chat to Client    | Respond to Webhook               | Returns chatbot response to live chat frontend     | Yes - End, No - Continue | —                           | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |
| Sticky Note            | Sticky Note                     | Documentation and overview of the workflow         | —                     | —                           | AI Chat Bot workflow for WordPress & Webhook Live Chats (full overview and setup instructions)              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Website Chat Messages`  
   - Type: Webhook (HTTP POST)  
   - Path: `demo-workflow`  
   - HTTP Method: POST  
   - Response Mode: `Response Node`  
   - Purpose: Accept incoming chat messages as JSON with property `chatinput`.

2. **Create Set Node**  
   - Name: `Convert Chat Text`  
   - Type: Set  
   - Add Assignment:  
     - Variable: `chatInput` (string)  
     - Value: Expression `{{$json["body"]["chatinput"]}}`  
   - Connect input from `Website Chat Messages`.

3. **Create LangChain Memory Node**  
   - Name: `Simple Memory`  
   - Type: LangChain Memory Buffer Window  
   - Session Key: `x-real-ip` (to track sessions by client IP)  
   - Context Window Length: 20 messages  
   - No inputs (standalone node).

4. **Create LangChain Think Tool Node**  
   - Name: `Think`  
   - Type: LangChain Tool (Think)  
   - Description Expression:  
     ```
     The current day and time is {{ new Date().toUTCString() }}, you don't need to ask for the time or date from the user.
     Ensure you have run the tool required for each query.
     ```  
   - No inputs (standalone node).

5. **Create LangChain LLM Chat Model Nodes for Each Provider:**  
   - **OpenRouter Chat Model**  
     - Type: LangChain LLM Chat Model  
     - Model: `google/gemini-2.5-flash`  
     - Set OpenRouter API credentials.  
   - **OpenAI Chat Model**  
     - Type: LangChain LLM Chat Model  
     - Model: `gpt-4`  
     - Set OpenAI API credentials.  
   - **Google Gemini Chat Model**  
     - Type: LangChain LLM Chat Model  
     - Model: `models/gemini-1.5-pro-latest`  
     - Set Google Palm API credentials.  
   - **Anthropic Chat Model**  
     - Type: LangChain LLM Chat Model  
     - Model: `claude-3-7-sonnet-20250219`  
     - Set Anthropic API credentials.  
   - **xAI Grok Chat Model**  
     - Type: LangChain LLM Chat Model  
     - Set xAi API credentials.

6. **Create LangChain AI Agent Node**  
   - Name: `Forerunner™ AI Agent`  
   - Type: LangChain AI Agent  
   - System Message: Paste the comprehensive brand voice, conversational guidelines, allowed topics, lead capture, escalation, compliance, accuracy, contact info, and conversation ending instructions as provided.  
   - Connect Inputs:  
     - From `Convert Chat Text` (main input)  
     - From `Simple Memory` (memory input)  
     - From `Think` (AI tool input)  
     - From `OpenRouter Chat Model` (AI language model input)  
   - Output: Connect to next logic node.

7. **Create If Node**  
   - Name: `End Conversation?`  
   - Type: If  
   - Condition: Check if AI Agent output field `output` contains string `[END_OF_CONVERSATION]` (case sensitive, strict).  
   - Connect input from `Forerunner™ AI Agent`.

8. **Create Set Node for Ending Conversation**  
   - Name: `Yes - End`  
   - Type: Set  
   - Assignments:  
     - `reply`: expression to replace `[END_OF_CONVERSATION]` with empty string and trim, e.g., `{{$json.output.replace('[END_OF_CONVERSATION]', '').trim()}}`  
     - `endOfConversation`: boolean `true`  
   - Connect input from `End Conversation?` True branch.

9. **Create Set Node for Continuing Conversation**  
   - Name: `No - Continue`  
   - Type: Set  
   - Assignments:  
     - `reply`: set to AI output string, e.g., `{{$json.output}}`  
   - Connect input from `End Conversation?` False branch.

10. **Create Respond to Webhook Node**  
    - Name: `Send Chat to Client`  
    - Type: Respond to Webhook  
    - Connect inputs from both `Yes - End` and `No - Continue`.  
    - Purpose: Returns the chatbot response JSON to the client.

11. **(Optional) Create Sticky Note Node**  
    - Name: `Sticky Note`  
    - Type: Sticky Note  
    - Add detailed workflow description content for internal documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow powers a versatile AI chatbot that can be integrated into any live chat interface, such as the free Forerunner™ AI Chat Bot for WordPress. It enables automated customer support and lead generation by handling user queries independently. Setup involves connecting your preferred LLM and live chat platform via webhooks. The AI Agent uses system messages to maintain brand voice and conversation guidelines. Conversation ends when `[END_OF_CONVERSATION]` tag is detected in AI output. | Embedded in Sticky Note node within the workflow for user reference.                                         |
| For brand voice and conversational guidelines, the AI Agent uses UK English spelling, concise, friendly language, and avoids jargon. It supports lead capture fields, escalation paths, and GDPR compliance. Conversations are retained for 12 months unless opted out.                                                                                                                                                                                                                                                                                                                                                             | System message in `Forerunner™ AI Agent` node.                                                               |
| Supported LLMs include OpenAI GPT-4, Google Gemini (via OpenRouter and Google API), Anthropic Claude, and xAI Grok, enabling flexible backend choice or fallback. Credentials must be configured for each LLM provider.                                                                                                                                                                                                                                                                                                                                                                                                        | Multiple LangChain LLM Chat Model nodes.                                                                     |
| To end a conversation properly, the AI should append the `[END_OF_CONVERSATION]` tag. The workflow detects this and cleans the response before sending it back. This prevents endless loops and signals the live chat interface to close the chat.                                                                                                                                                                                                                                                                                                                                                                           | Logic handled in `End Conversation?`, `Yes - End`, and `No - Continue` nodes.                                 |
| The workflow is designed for synchronous response delivery via webhook, enabling real-time chat interactions. The `Respond to Webhook` node ensures responses are sent back promptly.                                                                                                                                                                                                                                                                                                                                                                                                                                        | `Send Chat to Client` node functionality.                                                                    |
| For more information on n8n LangChain integrations and AI Agent setup, consult official n8n documentation and LangChain API references.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | External documentation (not linked here).                                                                    |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow. All data processed are legal and public, respecting current content policies with no illegal, offensive, or protected content.