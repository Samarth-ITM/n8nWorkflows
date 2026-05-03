Real Estate Lead Qualification Chatbot with GPT-4o-mini

https://n8nworkflows.xyz/workflows/real-estate-lead-qualification-chatbot-with-gpt-4o-mini-6162


# Real Estate Lead Qualification Chatbot with GPT-4o-mini

---

### 1. Workflow Overview

This workflow implements a **Real Estate Lead Qualification Chatbot** designed for a website integration, powered by GPT-4o-mini. Its main purpose is to receive user messages via a webhook, process them through an AI agent that acts as a pre-sales real estate assistant, and respond with personalized, trust-building messages aimed at qualifying leads and encouraging site visits.

**Target Use Cases:**
- Real estate businesses wanting an automated, intelligent chatbot to qualify incoming leads.
- Engaging visitors in a conversational manner using real estate sales frameworks (AIDA, BANT, SPIN, PAS).
- Handling multiple user sessions with memory retention.
- Responding in "Hinglish" (a mix of Hindi and English) with a friendly and confident tone.

The workflow logic is grouped into the following blocks:

- **1.1 Input Reception:** Handles inbound HTTP POST requests containing user messages.
- **1.2 Message Preparation:** Extracts and sets the user's message into a usable format.
- **1.3 AI Processing:** Uses LangChain AI agent with GPT-4o-mini model and session memory to generate relevant responses according to configured sales frameworks and persona.
- **1.4 Response Delivery:** Sends the AI-generated response back to the user via the webhook response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives incoming HTTP POST requests from website chatbot clients. This node is the entry point for user messages.

**Nodes Involved:**  
- Webhook

**Node Details:**  

- **Webhook**  
  - *Type and Role:* HTTP webhook node that listens for incoming POST requests on a defined path.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Path: `chatbot-webhook`  
    - Response Mode: `responseNode` (defers actual response until later node)  
  - *Key Expressions / Variables:* None at input stage.  
  - *Input Connections:* None (start node).  
  - *Output Connections:* Sends data to "Set User Message" node.  
  - *Version Requirements:* n8n v1+ compatible.  
  - *Potential Failures:* Incorrect HTTP method, path conflicts, malformed payloads, network timeouts.

#### 1.2 Message Preparation

**Overview:**  
Extracts the user's message from the webhook payload and sets it into a dedicated variable (`user_message`) for downstream AI processing.

**Nodes Involved:**  
- Set User Message

**Node Details:**  

- **Set User Message**  
  - *Type and Role:* Set node used to assign or transform incoming data.  
  - *Configuration:*  
    - Sets a string field named `user_message` with the value extracted from the webhook's JSON body: `{{$json.body.message}}`.  
  - *Key Expressions / Variables:*  
    - `user_message = {{$json.body.message}}`  
  - *Input Connections:* Receives from Webhook.  
  - *Output Connections:* Sends to AI Agent node.  
  - *Potential Failures:* Missing or malformed `message` field in the webhook JSON body, resulting in empty or undefined `user_message`.

#### 1.3 AI Processing

**Overview:**  
This block processes the user's message through an AI agent configured with a real estate sales persona. It incorporates session memory for contextual awareness and uses the GPT-4o-mini language model to generate responses.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Simple Memory

**Node Details:**  

