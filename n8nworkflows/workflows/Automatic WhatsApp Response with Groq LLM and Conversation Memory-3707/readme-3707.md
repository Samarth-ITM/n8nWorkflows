Automatic WhatsApp Response with Groq LLM and Conversation Memory

https://n8nworkflows.xyz/workflows/automatic-whatsapp-response-with-groq-llm-and-conversation-memory-3707


# Automatic WhatsApp Response with Groq LLM and Conversation Memory

### 1. Workflow Overview

This workflow automates WhatsApp responses for small businesses by leveraging the Groq large language model (LLM) integrated via n8n’s AI Agent node, combined with a simple conversation memory to maintain context. It targets individual users or teams who receive frequent WhatsApp inquiries about their products or services and want to automate replies efficiently without manual intervention.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures incoming WhatsApp messages via the WhatsApp Trigger node.
- **1.2 Conditional Routing:** Uses an If node (“Signpost”) to evaluate incoming messages and direct the flow accordingly.
- **1.3 AI Processing & Memory:** Utilizes the AI Agent node configured with the Groq Chat Model and Simple Memory to generate context-aware responses.
- **1.4 Output Delivery:** Sends the AI-generated response back to the WhatsApp user via the WhatsApp node (“Output”).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming WhatsApp messages and initiates the workflow by passing the message data downstream.

- **Nodes Involved:**  
  - Input Submissions (WhatsApp Trigger)

- **Node Details:**  

  - **Input Submissions**  
    - Type: WhatsApp Trigger  
    - Role: Entry point capturing incoming WhatsApp messages.  
    - Configuration: Requires WhatsApp credential setup to connect to the WhatsApp Business API or integration.  
    - Key Expressions: Captures message content, sender info, and metadata from WhatsApp.  
    - Input: External WhatsApp messages (via webhook).  
    - Output: Passes message data to the “Signpost” node.  
    - Edge Cases:  
      - Authentication failures if credentials are invalid or expired.  
      - Webhook connectivity issues or message format changes.  
      - Rate limiting or message flooding from WhatsApp side.  
    - Version: n8n WhatsApp Trigger node version 1.

#### 2.2 Conditional Routing

- **Overview:**  
  This block evaluates the incoming message to decide if it should be processed by the AI Agent or handled differently. It acts as a gatekeeper to control workflow branching.

- **Nodes Involved:**  
  - Signpost (If node)

- **Node Details:**  

  - **Signpost**  
    - Type: If node  
    - Role: Checks conditions on the incoming message to route the flow.  
    - Configuration: Conditions are user-defined (not explicitly detailed in the JSON), typically checking message content or metadata to decide if AI response is needed.  
    - Key Expressions: Likely uses expressions referencing the WhatsApp message text or sender attributes.  
    - Input: Receives data from “Input Submissions”.  
    - Output: On true, routes to “AI Agent”; on false, could stop or route elsewhere (not shown).  
    - Edge Cases:  
      - Expression evaluation errors if message data is missing or malformed.  
      - Unhandled false branch leading to dropped messages.  
    - Version: If node version 2.2.

#### 2.3 AI Processing & Memory

- **Overview:**  
  This block uses the AI Agent node configured with the Groq Chat Model and Simple Memory to generate intelligent, context-aware replies based on the incoming message and conversation history.

- **Nodes Involved:**  
  - AI Agent  
  - Groq Chat Model  
  - Simple Memory

- **Node Details:**  

  - **Groq Chat Model**  
    - Type: Langchain Groq Chat Model node  
    - Role: Provides the language model backend for generating AI responses.  
    - Configuration: Uses Groq credentials and model selection configured in n8n credentials.  
    - Input: Receives prompt text from AI Agent.  
    - Output: Returns generated text to AI Agent.  
    - Edge Cases:  
      - API authentication failures.  
      - Model timeout or rate limits.  
      - Unexpected model output or errors.  
    - Version: 1.

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Stores recent conversation history to maintain context across messages.  
    - Configuration: Uses session ID and key expressions to store/retrieve chat history.  
    - Input: Connected to AI Agent’s memory input.  
    - Output: Provides memory context to AI Agent.  
    - Edge Cases:  
      - Memory overflow or data loss if session ID is not properly defined.  
      - Incorrect key usage causing context mismatch.  
    - Version: 1.3.

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Orchestrates the AI workflow by combining the Groq Chat Model and Simple Memory to process input and generate output.  
    - Configuration: No explicit parameters shown, but internally configured to use Groq Chat Model as language model and Simple Memory for context.  
    - Input: Receives routed message data from “Signpost”.  
    - Output: Sends generated response to “Output” node.  
    - Edge Cases:  
      - Failures in linked nodes (Groq model or memory).  
      - Misconfiguration leading to empty or irrelevant responses.  
    - Version: 1.8.

#### 2.4 Output Delivery

- **Overview:**  
  Sends the AI-generated response back to the WhatsApp user, completing the interaction cycle.

