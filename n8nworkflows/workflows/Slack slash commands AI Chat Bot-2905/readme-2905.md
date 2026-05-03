Slack slash commands AI Chat Bot

https://n8nworkflows.xyz/workflows/slack-slash-commands-ai-chat-bot-2905


# Slack slash commands AI Chat Bot

### 1. Workflow Overview

This workflow implements an AI chatbot in Slack public channels that responds to slash commands, specifically the `/ask` command. When a user triggers the slash command in Slack, the request is sent to an n8n webhook. The workflow then branches based on the command received, processes the input text through an AI language model (OpenAI GPT-4o-mini), generates a response, and posts the answer back to the Slack channel.

Logical blocks:

- **1.1 Command Reception:** Receives slash command requests from Slack via a webhook.
- **1.2 Command Routing:** Uses a Switch node to route requests based on the slash command name.
- **1.3 AI Response Generation:** For the `/ask` command, sends the user’s question to an OpenAI chat model and generates a response.
- **1.4 Slack Response Posting:** Sends the AI-generated message back to the Slack channel where the command was invoked.

---

### 2. Block-by-Block Analysis

#### 2.1 Command Reception

- **Overview:**  
  This block receives incoming HTTP POST requests from Slack slash commands via a webhook URL. It captures the payload containing the command and message text.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook (HTTP POST listener)  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook ID path (`1bd05fcf-8286-491f-ae13-f0e6bff4aca6`)  
      - Response Code: 204 (No Content) to Slack, indicating receipt without additional content  
    - Key Expressions/Variables:  
      - Incoming Slack payload accessible via `$json.body` (e.g., `$json.body.command`, `$json.body.text`, `$json.body.channel_id`)  
    - Input/Output:  
      - Input: HTTP POST from Slack slash command  
      - Output: JSON payload forwarded to Switch node  
    - Edge Cases/Potential Failures:  
      - Slack request validation not explicitly handled (e.g., verification token or signing secret)  
      - If Slack sends malformed payload, downstream nodes may fail  
      - Timeout if Slack does not receive timely 204 response  
    - Version: 2

#### 2.2 Command Routing

- **Overview:**  
  Routes incoming requests based on the slash command invoked. Currently supports `/ask` and `/another` commands, branching accordingly.

- **Nodes Involved:**  
  - Switch

- **Node Details:**  
  - **Switch**  
    - Type: Switch node (conditional branching)  
    - Configuration:  
      - Rules check `$json.body.command` string equality:  
        - `/ask` → output "ask"  
        - `/another` → output "another" (not further processed in this workflow)  
    - Input/Output:  
      - Input: JSON from Webhook node  
      - Output: Branches to different nodes based on command  
    - Edge Cases/Potential Failures:  
      - If command is unrecognized, no branch is taken (no default branch configured)  
      - Case sensitivity enforced (`caseSensitive: true`)  
      - If `$json.body.command` is missing or malformed, may cause expression errors  
    - Version: 3.2

#### 2.3 AI Response Generation

- **Overview:**  
  Processes the user’s question text through an AI language model to generate a response message.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - OpenAI Chat Model

- **Node Details:**  
  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain node  
    - Configuration:  
      - Text input: `={{ $json.body.text }}` (the user’s question from Slack)  
      - Prompt Type: "define" (uses a predefined prompt template)  
      - Connected to OpenAI Chat Model as the language model backend  
    - Input/Output:  
      - Input: JSON from Switch node (only `/ask` branch)  
      - Output: JSON containing AI-generated text response, passed to Slack message node  
    - Edge Cases/Potential Failures:  
      - If `$json.body.text` is empty or missing, AI response may be irrelevant or error  
      - Prompt template errors or misconfiguration could cause failures  
      - Requires valid OpenAI credentials configured in the OpenAI Chat Model node  
    - Version: 1.5

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Configuration:  
      - Model: `gpt-4o-mini` (a GPT-4 variant)  
      - No additional options specified  
      - Connected as AI language model for Basic LLM Chain  
    - Input/Output:  
      - Input: Receives prompt from Basic LLM Chain  
      - Output: AI-generated completion passed back to Basic LLM Chain  
    - Edge Cases/Potential Failures:  
      - API authentication errors if OpenAI credentials are missing or invalid  
      - Rate limits or timeouts from OpenAI API  
      - Model-specific constraints or errors  
    - Version: 1.2

#### 2.4 Slack Response Posting

- **Overview:**  
  Sends the AI-generated response text back to the Slack channel where the slash command was invoked.

- **Nodes Involved:**  
  - Send a Message

- **Node Details:**  
  - **Send a Message**  
    - Type: Slack node (message sender)  
    - Configuration:  
      - Text: `={{ $json.text }}` (AI-generated response text)  
      - Channel: Selected by channel ID from webhook payload: `={{ $('Webhook').item.json.body.channel_id }}`  
      - Other options: `includeLinkToWorkflow` disabled (no workflow link in message)  
    - Input/Output:  
      - Input: AI response JSON from Basic LLM Chain  
      - Output: Slack API response (not further processed)  
    - Edge Cases/Potential Failures:  
      - Slack API authentication errors if Slack credentials are missing or invalid  
      - Channel ID invalid or bot not invited to channel → message send failure  
      - Rate limits or network errors  
    - Version: 2.3