- **AI Agent**  
  - *Type and Role:* LangChain AI Agent node orchestrating the prompt, memory, and language model to generate a response.  
  - *Configuration:*  
    - Input text: `{{$json["user_message"]}}` (the extracted user message)  
    - System message: Defines the agent persona as "Khusboo," a friendly real estate pre-sales agent for Alcove New Kolkata Sangam.  
    - Persona instructions include:  
      - Use of sales frameworks (AIDA, BANT, SPIN, PAS)  
      - Respond in Hinglish (mix of Hindi and English)  
      - Personal and trust-building tone  
      - Avoid disclosing prices unless asked  
      - Encourage site visits and share videos confidently  
    - Prompt Type: `define` (custom prompt configuration)  
  - *Key Expressions / Variables:*  
    - Input: `{{$json["user_message"]}}`  
    - System message embedded as a static string.  
  - *Input Connections:* Receives from "Set User Message" (main input), "Simple Memory" (ai_memory), and "OpenAI Chat Model" (ai_languageModel).  
  - *Output Connections:* Sends to "Respond to Webhook".  
  - *Version Requirements:* LangChain node version 2.  
  - *Potential Failures:*  
    - AI model call failures (auth, quota, network)  
    - Memory session key issues  
    - Expression evaluation errors  
    - Incoherent or off-topic responses if prompt or memory corrupted  
  - *Sub-workflow:* None; the node integrates directly with OpenAI and memory nodes.

- **OpenAI Chat Model**  
  - *Type and Role:* Language model node providing GPT-4o-mini as the underlying chat engine.  
  - *Configuration:*  
    - Model: `gpt-4o-mini` (a compact GPT-4 variant)  
    - No additional options set.  
  - *Input Connections:* Connected to AI Agent as `ai_languageModel` input.  
  - *Output Connections:* None beyond AI Agent.  
  - *Potential Failures:*  
    - OpenAI API authentication errors  
    - Rate limits or quota exceeded  
    - Model unavailability or version incompatibility

- **Simple Memory**  
  - *Type and Role:* Memory buffer node managing conversational context across sessions.  
  - *Configuration:*  
    - Session key: `memory_{{ $json.body.session_id || 'default' }}` (uses `session_id` from webhook body; defaults to 'default')  
    - Session ID type: Custom key  
    - Context window length: 20 (number of recent messages retained)  
  - *Input Connections:* None (implicit input from webhook data).  
  - *Output Connections:* Connected to AI Agent as `ai_memory` input.  
  - *Potential Failures:*  
    - Missing or inconsistent `session_id` leads to context loss or session mixing  
    - Memory storage failures  
    - Expression evaluation errors

#### 1.4 Response Delivery

**Overview:**  
Sends the AI-generated response back to the user through the webhook HTTP response, ensuring JSON format and proper escaping of special characters.

**Nodes Involved:**  
- Respond to Webhook

**Node Details:**  

- **Respond to Webhook**  
  - *Type and Role:* Sends the final JSON response to the HTTP caller of the webhook.  
  - *Configuration:*  
    - Response format: JSON  
    - Response body:  
      ```json
      {
        "Respond Immediately": "{{ $json.output.replaceAll('\n', '\\n').replaceAll('\"', '\\\"') }}"
      }
      ```  
    - This expression escapes newlines and quotes in the AI agent's output for JSON safety.  
  - *Input Connections:* Receives from AI Agent.  
  - *Output Connections:* None (terminal node).  
  - *Potential Failures:*  
    - Missing or undefined AI output causing empty responses  
    - JSON formatting errors if escaping fails  
    - HTTP response timeout or client disconnects

---

### 3. Summary Table

| Node Name          | Node Type                           | Functional Role                         | Input Node(s)       | Output Node(s)        | Sticky Note                                                                                                     |
|--------------------|-----------------------------------|---------------------------------------|---------------------|-----------------------|----------------------------------------------------------------------------------------------------------------|
| Webhook            | n8n-nodes-base.webhook            | Receive incoming HTTP POST requests   | None                | Set User Message       |                                                                                                                |
| Set User Message    | n8n-nodes-base.set                | Extract and store user message         | Webhook             | AI Agent               |                                                                                                                |
| AI Agent           | @n8n/n8n-nodes-langchain.agent    | Generate AI-driven real estate responses | Set User Message, Simple Memory, OpenAI Chat Model | Respond to Webhook |                                                                                                                |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o-mini model responses   | None                | AI Agent               |                                                                                                                |
| Simple Memory       | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintain conversation context per session | None                | AI Agent               |                                                                                                                |
| Respond to Webhook  | n8n-nodes-base.respondToWebhook   | Send AI response back via HTTP         | AI Agent            | None                   |                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**  
   - Type: `Webhook`  
   - HTTP Method: `POST`  
   - Path: `chatbot-webhook`  
   - Response Mode: `responseNode`  
   - No credentials needed.