- **Nodes Involved:**  
  - Output (WhatsApp node)

- **Node Details:**  

  - **Output**  
    - Type: WhatsApp node  
    - Role: Sends messages to WhatsApp users.  
    - Configuration: Requires WhatsApp credentials and message formatting.  
    - Input: Receives AI-generated response from “AI Agent”.  
    - Output: Sends message to WhatsApp API.  
    - Edge Cases:  
      - Authentication errors or expired tokens.  
      - Message formatting errors causing delivery failure.  
      - WhatsApp API rate limits or downtime.  
    - Version: 1.

---

### 3. Summary Table

| Node Name         | Node Type                          | Functional Role               | Input Node(s)       | Output Node(s) | Sticky Note                                                                                      |
|-------------------|----------------------------------|------------------------------|---------------------|----------------|-------------------------------------------------------------------------------------------------|
| Input Submissions  | WhatsApp Trigger                  | Receive incoming WhatsApp messages | None (Webhook)      | Signpost       | Must set up WhatsApp credentials; follow n8n guide to register credential account.              |
| Signpost          | If node                          | Conditional routing of messages | Input Submissions   | AI Agent       | Use conditions to direct workflow; ensure expressions match your business logic.                |
| Groq Chat Model    | Langchain Groq Chat Model        | AI language model for response generation | AI Agent (ai_languageModel) | AI Agent       | Requires Groq credentials; select appropriate model for your business needs.                    |
| Simple Memory      | Langchain Memory Buffer Window   | Stores conversation context   | AI Agent (ai_memory) | AI Agent       | Use session ID and key to maintain chat history; minimizes cost while preserving context.       |
| AI Agent          | Langchain Agent                  | Orchestrates AI processing and memory | Signpost           | Output         | Connects Groq model and memory; configure prompt and session parameters carefully.              |
| Output            | WhatsApp node                    | Sends AI-generated replies    | AI Agent            | None           | Requires WhatsApp credentials; format messages correctly for delivery.                          |
| Sticky Note       | Sticky Note                     | Documentation and notes       | None                | None           | Various notes on setup and customization (see detailed instructions in workflow description).  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure credentials by registering your WhatsApp Business API or integration in n8n.  
   - This node listens for incoming WhatsApp messages and triggers the workflow.  
   - Connect its output to the next node (“Signpost”).

2. **Add If Node (“Signpost”)**  
   - Type: If node  
   - Configure conditions to evaluate incoming messages (e.g., check if message text is not empty or matches certain keywords).  
   - Connect input from WhatsApp Trigger node.  
   - On true branch, connect output to “AI Agent” node.  
   - On false branch, optionally handle differently or leave unconnected.

3. **Add Langchain Groq Chat Model Node**  
   - Type: Langchain Groq Chat Model  
   - Set up Groq credentials in n8n (API key, model selection).  
   - No additional parameters needed unless customizing prompt or model settings.  
   - This node will be linked as the language model for the AI Agent.

4. **Add Langchain Simple Memory Node**  
   - Type: Langchain Memory Buffer Window  
   - Configure session ID (e.g., use WhatsApp sender ID) to maintain conversation context per user.  
   - Set key to store chat history.  
   - This node will be linked as the memory source for the AI Agent.

5. **Add Langchain AI Agent Node**  
   - Type: Langchain Agent  
   - Configure to use the Groq Chat Model node as its language model input.  
   - Configure to use the Simple Memory node as its memory input.  
   - Connect input from the true branch of the “Signpost” node.  
   - Connect output to the WhatsApp node (“Output”).  
   - Customize prompts or session parameters as needed to reflect your business knowledge base.

6. **Add WhatsApp Node (“Output”)**  
   - Type: WhatsApp node  
   - Configure WhatsApp credentials for sending messages.  
   - Connect input from the AI Agent node.  
   - Map AI Agent’s output text to the message body for WhatsApp.  
   - Ensure message formatting complies with WhatsApp API requirements.

7. **Final Steps**  
   - Save the workflow.  
   - Activate the workflow.  
   - Test by sending WhatsApp messages to your registered number and verify automated responses.  
   - Adjust conditions, prompts, and memory settings to optimize response accuracy.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is dedicated to small businesses to automate WhatsApp customer interactions using AI.             | Workflow description and motivation section.                                                   |
| Setup requires registering credentials for WhatsApp and Groq AI model in n8n.                                   | Follow official n8n documentation for WhatsApp and Groq credential setup.                      |
| Customize AI Agent prompts and memory keys to align with your specific business knowledge base for best results.| Instructions in workflow description under “How to customize this workflow to your needs”.     |
| For detailed WhatsApp credential setup, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.whatsApp/ | Official n8n WhatsApp node documentation.                                                      |
| For Groq model integration and credential setup, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.lmChatGroq/ | Official n8n Groq Chat Model node documentation.                                               |

---

This document provides a complete, structured reference to understand, reproduce, and customize the “Automatic WhatsApp Response with Groq LLM and Conversation Memory” workflow in n8n.