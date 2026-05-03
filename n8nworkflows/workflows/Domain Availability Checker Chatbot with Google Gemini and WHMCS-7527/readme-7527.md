Domain Availability Checker Chatbot with Google Gemini and WHMCS

https://n8nworkflows.xyz/workflows/domain-availability-checker-chatbot-with-google-gemini-and-whmcs-7527


# Domain Availability Checker Chatbot with Google Gemini and WHMCS

### 1. Workflow Overview

This workflow implements an AI-powered chatbot designed to check domain name availability for users by integrating with a WHMCS system via its API. It targets domain-selling websites aiming to provide immediate, accurate, and friendly customer support regarding domain availability inquiries. The chatbot uses Google Gemini (PaLM) as its language model and LangChain’s AI Agent node for orchestrating AI interactions, combined with a session-based memory to maintain conversational context.

Logical blocks in the workflow:

- **1.1 Input Reception and Response Handling:** Reception of user queries via a webhook and delivering responses back.
- **1.2 AI Processing and Memory Management:** Uses LangChain AI Agent to interpret user queries, maintain session memory, and interact with the language model.
- **1.3 Domain Availability Verification:** HTTP requests to the WHMCS API to verify domain availability based on user input.
- **1.4 Supportive Configuration and Metadata:** Sticky notes providing documentation and configuration instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Response Handling

**Overview:**  
This block handles incoming HTTP POST requests from users (likely from a frontend chatbot interface), passes the query to the AI Agent, and returns the AI-generated response to the user with appropriate CORS and content headers.

**Nodes Involved:**  
- Webhook  
- Respond to Webhook1

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook Trigger  
  - Role: Listens for incoming POST requests containing the user's chat input and session ID.  
  - Configuration:  
    - HTTP method: POST  
    - Path: Unique webhook path (`8a56ad98-d800-4296-9a12-e6472b5d46dd`)  
    - Response mode: Uses Respond to Webhook node for returning response  
  - Inputs: External HTTP POST requests  
  - Outputs: Triggers the AI Agent node with the received data  
  - Edge cases: Invalid HTTP method, missing or malformed JSON body, session ID absent or malformed  

- **Respond to Webhook1**  
  - Type: Respond to Webhook  
  - Role: Sends the AI Agent's response back to the caller with CORS headers allowing cross-origin requests.  
  - Configuration:  
    - Response headers set to permit POST, OPTIONS with required CORS headers and JSON content type  
  - Inputs: Response data from AI Agent node  
  - Outputs: HTTP response to the original caller  
  - Edge cases: Response generation failure, header misconfiguration leading to CORS issues  

---

#### 1.2 AI Processing and Memory Management

**Overview:**  
This block uses LangChain’s AI Agent to process user input, maintain chat context through session-based memory, and integrate the Google Gemini language model. It ensures responses are accurate, friendly, and strictly based on verified domain availability checks.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Simple Memory

**Node Details:**

- **AI Agent**  
  - Type: LangChain AI Agent Node  
  - Role: Orchestrates the AI conversation, interprets user queries, invokes tools (domain checker), and formulates responses.  
  - Configuration highlights:  
    - Input text dynamically set from webhook JSON body (`{{$json.body.chatInput}}`)  
    - System message defines the assistant’s persona, responsibilities, guidelines, and interaction examples  
    - Tools: Uses "Domain_Availability_Checker" node as AI tool for domain verification  
    - Integrates memory (Simple Memory) and language model (Google Gemini)  
  - Inputs: User input text, memory context, tool results, language model output  
  - Outputs: Final AI-generated response to be sent back to the user  
  - Edge cases: Expression evaluation errors, exceeding token limits, tool invocation failures, memory session key missing or corrupted, delayed responses not mimicking typing  

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat Model Node  
  - Role: Provides AI language model responses based on prompts from AI Agent  
  - Configuration: Uses authenticated Google PaLM API credentials  
  - Inputs: Prompt and context from AI Agent  
  - Outputs: Language model replies for AI Agent processing  
  - Edge cases: API authentication failure, quota limits, network timeouts  