2. **Add Set Node ("Set User Message")**  
   - Type: `Set`  
   - Add a new string field named `user_message`  
   - Value: Expression `{{$json.body.message}}` to extract the incoming message from webhook JSON body.  
   - Connect Webhook’s main output to this node’s input.

3. **Add Simple Memory Node ("Simple Memory")**  
   - Type: `Memory Buffer Window` (LangChain memory node)  
   - Set session key: `memory_{{ $json.body.session_id || 'default' }}` (expression)  
   - Session ID type: `customKey`  
   - Context window length: `20`  
   - No direct input connection needed but ensure it reads webhook body for `session_id`.

4. **Add OpenAI Chat Model Node ("OpenAI Chat Model")**  
   - Type: `LM Chat OpenAI` (LangChain node)  
   - Model selection: Choose `gpt-4o-mini` from model list  
   - Credentials: Configure with valid OpenAI API key with GPT-4o-mini access  
   - No additional options required.

5. **Create AI Agent Node ("AI Agent")**  
   - Type: `LangChain Agent`  
   - Text input: Expression `{{$json["user_message"]}}` (from Set User Message node)  
   - System message:  
     ```
     You are Khusboo, a friendly real estate pre-sales agent for Alcove New Kolkata Sangam. Your goal is to qualify leads and gently guide them toward a site visit. Use AIDA, BANT, SPIN, and PAS frameworks. Respond in Hinglish, be personal, share videos confidently, and never disclose price unless asked. Always aim to build trust and invite for a visit.
     ```  
   - Prompt Type: `define`  
   - Connect inputs:  
     - Main Input: from "Set User Message" node  
     - AI Memory: from "Simple Memory" node  
     - AI Language Model: from "OpenAI Chat Model" node

6. **Add Respond to Webhook Node ("Respond to Webhook")**  
   - Type: `Respond to Webhook`  
   - Respond with: JSON  
   - Response body expression:  
     ```json
     {
       "Respond Immediately": "{{ $json.output.replaceAll('\n', '\\n').replaceAll('\"', '\\\"') }}"
     }
     ```  
   - Connect input from "AI Agent" node main output.

7. **Verify Credentials**  
   - Ensure OpenAI API credentials are configured and valid.  
   - No specific credentials needed for webhook or other nodes.

8. **Test the Workflow**  
   - Trigger POST requests to `/webhook/chatbot-webhook` with JSON body containing:  
     ```json
     {
       "message": "Hello, I am interested in your properties.",
       "session_id": "user123"
     }
     ```  
   - Confirm chatbot responds with a Hinglish, friendly, and qualified lead message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The AI agent is configured to use multiple sales frameworks (AIDA, BANT, SPIN, PAS) to qualify leads effectively.               | Sales frameworks overview: https://blog.hubspot.com/sales/sales-frameworks                     |
| The chatbot responds in "Hinglish," a blend of Hindi and English, which is common in Indian urban communication styles.         | Cultural communication style note                                                              |
| Uses GPT-4o-mini, a compact yet capable GPT-4 variant suitable for conversational AI with cost-effective performance.           | OpenAI GPT-4o-mini documentation                                                               |
| Session memory ensures conversational continuity across multiple user inputs, enhancing user experience in chatbot dialogs.     | n8n LangChain memory buffer documentation                                                      |
| Response escaping replaces newlines and quotes to ensure valid JSON output, critical for frontend integration with JavaScript.  | JSON escaping best practices                                                                    |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---