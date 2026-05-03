Create a Slack Chatbot with GPT-4o for Automated Responses

https://n8nworkflows.xyz/workflows/create-a-slack-chatbot-with-gpt-4o-for-automated-responses-7734


# Create a Slack Chatbot with GPT-4o for Automated Responses

---

### 1. Workflow Overview

This workflow implements a **Slack chatbot powered by GPT-4o** to automate responses within Slack channels. It listens to messages triggered in Slack, sends user messages to an OpenAI GPT-4o language model for processing, and posts the AI-generated replies back into Slack. The use cases include customer support, e-commerce assistance, or any automated conversational agent within Slack.

The workflow is logically divided into the following blocks:

- **1.1 Slack Message Input Reception:** Capture user messages from Slack channels through a chat trigger.
- **1.2 User Message Echo to Slack:** Post the captured user message back to Slack for logging or transparency.
- **1.3 AI Processing with OpenAI GPT-4o:** Forward the user input to the GPT-4o model via Langchain nodes for generating a chatbot response.
- **1.4 Post AI Response to Slack:** Send the AI-generated response back to Slack in the same channel.
- **1.5 Response Formatting:** Format the AI response to a simplified JSON structure for consistent downstream consumption.

Sticky notes are included to provide setup instructions, including Slack app configuration and OAuth2 credential setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Slack Message Input Reception

- **Overview:**  
  This block listens for incoming messages in Slack to trigger the chatbot workflow.

- **Nodes Involved:**  
  - `Sample Chatbot`

- **Node Details:**  
  - **Sample Chatbot**  
    - *Type:* `@n8n/n8n-nodes-langchain.chatTrigger` (Langchain chat trigger node)  
    - *Role:* Entry point that listens for chat messages, triggering the workflow when a message is detected.  
    - *Configuration:* Default options enabled; webhook ID assigned for receiving Slack events.  
    - *Input/Output:* No input; outputs the captured chat input JSON object.  
    - *Version:* 1.3  
    - *Potential Failures:* Webhook connectivity issues, Slack event subscription misconfiguration, permission errors.  
    - *Notes:* This node requires Slack app permissions and proper webhook URL setup in Slack.

#### 2.2 User Message Echo to Slack

- **Overview:**  
  Posts the user’s original message back into Slack, possibly for transparency or logging.

- **Nodes Involved:**  
  - `Send User Message in Slack`

- **Node Details:**  
  - **Send User Message in Slack**  
    - *Type:* `n8n-nodes-base.slack`  
    - *Role:* Sends a message to Slack as the bot, echoing the user’s input.  
    - *Configuration:*  
      - Text template: `=*User:* {{ $json.chatInput }}` to prefix the message with "User:" and the input text.  
      - Sends as a user with fixed Slack User ID `U09ADJPB7QA`.  
      - Uses OAuth2 credentials for Slack API authentication.  
    - *Input:* Receives output from `Sample Chatbot` containing `chatInput`.  
    - *Output:* Passes data forward to `Sample Chat Agent`.  
    - *Version:* 2.3  
    - *Potential Failures:* Slack OAuth token expiry, insufficient scopes (`chat:write`), rate limits.

#### 2.3 AI Processing with OpenAI GPT-4o

- **Overview:**  
  Processes the user input through a GPT-4o powered Langchain agent to generate a chatbot reply.

- **Nodes Involved:**  
  - `OpenAI Chat Model8`  
  - `Sample Chat Agent`