---

### 3. Summary Table

| Node Name         | Node Type                      | Functional Role               | Input Node(s) | Output Node(s)     | Sticky Note                                                                                          |
|-------------------|--------------------------------|------------------------------|---------------|--------------------|----------------------------------------------------------------------------------------------------|
| Webhook           | Webhook (HTTP POST listener)   | Receives Slack slash commands | —             | Switch             | Copy the webhook URL, paste it into the Request URL of the Slack slash command, and complete creation. (Korean note also present) |
| Switch            | Switch (conditional branching) | Routes based on slash command | Webhook       | Basic LLM Chain    | Switch each slash command. (Korean note also present)                                              |
| Basic LLM Chain   | LangChain LLM Chain             | Generates AI response text    | Switch        | Send a Message     | Create AI Messages                                                                                  |
| OpenAI Chat Model | LangChain OpenAI Chat Model     | Provides AI language model    | — (linked to Basic LLM Chain) | Basic LLM Chain (AI model input) | —                                                                                                  |
| Send a Message    | Slack node                     | Sends message to Slack channel| Basic LLM Chain | —                  | Send a Slack Message                                                                                |
| Sticky Note       | Sticky Note                    | Instructional note            | —             | —                  | Command Trigger: Copy webhook URL to Slack slash command Request URL                               |
| Sticky Note1      | Sticky Note                    | Instructional note            | —             | —                  | Command Switch: Switch each slash command                                                          |
| Sticky Note2      | Sticky Note                    | Instructional note            | —             | —                  | Create AI Messages                                                                                  |
| Sticky Note3      | Sticky Note                    | Instructional note            | —             | —                  | Send a Slack Message                                                                                |
| Sticky Note8      | Sticky Note                    | Tutorial description (Korean) | —             | —                  | Slack slash command and channel message chatbot tutorial with YouTube link (Korean)                |
| Sticky Note10     | Sticky Note                    | Tutorial description (English)| —             | —                  | Create an AI chatbot with Slack slash commands tutorial with YouTube link                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Choose a unique path (e.g., `slack-command-webhook`)  
   - Response Code: 204 (No Content)  
   - Purpose: Receive Slack slash command requests.

2. **Create Switch Node**  
   - Type: Switch  
   - Add rules to check the incoming command:  
     - Expression: `{{$json.body.command}}`  
     - Rule 1: Equals `/ask` → output "ask"  
     - Rule 2: Equals `/another` → output "another" (optional, for future commands)  
   - Connect Webhook node output to Switch node input.

3. **Create OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: Select `gpt-4o-mini` or preferred GPT-4 variant  
   - Configure OpenAI credentials (API key) in n8n credentials manager.

4. **Create Basic LLM Chain Node**  
   - Type: LangChain LLM Chain  
   - Text input: Set to `={{ $json.body.text }}` (text from Slack command)  
   - Prompt Type: Use "define" or a custom prompt as needed  
   - Connect OpenAI Chat Model node as the AI language model for this chain.

5. **Create Slack Node to Send Message**  
   - Type: Slack  
   - Operation: Send Message  
   - Text: `={{ $json.text }}` (AI-generated response from Basic LLM Chain)  
   - Channel: Set to `={{ $('Webhook').item.json.body.channel_id }}` to reply in the same channel  
   - Configure Slack OAuth2 credentials with `chat:write` scope.

6. **Connect Nodes**  
   - Connect Switch node’s `/ask` output to Basic LLM Chain node input.  
   - Connect Basic LLM Chain node output to Slack Send Message node input.

7. **Slack App Setup**  
   - Create a Slack app in your workspace.  
   - Add `chat:write` permission under OAuth & Permissions > Scopes.  
   - Create a Slash Command (e.g., `/ask`) in Slack’s Slash Commands menu.  
   - Set the Request URL to the webhook URL generated by the Webhook node in n8n.  
   - Install the app to your workspace.

8. **Test the Workflow**  
   - Trigger the slash command `/ask` in a Slack channel.  
   - Confirm the AI chatbot responds with generated text.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                         |
|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow is demonstrated in a Korean YouTube video explaining the setup and usage in detail.                                  | https://www.youtube.com/watch?v=UpudYFCWaIM             |
| Slack app requires `chat:write` permission to post messages to channels.                                                           | Slack OAuth & Permissions > Scopes                      |
| The webhook URL must be copied exactly and set as the Request URL in Slack Slash Command configuration.                            | Slack Slash Commands menu                               |
| The workflow currently only processes the `/ask` command; other commands can be added in the Switch node for future expansion.     | Switch node configuration                               |
| Ensure OpenAI API credentials are configured in n8n for the AI nodes to function correctly.                                         | n8n Credentials manager                                 |
| Slack bot must be invited to the channels where it is expected to respond to slash commands.                                        | Slack channel management                                |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and maintaining the Slack slash commands AI chatbot workflow in n8n.