- **Simple Memory**  
  - Type: LangChain Memory Buffer (Window)  
  - Role: Maintains a sliding window of conversation context keyed by session ID to provide coherent multi-turn dialogue  
  - Configuration:  
    - Session key extracted dynamically from webhook JSON body (`{{$('Webhook').item.json.body.sessionId}}`)  
    - Context window length: 15 messages, balancing context size and performance  
  - Inputs: Incoming conversation data  
  - Outputs: Contextual memory passed to AI Agent  
  - Edge cases: Missing or malformed session ID, session memory overflow, inconsistent context leading to AI confusion  

---

#### 1.3 Domain Availability Verification

**Overview:**  
This block performs direct domain availability checks by sending HTTP POST requests to the WHMCS API, using credentials and parameters dynamically provided by the AI Agent.

**Nodes Involved:**  
- Domain_Availability_Checker

**Node Details:**

- **Domain_Availability_Checker**  
  - Type: HTTP Request Tool  
  - Role: Queries WHMCS API to check domain WHOIS status (availability) based on domain names extracted from user input.  
  - Configuration:  
    - URL: Custom WHMCS API endpoint (`https://your_whmcs_url.com/includes/api.php`) — must be replaced with actual URL  
    - Method: POST with form-urlencoded content type  
    - Headers: Content-Type set to `application/x-www-form-urlencoded`  
    - Body parameters include:  
      - `identifier`: WHMCS API identifier (to be configured)  
      - `secret`: WHMCS API secret (to be configured)  
      - `action`: Fixed as `DomainWhois` for domain checking  
      - `domain`: dynamically injected from AI Agent overrides (`$fromAI('parameters3_Value', '', 'string')`)  
    - Tool description documents its purpose clearly  
  - Inputs: Domain string from AI Agent’s tool invocation  
  - Outputs: JSON response from WHMCS API indicating domain availability  
  - Edge cases: Network errors, authentication failures (bad identifier/secret), invalid domain format, API rate limits, unexpected API response formats  

---

#### 1.4 Supportive Configuration and Metadata

**Overview:**  
Contains documentation and configuration instructions to assist developers in setting up and understanding the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Provides a concise overview of the workflow, its purpose, and a link to a live demo.  
  - Content highlights:  
    - Describes the workflow as an AI chatbot checking domain availability using WHMCS  
    - Includes a clickable link to a live demo webpage for reference  

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Instructions for configuring the WHMCS API credentials and endpoint within the “Domain_Availability_Checker” node.  
  - Content highlights:  
    - Step-by-step guide on replacing placeholders with actual WHMCS API identifier and secret  
    - Reminder to update the API URL to the actual WHMCS instance domain  

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                            | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                   |
|----------------------------|-----------------------------------|--------------------------------------------|-----------------------|-----------------------|-----------------------------------------------------------------------------------------------|
| Webhook                    | HTTP Webhook Trigger               | Receives user requests, triggers workflow  | —                     | AI Agent              |                                                                                               |
| AI Agent                   | LangChain AI Agent                 | Processes input, manages conversation, calls tools | Webhook               | Respond to Webhook1    |                                                                                               |
| Respond to Webhook1        | Respond to Webhook                 | Sends AI response back to user with CORS   | AI Agent              | —                     |                                                                                               |
| Simple Memory              | LangChain Memory Buffer Window    | Maintains session conversational context   | (Injected in AI Agent) | AI Agent (memory)      |                                                                                               |
| Google Gemini Chat Model   | LangChain Google Gemini LM        | Provides AI language model responses        | (Injected in AI Agent) | AI Agent (languageModel)|                                                                                               |
| Domain_Availability_Checker| HTTP Request Tool                 | Checks domain availability via WHMCS API   | AI Agent (tool)       | AI Agent (tool output) |                                                                                               |
| Sticky Note                | Sticky Note                       | Documentation: workflow overview and demo  | —                     | —                     | ## Domain Name Availability Check Workflow\n### [Live Demo](https://omerfayyaz.com/domain-name-availability-checker-with-n8n-using-whmcs-api/index.html) This n8n workflow creates an **AI-powered chatbot** that automatically checks domain availability using your WHMCS system. Customers can ask about domains in natural language, and the AI will verify availability through WHMCS API and suggest alternatives if needed. |
| Sticky Note1               | Sticky Note                       | Documentation: WHMCS API configuration guide | —                     | —                     | ### **Configure WHMCS API**\n- Open the "Domain_Availability_Checker" node\n- Replace `Your_WHMCS_Identifier` with your actual WHMCS API identifier\n- Replace `Your_WHMCS_Secret` with your actual WHMCS API secret\n- Update the URL to your WHMCS domain: `https://yourdomain.com/includes/api.php` |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook Trigger  
   - Configure:  
     - HTTP Method: POST  
     - Path: Unique path (e.g., `8a56ad98-d800-4296-9a12-e6472b5d46dd`)  
     - Response Mode: `Response Node`, to use Respond to Webhook node for responses  

