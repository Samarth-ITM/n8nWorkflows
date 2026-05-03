Build a Telegram Mental Health Support Bot with GPT-4o

https://n8nworkflows.xyz/workflows/build-a-telegram-mental-health-support-bot-with-gpt-4o-6556


# Build a Telegram Mental Health Support Bot with GPT-4o

### 1. Workflow Overview

This workflow implements a Telegram chatbot designed to provide mental health support using the GPT-4o AI model. It listens for incoming Telegram messages, categorizes them based on specific user tags (#vent, #insight, #cope), generates empathetic and context-aware responses using AI, and sends replies back to the user. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages and initiates the workflow.
- **1.2 User Intent Routing:** Determines the type of support requested by inspecting message prefixes.
- **1.3 Prompt Preparation:** Selects and prepares the AI prompt corresponding to the user's tag.
- **1.4 AI Response Generation:** Uses the GPT-4o model to generate a personalized response.
- **1.5 Messaging Back to User:** Sends the AI-generated reply back to the Telegram chat.
- **1.6 User Experience Enhancement:** Shows a “typing…” indicator while processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new incoming messages from Telegram users and triggers the workflow.

- **Nodes Involved:**  
  - Start: Receive Message on Telegram

- **Node Details:**  
  - **Start: Receive Message on Telegram**  
    - Type: Telegram Trigger  
    - Role: Listens for Telegram updates specifically for new messages.  
    - Configuration:  
      - Listens to "message" updates only.  
      - Uses a Telegram Bot API credential (configured via BotFather token).  
    - Inputs: None (trigger node).  
    - Outputs: Emits incoming message data including chat ID and message text.  
    - Edge Cases:  
      - Possible webhook registration error or credential issues.  
      - Message types other than text (not filtered here, but downstream routing expects text).  
    - Version: 1.1  

---

#### 2.2 User Experience Enhancement

- **Overview:**  
  Provides immediate feedback to the user by showing a typing indicator while the bot processes the request.

- **Nodes Involved:**  
  - Show Typing Indicator

- **Node Details:**  
  - **Show Typing Indicator**  
    - Type: Telegram Node  
    - Role: Sends a "typing…" chat action to Telegram to indicate the bot is processing.  
    - Configuration:  
      - Uses the chat ID from the incoming message to show typing in the correct chat.  
      - Operation set to "sendChatAction".  
      - Uses the same Telegram credential as the trigger.  
    - Inputs: From "Start: Receive Message on Telegram".  
    - Outputs: Passes the data forward unaltered.  
    - Edge Cases:  
      - Telegram API rate limits or network errors.  
    - Version: 1.2  

---

#### 2.3 User Intent Routing

- **Overview:**  
  Routes the conversation flow based on message content prefixes to determine the user’s intent: venting, insight seeking, coping strategies, or general support.

- **Nodes Involved:**  
  - Route by Input Type  
  - Sticky Note1 (annotation)

- **Node Details:**  
  - **Route by Input Type**  
    - Type: Switch Node  
    - Role: Examines the beginning of the text message to detect one of the tags: `#vent`, `#insight`, `#cope`. If none match, routes to a default case.  
    - Configuration:  
      - Four outputs named "Vent", "Insight", "Cope", and "Any other".  
      - Each output checks if the message text starts with the respective tag using a strict, case-sensitive, string "startsWith" condition.  
    - Inputs: From "Show Typing Indicator".  
    - Outputs: Routes to one of the four prompt-setting nodes.  
    - Edge Cases:  
      - Messages without text or empty text might fall into "Any other" catch-all branch.  
      - Case sensitivity means tags must be exactly lowercase as specified.  
    - Version: 3.2  
  - **Sticky Note1**  
    - Function: Explains the logic of the routing based on message prefixes.  

---

#### 2.4 Prompt Preparation

- **Overview:**  
  Based on the routing, selects and sets the AI prompt tailored to the user's intent to guide GPT-4o’s response style and content.

- **Nodes Involved:**  
  - Vent prompt  
  - Insight prompt  
  - Cope prompt  
  - Main prompt for all other messages  
  - Sticky Note3 (annotation)

- **Node Details:**  
  - **Vent prompt**  
    - Type: Set Node  
    - Role: Sets a JSON object with a compassionate therapist prompt encouraging empathetic validation and reflective listening without offering unsolicited solutions.  
    - Configuration: Raw JSON with prompt text.  
    - Inputs: From "Route by Input Type" → Vent output.  
    - Outputs: Passes the prompt JSON to the AI node.  
    - Edge Cases: None specific.  
    - Version: 3.4  

  - **Insight prompt**  
    - Type: Set Node  
    - Role: Sets a prompt instructing the AI to analyze the user’s message for psychological insights and inner patterns.  
    - Inputs: From "Route by Input Type" → Insight output.  
    - Version: 3.4  

  - **Cope prompt**  
    - Type: Set Node  
    - Role: Sets a prompt directing the AI to suggest 1–2 simple coping strategies in a warm and practical tone.  
    - Inputs: From "Route by Input Type" → Cope output.  
    - Version: 3.4  

  - **Main prompt for all other messages**  
    - Type: Set Node  
    - Role: Provides a general compassionate AI assistant prompt for messages without tags, interpreting user needs gently and responding warmly.  
    - Inputs: From "Route by Input Type" → Any other output.  
    - Version: 3.4  

  - **Sticky Note3**  
    - Content: Notes that this block chooses the prompt from the given options based on routing.  

---

#### 2.5 AI Response Generation

- **Overview:**  
  Sends the prepared prompt and the user’s original message to the GPT-4o AI model for generating a personalized mental health supportive response.

- **Nodes Involved:**  
  - Generate personalised answer  
  - Sticky Note4 (annotation)

- **Node Details:**  
  - **Generate personalised answer**  
    - Type: AI/ML API Node (n8n-nodes-aimlapi)  
    - Role: Calls the OpenAI GPT-4o model with the combined prompt and user message text.  
    - Configuration:  
      - Model: `openai/gpt-4o`  
      - Prompt: Combines the prompt set in previous node plus the user's message text dynamically using expressions.  
      - Uses an AI/ML API credential containing the OpenAI key.  
    - Inputs: From any of the prompt-setting nodes.  
    - Outputs: Contains the AI-generated content in `json.content`.  
    - Edge Cases:  
      - API quota limits, network errors, or invalid API keys.  
      - Possible prompt formatting issues if expressions fail.  
    - Version: 1  

  - **Sticky Note4**  
    - Content: Describes the node’s function to generate and send the answer.

---

#### 2.6 Messaging Back to User

- **Overview:**  
  Sends the AI-generated reply text back to the user’s Telegram chat.

- **Nodes Involved:**  
  - Send message to Telegram

- **Node Details:**  
  - **Send message to Telegram**  
    - Type: Telegram Node  
    - Role: Sends a message to a specific Telegram chat ID.  
    - Configuration:  
      - Text content is dynamically set from AI node output `json.content`.  
      - Chat ID dynamically taken from the original incoming message.  
      - No appended attribution to the message.  
      - Uses the same Telegram credential as the trigger node.  
    - Inputs: From “Generate personalised answer”.  
    - Outputs: None (end node).  
    - Edge Cases:  
      - Telegram API errors such as rate limits or invalid chat IDs.  
    - Version: 1.2  

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                           | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                  |
|------------------------------|-------------------------------|-----------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| Start: Receive Message on Telegram | Telegram Trigger              | Listens for incoming Telegram messages  | -                             | Show Typing Indicator          |                                                                                                              |
| Show Typing Indicator          | Telegram                      | Shows “typing…” indicator in chat       | Start: Receive Message on Telegram | Route by Input Type            |                                                                                                              |
| Route by Input Type            | Switch                       | Routes messages by tag prefix             | Show Typing Indicator          | Vent prompt, Insight prompt, Cope prompt, Main prompt for all other messages | Checks message.text for #vent, #insight, #cope, or falls back                                                  |
| Vent prompt                   | Set                          | Sets compassionate therapist prompt      | Route by Input Type (Vent)     | Generate personalised answer   | Chooses from given prompts                                                                                    |
| Insight prompt                | Set                          | Sets insightful psychologist prompt      | Route by Input Type (Insight)  | Generate personalised answer   | Chooses from given prompts                                                                                    |
| Cope prompt                   | Set                          | Sets coping strategies prompt             | Route by Input Type (Cope)     | Generate personalised answer   | Chooses from given prompts                                                                                    |
| Main prompt for all other messages | Set                          | Sets general compassionate AI prompt     | Route by Input Type (Any other) | Generate personalised answer   | Chooses from given prompts                                                                                    |
| Generate personalised answer  | AI/ML API (AimlApi)           | Generates AI response using GPT-4o       | Vent prompt, Insight prompt, Cope prompt, Main prompt for all other messages | Send message to Telegram       | Generates and sends answer to Telegram                                                                       |
| Send message to Telegram       | Telegram                      | Sends AI-generated reply back to chat    | Generate personalised answer   | -                             |                                                                                                              |
| Sticky Note                   | Sticky Note                  | Quick start guide and instructions        | -                             | -                             | See Quick Start Guide content                                                                                 |
| Sticky Note1                  | Sticky Note                  | Annotation for routing logic               | -                             | -                             | Checks message.text for #vent, #insight, #cope, or falls back                                                |
| Sticky Note2                  | Sticky Note                  | Node overview summary                      | -                             | -                             | Overview of all main nodes                                                                                     |
| Sticky Note3                  | Sticky Note                  | Annotation for prompt selection block     | -                             | -                             | Chooses from given prompts                                                                                    |
| Sticky Note4                  | Sticky Note                  | Annotation for AI response generation and sending | -                             | -                             | Generates and sends answer to Telegram                                                                       |
| Sticky Note5                  | Sticky Note                  | Example of #vent usage                      | -                             | -                             | Example: `#vent I’m feeling anxious about my presentation` → empathetic response                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Node Type: Telegram Trigger  
   - Set "Updates" to listen to "message" only.  
   - Connect Telegram API credentials linked to your bot token (via BotFather).  
   - Position this as the start node.

2. **Create Telegram Node to Show Typing Indicator**  
   - Node Type: Telegram  
   - Operation: "sendChatAction"  
   - Set Chat ID to use expression: `{{$node["Start: Receive Message on Telegram"].json["message"]["chat"]["id"]}}`  
   - Use same Telegram credentials as trigger.  
   - Connect output of Telegram Trigger to this node.

3. **Add a Switch Node to Route by Input Type**  
   - Node Type: Switch  
   - Add 4 outputs: "Vent", "Insight", "Cope", "Any other".  
   - For each output, create a condition on the incoming message text:  
     - Vent: text startsWith `#vent` (case sensitive).  
     - Insight: text startsWith `#insight`.  
     - Cope: text startsWith `#cope`.  
     - Any other: default catch-all condition (message text exists).  
   - Use expression: `{{$node["Start: Receive Message on Telegram"].json.message.text}}` for conditions.  
   - Connect output of "Show Typing Indicator" node to this Switch node.

4. **Create Four Set Nodes for Prompts**  
   - Node Type: Set (mode: raw JSON)  
   - Each sets a JSON object with a key `prompt` containing the respective prompt string:  
     - Vent prompt: Compassionate therapist prompt about empathetic validation.  
     - Insight prompt: Analytical psychologist prompt offering insights.  
     - Cope prompt: Supportive prompt suggesting coping strategies.  
     - Main prompt for all other messages: General mental health assistant prompt.  
   - Connect each Switch output to the respective Set node.

5. **Create AI/ML API Node to Generate Answer**  
   - Node Type: n8n-nodes-aimlapi.aimlApi  
   - Model: `openai/gpt-4o`  
   - Prompt: Use expression to combine prompt from Set node and original message text:  
     ```
     {{$json.prompt}}

     Message:
     {{$node["Start: Receive Message on Telegram"].json.message.text}}
     ```  
   - Connect all four Set nodes to this AI node.  
   - Use AI/ML API credentials with your OpenAI key.

6. **Create Telegram Node to Send Message Back**  
   - Node Type: Telegram  
   - Operation: Send Message (default)  
   - Text: Use expression to output AI-generated content: `{{$json["content"]}}`  
   - Chat ID: Use expression: `{{$node["Start: Receive Message on Telegram"].json.message.chat.id}}`  
   - Disable append attribution.  
   - Use same Telegram credentials.  
   - Connect the AI node output to this node.

7. **Activate the Workflow**  
   - Ensure Telegram bot webhook is properly set.  
   - Test by sending messages to your Telegram bot with tags `#vent`, `#insight`, `#cope`, or no tag.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                      |
|------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Quick Start Guide embedded as sticky note in the workflow canvas.            | See workflow notes for stepwise usage instructions.                                                |
| Example usage: `#vent I’m feeling anxious about my presentation` triggers empathetic response. | See Sticky Note5 for example prompt and response style.                                           |
| Node overview and flow summary provided in Sticky Note2 for quick understanding. | Helpful for onboarding new users or developers.                                                   |
| Telegram Bot credentials must be generated via BotFather on Telegram.        | https://core.telegram.org/bots#6-botfather                                                          |
| OpenAI GPT-4o usage requires API key and subscription to OpenAI services.    | https://platform.openai.com/account/api-keys                                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.