- **Node Details:**  
  - **OpenAI Chat Model8**  
    - *Type:* `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - *Role:* Calls OpenAI’s GPT-4o model via Langchain integration.  
    - *Configuration:*  
      - Model selected: `gpt-4o`.  
      - No additional options configured.  
      - Uses OpenAI API credentials (`openAiApi`).  
    - *Input:* No direct input; connected as language model source for agent.  
    - *Output:* Provides language model results to `Sample Chat Agent`.  
    - *Version:* 1.2  
    - *Potential Failures:* API rate limits, invalid API key, network errors, model availability.  

  - **Sample Chat Agent**  
    - *Type:* `@n8n/n8n-nodes-langchain.agent`  
    - *Role:* Acts as a Langchain agent that inputs user chat and uses the OpenAI Chat Model to generate a response.  
    - *Configuration:*  
      - Text input bound to the last message’s `chatInput` from `Sample Chatbot`.  
      - System message prompt: "you are an ecommerce bot. help the user as if you were working for a mock store."  
      - Prompt type: Defined prompt.  
    - *Input:* Receives user message from `Send User Message in Slack` and language model from `OpenAI Chat Model8`.  
    - *Output:* Produces chatbot's generated output.  
    - *Version:* 2.2  
    - *Potential Failures:* Expression evaluation errors, prompt formatting issues, API errors propagated from `OpenAI Chat Model8`.

#### 2.4 Post AI Response to Slack

- **Overview:**  
  Sends the AI-generated response back into Slack as a message in the channel.

- **Nodes Involved:**  
  - `Send Agents Response in Slack`

- **Node Details:**  
  - **Send Agents Response in Slack**  
    - *Type:* `n8n-nodes-base.slack`  
    - *Role:* Posts the chatbot’s response to Slack channel.  
    - *Configuration:*  
      - Text template: `=*Chatbot:* {{ $json.output }}`.  
      - Posts as user with fixed Slack User ID `U09ADJPB7QA`.  
      - OAuth2 Slack credentials used.  
    - *Input:* Receives output from `Sample Chat Agent` node.  
    - *Output:* Passes data forward to `Format Response`.  
    - *Version:* 2.3  
    - *Potential Failures:* Same as previous Slack node (token issues, scopes, rate limits).

#### 2.5 Response Formatting

- **Overview:**  
  Formats the raw AI output into a clean JSON structure for consistency.

- **Nodes Involved:**  
  - `Format Response`

- **Node Details:**  
  - **Format Response**  
    - *Type:* `n8n-nodes-base.code` (JavaScript code node)  
    - *Role:* Extracts the `output` field from `Sample Chat Agent` and returns it as `text` in JSON.  
    - *Code:*  
      ```js
      return [{ json: { text:$('Sample Chat Agent').first().json.output } }];
      ```  
    - *Input:* Takes input from `Send Agents Response in Slack`.  
    - *Output:* Outputs a simplified JSON object with the chatbot’s text response.  
    - *Version:* 2  
    - *Potential Failures:* Runtime JavaScript errors if input data is missing or malformed.

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                      | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                                            |
|-----------------------------|-----------------------------------------|------------------------------------|--------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Sample Chatbot              | @n8n/n8n-nodes-langchain.chatTrigger    | Slack message input reception      | None                           | Send User Message in Slack   | # 💬 OpenAI Chat Agent with Slack Listening: connects chatbot to Slack, listens, sends to OpenAI, replies in Slack.      |
| Send User Message in Slack  | n8n-nodes-base.slack                     | Echo user message to Slack         | Sample Chatbot                 | Sample Chat Agent            | Setup instructions for Slack API and OAuth2 credentials. See Sticky Note20 and Sticky Note66 for details.                |
| OpenAI Chat Model8          | @n8n/n8n-nodes-langchain.lmChatOpenAi   | GPT-4o language model integration  | None (Langchain model source) | Sample Chat Agent            |                                                                                                                         |
| Sample Chat Agent           | @n8n/n8n-nodes-langchain.agent           | AI processing and chatbot response | Send User Message in Slack, OpenAI Chat Model8 | Send Agents Response in Slack |                                                                                                                         |
| Send Agents Response in Slack | n8n-nodes-base.slack                   | Post AI response to Slack          | Sample Chat Agent              | Format Response             |                                                                                                                         |
| Format Response             | n8n-nodes-base.code                      | Format chatbot response JSON       | Send Agents Response in Slack | None                        |                                                                                                                         |
| Sticky Note61               | n8n-nodes-base.stickyNote                | Documentation                      | None                           | None                        | # 💬 OpenAI Chat Agent with Slack Listening: This workflow connects a chatbot to Slack.                                   |
| Sticky Note20               | n8n-nodes-base.stickyNote                | Setup instructions                 | None                           | None                        | Setup instructions for Slack app creation and OAuth2 token configuration.                                                |
| Sticky Note66               | n8n-nodes-base.stickyNote                | Setup instructions                 | None                           | None                        | Details on connecting Slack API including required scopes and OAuth2 setup.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack app and configure permissions:**
   - Go to <https://api.slack.com/apps>.
   - Create a new app and add OAuth scopes:  
     `channels:history`, `groups:history`, `im:history`, `mpim:history`, `channels:read`, `groups:read`, `users:read`, and `chat:write`.
   - Install the app to your workspace.
   - Copy the **Bot User OAuth Token**.

2. **Create n8n Slack OAuth2 credentials:**
   - In n8n, go to Credentials → New → Slack OAuth2 API.
   - Paste the Bot User OAuth Token and save.

3. **Add `Sample Chatbot` node:**
   - Node type: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Set webhook ID (auto-generated).
   - No additional parameters needed.
   - This node listens for Slack chat messages and triggers the workflow.

4. **Add `Send User Message in Slack` node:**
   - Node type: `Slack` (n8n-nodes-base.slack).
   - Set parameters:  
     - Text: `=*User:* {{ $json.chatInput }}`  
     - User: Slack User ID (e.g., `U09ADJPB7QA`).  
     - Authentication: OAuth2 using your Slack credential.
   - Connect input from `Sample Chatbot`.

5. **Add `OpenAI Chat Model8` node:**
   - Node type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`.
   - Configure model to `gpt-4o`.
   - Assign OpenAI API credentials.
   - No direct input connection (used as language model for agent).

