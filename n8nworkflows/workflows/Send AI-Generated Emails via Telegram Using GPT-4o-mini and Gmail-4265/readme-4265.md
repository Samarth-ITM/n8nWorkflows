Send AI-Generated Emails via Telegram Using GPT-4o-mini and Gmail

https://n8nworkflows.xyz/workflows/send-ai-generated-emails-via-telegram-using-gpt-4o-mini-and-gmail-4265


# Send AI-Generated Emails via Telegram Using GPT-4o-mini and Gmail

---

### 1. Workflow Overview

This workflow automates email generation and sending via Gmail, triggered by incoming Telegram messages, using an AI agent powered by the GPT-4o-mini model. It is designed to receive user messages on Telegram, process them through an AI language model configured as an email agent, compose emails with dynamic subject, recipient, and message content, and send them via Gmail. The AI agent also responds back to the Telegram user with the generated output to maintain conversational context.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages to trigger the workflow.
- **1.2 AI Processing:** Uses Langchain AI agent leveraging GPT-4o-mini to interpret messages and generate email content.
- **1.3 Memory Management:** Maintains session-based conversational context for the AI agent.
- **1.4 Email Dispatch:** Sends the AI-generated email using Gmail.
- **1.5 Telegram Response:** Sends the AI agent’s response back to the user on Telegram.
- **1.6 Documentation:** Sticky notes providing contextual information and comments.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives Telegram messages that trigger the workflow execution.
- **Nodes Involved:**  
  - Telegram Trigger
  - Sticky Note (Telegram Message Received Trigger Node)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram updates  
    - Configuration: Listens specifically for "message" updates to capture incoming user messages.  
    - Expressions: None beyond the update type.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "AI Agent" node main input.  
    - Edge cases: Telegram API rate limits, message format variations, or unsupported message types could cause missed triggers or errors.  
    - Version-specific: Uses version 1.2, compatible with current Telegram node specifications.  

  - **Sticky Note (Telegram Message Received Trigger Node)**  
    - Type: Comment / documentation node  
    - Purpose: Describes the role of the Telegram Trigger node for clarity in workflow visualization.  

#### 2.2 AI Processing

- **Overview:** Processes the incoming Telegram message text with an AI agent powered by GPT-4o-mini to generate email components (To, Subject, Message) and conversational output.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  
  - Sticky Note (AI Agent with Email Tool)

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain AI Agent node  
    - Role: Receives text input from Telegram, applies AI logic to interpret and generate structured email content and responses.  
    - Configuration:  
      - Receives input text dynamically from Telegram message text (`{{$json.message.text}}`).  
      - Uses "define" prompt type for detailed AI instructions.  
      - Connects to AI language model (OpenAI Chat Model) for generation.  
      - Connects to a memory buffer node (Simple Memory) to maintain session context keyed by chat ID.  
      - Connects output to both Gmail (as ai_tool) and Telegram (main output).  
    - Expressions:  
      - Input text: `={{ $json.message.text }}`  
      - Session key for memory: `={{ $json.message.chat.id }}`  
    - Inputs: From Telegram Trigger (main input), OpenAI Chat Model (ai_languageModel), Simple Memory (ai_memory)  
    - Outputs: To Gmail (ai_tool), Telegram (main)  
    - Edge cases:  
      - AI model response errors or timeouts.  
      - Invalid or ambiguous input text from Telegram.  
      - Memory buffer overflow or session key mismatches.  
    - Version-specific: Version 1.9, compatible with Langchain agent node enhancements.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node  
    - Role: Provides AI language model responses using GPT-4o-mini.  
    - Configuration:  
      - Model: gpt-4o-mini selected from list, cached for efficiency.  
      - No additional options enabled.  
    - Inputs: From AI Agent (ai_languageModel)  
    - Outputs: To AI Agent (ai_languageModel)  
    - Edge cases: API authentication failures, model rate limits, network timeouts.  
    - Version-specific: Version 1.2, supporting GPT-4o-mini model integration.

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window node  
    - Role: Maintains conversational session memory for AI agent.  
    - Configuration:  
      - Session key based on Telegram chat ID: `={{ $json.message.chat.id }}`  
      - Session ID type set to "customKey" for proper session isolation.  
    - Inputs: From AI Agent (ai_memory)  
    - Outputs: To AI Agent (ai_memory)  
    - Edge cases: Large conversation history causing memory overflow or stale context.  
    - Version-specific: Version 1.3.

  - **Sticky Note (AI Agent with Email Tool)**  
    - Type: Comment node describing AI agent role and integration with email tool.