2. **Create AI Agent Node**  
   - Type: LangChain AI Agent  
   - Parameters:  
     - Text Input: Set to expression `={{ $json.body.chatInput }}` (extract user message from webhook body)  
     - System Message: Copy detailed assistant instructions (see 2.2 AI Agent section for full content)  
     - Prompt Type: Define  
     - Add tools: `Domain_Availability_Checker` (to be created in next step)  
     - Configure memory: Link to `Simple Memory` node  
     - Configure language model: Link to `Google Gemini Chat Model` node  

3. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Configure:  
     - Set Response Headers:  
       - `Content-Type`: `application/json`  
       - `Access-Control-Allow-Origin`: `*`  
       - `Access-Control-Allow-Headers`: `Content-Type, x-api-key`  
       - `Access-Control-Allow-Methods`: `POST, OPTIONS`  
   - Connect AI Agent output to this node’s input  

4. **Create Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Parameters:  
     - Session Key: Expression `={{ $('Webhook').item.json.body.sessionId }}` to uniquely identify chat sessions  
     - SessionId Type: Custom Key  
     - Context Window Length: `15` messages  

5. **Create Google Gemini Chat Model Node**  
   - Type: LangChain Google Gemini Chat Model  
   - Credentials: Set up Google PaLM API credentials (OAuth2 or API key)  
   - No additional parameters needed  

6. **Create Domain_Availability_Checker Node**  
   - Type: HTTP Request Tool  
   - Configure:  
     - URL: Replace with your WHMCS API URL, e.g., `https://yourdomain.com/includes/api.php`  
     - Method: POST  
     - Content Type: `application/x-www-form-urlencoded`  
     - Headers: Add `Content-Type`: `application/x-www-form-urlencoded`  
     - Body Parameters (form data):  
       - `identifier`: Your actual WHMCS API identifier (string)  
       - `secret`: Your actual WHMCS API secret (string)  
       - `action`: Set to `DomainWhois` (string)  
       - `domain`: Expression to dynamically receive from AI Agent’s tool input, e.g., `={{ $fromAI('parameters3_Value', '', 'string') }}`  
     - Enable sending body and headers appropriately  

7. **Connect Nodes**  
   - Webhook main output → AI Agent input  
   - Simple Memory output → AI Agent memory input  
   - Google Gemini Chat Model output → AI Agent language model input  
   - Domain_Availability_Checker output → AI Agent tool input  
   - AI Agent main output → Respond to Webhook input  

8. **Final Checks and Activation**  
   - Verify all credentials (Google Gemini and WHMCS) are correctly configured and tested separately.  
   - Replace all placeholder values in Domain_Availability_Checker node.  
   - Test webhook endpoint by sending sample POST requests with JSON body containing `chatInput` and `sessionId`.  
   - Monitor logs for API errors, memory session issues, and AI response accuracy.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow creates an AI chatbot that automatically verifies domain availability using WHMCS API integration.             | Live Demo Link: https://omerfayyaz.com/domain-name-availability-checker-with-n8n-using-whmcs-api/index.html          |
| WHMCS API credentials and URL must be properly configured in the HTTP Request node to enable domain availability checks.     | Refer to Sticky Note1 instructions inside the workflow                                                              |
| The AI Agent's system message enforces strict rules: no speculative answers, no unverified domain mentions, and positive tone. | Ensures compliance to business rules and professional customer support standards                                     |
| Google Gemini requires valid API credentials and may have usage quotas or rate limits to consider during production use.     | Google PaLM API documentation and quota management                                                                    |
| Session-based memory ensures multi-turn conversations remain contextually relevant but requires correct session ID handling.| Improper session ID management could cause memory leakage or incoherent conversations                                 |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and public.