6. **Add `Sample Chat Agent` node:**
   - Node type: `@n8n/n8n-nodes-langchain.agent`.
   - Text input expression: `={{ $('Sample Chatbot').item.json.chatInput }}`
   - Prompt system message: `you are an ecommerce bot. help the user as if you were working for a mock store.`
   - Prompt type: Define.
   - Connect this agent's language model input to `OpenAI Chat Model8`.
   - Connect input from `Send User Message in Slack`.

7. **Add `Send Agents Response in Slack` node:**
   - Node type: `Slack`.
   - Text: `=*Chatbot:* {{ $json.output }}`
   - User: same Slack User ID as before.
   - OAuth2 Slack credential.
   - Connect input from `Sample Chat Agent`.

8. **Add `Format Response` node:**
   - Node type: `Code`.
   - JavaScript code:
     ```js
     return [{ json: { text:$('Sample Chat Agent').first().json.output } }];
     ```
   - Connect input from `Send Agents Response in Slack`.

9. **Connect node outputs:**
   - `Sample Chatbot` → `Send User Message in Slack`
   - `Send User Message in Slack` → `Sample Chat Agent`
   - `OpenAI Chat Model8` → `Sample Chat Agent` (as AI model)
   - `Sample Chat Agent` → `Send Agents Response in Slack`
   - `Send Agents Response in Slack` → `Format Response`

10. **Test the workflow:**
    - Ensure webhook URLs are registered with Slack events.
    - Verify OAuth2 token scopes and validity.
    - Send a message in Slack to trigger the bot.
    - Observe user message echo and AI response in Slack channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Setup instructions for Slack API including OAuth scopes and token creation, crucial for Slack integration.                                                   | <https://api.slack.com/apps>                                                                     |
| Contact for customization requests (e.g., filtering or reports): robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/          | Contact information included in Sticky Note20                                                   |
| Workflow uses Langchain nodes integrated in n8n to leverage OpenAI GPT-4o for conversational AI.                                                              | n8n Langchain nodes documentation                                                               |
| User and bot Slack User ID fixed to `U09ADJPB7QA` – must be updated according to your Slack workspace users or bot user.                                      | Slack user management                                                                             |
| GPT-4o is a premium OpenAI model; ensure your API key has access and monitor usage costs.                                                                      | OpenAI account management                                                                        |
| Slack rate limits and token expiration can cause message sending failures; implement error handling and token refresh if deploying at scale.                  | Slack API rate limits documentation                                                              |

---

**Disclaimer:** The provided text is extracted solely from an automated workflow built with n8n, an integration and automation tool. This processing strictly follows current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---