#### 2.3 Email Dispatch

- **Overview:** Sends the generated email content via Gmail using data populated by the AI agent.
- **Nodes Involved:**  
  - Gmail  
 
- **Node Details:**

  - **Gmail**  
    - Type: Gmail Send Email node  
    - Role: Sends an email using AI-generated recipient, subject, and message text.  
    - Configuration:  
      - Recipient (`sendTo`): extracted dynamically from AI agent output field "To".  
      - Subject: from AI agent output "Subject".  
      - Message body: from AI agent output "Message".  
      - Email type set to plain text.  
      - No additional sending options configured.  
    - Expressions:  
      - `={{ $fromAI('To', '', 'string') }}`  
      - `={{ $fromAI('Subject', '', 'string') }}`  
      - `={{ $fromAI('Message', '', 'string') }}`  
    - Inputs: From AI Agent (ai_tool)  
    - Outputs: None (terminal node)  
    - Edge cases:  
      - Gmail API authentication failures or expired OAuth tokens.  
      - Invalid email addresses or malformed email content.  
      - API rate limits or quota exhaustion.  
    - Version-specific: Version 2.1.

#### 2.4 Telegram Response

- **Overview:** Sends the AI agent’s conversational output back to the Telegram user, enabling real-time interaction.
- **Nodes Involved:**  
  - Telegram  
  - Sticky Note (Telegram Send Message Action Node)

- **Node Details:**

  - **Telegram**  
    - Type: Telegram Send Message node  
    - Role: Sends text message back to the originating Telegram chat ID.  
    - Configuration:  
      - Text message content taken from AI agent output field `output`.  
      - Chat ID dynamically extracted from Telegram Trigger node input (`$('Telegram Trigger').item.json.message.chat.id`).  
      - Appending attribution disabled.  
    - Expressions:  
      - Text: `={{ $json.output }}`  
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Inputs: From AI Agent (main output)  
    - Outputs: None (terminal node)  
    - Edge cases: Telegram API rate limits, chat ID errors, or message format restrictions.  
    - Version-specific: Version 1.2.

  - **Sticky Note (Telegram Send Message Action Node)**  
    - Type: Comment node clarifying the purpose of the Telegram send message node.

#### 2.5 Documentation

- **Overview:** Provides contextual information and workflow explanation for maintainers and users.
- **Nodes Involved:**  
  - Sticky Note (General workflow description)

- **Node Details:**

  - **Sticky Note**  
    - Content:  
      - Notes that this is a simple email agent connected to Telegram on message trigger.  
      - Highlights Gmail as the email sending tool.  
      - Mentions that Telegram message output is enabled to allow AI interaction.  
    - Position: Visible on workflow canvas for quick reference.

---

### 3. Summary Table

| Node Name         | Node Type                         | Functional Role                       | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                     |
|-------------------|----------------------------------|------------------------------------|------------------------|-----------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger   | n8n-nodes-base.telegramTrigger   | Entry point, receives Telegram messages  | None                   | AI Agent              | Telegram Message Received Trigger Node                                                         |
| AI Agent          | @n8n/n8n-nodes-langchain.agent   | Processes text input via AI, generates email content and response | Telegram Trigger, OpenAI Chat Model, Simple Memory | Gmail, Telegram          | AI Agent with Email Tool                                                                       |
| OpenAI Chat Model  | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini AI language model responses | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) |                                                                                                |
| Simple Memory     | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational session memory | AI Agent (ai_memory)    | AI Agent (ai_memory)   |                                                                                                |
| Gmail             | n8n-nodes-base.gmailTool          | Sends email with AI-generated content  | AI Agent (ai_tool)      | None                  |                                                                                                |
| Telegram          | n8n-nodes-base.telegram           | Sends AI response message back to Telegram user | AI Agent (main)          | None                  | Telegram Send Message Action Node                                                              |
| Sticky Note       | n8n-nodes-base.stickyNote         | Documentation / comments             | None                   | None                  | This is a simple workflow for an Email Agent connected to Telegram on message trigger. Gmail is connected as an Email Sending Tool. Telegram send message output is enabled to allow AI Agent to interact with user. |
| Sticky Note (Telegram Message Received Trigger Node) | n8n-nodes-base.stickyNote | Documentation                      | None                   | None                  | Telegram Message Received Trigger Node                                                         |
| Sticky Note1 (AI Agent with Email Tool) | n8n-nodes-base.stickyNote | Documentation                      | None                   | None                  | AI Agent with Email Tool                                                                       |
| Sticky Note2 (Telegram Send Message Action Node) | n8n-nodes-base.stickyNote | Documentation                      | None                   | None                  | Telegram Send Message Action Node                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Telegram Trigger node**  
   - Type: `Telegram Trigger`  
   - Configure to listen for update type: `message` only.  
   - No additional fields needed.  
   - This node will serve as the workflow entry point.

2. **Add OpenAI Chat Model node**  
   - Type: `Langchain OpenAI Chat Model`  
   - Select model: `gpt-4o-mini` from the list.  
   - Leave options default (no extra parameters).  
   - This node will serve as the language model provider.

3. **Add Simple Memory node**  
   - Type: `Langchain Memory Buffer Window`  
   - Set `sessionKey` to: `{{$json.message.chat.id}}` to isolate memory per Telegram user.  
   - Set `sessionIdType` to `customKey`.

4. **Add AI Agent node**  
   - Type: `Langchain Agent`  
   - Set `text` input parameter to: `{{$json.message.text}}` (dynamic expression from Telegram trigger message).  
   - Set `promptType` to `define`.  
   - Connect the AI Agent inputs:  
     - Main input from Telegram Trigger main output.  
     - `ai_languageModel` input from OpenAI Chat Model output.  
     - `ai_memory` input from Simple Memory output.  
   - Configure AI Agent outputs:  
     - `ai_tool` output connected to Gmail node (to be created next).  
     - Main output connected to Telegram node (to be created next).

5. **Add Gmail node**  
   - Type: `Gmail` (Send Email)  
   - Configure parameters using AI Agent output:  
     - `sendTo` set to: `{{$fromAI('To', '', 'string')}}`  
     - `subject` set to: `{{$fromAI('Subject', '', 'string')}}`  
     - `message` set to: `{{$fromAI('Message', '', 'string')}}`  
     - `emailType` set to: `text` (plain text email).  
   - Connect Gmail node input from AI Agent’s `ai_tool` output.

6. **Add Telegram node (Send Message)**  
   - Type: `Telegram`  
   - Configure parameters:  
     - `text` set to: `{{$json.output}}` (the AI Agent’s output text).  
     - `chatId` set to: `{{$('Telegram Trigger').item.json.message.chat.id}}` (to send reply to the correct user).  
     - Disable `appendAttribution`.  
   - Connect Telegram node input from AI Agent main output.

7. **Add Sticky Notes for Documentation** (optional but recommended)  
   - Add descriptive sticky notes near each logical block to clarify functionality:  
     - Near Telegram Trigger: "Telegram Message Received Trigger Node"  
     - Near AI Agent and related nodes: "AI Agent with Email Tool"  
     - Near Telegram send node: "Telegram Send Message Action Node"  
     - General note describing overall workflow purpose and integrations.

8. **Credential Setup**  
   - For Telegram nodes, configure Telegram credentials with appropriate Bot token.  
   - For OpenAI Chat Model, configure OpenAI API credentials with valid API key.  
   - For Gmail node, set up OAuth2 credentials with permissions to send emails. Ensure refresh tokens are valid.  

9. **Validation and Testing**  
   - Test the Telegram trigger by sending a message to the bot.  
   - Verify AI agent processes message and outputs structured email content.  
   - Confirm email is sent via Gmail and that Telegram receives the AI agent’s reply message.  
   - Monitor for errors such as authentication failures, invalid email addresses, or rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow enables dynamic email sending initiated by Telegram messages, leveraging GPT-4o-mini AI model through Langchain. | Workflow description embedded in Sticky Note node on canvas.                                      |
| Gmail node requires OAuth2 credentials with "send email" scope properly configured for seamless email delivery.               | Gmail API documentation: https://developers.google.com/gmail/api/guides/sending                     |
| Telegram Bot API token must have permissions to read messages and send messages to users.                                    | Telegram Bot API docs: https://core.telegram.org/bots/api                                          |
| GPT-4o-mini is a lightweight GPT-4 variant optimized for cost and speed; ensure API quota and usage limits are monitored.    | OpenAI GPT-4o-mini details: https://openai.com/research/gpt-4o-mini                                |

---

**Disclaimer:** The provided content derives solely from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements. All manipulated data is legal and public.